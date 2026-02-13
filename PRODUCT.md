# SharedStorage Plugin - Product Documentation

## Product Overview

SharedStorage is a unified storage abstraction plugin for RDK-based devices that provides applications with seamless access to both local device storage and cloud-based storage through a single, consistent API. The plugin eliminates the complexity of managing multiple storage backends by intelligently routing operations based on data scope requirements, enabling developers to focus on application logic rather than storage infrastructure.

## Key Features

### Unified Storage Interface
- **Single API for Multiple Backends**: Access both local (PersistentStore) and cloud (CloudStore) storage through one unified interface
- **Automatic Scope Routing**: Operations automatically directed to appropriate storage backend based on DEVICE or ACCOUNT scope
- **Consistent Error Handling**: Uniform error codes and response structures across all storage operations
- **Type-Safe Operations**: Strongly-typed interfaces prevent common storage access errors

### Storage Operations
- **SetValue**: Store key-value pairs with optional TTL (time-to-live) expiration
- **GetValue**: Retrieve values with TTL information
- **DeleteKey**: Remove specific keys from storage
- **DeleteNamespace**: Bulk deletion of entire namespace contents
- **GetKeys**: Enumerate all keys within a namespace
- **GetNamespaces**: List all available namespaces in a scope

### Advanced Management
- **Storage Inspection**: Query namespace sizes, enumerate keys and namespaces
- **Quota Management**: Set and retrieve storage limits per namespace
- **Cache Control**: Force cache flush to ensure data persistence
- **Real-time Notifications**: Subscribe to value change events across storage backends

### Multi-Scope Support
- **DEVICE Scope**: Local, device-specific storage (fast, always available)
- **ACCOUNT Scope**: Cloud-based, user-account storage (synchronized across devices)

## Use Cases and Target Scenarios

### Application Settings Management
**Scenario**: An application needs to persist user preferences both locally and in the cloud.

**Solution**: 
- Use DEVICE scope for local settings (UI preferences, cache data)
- Use ACCOUNT scope for user preferences that should sync across devices
- Subscribe to change notifications to update UI when settings change remotely

**Benefits**: Seamless experience across devices, offline capability for local settings

### Multi-Device State Synchronization
**Scenario**: A streaming application wants to maintain watch history and continue watching position across multiple devices.

**Solution**:
- Store viewing progress using ACCOUNT scope
- Leverage CloudStore backend for automatic synchronization
- Use change notifications to update UI when progress changes from another device

**Benefits**: Users can seamlessly switch devices without losing context

### Device Configuration Management
**Scenario**: System services need to store device-specific configuration that persists across reboots.

**Solution**:
- Use DEVICE scope with PersistentStore backend
- Organize configs by namespace (e.g., "system", "network", "display")
- Set storage limits per namespace to prevent unlimited growth

**Benefits**: Reliable, persistent configuration storage with quota management

### Multi-Tenancy Support
**Scenario**: A multi-user device needs isolated storage per user account.

**Solution**:
- Use ACCOUNT scope with different namespace per user
- CloudStore handles authentication and isolation
- SharedStorage provides unified access pattern

**Benefits**: Secure, isolated storage without application complexity

### Feature Flags and Remote Configuration
**Scenario**: Enable/disable features remotely without application updates.

**Solution**:
- Store feature flags in ACCOUNT scope
- Subscribe to change notifications
- Application updates behavior when flags change

**Benefits**: Dynamic feature control, A/B testing capability

## API Capabilities and Integration

### JSON-RPC API Methods

#### Core Storage Operations
```json
// Set value
{
  "jsonrpc": "2.0",
  "method": "SharedStorage.1.setValue",
  "params": {
    "scope": 0,           // 0=DEVICE, 1=ACCOUNT
    "namespace": "app1",
    "key": "setting",
    "value": "enabled",
    "ttl": 0              // 0=never expire
  }
}

// Get value
{
  "jsonrpc": "2.0",
  "method": "SharedStorage.1.getValue",
  "params": {
    "scope": 0,
    "namespace": "app1",
    "key": "setting"
  }
}
```

#### Inspection Operations
```json
// Get keys in namespace
{
  "jsonrpc": "2.0",
  "method": "SharedStorage.1.getKeys",
  "params": {
    "scope": 0,
    "namespace": "app1"
  }
}

// Get all namespaces
{
  "jsonrpc": "2.0",
  "method": "SharedStorage.1.getNamespaces",
  "params": {
    "scope": 0
  }
}

// Get storage sizes
{
  "jsonrpc": "2.0",
  "method": "SharedStorage.1.getStorageSizes",
  "params": {
    "scope": 0
  }
}
```

#### Management Operations
```json
// Set namespace storage limit
{
  "jsonrpc": "2.0",
  "method": "SharedStorage.1.setNamespaceStorageLimit",
  "params": {
    "scope": 0,
    "namespace": "app1",
    "storageLimit": 10485760  // 10MB in bytes
  }
}

// Flush cache
{
  "jsonrpc": "2.0",
  "method": "SharedStorage.1.flushCache",
  "params": {}
}
```

### Event Notifications
```json
// Subscribe to changes
{
  "jsonrpc": "2.0",
  "method": "SharedStorage.1.register",
  "params": {
    "event": "onValueChanged",
    "id": "client-1"
  }
}

// Event notification
{
  "jsonrpc": "2.0",
  "method": "client.events.1.onValueChanged",
  "params": {
    "scope": 0,
    "namespace": "app1",
    "key": "setting",
    "value": "disabled"
  }
}
```

