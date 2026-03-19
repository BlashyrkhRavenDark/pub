# Secure Architecture for Headless Mobile Fleet Management

**Author:** antoine lagier
**Date:** Dec 2025

## 1. Executive Summary and Operational Context
Managing a large-scale fleet of mobile devices for continuous integration, automated testing, or dynamic monitoring requires a hardened, headless control plane. The architectural imperative is isolating the unpredictable nature of mobile hardware interfaces from the stability of the central management API. This paper defines the security patterns, isolation constraints, and threat mitigation strategies required to build a resilient, asynchronous command dispatch system for connected Android devices.

## 2. System Architecture and Component Isolation
The architecture mandates strict decoupling of the control plane from the execution plane. This separation prevents cascading failures originating from hardware faults or malicious inputs.

### 2.1. The Control Plane (Frontend)
A stateless Single Page Application (SPA) serves as the operational dashboard. It communicates exclusively via authenticated REST API endpoints and secure WebSockets. It possesses zero direct hardware access and relies entirely on backend state projections.

### 2.2. The Execution Plane (Backend)
An asynchronous API framework functions as the central orchestrator. It manages state persistence, connection multiplexing, and hardware discovery. The execution plane operates under a strict non-blocking mandate.

### 2.3. The Hardware Abstraction Layer
Physical or network-attached devices are treated as untrusted endpoints. Communication occurs over localized proxy connections, abstracting the underlying protocol (e.g., ADB) into standardized, observable data streams.

## 3. Resilience and State Management Patterns

### 3.1. Process Isolation and the Global Interpreter Lock (GIL)
Mobile debugging protocols inherently utilize blocking socket operations. Integrating these protocols into an asynchronous event loop causes catastrophic latency spikes if mismanaged.
* **Pattern Implementation:** The system deploys distributed queue workers bounded to independent threads. Each connected hardware endpoint is mapped to a dedicated asynchronous worker loop. Blocking calls are explicitly delegated to separate thread pools using `asyncio.to_thread()` equivalents.
* **Security Outcome:** Resource exhaustion on a single device connection (e.g., a hung TCP socket) is contained within its isolated thread, preserving the event loop for unaffected devices and API consumers.

### 3.2. Deterministic State Machines via Circuit Breakers
Hardware connections are transient and unreliable. Repeatedly dispatching commands to an unresponsive endpoint degrades system performance and complicates audit logging.
* **Pattern Implementation:** A centralized `HealthMonitor` orchestrates a Circuit Breaker pattern for every recognized device.
    * **CLOSED State:** Normal operation. Commands are dequeued and executed.
    * **OPEN State:** Triggered after a defined threshold of consecutive command timeouts. The queue is halted. Incoming commands are instantly rejected with a `429 Too Many Requests` or `503 Service Unavailable` response.
    * **HALF-OPEN State:** Activated after a defined cooldown period. A single, low-impact diagnostic command (e.g., a generic device stat request) is permitted. Success transitions the state to CLOSED; failure reverts to OPEN.
* **Security Outcome:** Prevents systemic denial-of-service conditions caused by infinite retry loops and localized network degradation.

## 4. Threat Modeling and Mitigation

### 4.1. Command Injection Vectors
The core function of the system involves dispatching shell commands to remote operating systems. This presents a critical remote code execution (RCE) vulnerability if inputs are not sanitized.
* **Mitigation Strategy:** Implementation of a strict `ActionPayloadValidator`. This component utilizes a whitelist approach for permitted command structures.
* **Sanitization Rules:**
    * Absolute prohibition of shell metacharacters: `;`, `&`, `|`, `>`, `<`, `$`, `` ` ``.
    * Coordinate bounds checking for touch inputs (validating against known screen resolutions).
    * Type enforcement on all variable inputs (e.g., enforcing integer types for delay timers, alphanumeric constraints for package names).

### 4.2. Connection Hijacking and Unauthorized Access
Remote interaction requires streaming raw video and injecting input commands over network protocols.
* **Mitigation Strategy:** Localized network containment. The on-device streaming daemon binds exclusively to localhost. The backend execution plane establishes a secure, dynamically allocated proxy tunnel to the device. Port forwarding is authenticated, managed per-session, and systematically destroyed upon session termination to prevent orphaned, exposed interfaces.

## 5. Identity and Audit Integrity
Device tracking via transient identifiers (IP addresses, USB mount points) corrupts audit logs and access controls.
* **Implementation:** The system extracts the immutable hardware serial number during the initial discovery phase. This serial number serves as the primary cryptographic key for all database associations, session tracking, and audit logging.
* **Result:** A continuous, non-repudiable audit trail of all commands executed against a specific piece of hardware, regardless of how it physically connects to the network.
