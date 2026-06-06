#### This repository contains notes for the below topics and sub-topics

## Node.js Internals

### V8 Engine Under the Hood

* V8 Heap vs. Stack Memory Architecture
* Generational Garbage Collection: Scavenger (New Space) vs. Mark-Sweep-Compact (Old Space)
* Hidden Classes (`Shapes`) and Inline Caches (IC) Optimization
* Compiling and Executing JavaScript: Ignition Interpreter & TurboFan JIT Compiler
* Diagnosing Memory Leaks via V8 Heap Snapshots and Core Dumps

### Libuv & Native Bindings

* Libuv Thread Pool Architecture and C++ Asynchronous Worker Threads
* The Node.js Layering: JavaScript V8 $\rightarrow$ Node API (C++) $\rightarrow$ Libuv (C)
* System Call Multiplexing: `epoll` (Linux), `kqueue` (macOS), and `IOCP` (Windows)
* CPU-Intensive Offloading: `worker_threads` vs. `child_process` Optimization

---

## The Event Loop

### Advanced Phase Mechanics

* Execution Priority Mechanics: Timers, Pending Callbacks, Idle/Prepare, Poll, Check, Close Callbacks
* Microtask Prioritization: `process.nextTick` vs. `Promise.then` (Miclrotask Checkpoints)
* Event Loop Starvation and Macro-Queue Delay Tracking
* Execution Anomalies: `setTimeout(fn, 0)` vs. `setImmediate(fn)` in I/O Boundaries

### Production Diagnostics & Tuning

* Thread Pool Saturation: Overriding `$UV_THREADPOOL_SIZE` for Heavy Crypto/FS Tasks
* Tracking Event Loop Blockage Metrics using `perf_hooks` and Block Alerts
* Interpreting Flame Graphs and CPU Profiles under Peak Production Traffic

---
## Asynchronous Programming in Node.js

### Advanced Event Loop Dynamics & Architecture

* Phase Orchestration & Execution Priority ($Ticking$ Mechanics)
* Microtask Queue vs. Macrotask Queue (NextTick vs. Promises)
* Thread Pool ($UV\_THREADPOOL\_SIZE$) Bottlenecks & Optimization
* Event Loop Blocking Diagnostics via Flame Graphs & Core Dumps

### Context & State Management

* Managing Thread-Safe Contexts with `AsyncLocalStorage`
* Correlating Logs Across Async Boundaries without Propagating Parameters
* Tracking Execution Scopes in Multi-Tenant Architectures

### High-Performance Data Streaming & Backpressure

* Readable, Writable, Duplex, and Transform Stream Architectures
* Memory Leak Mitigations in Network Sockets & File Descriptors
* Managing Native Backpressure in Pipeline Topologies (`stream/promises`)
* Memory-Bounded Processing of Large JSON via SAX/Tokenized Ingestion

### Advanced Concurrency Patterns

* Custom Thread Management via Worker Threads (`worker_threads`)
* Shared Memory Spaces (`SharedArrayBuffer` & `Atomics`)
* Resilient Execution Policies: Circuit Breakers, Bulkheads, and Racing Strategies

---

## Express.js Production-Grade Architecture

### Architectural Foundations & Decoupled Design

* Clean Architecture Boundaries: Inward Dependency Rule Realization
* Decoupling Inversion of Control (IoC) Frameworks from HTTP Infrastructure
* Structural Data Flow Layouts: Controllers vs. Domain Services vs. Data Access Layer
* Lifecycle Event Hooks and Graceful Termination Pipeline Handling

### Route Orchestration & Scale Management

* Dynamic Polymorphic Routing Models via Runtime Scoping Middleware
* Decoupled Modular Sub-Router Pipelines for Large Domain Models
* Mitigating State Leaks Caused by Middleware Chain Context Binding
* Express v5 Native Async Error Scoping vs. Monkey-Patching in Express v4

### Production Resource Hardening

* Single-Threaded Memory Bounds and V8 Garbage Collection Tuning under Express Load
* Event-Loop Friendly JSON Serialization and High-Throughput Body Parsing Alternate Drivers
* Strategic Reverse Proxy Alignment via `trust proxy` Headers
* Session Hardening and Secure Shared Application Context State Management

---

## Enterprise REST API Design

### Maturity & Representation Architecture

