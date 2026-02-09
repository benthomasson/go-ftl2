# Execution Engine

This document describes the concurrent execution model for Go-FTL2.

## Concurrency Model Diagram

![Concurrency Model](../diagrams/concurrency-model.png)

## Execution Flow Diagram

![Execution Flow](../diagrams/execution-flow.png)

## Overview

The execution engine processes hosts in chunks with bounded concurrency. This approach:

- Prevents overwhelming remote systems with too many connections
- Allows controlled failure handling (fail-fast or continue)
- Provides predictable resource usage

## ModuleExecutor

The `ModuleExecutor` is the main orchestration point:

```go
package executor

import (
    "context"
    "sync"

    "golang.org/x/sync/errgroup"
)

type Executor struct {
    // Configuration
    chunkSize int
    failFast  bool

    // Runners
    localRunner  Runner
    remoteRunner Runner

    // Logging
    logger *slog.Logger
}

type ExecutorOption func(*Executor)

func NewExecutor(opts ...ExecutorOption) *Executor {
    e := &Executor{
        chunkSize: 10, // default
        failFast:  false,
    }
    for _, opt := range opts {
        opt(e)
    }
    return e
}

func WithChunkSize(size int) ExecutorOption {
    return func(e *Executor) {
        e.chunkSize = size
    }
}

func WithFailFast(enabled bool) ExecutorOption {
    return func(e *Executor) {
        e.failFast = enabled
    }
}

func WithLocalRunner(r Runner) ExecutorOption {
    return func(e *Executor) {
        e.localRunner = r
    }
}

func WithRemoteRunner(r Runner) ExecutorOption {
    return func(e *Executor) {
        e.remoteRunner = r
    }
}
```

## Execution Logic

### Main Execute Method

```go
func (e *Executor) Execute(
    ctx context.Context,
    hosts []*inventory.Host,
    config *types.ExecutionConfig,
) (*types.ExecutionResults, error) {
    startTime := time.Now()

    results := &types.ExecutionResults{
        Results:   make(map[string]*types.ModuleResult),
        StartTime: startTime,
    }

    // Process hosts in chunks
    chunks := chunk(hosts, e.chunkSize)

    for _, hostChunk := range chunks {
        if err := e.executeChunk(ctx, hostChunk, config, results); err != nil {
            if e.failFast {
                results.Duration = time.Since(startTime)
                return results, err
            }
            // Continue on non-fail-fast
        }
    }

    results.Duration = time.Since(startTime)
    return results, nil
}
```

### Chunk Processing

```go
func (e *Executor) executeChunk(
    ctx context.Context,
    hosts []*inventory.Host,
    config *types.ExecutionConfig,
    results *types.ExecutionResults,
) error {
    var mu sync.Mutex
    g, gctx := errgroup.WithContext(ctx)

    // Limit concurrency to chunk size
    g.SetLimit(len(hosts))

    for _, host := range hosts {
        host := host // capture for goroutine

        g.Go(func() error {
            result, err := e.executeOnHost(gctx, host, config)

            mu.Lock()
            defer mu.Unlock()

            if err != nil {
                results.Results[host.Name] = &types.ModuleResult{
                    Failed: true,
                    Msg:    err.Error(),
                }
                results.Failed++
                if e.failFast {
                    return err
                }
                return nil // Continue on failure
            }

            results.Results[host.Name] = result
            if result.Failed {
                results.Failed++
            } else {
                results.Successful++
            }
            if result.Changed {
                results.Changed++
            }

            return nil
        })
    }

    return g.Wait()
}
```

### Host Execution

```go
func (e *Executor) executeOnHost(
    ctx context.Context,
    host *inventory.Host,
    config *types.ExecutionConfig,
) (*types.ModuleResult, error) {
    // Select runner based on host type
    var runner Runner
    if host.IsLocal() {
        runner = e.localRunner
    } else {
        runner = e.remoteRunner
    }

    if runner == nil {
        return nil, fmt.Errorf("no runner available for host %s", host.Name)
    }

    // Execute with timing
    startTime := time.Now()
    result, err := runner.Run(ctx, host, config)
    if err != nil {
        return nil, &types.ModuleError{
            Host:   host.Name,
            Module: config.ModuleName,
            Err:    err,
        }
    }

    result.StartTime = startTime
    result.Duration = time.Since(startTime)

    return result, nil
}
```

## Chunking Utility

```go
package executor

// chunk splits a slice into chunks of the given size
func chunk[T any](items []T, size int) [][]T {
    if size <= 0 {
        size = 1
    }

    var chunks [][]T
    for size < len(items) {
        items, chunks = items[size:], append(chunks, items[0:size:size])
    }
    return append(chunks, items)
}
```

## Runner Interface

```go
package executor

// Runner executes a module on a single host
type Runner interface {
    Run(ctx context.Context, host *inventory.Host, config *types.ExecutionConfig) (*types.ModuleResult, error)
    Close() error
}
```

## LocalRunner

Executes modules on the local machine:

