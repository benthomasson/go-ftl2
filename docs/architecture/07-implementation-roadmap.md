# Implementation Roadmap

This document outlines the phased approach for implementing Go-FTL2.

## Phase Overview

| Phase | Focus | Outcome |
|-------|-------|---------|
| 1 | Foundation | Core types, inventory, basic execution |
| 2 | Local Execution | FTL modules, local runner |
| 3 | Remote Execution | SSH, gate protocol |
| 4 | State & Persistence | State management, audit logging |
| 5 | CLI & Polish | Command-line interface, documentation |

## Phase 1: Foundation

**Goal**: Establish the core type system and package structure.

### Tasks

1. **Initialize Go module**
   ```bash
   go mod init github.com/org/go-ftl2
   ```

2. **Create package structure**
   - `pkg/types/` - Core type definitions
   - `pkg/inventory/` - Inventory loading
   - `pkg/automation/` - Context and options

3. **Implement core types** (`pkg/types/`)
   - [ ] `ExecutionConfig` struct
   - [ ] `ModuleResult` struct
   - [ ] `ExecutionResults` struct
   - [ ] Error types and sentinel errors

4. **Implement inventory** (`pkg/inventory/`)
   - [ ] `Host` struct with methods
   - [ ] `HostGroup` struct with methods
   - [ ] `Inventory` struct with YAML loading
   - [ ] Target pattern matching (group names, "all", patterns)

5. **Basic AutomationContext** (`pkg/automation/`)
   - [ ] `Context` struct
   - [ ] Functional options pattern
   - [ ] `New()` constructor
   - [ ] `Close()` cleanup

### Deliverables

- Inventory loading from YAML files
- Basic context creation
- Unit tests for all types

### Dependencies

```go
require (
    gopkg.in/yaml.v3 v3.0.1
)
```

---

## Phase 2: Local Execution

**Goal**: Execute FTL modules on the local machine.

### Tasks

1. **Module system** (`pkg/modules/`)
   - [ ] `FTLModule` interface
   - [ ] Module registry
   - [ ] FQCN parsing
   - [ ] Module loader

2. **FTL modules** (`pkg/modules/ftl/`)
   - [ ] `ftl_file` - File/directory management
   - [ ] `ftl_command` - Command execution
   - [ ] `ftl_copy` - File copying
   - [ ] `ftl_uri` - HTTP requests

3. **Executor foundation** (`pkg/executor/`)
   - [ ] `Runner` interface
   - [ ] `LocalRunner` implementation
   - [ ] `Executor` with chunking

4. **Context integration**
   - [ ] `Execute()` method on Context
   - [ ] Result aggregation

### Deliverables

- Working local execution
- Basic FTL modules
- Example: local file operations

### Example Usage

```go
ctx, _ := automation.New(
    automation.WithInventory("inventory.yaml"),
)
defer ctx.Close()

results, err := ctx.Execute(context.Background(), "localhost", "ftl_file", map[string]any{
    "path":  "/tmp/test",
    "state": "directory",
})
```

---

## Phase 3: Remote Execution

**Goal**: Execute modules on remote hosts via SSH.

### Tasks

1. **SSH client** (`pkg/ssh/`)
   - [ ] `Client` with connection management
   - [ ] `Connection` wrapper
   - [ ] Key-based authentication
   - [ ] Connection pooling

2. **Gate protocol** (`pkg/gate/`)
   - [ ] `Protocol` for message encoding/decoding
   - [ ] Message type definitions
   - [ ] `Connection` for gate communication

3. **Gate builder** (`pkg/gate/`)
   - [ ] Embed Python gate source
   - [ ] Gate building with caching
   - [ ] Hash-based invalidation

4. **Remote runner** (`pkg/executor/`)
   - [ ] `RemoteRunner` implementation
   - [ ] Gate deployment
   - [ ] Module execution via gate

5. **Executor integration**
   - [ ] Runner selection (local vs remote)
   - [ ] Connection reuse

### Deliverables

- SSH-based remote execution
- Gate deployment and communication
- Example: remote file operations

### Dependencies

```go
require (
    golang.org/x/crypto v0.18.0  // SSH
)
```

---

## Phase 4: State & Persistence

**Goal**: Add state tracking and audit logging.

### Tasks

1. **State management** (`pkg/state/`)
   - [ ] `State` struct with operations
   - [ ] `HostState` and `ResourceState`
   - [ ] JSON file backend
   - [ ] Atomic writes

