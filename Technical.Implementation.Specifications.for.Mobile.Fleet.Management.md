# Technical Implementation Specifications for Mobile Fleet Management

**Author:** antoine lagier
**Date:** Feb 2026

## 1. Execution Environment and State Persistence
The implementation relies on an asynchronous Python environment interfacing with a lightweight, embedded SQL database.

### 1.1. Concurrency Handling in SQLite
Managing high-frequency, concurrent state updates from dozens of independent device worker threads requires specific database configurations to prevent locking collisions.
* **Configuration:** The database must be initialized in Write-Ahead Logging (WAL) mode.
* **Isolation Level:** Transactions utilize `DEFERRED` isolation.
* **Connection Pooling:** A strict connection pool is enforced, paired with elevated `busy_timeout` pragmas to allow queries to queue seamlessly during microscopic lock contentions rather than failing immediately.

## 2. Network Discovery and Lifecycle Management
The system requires an autonomous mechanism to detect, register, and monitor hardware endpoints without human intervention.

### 2.1. Autonomous Discovery Protocol
A detached background process executes continuous polling cycles.
* **Subnet Scanning:** Non-intrusive TCP port scans across predefined local subnets to identify network-attached debugging interfaces.
* **Physical Interrogation:** Polling USB buses for recognized device vendor IDs.
* **State Synchronization:** Discovered devices are matched against the database using their hardware serials. Unreachable devices are flagged as offline, and their associated queue workers are systematically terminated to reclaim memory.

### 2.2. The Worker Lifecycle
1.  **Initialization:** Upon discovery, a device transitions to `ONLINE`. The central orchestrator spawns a dedicated async task.
2.  **Processing:** The worker polls the localized database queue for commands assigned to its specific hardware serial.
3.  **Execution:** Commands are dispatched via the thread-pooled hardware abstraction layer.
4.  **Termination:** If the discovery scanner marks the device `OFFLINE`, a termination signal is sent to the worker, ensuring proper socket closure and thread cleanup.

## 3. Low-Latency Video Streaming and WebSocket Proxying
Remote human interaction mandates sub-100ms video streaming latency.

### 3.1. Stream Ingestion and Proxy Mechanics
* The hardware endpoint runs a daemon capturing the framebuffer and encoding it into raw H.264 Network Abstraction Layer (NAL) units.
* The backend initiates a local socket connection to the device's proxy port.
* A FastAPI WebSocket route reads the raw NAL units from the local socket and streams them directly to the authenticated frontend client.

### 3.2. Frontend Decoding Constraints
* The frontend utilizes a JavaScript-based H.264 decoder (e.g., JMuxer) operating within an HTML5 Canvas element.
* **Integrity Enforcement:** The dimensions configured in the video encoder on the hardware must cryptographically match the dimensions defined in the touch-event headers sent by the frontend. Mismatched dimensions result in coordinate translation failures and the immediate rejection of all input events by the hardware daemon.

## 4. Development Constraints and Coding Standards
To maintain architectural integrity during continuous development, the following immutable rules apply to all code modifications:

1.  **Zero-Blocking Rule:** No direct hardware interaction library (e.g., `ppadb`) may be invoked directly within an `async def` route. Violations degrade API performance globally.
2.  **Resilience via Tenacity:** Network instability is assumed. All outbound hardware commands must be wrapped in exponential backoff retry logic, specifically respecting the external state of the Circuit Breaker.
3.  **Explicit Cleanup:** Streaming daemons running on hardware endpoints must be explicitly killed (`process.stop()`) and their associated threads joined before a new session is permitted. Failure to do so results in hardware-level memory leaks and required physical reboots.
