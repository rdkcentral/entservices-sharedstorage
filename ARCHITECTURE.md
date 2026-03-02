# SharedStorage Plugin - Architecture Documentation

## Overview

The SharedStorage plugin is a WPEFramework (Thunder) plugin that provides a unified abstraction layer for key-value storage operations across multiple storage backends. It acts as a facade to consolidate access to both device-local (PersistentStore) and cloud-based (CloudStore) storage systems, enabling applications to manage data with different scope requirements through a single, consistent interface.

## System Architecture

### Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   Client Applications                       │
│          (Apps requiring storage capabilities)              │
└────────────────────┬────────────────────────────────────────┘
                     │ JSON-RPC
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              SharedStorage Plugin (Thunder)                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │   SharedStorage JSONRPC Handler                       │  │
│  │   - SetValue, GetValue, DeleteKey, DeleteNamespace    │  │
│  │   - Event notifications (OnValueChanged)              │  │
│  └───────────────────┬───────────────────────────────────┘  │
│                      │                                       │
│  ┌───────────────────▼───────────────────────────────────┐  │
│  │   SharedStorageImplementation (OOP)                   │  │
│  │   - Scope routing (DEVICE vs ACCOUNT)                 │  │
│  │   - Interface management & lifecycle                  │  │
│  │   - Event dispatch & notification handling            │  │
│  └───────┬───────────────────────────────────┬───────────┘  │
└──────────┼───────────────────────────────────┼──────────────┘
           │                                   │
    DEVICE Scope                        ACCOUNT Scope
           │                                   │
           ▼                                   ▼
┌──────────────────────┐          ┌──────────────────────┐
│  PersistentStore     │          │    CloudStore        │
│  (IStore2)           │          │    (IStore2)         │
│                      │          │                      │
│  - Local SQLite DB   │          │  - Remote Cloud API  │
│  - Device storage    │          │  - Account sync      │
│  - Fast access       │          │  - Cross-device data │
└──────────────────────┘          └──────────────────────┘
```

### Component Interactions

1. **SharedStorage Plugin (Main)**: Entry point for the Thunder framework
   - Manages plugin lifecycle (Initialize/Deinitialize)
   - Creates out-of-process implementation instance
   - Registers JSON-RPC handlers for client communication
   - Provides multiple interfaces: ISharedStorage, ISharedStorageInspector, ISharedStorageLimit, ISharedStorageCache

2. **SharedStorageImplementation (OOP)**: Core business logic running in separate process
   - Routes storage operations based on scope (DEVICE → PersistentStore, ACCOUNT → CloudStore)
   - Maintains connections to backend storage services via QueryInterfaceByCallsign
   - Implements event aggregation and notification dispatch
   - Thread-safe notification handling with worker pool dispatch

3. **Backend Storage Interfaces**:
   - **PersistentStore (IStore2)**: Device-scoped local storage
   - **CloudStore (IStore2)**: Account-scoped cloud storage
   - Both implement the same IStore2 interface for consistency

## Data Flow

### Write Operation (SetValue)
```
Client → JSON-RPC → SharedStorage → Scope Router → 
  [DEVICE → PersistentStore] OR [ACCOUNT → CloudStore] → 
  Backend Storage → Success Response → Client
```

### Read Operation (GetValue)
```
Client → JSON-RPC → SharedStorage → Scope Router → 
  Backend Storage → Value Retrieval → Response → Client
```

### Change Notification
```
Backend Store (PS/CS) → ValueChanged Event → 
  SharedStorageImpl Listener → Event Dispatch (Worker Pool) → 
  Registered Notification Clients → OnValueChanged Callback
```

## Plugin Framework Integration

### WPEFramework/Thunder Integration

The plugin integrates with Thunder framework through:

- **IPlugin Interface**: Lifecycle management (Initialize, Deinitialize)
- **JSONRPC**: Client-facing API exposure via Thunder controller
- **IConfiguration**: Runtime configuration through Configure() method
- **Out-of-Process Execution**: Implementation runs in separate process for stability and isolation
- **Service Registration**: Plugin registered with Thunder using SERVICE_REGISTRATION macro

### Interface Hierarchy

```
SharedStorage (IPlugin, JSONRPC)
    └─> SharedStorageImplementation
            ├─> ISharedStorage (core storage ops)
            ├─> ISharedStorageInspector (enumeration)
            ├─> ISharedStorageLimit (quota management)
            ├─> ISharedStorageCache (cache control)
            └─> IConfiguration (setup)
```

## Dependencies and Interfaces

### External Dependencies

1. **WPEFramework Core**
   - Core::ERROR_* status codes
   - Core::IWorkerPool for async dispatch
   - RPC::IRemoteConnection for OOP communication

2. **Thunder Plugins**
   - ${NAMESPACE}Plugins (plugin framework)
   - ${NAMESPACE}Definitions (type definitions)

3. **Exchange Interfaces** (entservices-apis)
   - IStore2: Primary storage interface
   - IStoreInspector: Namespace/key enumeration
   - IStoreLimit: Storage quota management
   - IStoreCache: Cache flush operations
   - ISharedStorage: Unified storage facade

4. **Backend Services**
   - PersistentStore plugin ("org.rdk.PersistentStore")
   - CloudStore plugin ("org.rdk.CloudStore")

### Internal Interfaces

- **ISharedStorage::INotification**: Event callback interface for value change notifications
- **Store2Notification**: Internal adapter for backend store events
- **RemoteConnectionNotification**: OOP connection lifecycle management

## Technical Implementation Details

### Scope-Based Routing
The plugin uses an enum-based scope system:
- **DEVICE (0)**: Routes to PersistentStore for local device storage
- **ACCOUNT (1)**: Routes to CloudStore for cloud/account-scoped data

The `getRemoteStoreObject()` method performs dynamic routing based on scope value.

### Thread Safety
- Uses `Core::CriticalSection _adminLock` for notification list protection
- Event dispatch delegated to Thunder's worker pool (Core::IWorkerPool)
- Job-based async processing prevents blocking on notification delivery

### Error Handling
- Returns Thunder Core error codes (Core::ERROR_NONE, Core::ERROR_GENERAL, etc.)
- Supports both return status and success flag output parameters
- Comprehensive logging via UtilsLogging.h macros (LOGINFO, LOGERR, TRACE)

### Memory Management
- Reference-counted interfaces (AddRef/Release pattern)
- Explicit cleanup in destructor
- Connection lifecycle tied to plugin deinitialization

### Configuration
- Config files: SharedStorage.config, SharedStorage.conf.in
- Runtime configuration via IConfiguration::Configure()
- CMake options: PLUGIN_SHAREDSTORAGE_MODE, PLUGIN_SHAREDSTORAGE_STARTUPORDER

### Event Mechanism
- Implements observer pattern for value change notifications
- Multiple clients can register for notifications
- Thread-safe registration/unregistration
- Events dispatched via Job objects to worker pool threads

## Build System Integration

- CMake-based build with Thunder/WPEFramework dependencies
- Separate libraries: Plugin (SharedStorage) and Implementation (SharedStorageImplementation)
- Installed to: `lib/${STORAGE_DIRECTORY}/plugins`
- Test support with mock library integration (RDK_SERVICE_L2_TEST)
- Helper utilities located in `../helpers` (UtilsLogging.h)

## Version Information

- API Version: 1.0.1 (Major.Minor.Patch)
- Plugin registered with Thunder using metadata system
- Version compatibility managed through Thunder's interface versioning