### C++ Integration (Native)
```cpp
// Get SharedStorage interface
auto sharedStorage = service->QueryInterfaceByCallsign<Exchange::ISharedStorage>(
    "org.rdk.SharedStorage");

// Set value
Exchange::ISharedStorage::Success result;
sharedStorage->SetValue(
    Exchange::ISharedStorage::ScopeType::DEVICE,
    "app1",
    "setting",
    "enabled",
    0,  // ttl
    result);

// Register for notifications
class MyNotification : public Exchange::ISharedStorage::INotification {
    void OnValueChanged(const ScopeType scope, const string& ns, 
                       const string& key, const string& value) override {
        // Handle change
    }
};
sharedStorage->Register(&myNotification);
```

## Integration Benefits

### For Application Developers
- **Simplified Storage Access**: No need to understand PersistentStore vs CloudStore differences
- **Automatic Scope Handling**: Storage backend selection handled transparently
- **Event-Driven Architecture**: React to storage changes in real-time
- **Namespace Organization**: Logical data grouping prevents key collisions

### For System Integrators
- **Centralized Storage Management**: Single plugin to monitor and control
- **Quota Enforcement**: Prevent storage exhaustion with per-namespace limits
- **Storage Inspection**: Debug and audit storage usage across applications
- **Backend Abstraction**: Swap storage implementations without affecting clients

### For Platform Providers
- **Modular Architecture**: Plugin can be enabled/disabled via build configuration
- **Cloud Integration**: Built-in cloud storage support via CloudStore backend
- **Standards-Based**: Uses Thunder/WPEFramework standard patterns
- **Process Isolation**: Out-of-process implementation protects main Thunder process

## Performance and Reliability Characteristics

### Performance Profile

#### Latency
- **DEVICE Scope (PersistentStore)**: 
  - Typical latency: <10ms for read/write operations
  - Local SQLite database with optimized indexing
  - In-memory caching for frequently accessed data

- **ACCOUNT Scope (CloudStore)**:
  - Latency depends on network conditions
  - Typical range: 50-500ms for cloud round-trip
  - Asynchronous operations recommended

#### Throughput
- **Concurrent Operations**: Thread-safe implementation supports multiple simultaneous clients
- **Event Dispatch**: Non-blocking notification delivery via worker pool
- **Batch Operations**: Namespace deletion optimized for bulk key removal

### Reliability Features

#### Data Durability
- **DEVICE Scope**: SQLite with transaction support ensures ACID properties
- **ACCOUNT Scope**: Cloud backend handles replication and redundancy
- **Cache Management**: FlushCache() forces write-through to persistent storage

#### Error Handling
- **Graceful Degradation**: Backend unavailability returns error codes, doesn't crash
- **Automatic Reconnection**: Backend connections re-established on recovery
- **Transaction Safety**: Failed operations leave storage in consistent state

#### Process Isolation
- **Out-of-Process Design**: Implementation crash doesn't affect Thunder controller
- **Connection Monitoring**: Automatic cleanup on implementation process failure
- **Recovery Mechanism**: Thunder can restart implementation process

### Scalability Characteristics

#### Storage Capacity
- **DEVICE Scope**: Limited by local filesystem capacity
- **Namespace Quotas**: Configurable per-namespace limits prevent unbounded growth
- **Storage Monitoring**: GetStorageSizes() enables proactive capacity management

#### Client Scalability
- **Multiple Subscribers**: Supports numerous concurrent notification clients
- **Efficient Dispatch**: Worker pool prevents thread exhaustion
- **Memory Management**: Reference-counted interfaces prevent leaks

### Monitoring and Observability

#### Logging
- Comprehensive logging via UtilsLogging framework
- Configurable log levels (INFO, ERROR, TRACE)
- Operation tracing for debugging

#### Metrics
- Storage size reporting per namespace
- Operation success/failure tracking
- Backend availability status

## Configuration and Deployment

### Build Configuration
```cmake
# Enable SharedStorage plugin
-DPLUGIN_SHAREDSTORAGE=ON

# Configure startup mode (Off, Local, Container, Distributed)
-DPLUGIN_SHAREDSTORAGE_MODE="Off"

# Set startup order (higher = later in boot sequence)
-DPLUGIN_SHAREDSTORAGE_STARTUPORDER="51"
```

### Runtime Configuration
- Configuration files: `SharedStorage.config`, `SharedStorage.conf.in`
- Installed to: `/etc/entservices/`
- Thunder controller configuration: Plugin registered as "SharedStorage"

### Prerequisites
- **Required Plugins**: 
  - PersistentStore (for DEVICE scope)
  - CloudStore (for ACCOUNT scope, optional)
- **Thunder Version**: Compatible with WPEFramework R4+
- **System Requirements**: Standard RDK environment

## Version and Compatibility

- **Plugin API Version**: 1.0.1
- **Interface Compatibility**: IStore2-based backends
- **Thunder Framework**: R4.x compatible
- **Backward Compatibility**: Maintains stable JSON-RPC API

## Summary

SharedStorage provides a production-ready, enterprise-grade storage abstraction layer that simplifies application development while delivering robust performance and reliability. Its unified interface, scope-based routing, and real-time notification system make it an essential component for modern RDK-based applications requiring flexible, scalable storage solutions.
