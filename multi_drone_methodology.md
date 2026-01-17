# Multi-Drone Control System: Architecture & Methodology

> [!NOTE]
> This document presents a scientifically-backed methodology for building a **scalable multi-drone control system** based on analysis of existing ArduPilot and PX4 implementations, Skydio's fleet management approach, and MAVLink best practices.

---

## 1. Existing Architecture Analysis

### 1.1 Current Single-Drone Architecture

#### ArduPilot C++ Backend ([guided_server.cpp](file:///home/salty/.gemini/antigravity/scratch/YZN/Aurdopilot-SubSystem--main/guided_server.cpp))

**Architecture Pattern:** Monolithic Single-Instance Server

```mermaid
graph TB
    A[HTTP Server :8080] --> B[MAVSDK Instance]
    B --> C[MAVLink Connection<br/>udpin://0.0.0.0:14540]
    C --> D[Single Drone System]
    B --> E[Telemetry Plugin]
    B --> F[Action Plugin]
    B --> G[MavlinkPassthrough Plugin]
    A --> H[REST API Endpoints]
    H --> I[/status - GET]
    H --> J[/arm_takeoff - POST]
    H --> K[/goto - POST]
    H --> L[/land, /rtl, /disarm - POST]
```

**Key Components:**
- **Single `Mavsdk` instance** managing one autopilot connection
- **HTTP Server** (cpp-httplib) providing RESTful API
- **Thread-safe API** using `std::mutex` for command serialization
- **Non-blocking telemetry** endpoint (`/status`) for high-frequency polling
- **Event logging** with rolling buffer (max 500 events)
- **Bearer token authentication** (optional)

**Critical Observation:**
```cpp
// Lines 228-246: Single system discovery
std::shared_ptr<System> system;
if (auto opt = mavsdk.first_autopilot(10.0); opt) {
    system = opt.value();
}
```
The current architecture **assumes a single drone** and blocks until discovery.

---

### 1.2 Python Implementation Analysis

#### ArduPilot Python Backend ([guided_server.py](file:///home/salty/.gemini/antigravity/scratch/YZN/Aurdopilot-SubSystem--main-python/guided_server.py))

**Key Insights:**
- Uses **MAVSDK-Python** wrapper (simplified async API)
- **Flask** for HTTP server (easier concurrency via threading)
- **Separate event loops** per API call (creates new loop in each endpoint)
- **No connection pooling** - reconnects for each request

**Advantages over C++:**
- **Easier async handling** with Python's asyncio
- **Simpler code** (565 lines vs 682 lines)
- **Better for rapid prototyping**

**Disadvantages:**
- **Performance overhead** (event loop recreation)
- **GIL limitations** for true multi-threading
- **Memory overhead** vs C++

---

### 1.3 PX4 Implementation Comparison