* Richardson Maturity Model Level 3 Implementation: Dynamic State Graph Topologies
* Contextual Media-Type Content Negotiation Strategies
* Dynamic API Version Management: URL Pathing vs. Accept Media Header Poly-Routing
* Self-Descriptive Resource Lifecycles (HATEOAS Link Schema Architectures)

### Data Validation & Integrity Contracts

* Compile-Time Type Inference vs. High-Throughput Runtime Validation Engines
* Mitigating Prototype Injections and Prototype Pollution Vectors via Input Schemas
* Canonical Resource Representation Modeling: Data Transfer Objects (DTOs) vs. Presentation Layer Serializers
* Declarative Sanitization of Output Interfaces

### Distributed State Synchronization & Cache Hardening

* Distributed Key Mutation Protection: Two-Phase Commit Locks vs. Database Unique Indexes
* Idempotency State Engines with Automatic Ghost-Lock Clearing Strategies
* Multi-Tier Edge Caching Mechanics: Proxies, CDNs, and `Vary` Response Header Optimization
* Conditional Execution Mechanisms (`ETag`, `If-None-Match`, `If-Modified-Since`)

### Enterprise Fault Engineering

* Operational Domain Failures vs. Catastrophic Structural Outages Separation
* Machine-Readable Standardization Protocols (RFC 7807 Problem Details Specifications)
* Distributed Correlation ID Traversal Across Distributed Infrastructure Networks
* Preventing Information Disclosure via Clean Straces and Internal Metric Masking

---

## Advanced Middleware Mechanics

### Pipeline Execution Order & Interception

* Intercepting Output Buffers to Dynamically Commit Payload Modifications Under-the-Hood
* Advanced Interceptor Implementations: Overriding Native Res State Transmissions Safely
* Functional Middleware Composition: Curried Setup Factories and Dynamic Argument Injections
* Short-Circuit Evaluation and Fail-Fast Pipeline Gateways

### Cross-Cutting Security Engineering

* Request Rate-Limiting Mechanics: Token Bucket vs. Sliding Window Log Distributed Implementations
* Secure Header Injections (CORS Controls, CSP Specifications, HSTS Enforcement)
* Identity Scopes and Granular Policy Checks Execution
* Defense Isolation against Injection Exploits, Parameter Tampering, and Resource Exhaustion Attacks

### Performance, Analytics, & Observability

* Distributed Telemetry Interception Hooks: OpenTelemetry Extraction & Context Propagation
* Non-Blocking Streaming Analytics Ingestion: Safe `stdout` Piping Pipelines
* Memory Tracking Interceptors for Profiling Endpoint Ingestion Overhead
* Request Context Deallocation and Resource Teardown Hooks upon Connection Closure


---
<!-- 
## Advanced MongoDB Infrastructure

### Storage Engines & Sharding Topologies

* WiredTiger Storage Engine Architecture: Concurrency, Journaling, and Checkpoints
* WiredTiger Cache Tuning and Out-of-Memory (OOM) Container Crash Mitigations
* Distributed Partitioning: Sharding Configurations, Choice of Shard Keys, and Jumbo Chunk Mitigation
* Replica Set Internals: Oplog Sizing, Raft-like Consensus Elections, and Heartbeats

### Query Optimization & Locking Architecture

* Execution Plan Invalidation: Index Selection, Collation, and Collection Scans (`COLLSCAN`)
* Partial, Sparse, Compound, and Covered Indexes Execution Boundaries
* Lock Scoping Gradients: Global vs. Database vs. Collection vs. Document-level Intent Locking
* The Aggregation Pipeline Engine: Memory Allocation Thresholds ($100\text{MB}$ Limit) and Disk Spilling

---

## Mongoose ODM

### Schema Compilation & Document Hydration

* Document Hydration Overheads: Cast Hooks, Internal Getters/Setters, and Dependency Graphs
* Performance Invalidation: Raw Query Extraction via `.lean()` bypassing Document Archetypes
* Virtuals Architecture: Dynamic Extraction, Population Boundaries, and Transformation Overrides
* Mongoose 9.x Schema Composition: Strict Query Injection Guards vs. Discriminators

### Hooks & Advanced Transaction Controls