2. **State merging**
   - [ ] Merge state hosts to inventory
   - [ ] Dynamic host addition

3. **Secrets management** (`pkg/internal/secrets/`)
   - [ ] Environment variable loading
   - [ ] Secret bindings
   - [ ] Secret filtering in output

4. **Audit logging**
   - [ ] Audit log format
   - [ ] Secret redaction
   - [ ] JSON output

5. **Context integration**
   - [ ] State loading on startup
   - [ ] `AddHost()` method
   - [ ] Audit log writing

### Deliverables

- Persistent state across runs
- Secret injection
- Audit trail

---

## Phase 5: CLI & Polish

**Goal**: Production-ready command-line interface.

### Tasks

1. **CLI framework** (`cmd/ftl2/`)
   - [ ] Cobra-based CLI
   - [ ] `run` command
   - [ ] `inventory` command
   - [ ] `state` command
   - [ ] `version` command

2. **Progress reporting**
   - [ ] Rich terminal output
   - [ ] Progress indicators
   - [ ] Color support

3. **Configuration**
   - [ ] Config file support
   - [ ] Environment variables
   - [ ] Command-line flags

4. **Error handling**
   - [ ] User-friendly error messages
   - [ ] Verbose mode
   - [ ] Debug logging

5. **Documentation**
   - [ ] README.md
   - [ ] CLI help text
   - [ ] Examples

### Dependencies

```go
require (
    github.com/spf13/cobra v1.8.0
    github.com/fatih/color v1.16.0
)
```

### Deliverables

- `ftl2` binary
- Comprehensive CLI
- Documentation

---

## Testing Strategy

### Unit Tests

Each package has comprehensive unit tests:

```bash
go test ./pkg/...
```

### Integration Tests

Test full workflows:

```bash
go test ./test/integration/... -tags=integration
```

### Test Coverage

Target: 80% coverage minimum

```bash
go test -cover ./pkg/...
```

---

## Milestones

| Milestone | Target | Criteria |
|-----------|--------|----------|
| M1: Foundation | Week 2 | Inventory loading, types defined |
| M2: Local Exec | Week 4 | FTL modules working locally |
| M3: Remote Exec | Week 7 | SSH gate execution working |
| M4: State | Week 9 | State persistence, secrets |
| M5: CLI | Week 11 | Full CLI, documentation |
| M6: Release | Week 12 | v0.1.0 release |

---

## Technical Decisions

### Why Go?

1. **Single binary** - No runtime dependencies
2. **Native performance** - Faster than Python
3. **Concurrency** - goroutines vs asyncio
4. **Cross-compilation** - Easy multi-platform builds
5. **Memory efficiency** - Lower footprint

### Why Keep Python Gate?

1. **Ansible compatibility** - Must run Python modules
2. **Module ecosystem** - Leverage existing modules
3. **Gradual migration** - Can port modules to Go over time

### Key Libraries

| Purpose | Library | Rationale |
|---------|---------|-----------|
| YAML | `gopkg.in/yaml.v3` | Standard, well-maintained |
| SSH | `golang.org/x/crypto/ssh` | Official Go crypto library |
| CLI | `github.com/spf13/cobra` | Industry standard |
| Concurrency | `golang.org/x/sync/errgroup` | Clean error handling |
| JSON | `encoding/json` | Standard library |

---

## Risk Mitigation

### Risk: Ansible Module Compatibility

**Mitigation**: Keep Python gate for Ansible modules, only port high-value modules to Go.

### Risk: SSH Connection Issues

**Mitigation**: Implement robust connection pooling, retry logic, and timeout handling.

### Risk: State Corruption

**Mitigation**: Atomic file writes, file locking, validation on load.

### Risk: Performance Regression

**Mitigation**: Benchmark critical paths, profile regularly, compare with Python version.

---

## Success Criteria

### Functional

- [ ] All FTL modules working
- [ ] Remote execution via SSH gate
- [ ] State persistence
- [ ] Secret injection
- [ ] CLI commands working

### Performance

- [ ] 2x faster than Python FTL2 for local execution
- [ ] 1.5x faster for remote execution
- [ ] Memory usage < 50MB for typical workloads

### Quality

- [ ] 80%+ test coverage
- [ ] No critical bugs
- [ ] Documentation complete
- [ ] Example playbooks working