```go
package executor

type LocalRunner struct {
    moduleLoader *modules.Loader
}

func NewLocalRunner(loader *modules.Loader) *LocalRunner {
    return &LocalRunner{moduleLoader: loader}
}

func (r *LocalRunner) Run(
    ctx context.Context,
    host *inventory.Host,
    config *types.ExecutionConfig,
) (*types.ModuleResult, error) {
    // Check for FTL module first
    if ftlMod, ok := r.moduleLoader.GetFTLModule(config.ModuleName); ok {
        return ftlMod.Run(ctx, config.ModuleArgs)
    }

    // Fall back to Ansible module via subprocess
    return r.runAnsibleModule(ctx, config)
}

func (r *LocalRunner) runAnsibleModule(
    ctx context.Context,
    config *types.ExecutionConfig,
) (*types.ModuleResult, error) {
    // Serialize args to JSON
    argsJSON, err := json.Marshal(config.ModuleArgs)
    if err != nil {
        return nil, fmt.Errorf("failed to serialize args: %w", err)
    }

    // Find and execute the Ansible module
    modulePath, err := r.moduleLoader.FindModule(config.ModuleName)
    if err != nil {
        return nil, err
    }

    cmd := exec.CommandContext(ctx, "python3", modulePath)
    cmd.Stdin = bytes.NewReader(argsJSON)

    output, err := cmd.Output()
    if err != nil {
        return nil, fmt.Errorf("module execution failed: %w", err)
    }

    var result types.ModuleResult
    if err := json.Unmarshal(output, &result); err != nil {
        return nil, fmt.Errorf("failed to parse module output: %w", err)
    }

    return &result, nil
}

func (r *LocalRunner) Close() error {
    return nil
}
```

## RemoteRunner

Executes modules on remote hosts via SSH gate:

```go
package executor

type RemoteRunner struct {
    sshClient   *ssh.Client
    gateBuilder *gate.Builder
    gateCache   map[string]*gate.Connection // host -> gate connection
    mu          sync.RWMutex
}

func NewRemoteRunner(sshClient *ssh.Client, gateBuilder *gate.Builder) *RemoteRunner {
    return &RemoteRunner{
        sshClient:   sshClient,
        gateBuilder: gateBuilder,
        gateCache:   make(map[string]*gate.Connection),
    }
}

func (r *RemoteRunner) Run(
    ctx context.Context,
    host *inventory.Host,
    config *types.ExecutionConfig,
) (*types.ModuleResult, error) {
    // Get or create gate connection
    gateConn, err := r.getGateConnection(ctx, host)
    if err != nil {
        return nil, err
    }

    // Send module message
    msg := &gate.ModuleMessage{
        Type:       "module",
        Name:       config.ModuleName,
        Args:       config.ModuleArgs,
        CheckMode:  config.DryRun,
    }

    result, err := gateConn.SendModule(ctx, msg)
    if err != nil {
        return nil, fmt.Errorf("gate execution failed: %w", err)
    }

    return result, nil
}

func (r *RemoteRunner) getGateConnection(
    ctx context.Context,
    host *inventory.Host,
) (*gate.Connection, error) {
    r.mu.RLock()
    if conn, ok := r.gateCache[host.Name]; ok {
        r.mu.RUnlock()
        return conn, nil
    }
    r.mu.RUnlock()

    r.mu.Lock()
    defer r.mu.Unlock()

    // Double-check after acquiring write lock
    if conn, ok := r.gateCache[host.Name]; ok {
        return conn, nil
    }

    // Build and deploy gate
    gatePath, err := r.gateBuilder.Build(ctx)
    if err != nil {
        return nil, fmt.Errorf("failed to build gate: %w", err)
    }

    // Connect via SSH
    sshConn, err := r.sshClient.Connect(ctx, host)
    if err != nil {
        return nil, err
    }

    // Deploy gate
    remotePath := fmt.Sprintf("/tmp/gate-%s.pyz", r.gateBuilder.Hash())
    if err := sshConn.Upload(ctx, gatePath, remotePath); err != nil {
        return nil, fmt.Errorf("failed to deploy gate: %w", err)
    }

    // Start gate process
    gateConn, err := gate.NewConnection(ctx, sshConn, remotePath)
    if err != nil {
        return nil, fmt.Errorf("failed to start gate: %w", err)
    }

    r.gateCache[host.Name] = gateConn
    return gateConn, nil
}

func (r *RemoteRunner) Close() error {
    r.mu.Lock()
    defer r.mu.Unlock()

    var errs []error
    for _, conn := range r.gateCache {
        if err := conn.Shutdown(); err != nil {
            errs = append(errs, err)
        }
    }
    r.gateCache = make(map[string]*gate.Connection)

    if len(errs) > 0 {
        return fmt.Errorf("failed to close gate connections: %v", errs)
    }
    return nil
}
```

## Context Cancellation

The execution engine respects context cancellation:

```go
func (e *Executor) executeOnHost(
    ctx context.Context,
    host *inventory.Host,
    config *types.ExecutionConfig,
) (*types.ModuleResult, error) {
    // Check for cancellation before starting
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }

    // ... execution logic ...
}
```

## Fail-Fast Behavior

When `failFast` is enabled:

1. First failure cancels the context for the current chunk
2. Remaining hosts in the chunk are abandoned
3. No subsequent chunks are processed
4. Partial results are returned

```go
if e.failFast {
    return err // This cancels errgroup context
}
return nil // Continue despite failure
```

## Retry Logic

For transient failures, the executor supports retry:

```go
package internal/retry

type Config struct {
    MaxAttempts int
    InitialDelay time.Duration
    MaxDelay     time.Duration
    Multiplier   float64
}

func WithRetry[T any](
    ctx context.Context,
    cfg Config,
    fn func() (T, error),
) (T, error) {
    var result T
    var err error
    delay := cfg.InitialDelay

    for attempt := 0; attempt < cfg.MaxAttempts; attempt++ {
        result, err = fn()
        if err == nil {
            return result, nil
        }

        // Check if error is retryable
        if !isRetryable(err) {
            return result, err
        }

        // Wait before retry
        select {
        case <-ctx.Done():
            return result, ctx.Err()
        case <-time.After(delay):
        }

        // Exponential backoff
        delay = time.Duration(float64(delay) * cfg.Multiplier)
        if delay > cfg.MaxDelay {
            delay = cfg.MaxDelay
        }
    }

    return result, fmt.Errorf("max retries exceeded: %w", err)
}
```
