# State Management

This document describes how Go-FTL2 tracks and persists state across runs.

## Overview

State management enables:

1. **Idempotent provisioning** - Track what resources have been created
2. **Crash recovery** - Resume from where execution left off
3. **Dynamic inventory** - Add hosts during execution that persist
4. **Resource tracking** - Track cloud resources, IPs, etc.

## State Structure

```go
package state

import (
    "encoding/json"
    "os"
    "sync"
    "time"
)

// State holds all persistent state
type State struct {
    // File path for persistence
    path string

    // Host state
    hosts map[string]*HostState

    // Resource state (cloud instances, IPs, etc.)
    resources map[string]*ResourceState

    // Metadata
    metadata *Metadata

    // Concurrency
    mu sync.RWMutex

    // Backend
    backend Backend
}

// HostState tracks a provisioned host
type HostState struct {
    Name      string         `json:"name"`
    Address   string         `json:"address"`
    Port      int            `json:"port"`
    User      string         `json:"user"`
    Vars      map[string]any `json:"vars,omitempty"`
    Group     string         `json:"group,omitempty"`
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
}

// ResourceState tracks a provisioned resource
type ResourceState struct {
    Type      string         `json:"type"`      // e.g., "ec2_instance", "linode"
    ID        string         `json:"id"`        // Provider-specific ID
    Name      string         `json:"name"`      // Human-readable name
    Data      map[string]any `json:"data"`      // Resource-specific data
    Tags      map[string]string `json:"tags,omitempty"`
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
}

// Metadata contains state file metadata
type Metadata struct {
    Version   string    `json:"version"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}
```

## State Operations

### Creating State

```go
package state

// New creates a new state instance
func New(path string, opts ...Option) (*State, error) {
    s := &State{
        path:      path,
        hosts:     make(map[string]*HostState),
        resources: make(map[string]*ResourceState),
        metadata: &Metadata{
            Version:   "1.0",
            CreatedAt: time.Now(),
            UpdatedAt: time.Now(),
        },
        backend: &FileBackend{path: path},
    }

    for _, opt := range opts {
        opt(s)
    }

    return s, nil
}

// Load reads state from the backend
func Load(path string) (*State, error) {
    s, err := New(path)
    if err != nil {
        return nil, err
    }

    if err := s.backend.Load(s); err != nil {
        if os.IsNotExist(err) {
            // New state file
            return s, nil
        }
        return nil, fmt.Errorf("failed to load state: %w", err)
    }

    return s, nil
}
```

### Host Operations

```go
// AddHost adds or updates a host in state
func (s *State) AddHost(host *HostState) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    now := time.Now()

    if existing, ok := s.hosts[host.Name]; ok {
        // Update existing
        existing.Address = host.Address
        existing.Port = host.Port
        existing.User = host.User
        existing.Vars = host.Vars
        existing.Group = host.Group
        existing.UpdatedAt = now
    } else {
        // Add new
        host.CreatedAt = now
        host.UpdatedAt = now
        s.hosts[host.Name] = host
    }

    s.metadata.UpdatedAt = now

    // Persist immediately
    return s.save()
}

// GetHost retrieves a host by name
func (s *State) GetHost(name string) (*HostState, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    host, ok := s.hosts[name]
    return host, ok
}

// RemoveHost removes a host from state
func (s *State) RemoveHost(name string) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    delete(s.hosts, name)
    s.metadata.UpdatedAt = time.Now()

    return s.save()
}

// HasHost returns true if the host exists in state
func (s *State) HasHost(name string) bool {
    s.mu.RLock()
    defer s.mu.RUnlock()

    _, ok := s.hosts[name]
    return ok
}

// AllHosts returns all hosts in state
func (s *State) AllHosts() []*HostState {
    s.mu.RLock()
    defer s.mu.RUnlock()

    hosts := make([]*HostState, 0, len(s.hosts))
    for _, h := range s.hosts {
        hosts = append(hosts, h)
    }
    return hosts
}
```

### Resource Operations

```go
// AddResource adds or updates a resource in state
func (s *State) AddResource(key string, res *ResourceState) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    now := time.Now()

    if existing, ok := s.resources[key]; ok {
        existing.Data = res.Data
        existing.Tags = res.Tags
        existing.UpdatedAt = now
    } else {
        res.CreatedAt = now
        res.UpdatedAt = now
        s.resources[key] = res
    }

    s.metadata.UpdatedAt = now
    return s.save()
}

// GetResource retrieves a resource by key
func (s *State) GetResource(key string) (*ResourceState, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    res, ok := s.resources[key]
    return res, ok
}

// RemoveResource removes a resource from state
func (s *State) RemoveResource(key string) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    delete(s.resources, key)
    s.metadata.UpdatedAt = time.Now()

    return s.save()
}

// GetResourcesByType returns all resources of a given type
func (s *State) GetResourcesByType(resourceType string) []*ResourceState {
    s.mu.RLock()
    defer s.mu.RUnlock()

    var resources []*ResourceState
    for _, res := range s.resources {
        if res.Type == resourceType {
            resources = append(resources, res)
        }
    }
    return resources
}
```

### Persistence

```go
// save writes state to the backend
func (s *State) save() error {
    return s.backend.Save(s)
}

