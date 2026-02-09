# go-ftl2

A Go port of [FTL2](https://github.com/benthomasson/ftl2/) (Faster Than Light v2), a high-performance automation framework that runs Ansible modules in-process.

## Overview

FTL2 achieves 3-17x speedup over traditional `ansible-playbook` by running modules directly rather than spawning subprocesses. This Go port aims to provide:

- **Native performance** - Compiled binary with efficient concurrency
- **Single binary distribution** - No Python runtime for the controller
- **Lower memory footprint** - ~50MB vs ~200MB for Python
- **Fast startup** - ~10ms vs ~500ms for Python CLI

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Go Controller                            │
├─────────────┬─────────────┬─────────────┬──────────────────┤
│ Automation  │  Executor   │   Modules   │      SSH         │
│  Context    │  (chunked)  │  (FTL/FQCN) │    Client        │
└──────┬──────┴──────┬──────┴──────┬──────┴────────┬─────────┘
       │             │             │               │
       ▼             ▼             ▼               ▼
┌─────────────────────────────────────────────────────────────┐
│                    Remote Hosts                              │
├─────────────────────────────────────────────────────────────┤
│  Python Gate (.pyz) - Executes Ansible/FTL modules          │
└─────────────────────────────────────────────────────────────┘
```

## Status

**Design Phase** - Architecture documentation complete, implementation not yet started.

See [docs/architecture/](docs/architecture/) for detailed design documents.

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture Overview](docs/architecture/00-overview.md) | High-level design and principles |
| [Package Structure](docs/architecture/01-package-structure.md) | Go package layout |
| [Core Types](docs/architecture/02-core-types.md) | Type definitions and interfaces |
| [Execution Engine](docs/architecture/03-execution-engine.md) | Concurrency model |
| [Gate Protocol](docs/architecture/04-gate-protocol.md) | Remote execution protocol |
| [Module System](docs/architecture/05-module-system.md) | Module resolution |
| [State Management](docs/architecture/06-state-management.md) | Persistent state |
| [Implementation Roadmap](docs/architecture/07-implementation-roadmap.md) | Phased plan |
| [Concerns and Tradeoffs](docs/architecture/08-concerns-and-tradeoffs.md) | Risks and limitations |

## Planned Usage

```go
package main

import (
    "context"
    "log"

    "github.com/benthomasson/go-ftl2/pkg/automation"
)

func main() {
    ctx, err := automation.New(
        automation.WithInventory("inventory.yaml"),
        automation.WithSecrets("AWS_ACCESS_KEY", "AWS_SECRET_KEY"),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer ctx.Close()

    results, err := ctx.Execute(
        context.Background(),
        "webservers",
        "dnf",
        map[string]any{"name": "nginx", "state": "present"},
    )
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Changed: %d, Failed: %d", results.Changed, results.Failed)
}
```

## Key Differences from Python FTL2

| Aspect | Python FTL2 | Go FTL2 |
|--------|-------------|---------|
| Module syntax | `ftl.webservers.dnf()` | `ctx.Execute("webservers", "dnf", args)` |
| Concurrency | asyncio | goroutines + channels |
| Configuration | Dataclasses | Structs + functional options |
| Error handling | Exceptions | Explicit error returns |
| Distribution | pip package | Single binary |

## Requirements

- Go 1.21+
- Python 3.13+ (on remote hosts, for gate execution)

## License

MIT License - See [LICENSE](LICENSE) for details.

## Related Projects

- [ftl2](https://github.com/benthomasson/ftl2/) - Original Python implementation
- [Ansible](https://github.com/ansible/ansible) - Automation platform (module compatibility)
