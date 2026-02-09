# Concerns and Tradeoffs

This document outlines concerns, risks, and open questions about the Go port of FTL2. These should be considered before and during implementation.

## Major Concerns

### 1. Limited Performance Benefit for Remote Execution

**The Issue**: The core value proposition of FTL2 is running Ansible modules **in-process** rather than as subprocesses, achieving 250x speedup for local execution. However, for remote hosts, modules must still execute in Python on the remote machine via the gate.

**What Go Speeds Up**:
- CLI startup time (~10ms vs ~500ms for Python)
- Local FTL module execution
- SSH connection management and orchestration
- Result aggregation and processing
- Memory usage (~50MB vs ~200MB for Python)

**What Remains Python-Bound**:
- Remote module execution (the gate is Python)
- Ansible module compatibility layer
- Module dependency resolution on remote hosts

**Impact**: For workloads that are primarily remote (the common case), the performance improvement may be modest. The gate execution time dominates, and that remains unchanged.

**Mitigation**: Focus on optimizing orchestration overhead (connection pooling, parallel SSH, result streaming) where Go excels.

---

### 2. Python Gate Dependency

**The Issue**: The architecture retains a Python gate for remote execution, which undermines some benefits of a Go port.

**Implications**:

| Benefit Claimed | Reality |
|----------------|---------|
| Single binary distribution | Gate source must be embedded; remote hosts need Python |
| No runtime dependencies | Python 3.13+ required on all remote hosts |
| Simpler deployment | Must deploy gate.pyz to each remote host |
| One codebase | Two codebases: Go controller + Python gate |

**Maintenance Burden**:
- Gate protocol changes require updates in both languages
- Bug fixes may need to be applied twice
- Testing matrix expands (Go versions × Python versions)

**Alternatives Considered**:

1. **Pure Go gate**: Would require reimplementing Ansible module execution in Go, which is impractical given Ansible modules are Python.

2. **Embedded Python interpreter**: Use cgo to embed Python. Adds complexity, build issues, and doesn't help remote execution.

3. **Compile gate to binary**: Use PyInstaller/Nuitka. Increases gate size significantly (~50MB+), complicates updates.

**Recommendation**: Accept the Python gate as a pragmatic necessity. Document it clearly. Consider it a feature (Ansible compatibility) rather than a limitation.

---

### 3. API Ergonomics Regression

**The Issue**: Python's dynamic attribute access enables an elegant, fluent API that Go cannot replicate.

**Python API**:
```python
async with automation(inventory="inventory.yaml") as ftl:
    await ftl.webservers.dnf(name="nginx", state="present")
    await ftl.databases.postgresql_db(name="myapp")
```

**Go API**:
```go
ctx, _ := automation.New(automation.WithInventory("inventory.yaml"))
defer ctx.Close()

ctx.Execute(context.Background(), "webservers", "dnf", map[string]any{
    "name":  "nginx",
    "state": "present",
})
ctx.Execute(context.Background(), "databases", "postgresql_db", map[string]any{
    "name": "myapp",
})
```

**Differences**:

| Aspect | Python | Go |
|--------|--------|-----|
| Host scoping | `ftl.webservers.module()` | `Execute("webservers", "module", ...)` |
| Arguments | Keyword args | `map[string]any` |
| Async | `await` | `context.Context` |
| Cleanup | `async with` | `defer Close()` |

**Impact**: Users writing automation scripts may find Go less pleasant. The verbosity is a step backward in developer experience.

**Mitigations**:

1. **Builder pattern** for common operations:
   ```go
   ctx.On("webservers").DNF().Name("nginx").State("present").Run()
   ```

2. **Code generation** for type-safe module wrappers:
   ```go
   modules.DNF(ctx, "webservers", modules.DNFArgs{Name: "nginx", State: "present"})
   ```

3. **Accept the tradeoff**: Go users expect explicit APIs. The verbosity provides clarity and type safety.

---

### 4. Module System Complexity

**The Issue**: FTL2's module system involves intricate logic that is difficult to port correctly.

**Complex Components**:

1. **FQCN Resolution**: Parsing `amazon.aws.ec2_instance` into namespace/collection/module, then finding the actual file in collections paths.

2. **Dependency Detection**: Scanning Python module imports to bundle required packages.

3. **Module Bundling**: Creating ZIP archives with modules and dependencies for gate deployment.

4. **Shadowed Modules**: Mapping Ansible modules to FTL replacements (`ansible.builtin.file` → `ftl_file`).

5. **Excluded Modules**: Blocking dangerous modules for safety.

**Risks**:
- Subtle bugs in resolution logic cause wrong module to execute
- Missing dependencies cause remote failures
- Edge cases in Ansible's module loading not handled
- Collection structure changes break resolution