// Save explicitly saves state (for manual control)
func (s *State) Save() error {
    s.mu.Lock()
    defer s.mu.Unlock()

    return s.save()
}
```

## File Backend

```go
package state

import (
    "encoding/json"
    "os"
)

// FileBackend stores state in a JSON file
type FileBackend struct {
    path string
}

// stateFile is the serialization format
type stateFile struct {
    Metadata  *Metadata                 `json:"metadata"`
    Hosts     map[string]*HostState     `json:"hosts"`
    Resources map[string]*ResourceState `json:"resources"`
}

func (b *FileBackend) Load(s *State) error {
    data, err := os.ReadFile(b.path)
    if err != nil {
        return err
    }

    var sf stateFile
    if err := json.Unmarshal(data, &sf); err != nil {
        return fmt.Errorf("invalid state file: %w", err)
    }

    s.metadata = sf.Metadata
    s.hosts = sf.Hosts
    s.resources = sf.Resources

    if s.hosts == nil {
        s.hosts = make(map[string]*HostState)
    }
    if s.resources == nil {
        s.resources = make(map[string]*ResourceState)
    }

    return nil
}

func (b *FileBackend) Save(s *State) error {
    sf := stateFile{
        Metadata:  s.metadata,
        Hosts:     s.hosts,
        Resources: s.resources,
    }

    data, err := json.MarshalIndent(sf, "", "  ")
    if err != nil {
        return fmt.Errorf("failed to marshal state: %w", err)
    }

    // Write atomically
    tmpPath := b.path + ".tmp"
    if err := os.WriteFile(tmpPath, data, 0644); err != nil {
        return fmt.Errorf("failed to write state: %w", err)
    }

    if err := os.Rename(tmpPath, b.path); err != nil {
        return fmt.Errorf("failed to rename state file: %w", err)
    }

    return nil
}
```

## State Merging

Merge state hosts into inventory for dynamic inventory:

```go
package state

import (
    "github.com/org/go-ftl2/pkg/inventory"
)

// MergeToInventory adds state hosts to an inventory
func (s *State) MergeToInventory(inv *inventory.Inventory) error {
    s.mu.RLock()
    defer s.mu.RUnlock()

    for _, hostState := range s.hosts {
        host := &inventory.Host{
            Name:    hostState.Name,
            Address: hostState.Address,
            Port:    hostState.Port,
            User:    hostState.User,
            Vars:    hostState.Vars,
        }

        groupName := hostState.Group
        if groupName == "" {
            groupName = "all"
        }

        if err := inv.AddHost(groupName, host); err != nil {
            return fmt.Errorf("failed to add host %s: %w", host.Name, err)
        }
    }

    return nil
}

// FromInventoryHost creates HostState from inventory Host
func FromInventoryHost(host *inventory.Host, group string) *HostState {
    return &HostState{
        Name:    host.Name,
        Address: host.Address,
        Port:    host.Port,
        User:    host.User,
        Vars:    host.Vars,
        Group:   group,
    }
}
```

## State File Format

Example `state.json`:

```json
{
  "metadata": {
    "version": "1.0",
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-15T14:45:00Z"
  },
  "hosts": {
    "web01": {
      "name": "web01",
      "address": "192.168.1.10",
      "port": 22,
      "user": "admin",
      "group": "webservers",
      "vars": {
        "http_port": 80
      },
      "created_at": "2024-01-15T10:30:00Z",
      "updated_at": "2024-01-15T10:30:00Z"
    },
    "db01": {
      "name": "db01",
      "address": "192.168.1.20",
      "port": 22,
      "user": "admin",
      "group": "databases",
      "vars": {
        "postgres_version": "15"
      },
      "created_at": "2024-01-15T10:35:00Z",
      "updated_at": "2024-01-15T14:45:00Z"
    }
  },
  "resources": {
    "linode:12345678": {
      "type": "linode",
      "id": "12345678",
      "name": "web01",
      "data": {
        "label": "web01",
        "region": "us-east",
        "type": "g6-standard-2",
        "ipv4": ["192.168.1.10"]
      },
      "tags": {
        "environment": "production",
        "role": "webserver"
      },
      "created_at": "2024-01-15T10:30:00Z",
      "updated_at": "2024-01-15T10:30:00Z"
    }
  }
}
```

## Integration with AutomationContext

```go
package automation

func (c *Context) AddHost(name string, config HostConfig) error {
    // Add to inventory
    host := &inventory.Host{
        Name:    name,
        Address: config.Address,
        Port:    config.Port,
        User:    config.User,
        Vars:    config.Vars,
    }

    if err := c.inventory.AddHost(config.Group, host); err != nil {
        return err
    }

    // Add to state if state is enabled
    if c.state != nil {
        hostState := &state.HostState{
            Name:    name,
            Address: config.Address,
            Port:    config.Port,
            User:    config.User,
            Vars:    config.Vars,
            Group:   config.Group,
        }

        if err := c.state.AddHost(hostState); err != nil {
            return fmt.Errorf("failed to persist host: %w", err)
        }
    }

    return nil
}
```

## Concurrency Safety

All state operations are thread-safe:

1. Read operations use `RLock()` for concurrent reads
2. Write operations use `Lock()` for exclusive access
3. Writes persist immediately to prevent data loss
4. Atomic file writes prevent corruption
