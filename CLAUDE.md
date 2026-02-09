# CLAUDE.md - Development Guide for Claude

This file provides context and instructions for Claude when working on the go-ftl2 codebase.

## Project Overview

go-ftl2 is a Go port of FTL2, a high-performance Python automation framework. The goal is to provide a fast, single-binary controller that orchestrates module execution on local and remote hosts.

## Key Context

- **Reference implementation**: `/Users/ben/git/faster-than-light2` (Python FTL2)
- **Deep context/notes**: `/Users/ben/git/faster-than-light-refactor`
- **Architecture docs**: `docs/architecture/` in this repo

## Project Structure

```
go-ftl2/
├── cmd/ftl2/           # CLI entrypoint (planned)
├── pkg/
│   ├── automation/     # User-facing API, context management
│   ├── executor/       # Concurrent execution, runners
│   ├── gate/           # Remote gate protocol
│   ├── modules/        # Module resolution, FTL modules
│   ├── inventory/      # Host/group management
│   ├── state/          # Persistent state
│   ├── ssh/            # SSH client
│   └── types/          # Shared types
└── docs/
    ├── architecture/   # Design documents
    └── diagrams/       # Mermaid diagrams + PNGs
```

## Development Guidelines

### Code Style

- Follow standard Go conventions (`gofmt`, `golint`)
- Use `context.Context` for cancellation throughout
- Prefer functional options pattern for configuration
- Use interfaces for testability (Runner, Backend, etc.)
- Keep packages focused and loosely coupled

### Error Handling

- Return errors explicitly, don't panic
- Wrap errors with context: `fmt.Errorf("failed to X: %w", err)`
- Use sentinel errors for common cases: `var ErrNotFound = errors.New(...)`
- Create typed errors when additional context is needed

### Concurrency

- Use `errgroup.Group` for parallel operations with error handling
- Use `sync.RWMutex` for shared state
- Pass `context.Context` to all long-running operations
- Respect context cancellation

### Testing

- Write table-driven tests
- Use `t.Parallel()` where safe
- Mock external dependencies (SSH, filesystem)
- Target 80% coverage minimum

## Key Design Decisions

1. **Python gate retained**: Remote hosts run a Python gate for Ansible module compatibility. This is intentional - porting Ansible modules to Go is impractical.

2. **Functional options**: Use `WithX()` pattern for configuration rather than config structs.

3. **Chunked execution**: Process hosts in chunks (default 10) to limit concurrent connections.

4. **Length-prefixed JSON protocol**: Gate communication uses 8-byte hex length prefix + JSON body.

## Common Tasks

### Adding a new FTL module

1. Create `pkg/modules/ftl/<module_name>.go`
2. Implement the `FTLModule` interface
3. Register in `pkg/modules/ftl/registry.go`
4. Add tests in `pkg/modules/ftl/<module_name>_test.go`

### Updating the gate protocol

1. Update message types in `pkg/gate/messages.go`
2. Update protocol handling in `pkg/gate/protocol.go`
3. Update embedded Python gate source
4. Update `docs/architecture/04-gate-protocol.md`

### Adding a new diagram

1. Create `.mmd` file in `docs/diagrams/`
2. Render with: `mmdc -i file.mmd -o file.png -s 2 -b transparent`
3. Reference in architecture docs

## Important Files

| File | Purpose |
|------|---------|
| `pkg/types/config.go` | Core configuration types |
| `pkg/types/result.go` | Module result types |
| `pkg/executor/runner.go` | Runner interface definition |
| `pkg/gate/protocol.go` | Gate message encoding/decoding |
| `pkg/automation/context.go` | Main user-facing API |

## Dependencies

Planned dependencies:
- `gopkg.in/yaml.v3` - YAML parsing
- `golang.org/x/crypto/ssh` - SSH client
- `golang.org/x/sync/errgroup` - Error group for concurrency
- `github.com/spf13/cobra` - CLI framework

## Known Concerns

See `docs/architecture/08-concerns-and-tradeoffs.md` for detailed discussion of:
- Limited remote execution speedup (gate is still Python)
- API ergonomics vs Python version
- Module system complexity
- Target audience considerations

## Commands

```bash
# Run tests
go test ./...

# Run tests with coverage
go test -cover ./...

# Build
go build ./cmd/ftl2

# Render diagrams
cd docs/diagrams && mmdc -i *.mmd -o *.png -s 2 -b transparent

# Lint
golangci-lint run
```

## References

- [Python FTL2 source](https://github.com/benthomasson/ftl2/)
- [Ansible module development](https://docs.ansible.com/ansible/latest/dev_guide/developing_modules.html)
- [Go project layout](https://github.com/golang-standards/project-layout)