* Pre/Post Middleware Hook Interception: Document vs. Query Scoping Context Bindings
* Multi-Document ACID Transactions: Session Tracking, Two-Phase Commits, and Retry Policies
* Safeguarding Operations: Bypassing Query Sanitization Attacks (e.g., `$nor` / `$ne` NoSQL Injection Vulnerabilities)
* Managing Schema Migrations and Field Deprecations In-App

---

## JWT Authentication

### Cryptographic Signing & Key Topologies

* Symmetric Invalidation (HMAC SHA256) vs. Asymmetric Topology (RS256/EdDSA) Security
* Key Rotation Management: Structuring Resilient JSON Web Key Sets (JWKS) Gateways
* Cryptographic Replay Protection Mechanisms: Token Fingerprinting and `jti` Claims
* Payload Obfuscation Specifications (JWS vs. JWE Architecture)

### Token Lifecycles & Revocation Engines

* Stateless vs. Stateful Token Paradigms: Distributed Blacklisting via High-Performance Redis Layers
* Slidewindow Token Refresh Policies: Rotating Refresh Tokens and Replay Detection
* Storage Strategy Trade-offs: `HttpOnly` / `SameSite=Strict` Cookies vs. Memory Injection Storage
* Mitigating Token Leakage via Single Log-Out (SLO) Infrastructure Architectures

---

## Enterprise Authorization (AuthZ)

### Access Control Topologies

* Role-Based Access Control (RBAC) Hierarchies vs. Attribute-Based Access Control (ABAC) Contextual Policies
* Relationship-Based Access Control (ReBAC) Foundations (Google Zanzibar Google-scale topologies)
* Decoupled Authorization Core Architecture: Policy Decision Points (PDP) vs. Policy Enforcement Points (PEP)
* Cloud-Native External Policy Engines: Open Policy Agent (OPA) Integration

### Performance & Multi-Tenancy Hardening

* Caching Complex Authorization Graphs Safely without Introducing Stale Privileges
* Context Isolation Patterns: Dynamic Database Connection Pooling vs. Logical Tenant Filtering
* Enforcing Global Invariant Scopes across Shared Infrastructure Topologies

---

## Advanced API Security

### Threat Detection & Perimeter Defense

* Distributed Denial of Service (DDoS) Isolation: Rate Limiting Algorithms (Token Bucket, Leaky Bucket, Sliding Window Log)
* Content Security Hardening: Helmet Middleware Strategy, Content Security Policies (CSP), HSTS Core Enforcement
* Mass Assignment Prevention: Request Parameter Whitelisting and Strict Data Transfer Objects (DTOs)
* Cross-Origin Resource Sharing (CORS) Isolation and Preflight Optimization

### Input Sanitization & Attack Mitigations

* Automated NoSQL/SQL Sanitization Filters and Query Parameter Strict Cast Controls
* Protecting against Advanced XML/JSON Injection and Billion Laughs Processing Overloads
* Timing Attack Defense: Enforcing Constant-Time Cryptographic String Comparison Implementations

---

## Runtime Validation

### Schema Compilers vs. Evaluators

* High-Throughput Request Validation: Compilation (Ajv) vs. Object Parsers (Zod, Joi) Execution Metrics
* Strict Schema Bounds Enforcements: Strip Patterns (`.strict()`) vs. Implicit Variable Mappings
* Structural Asynchronous Dynamic Validations: Database Presence Evaluation and Identity Checks
* Type Coercion Safety: Sanitizing Matrix Arrays, Boolean Mapping Quirks, and Transformation Boundaries

---

## Observability & Distributed Logging

### High-Performance Ingestion Architecture

* Synchronous Logging Event Closures vs. Asymmetric Non-Blocking Structured Output Pipelines (Pino, Winston)
* Asynchronous Standard Streams Strategy: Decoupling Log Formats to `stdout` for Container Ingestion (Vector, FluentBit)
* Logging Levels, Context Overhead Management, and Dynamic Log-Level Adjustments in Live Production

### Distributed Tracing & Compliance

* OpenTelemetry Context Propagation Standards Across Asynchronous Microservice Networks
* Correlating Ingestion Paths via Unique Structural Context Tokens (X-Correlation-ID tracing)
* PII Data Scrubbing Invariants: Automated Context Masking and Cryptographic Hashing at Ingestion Boundaries
* Audit Trail Engineering: Generating Tamper-Evident System Logs -->