The [px4_server.cpp](file:///home/salty/.gemini/antigravity/scratch/YZN/PX4-GCS-Backend/px4_server.cpp) uses **Offboard mode** instead of ArduPilot's GUIDED mode:

```cpp
// Lines 175-219: PX4 goto using Offboard control
static bool goto_offboard(Offboard &offboard, Telemetry &tel, 
                          double target_lat, double target_lon, 
                          float target_rel_alt, double timeout_s)
{
    // Convert GPS to NED coordinates
    NED ned = gps_to_ned(origin, target_lat, target_lon, target_rel_alt);
    
    // Set position setpoint and start Offboard
    Offboard::PositionNedYaw sp{ned.north_m, ned.east_m, ned.down_m, 0.0f};
    offboard.set_position_ned(sp);
    offboard.start();
    
    // Stream setpoints until reached
    while (!reached) {
        offboard.set_position_ned(sp); // Keep streaming!
        std::this_thread::sleep_for(100ms);
    }
}
```

**Key Difference:** PX4 requires **continuous setpoint streaming** (10Hz+) during navigation.

---

## 2. Industry Best Practices Research

### 2.1 Skydio Fleet Management Architecture

Based on research, Skydio's multi-drone system uses:

#### Cloud-Native Architecture
- **Skydio Cloud** (AWS-hosted) as central orchestration layer
- **Fleet Manager** for centralized monitoring and control
- **Remote Ops** for autonomous multi-drone operations
- **Dock Integration** for pre-positioned autonomous launches

#### Key Architectural Principles:
1. **Autonomy-First Design** - AI-powered navigation enables trust in remote ops
2. **Centralized Coordination** - Single operator manages multiple drones
3. **Real-Time Streaming** - Live mission viewing from any browser
4. **Open API Integration** - RESTful APIs for fleet data access
5. **Scalable Backend** - Cloud infrastructure with horizontal scaling

> [!IMPORTANT]
> **Key Takeaway:** Skydio separates **fleet coordination** (cloud) from **drone autonomy** (edge). This allows N drones to scale without operator cognitive overload.

---

### 2.2 MAVLink Multi-Drone Patterns

#### MAVLink Design for Multi-Vehicle Systems

MAVLink protocol inherently supports **255 concurrent systems** (system IDs 1-255):

```
┌─────────────────────────────────────┐
│   Ground Control Station (ID: 255)  │
│                                     │
│  ┌──────────────────────────────┐  │
│  │   MAVSDK Instance            │  │
│  │                              │  │
│  │  • Connection Manager        │  │
│  │  • System Registry           │  │
│  │  • Plugin Factory            │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
           │
           ├─────────────────┬─────────────────┬─────────────────┐
           │                 │                 │                 │
           ▼                 ▼                 ▼                 ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐     ┌──────────┐
    │ Drone 1  │      │ Drone 2  │      │ Drone 3  │ ... │ Drone N  │
    │ ID: 1    │      │ ID: 2    │      │ ID: 3    │     │ ID: N    │
    │ Port:    │      │ Port:    │      │ Port:    │     │ Port:    │
    │ 14540    │      │ 14541    │      │ 14542    │     │ 14540+N  │
    └──────────┘      └──────────┘      └──────────┘     └──────────┘
```

#### Design Patterns for MAVSDK Multi-Drone

**Pattern 1: Single MAVSDK Instance with Multiple Systems** (C++ Recommended)

```cpp
Mavsdk mavsdk;

// Add connections for all drones
mavsdk.add_any_connection("udp://:14540");  // Drone 1
mavsdk.add_any_connection("udp://:14541");  // Drone 2
mavsdk.add_any_connection("udp://:14542");  // Drone 3

// Discover all systems
mavsdk.subscribe_on_new_system([&]() {
    for (auto system : mavsdk.systems()) {
        uint8_t id = system->get_system_id();
        // Create plugins per drone
        drone_registry[id] = std::make_unique<DroneController>(system);
    }
});
```

**Advantages:**
- ✅ **Optimal resource usage** (single MAVSDK instance)
- ✅ **Shared connection pool**
- ✅ **Native C++ performance**
- ✅ **Thread-safe by design**

**Pattern 2: Multiple MAVSDK Instances** (Python typical)

```python
# Separate MAVSDK instance per drone
drones = {}
for i, port in enumerate(drone_ports):
    drone = System()
    await drone.connect(f"udp://:{port}")
    drones[i] = drone
```

**Disadvantages:**
- ❌ **Higher memory overhead**
- ❌ **Duplicate resource allocation**
- ❌ **More complex synchronization**

---

## 3. Proposed Multi-Drone Architecture

### 3.1 Core Design Principles

> [!IMPORTANT]
> **Architectural Decisions:**
>
> 1. **Single-Process, Multi-System** - One backend instance managing N drones
> 2. **Connection-Based Routing** - Each drone on unique UDP port
> 3. **Query Parameter API** - `/endpoint?drone=N` for drone selection
> 4. **Thread-Safe State Management** - Per-drone mutex locks
> 5. **Scalable from Day 1** - Architecture supports 2-100+ drones

### 3.2 High-Level Architecture

```mermaid
graph TB
    subgraph Frontend["Frontend Dashboard"]
        UI[React/Vue UI]
        DS[Drone Selector]
        MAP[Multi-Drone Map View]
    end
    
    subgraph Backend["Multi-Drone Backend (C++)"]
        HTTP[HTTP Server :8080]
        DM[Drone Manager]
        
        subgraph Registry["Drone Registry"]
            D1[Drone Controller 1<br/>System ID: 1]
            D2[Drone Controller 2<br/>System ID: 2]
            DN[Drone Controller N<br/>System ID: N]
        end
        
        MAVSDK[MAVSDK Instance]
    end
    
    subgraph Connections["MAVLink Connections"]
        C1[UDP :14540]
        C2[UDP :14541]
        CN[UDP :14540+N]
    end
    
    subgraph Drones["SITL/Physical Drones"]
        DR1[ArduPilot SITL 1]
        DR2[ArduPilot SITL 2]
        DRN[ArduPilot SITL N]
    end
    
    UI --> HTTP
    DS -->|Select Drone| HTTP
    MAP --> HTTP
    
    HTTP -->|/?drone=1| DM
    HTTP -->|/?drone=2| DM
    HTTP -->|/?drone=N| DM
    
    DM --> D1
    DM --> D2
    DM --> DN
    
    D1 -->|Telemetry, Action| MAVSDK
    D2 -->|Telemetry, Action| MAVSDK
    DN -->|Telemetry, Action| MAVSDK
    
    MAVSDK --> C1
    MAVSDK --> C2
    MAVSDK --> CN
    
    C1 --> DR1
    C2 --> DR2
    CN --> DRN
```

---

### 3.3 Core Components Design

#### 3.3.1 Drone Controller Class

```cpp
class DroneController {
private:
    std::shared_ptr<System> system_;
    std::unique_ptr<Telemetry> telemetry_;
    std::unique_ptr<Action> action_;
    std::unique_ptr<MavlinkPassthrough> mavlink_;
    
    std::mutex command_mutex_;  // Serialize commands per drone
    std::string drone_id_;
    uint8_t system_id_;
    
    // Cached telemetry (for non-blocking status)
    std::atomic<bool> connected_{false};
    TelemetryCache telemetry_cache_;
    std::mutex cache_mutex_;

public:
    DroneController(std::shared_ptr<System> system, const std::string& id);
    
    // Non-blocking telemetry
    json get_status();
    
    // Blocking commands (thread-safe)
    bool arm_and_takeoff(float altitude_m);
    bool goto_location(double lat, double lon, float rel_alt_m);
    bool land();
    bool rtl();
    bool set_mode(const std::string& mode);
    
    // Telemetry streaming (background thread)
    void start_telemetry_streaming();
    void stop_telemetry_streaming();
};
```

**Scientific Rationale:**
- **Encapsulation** - Each drone is an independent entity
- **Thread-Safety** - Per-drone locks prevent race conditions
- **Telemetry Caching** - Background thread updates cache, `/status` reads cache (non-blocking)
- **Resource Isolation** - Drone failures don't affect others

---

#### 3.3.2 Drone Manager (Registry Pattern)

```cpp
class DroneManager {
private:
    std::map<std::string, std::unique_ptr<DroneController>> drones_;
    std::mutex registry_mutex_;
    std::shared_ptr<Mavsdk> mavsdk_;

public:
    // Add drone dynamically
    void register_drone(const DroneConfig& config);
    
    // Get drone by ID (thread-safe)
    DroneController* get_drone(const std::string& drone_id);
    
    // Get all drones
    std::vector<std::string> get_drone_ids();
    
    // Broadcast command to all drones
    void broadcast(std::function<void(DroneController*)> command);
    
    // System discovery callback
    void on_system_discovered(std::shared_ptr<System> system);
};
```

**Design Pattern:** **Factory + Registry Pattern**

**Benefits:**
- ✅ **Dynamic drone addition** - No recompilation needed
- ✅ **Centralized management** - Single source of truth
- ✅ **Scalable lookup** - O(log N) drone access
- ✅ **Thread-safe access** - Mutex-protected registry

---

#### 3.3.3 HTTP API Design

**RESTful API with Drone Selection:**

| Endpoint | Method | Description | URL Pattern |
|----------|--------|-------------|-------------|
| `/drones` | GET | List all connected drones | `/drones` |
| `/status` | GET | Get drone telemetry | `/status?drone=1` |
| `/arm_takeoff` | POST | Arm and takeoff | `/arm_takeoff?drone=1` |
| `/goto` | POST | Navigate to waypoint | `/goto?drone=1` |
| `/land` | POST | Land drone | `/land?drone=1` |
| `/rtl` | POST | Return to launch | `/rtl?drone=1` |
| `/broadcast/command` | POST | Send command to all drones | `/broadcast/land` |

**Example Request:**
```http
POST /arm_takeoff?drone=1 HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>

{
  "altitude_m": 10.0
}
```

**Example Response:**
```json
{
  "ok": true,
  "drone_id": "1",
  "timestamp_ms": 1705435200000
}
```

---

### 3.4 Concurrency & Thread Safety Strategy

```mermaid
graph LR
    subgraph Main Thread
        HTTP[HTTP Server Loop]
    end
    
    subgraph Worker Threads
        CMD1[Command Thread 1]
        CMD2[Command Thread 2]
        CMDN[Command Thread N]
    end
    
    subgraph Telemetry Threads
        TEL1[Telemetry Streamer 1]
        TEL2[Telemetry Streamer 2]
        TELN[Telemetry Streamer N]
    end
    
    HTTP -->|Queue Command| CMD1
    HTTP -->|Queue Command| CMD2
    HTTP -->|Queue Command| CMDN
    
    TEL1 -->|Update Cache| CACHE1[(Cache 1)]
    TEL2 -->|Update Cache| CACHE2[(Cache 2)]
    TELN -->|Update Cache| CACHEN[(Cache N)]
    
    HTTP -->|Read| CACHE1
    HTTP -->|Read| CACHE2
    HTTP -->|Read| CACHEN
```

**Thread Safety Mechanisms:**

1. **Per-Drone Command Mutex** - Only one command executes per drone at a time
2. **Lockless Reads** - Telemetry cache uses atomic operations
3. **Connection Pool** - MAVSDK handles internal thread safety
4. **HTTP Thread Pool** - cpp-httplib manages request concurrency

**Scientific Basis:**
- **Producer-Consumer Pattern** - Telemetry threads produce, HTTP consumes
- **Lock-Free Data Structures** - Atomic<T> for connection status
- **Thread-Per-Drone** - Parallel command execution across drones
- **Read-Write Lock Pattern** - Many readers (status), few writers (commands)

---

## 4. Scalability Analysis

### 4.1 Resource Scaling

**Computational Complexity:**

| Metric | Single Drone | N Drones | Complexity |
|--------|-------------|----------|------------|
| Memory (MAVSDK) | ~10 MB | ~10 MB + (N × 2 MB) | O(N) |
| Threads | 3-5 | 3 + (N × 2) | O(N) |
| HTTP Latency | 5-10 ms | 5-10 ms | O(1) |
| Telemetry Rate | 10 Hz | 10 Hz × N | O(N) |
| Command Throughput | 1-5 Hz | (1-5 Hz) × N | O(N) |

**Practical Limits:**

- **2-10 drones:** No bottlenecks, standard hardware
- **10-50 drones:** Requires multi-core CPU, efficient caching
- **50-100 drones:** Consider load balancing or distributed architecture
- **100+ drones:** Cloud-native horizontal scaling (Kubernetes)

### 4.2 Network Architecture for SITL Testing

**Port Allocation Strategy:**

```bash
# Drone 1
sim_vehicle.py --instance 0 --out udp:127.0.0.1:14540
# Backend connects: udp://127.0.0.1:14540

# Drone 2
sim_vehicle.py --instance 1 --out udp:127.0.0.1:14541
# Backend connects: udp://127.0.0.1:14541

# Drone N
sim_vehicle.py --instance N-1 --out udp:127.0.0.1:14540+N
```

**Backend Connection Configuration:**

```cpp
struct DroneConfig {
    std::string id;          // "drone_1", "drone_2", etc.
    std::string connection;  // "udp://127.0.0.1:14540"
    uint8_t expected_sysid;  // 1, 2, 3, ...
};

std::vector<DroneConfig> drones = {
    {"drone_1", "udp://127.0.0.1:14540", 1},
    {"drone_2", "udp://127.0.0.1:14541", 2},
    // ... add more drones
};
```

---

## 5. Implementation Methodology

### 5.1 Phased Development Approach

#### Phase 1: Core Multi-Drone Architecture (Week 1-2)
- [ ] Implement `DroneController` class
- [ ] Implement `DroneManager` registry
- [ ] Add query parameter routing to HTTP server
- [ ] Test with 2 SITL drones

#### Phase 2: Telemetry Optimization (Week 2-3)
- [ ] Implement background telemetry streaming
- [ ] Add telemetry caching per drone
- [ ] Optimize `/status` endpoint for multi-drone polling

#### Phase 3: Frontend Integration (Week 3-4)
- [ ] Add drone selector UI component
- [ ] Update map to show multiple drones
- [ ] Implement per-drone command dispatch
- [ ] Add broadcast commands UI

#### Phase 4: Testing & Validation (Week 4-5)
- [ ] SITL testing with 2, 4, 8 drones
- [ ] Load testing for 10+ drones
- [ ] Failover testing (drone disconnect/reconnect)
- [ ] Documentation and deployment guide

---

### 5.2 Technology Stack Decisions

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Language** | C++ (C++17) | Performance, MAVSDK native support, low latency |
| **HTTP Server** | cpp-httplib | Single-header, proven in current code |
| **JSON Library** | nlohmann/json | Already integrated, header-only |
| **MAVLink Library** | MAVSDK v2.12.x | Industry standard, multi-system support |
| **Concurrency** | std::thread + std::mutex | Standard library, portable |
| **Build System** | CMake | Existing setup, cross-platform |

**Why NOT Python for Multi-Drone?**
- ❌ **GIL limitations** - True parallelism difficult
- ❌ **Event loop complexity** - MAVSDK-Python requires separate instances
- ❌ **Memory overhead** - Higher per-drone cost
- ✅ **C++ Advantages** - 5-10x better throughput, native threading

---

### 5.3 Testing Strategy

#### Unit Testing
```cpp
TEST(DroneController, ArmAndTakeoff) {
    auto drone = create_mock_drone();
    ASSERT_TRUE(drone.arm_and_takeoff(10.0));
}

TEST(DroneManager, MultiDroneRegistry) {
    DroneManager mgr;
    mgr.register_drone({"drone_1", "udp://:14540", 1});
    mgr.register_drone({"drone_2", "udp://:14541", 2});
    ASSERT_EQ(mgr.get_drone_ids().size(), 2);
}
```

#### Integration Testing (SITL)
```bash
# Terminal 1: Start SITL drone 1
sim_vehicle.py --instance 0 --out udp:127.0.0.1:14540

# Terminal 2: Start SITL drone 2
sim_vehicle.py --instance 1 --out udp:127.0.0.1:14541

# Terminal 3: Start backend
./multi_drone_server --config drones.json

# Terminal 4: Test commands
curl -X POST "http://localhost:8080/arm_takeoff?drone=1" -d '{"altitude_m":10}'
curl -X POST "http://localhost:8080/arm_takeoff?drone=2" -d '{"altitude_m":10}'
```

---

## 6. Risk Mitigation & Failover

### 6.1 Failure Scenarios

| Scenario | Detection | Mitigation |
|----------|-----------|------------|
| **Drone disconnects** | `system->is_connected()` returns false | Mark as offline, stop command queue, alert UI |
| **Command timeout** | Async result with timeout | Retry with exponential backoff (3 attempts) |
| **Telemetry stale** | Timestamp check | Fall back to last-known-good, show warning |
| **Port conflict** | Connection failure | Auto-increment port, log error |
| **Resource exhaustion** | Thread/memory monitoring | Reject new connections, graceful degradation |

### 6.2 Graceful Degradation Strategy

```cpp
enum class DroneState {
    DISCOVERING,    // Initial connection
    CONNECTED,      // Healthy communication
    DEGRADED,       // Intermittent connection
    OFFLINE         // Lost connection
};

class DroneController {
    DroneState state_;
    int consecutive_failures_ = 0;
    
    void update_state() {
        if (!system_->is_connected()) {
            consecutive_failures_++;
            if (consecutive_failures_ > 3) {
                state_ = DroneState::OFFLINE;
            } else {
                state_ = DroneState::DEGRADED;
            }
        } else {
            consecutive_failures_ = 0;
            state_ = DroneState::CONNECTED;
        }
    }
};
```

---

## 7. Scientific Validation & References

### 7.1 Design Pattern Validation

This architecture follows **proven software engineering patterns**:

1. **Registry Pattern** (Gang of Four) - DroneManager
2. **Facade Pattern** - DroneController abstracts MAVSDK complexity
3. **Producer-Consumer** - Telemetry streaming
4. **Thread-Safe Singleton** - MAVSDK instance
5. **RESTful API Design** - HTTP endpoints with query parameters

### 7.2 Industry Standards Compliance

- ✅ **MAVLink v2** - Supports 255 systems, message signing
- ✅ **RESTful HTTP** - Stateless, cacheable, uniform interface
- ✅ **JSON API** - Industry-standard data interchange
- ✅ **Bearer Token Auth** - OAuth 2.0 pattern

### 7.3 References

1. **Skydio Cloud Architecture**: AWS-based fleet management with API-first design
   - Source: [Skydio Cloud Documentation](https://www.skydio.com/pages/skydio-cloud), [Fleet Manager Guide](https://support.skydio.com/hc/en-us/articles/360058864793-How-to-use-Skydio-Fleet-Manager)
   
2. **MAVLink Multi-System Support**: Protocol designed for 255 concurrent vehicles
   - Source: [MAVLink Developer Guide - Routing](https://mavlink.io/en/guide/routing.html), [MAVLink FAQ](https://mavlink.io/en/faq/)
   
3. **MAVSDK Multi-Drone Patterns**: Centralized orchestration with per-system plugins
   - Source: [MAVSDK C++ Guide - Multiple Systems](https://mavsdk.mavlink.io/main/en/cpp/guide/connections.html), [Fly Multiple Drones Example](https://mavsdk.mavlink.io/main/en/cpp/examples/fly_multiple_drones.html)
   
4. **Decentralized Swarm Control**: For future enhancement (not current scope)
   - Source: [Decentralized Control of Quadrotor Swarms (arXiv)](https://doi.org/10.48550/arXiv.2109.07735), [Decentralized UAV Swarm Control (MDPI)](https://www.mdpi.com/2504-446X/6/5/209)

---

## 8. Next Steps

> [!CAUTION]
> **Before Implementation:**
>
> 1. **User Review Required** - This methodology needs approval
> 2. **Configuration Decision** - JSON config file vs command-line args?
> 3. **Frontend Scope** - Minimal updates or full dashboard redesign?
> 4. **Testing Environment** - Confirm SITL multi-instance setup works
> 5. **Deployment Target** - Development only or production-ready?

### Recommended Implementation Order

1. ✅ **Phase 1:** Build `DroneController` and `DroneManager` classes
2. ✅ **Phase 2:** Integrate with existing HTTP server (query param routing)
3. ✅ **Phase 3:** Test with 2 SITL drones locally
4. ✅ **Phase 4:** Update frontend for drone selection
5. ✅ **Phase 5:** Scale test to N drones (4, 8, 16)

---

## 9. Comparison: Current vs Proposed Architecture

| Aspect | **Current (Single Drone)** | **Proposed (Multi-Drone)** |
|--------|---------------------------|----------------------------|
| **Systems Supported** | 1 | N (scalable to 100+) |
| **MAVSDK Instances** | 1 | 1 (shared) |
| **Connection Mgmt** | Hardcoded | Dynamic registration |
| **API Routing** | Direct endpoints | Query parameter (`?drone=N`) |
| **Concurrency** | Single mutex | Per-drone mutex |
| **Telemetry** | Blocking calls | Cached, non-blocking |
| **Scalability** | Requires new process | Scales within process |
| **Configuration** | Command-line args | Config file (JSON/YAML) |
| **Frontend Changes** | None needed | Drone selector required |
| **Code Complexity** | 682 lines | ~1500 lines (estimated) |

---

## 10. Questions for User Decision

1. **Configuration Format**: JSON config file or extend command-line args?
   ```json
   {
     "drones": [
       {"id": "drone_1", "connection": "udp://127.0.0.1:14540", "sysid": 1},
       {"id": "drone_2", "connection": "udp://127.0.0.1:14541", "sysid": 2}
     ],
     "http_port": 8080
   }
   ```

2. **Dynamic Addition**: Should drones auto-discover, or require pre-configuration?
   - Auto-discover: MAVSDK finds all systems on network
   - Pre-config: Only connect to specified drones (safer)

3. **Broadcast Commands**: Should we support "all drones land" functionality?

4. **Frontend Priority**: Minimal changes (dropdown), or full multi-drone dashboard?

5. **Backwards Compatibility**: Keep `/endpoint` (drone 1) or require `?drone=1` always?

---

**Ready to proceed with implementation plan after user review and feedback.**