**Mitigation**:
- Port logic carefully with extensive test coverage
- Test against real Ansible collections (amazon.aws, community.general)
- Consider shelling out to Python for complex resolution initially
- Add integration tests that run actual modules

---

## Moderate Concerns

### 5. Feature Drift Risk

**The Issue**: Two implementations will naturally diverge over time.

**Symptoms**:
- Bugs fixed in Python not fixed in Go (and vice versa)
- New features added to one but not the other
- Behavioral differences confuse users
- Documentation becomes inconsistent

**Mitigation**:
- Define clear feature parity requirements
- Shared test suites (protocol tests, integration tests)
- Single source of truth for protocol specification
- Consider which version is "primary"

---

### 6. Target Audience Clarity

**The Issue**: It's unclear who the Go port serves best.

**User Profiles**:

| User Type | Best Choice | Reason |
|-----------|-------------|--------|
| Python developers writing automation | Python FTL2 | Native language, better API |
| Ops teams wanting a CLI tool | Go FTL2 | Fast startup, single binary |
| CI/CD pipelines | Go FTL2 | Predictable resource usage |
| Ansible power users | Python FTL2 | Full module ecosystem access |
| Embedded/resource-constrained | Go FTL2 | Lower memory footprint |

**Recommendation**: Explicitly define and document the target audience. Consider whether both versions are necessary or if one should be primary.

---

### 7. Loss of Python Ecosystem

**The Issue**: Python has rich libraries that FTL2 leverages. Go equivalents may be less mature or missing.

**Examples**:

| Python | Go Equivalent | Concern |
|--------|---------------|---------|
| `asyncssh` | `golang.org/x/crypto/ssh` | Go's is lower-level, more manual |
| `httpx` | `net/http` | Go's is excellent, no issue |
| `jinja2` | `text/template` | Different syntax, not compatible |
| `rich` | `github.com/charmbracelet/lipgloss` | Different API, learning curve |
| `pyyaml` | `gopkg.in/yaml.v3` | Go's is good, minor differences |

**Mitigation**: Most Go equivalents are adequate. Document differences in behavior.

---

## Open Questions

### Q1: Would a Hybrid Approach Work Better?

Instead of a full Go port, consider:
- Go CLI that invokes Python FTL2 as a subprocess
- Go handles CLI parsing, progress display, configuration
- Python handles actual execution

**Pros**: Simpler, no feature drift, leverages both languages' strengths
**Cons**: Still need Python installed, subprocess overhead

### Q2: Is the Goal to Eliminate Python Entirely?

If yes, this requires:
- Porting all FTL modules to Go
- Reimplementing Ansible module execution in Go (impractical)
- Abandoning Ansible module compatibility

If no, the Python gate is permanent, and the port's value is limited to orchestration.

### Q3: What's the Primary Driver?

| Driver | Implication |
|--------|-------------|
| Startup time | Go helps significantly |
| Distribution simplicity | Go helps partially (gate still needs Python) |
| Memory usage | Go helps significantly |
| Execution speed | Go helps minimally (gate is the bottleneck) |
| Type safety | Go helps (but users write less code) |
| Cross-platform | Go helps (easier Windows support) |

### Q4: Should FTL Modules Be the Focus?

If the primary use case is native FTL modules (`ftl_file`, `ftl_command`, `ftl_copy`, `ftl_uri`), the Go port provides substantial value:
- These can be pure Go
- No Python needed for local execution
- No gate needed for remote FTL modules (could SSH + execute directly)

If the primary use case is Ansible collections (`amazon.aws`, `community.general`), the value is limited.

---

## Recommendations

### 1. Validate the Use Case First

Before investing heavily in the port:
- Survey potential users
- Identify the top 10 modules used
- Determine what percentage are FTL-portable vs Ansible-only

### 2. Consider a Phased Approach

**Phase 1**: Go CLI + Python FTL2 backend (hybrid)
- Fast CLI, leverage existing Python code
- Validate user interest

**Phase 2**: Go orchestration + Python gate (current design)
- Port orchestration layer
- Keep gate for compatibility

**Phase 3**: Native Go modules (optional)
- Port high-value modules to Go
- Reduce Python dependency for common cases

### 3. Define Success Criteria

What does success look like?
- [ ] 2x faster end-to-end execution?
- [ ] <100MB memory usage?
- [ ] Single binary distribution (accepting gate limitation)?
- [ ] Feature parity with Python version?
- [ ] Active user adoption?

### 4. Accept Pragmatic Tradeoffs

Some tradeoffs are acceptable:
- Python gate is necessary for Ansible compatibility
- Go API will be more verbose than Python
- Not every Python feature needs to be ported

---

## Summary

The Go port is technically feasible but comes with meaningful tradeoffs. The primary benefits (startup time, memory, distribution) must be weighed against the costs (maintenance burden, API regression, limited remote speedup).

The decision should be driven by clear use cases and target audience definition rather than technical preference alone.
