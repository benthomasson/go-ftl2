# Package Structure

This document describes the Go package layout for FTL2.

## Directory Layout

```
go-ftl2/
├── cmd/
│   └── ftl2/
│       └── main.go              # CLI entrypoint
│
├── pkg/
│   ├── automation/              # User-facing API
│   │   ├── context.go           # AutomationContext
│   │   ├── proxy.go             # ModuleProxy for host scoping
│   │   ├── options.go           # Functional options
│   │   └── automation.go        # Package exports
│   │
│   ├── executor/                # Execution orchestration
│   │   ├── executor.go          # ModuleExecutor
│   │   ├── runner.go            # Runner interface
│   │   ├── local.go             # LocalRunner implementation
│   │   ├── remote.go            # RemoteRunner implementation
│   │   └── chunk.go             # Host chunking utilities
│   │
│   ├── gate/                    # Remote gate system
│   │   ├── builder.go           # GateBuilder
│   │   ├── protocol.go          # Message encoding/decoding
│   │   ├── messages.go          # Message type definitions
│   │   └── cache.go             # Gate caching
│   │
│   ├── modules/                 # Module system
│   │   ├── ftl/                 # Native FTL module implementations
│   │   │   ├── file.go
│   │   │   ├── command.go
│   │   │   ├── copy.go
│   │   │   └── registry.go
│   │   ├── loader.go            # Module resolution
│   │   ├── fqcn.go              # FQCN parsing
│   │   ├── bundle.go            # Module bundling
│   │   └── ansible.go           # Ansible module execution
│   │
│   ├── inventory/               # Inventory management
│   │   ├── inventory.go         # Inventory loading
│   │   ├── host.go              # Host type
│   │   ├── group.go             # HostGroup type
│   │   └── parser.go            # YAML parsing
│   │
│   ├── state/                   # State persistence
│   │   ├── state.go             # State manager
│   │   ├── file.go              # JSON file I/O
│   │   └── merge.go             # State → inventory merging
│   │
│   ├── ssh/                     # SSH client
│   │   ├── client.go            # SSH client wrapper
│   │   ├── connection.go        # Connection management
│   │   └── pool.go              # Connection pooling
│   │
│   ├── types/                   # Shared types
│   │   ├── config.go            # Configuration types
│   │   ├── result.go            # Result types
│   │   └── errors.go            # Error types
│   │
│   └── internal/                # Internal utilities
│       ├── logging/             # Structured logging
│       ├── secrets/             # Secret management
│       └── retry/               # Retry logic
│
├── docs/                        # Documentation
│   ├── architecture/            # Architecture docs
│   └── diagrams/                # Mermaid diagrams
│
├── go.mod
├── go.sum
└── README.md
```

## Package Diagram

![Package Structure](../diagrams/package-structure.png)

## Package Responsibilities

### `cmd/ftl2`

CLI entrypoint using [cobra](https://github.com/spf13/cobra) for command parsing:

```go
// Commands:
// ftl2 run <playbook.go>     - Execute a playbook
// ftl2 inventory <file>      - List inventory
// ftl2 state <file>          - Inspect state
// ftl2 version               - Show version
```

### `pkg/automation`

User-facing API that orchestrates all other packages:

```go
package automation

// New creates a new AutomationContext with the given options
func New(opts ...Option) (*Context, error)

// Context is the main orchestration point
type Context struct {
    inventory *inventory.Inventory
    state     *state.State
    executor  *executor.Executor
    secrets   *secrets.Manager
    // ...
}

// Execute runs a module against target hosts
func (c *Context) Execute(ctx context.Context, target, module string, args map[string]any) (*Results, error)

// Close cleans up resources
func (c *Context) Close() error
```

### `pkg/executor`

Concurrent execution engine:

```go
package executor

// Executor orchestrates module execution across hosts
type Executor struct {
    chunkSize int
    failFast  bool
    runner    Runner
}

// Runner is the interface for executing modules
type Runner interface {
    Run(ctx context.Context, host *inventory.Host, module string, args map[string]any) (*types.ModuleResult, error)
}

// LocalRunner executes modules locally
type LocalRunner struct{}

// RemoteRunner executes modules via SSH gate
type RemoteRunner struct {
    sshClient   *ssh.Client
    gateBuilder *gate.Builder
}
```

### `pkg/gate`

Remote execution protocol:

```go
package gate

// Builder creates and caches gate executables
type Builder struct {
    cacheDir string
    modules  []string
}

// Protocol handles message encoding/decoding
type Protocol struct {
    reader io.Reader
    writer io.Writer
}

// Message types
type HelloMessage struct { ... }
type ModuleMessage struct { ... }
type ResultMessage struct { ... }
type ShutdownMessage struct { ... }
```

### `pkg/modules`

Module resolution and execution:

```go
package modules

// Loader resolves module names to executables
type Loader struct {
    searchPaths []string
    ftlModules  map[string]FTLModule
}

// FTLModule is a native Go module implementation
type FTLModule interface {
    Run(ctx context.Context, args map[string]any) (*types.ModuleResult, error)
}

// FQCN represents a fully qualified collection name
type FQCN struct {
    Namespace  string
    Collection string
    Module     string
}
```

### `pkg/inventory`

Host and group management:

```go
package inventory

// Inventory holds all hosts and groups
type Inventory struct {
    groups map[string]*HostGroup
}

// HostGroup is a named collection of hosts
type HostGroup struct {
    Name  string
    Hosts map[string]*Host
    Vars  map[string]any
}

// Host represents a target machine
type Host struct {
    Name       string
    Address    string
    Port       int
    User       string
    PrivateKey string
    Vars       map[string]any
}
```

### `pkg/state`

Persistent state tracking:

```go
package state

// State tracks provisioned resources
type State struct {
    path      string
    hosts     map[string]*HostState
    resources map[string]*ResourceState
}

// Add adds or updates a resource in state
func (s *State) Add(key string, value any) error

// Get retrieves a resource from state
func (s *State) Get(key string) (any, bool)

// Save persists state to disk
func (s *State) Save() error
```

### `pkg/ssh`

SSH connection management:

```go
package ssh

// Client wraps SSH connections
type Client struct {
    config *ssh.ClientConfig
    pool   *Pool
}

// Connect establishes a connection to a host
func (c *Client) Connect(ctx context.Context, host *inventory.Host) (*Connection, error)

// Connection represents an active SSH connection
type Connection struct {
    client  *ssh.Client
    session *ssh.Session
}
```

### `pkg/types`

Shared type definitions:

```go
package types

// ExecutionConfig holds module execution parameters
type ExecutionConfig struct {
    ModuleName string
    ModuleArgs map[string]any
    DryRun     bool
    Diff       bool
}

// ModuleResult is the outcome of module execution
type ModuleResult struct {
    Changed  bool
    Failed   bool
    Msg      string
    Data     map[string]any
    Warnings []string
}

// ExecutionResults aggregates results across hosts
type ExecutionResults struct {
    Results    map[string]*ModuleResult
    Successful int
    Failed     int
}
```

## Import Graph

```
cmd/ftl2
    └── pkg/automation
            ├── pkg/executor
            │       ├── pkg/ssh
            │       ├── pkg/gate
            │       └── pkg/modules
            ├── pkg/inventory
            ├── pkg/state
            └── pkg/types
```

## Visibility Rules

- `pkg/*` - Public packages, importable by external code
- `pkg/internal/*` - Internal utilities, not importable externally
- Types in `pkg/types` are shared across packages
- Each package owns its primary types (e.g., `inventory.Host`, `executor.Runner`)
