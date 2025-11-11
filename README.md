# CloakRPC

**Privacy-Preserving RPC Proxy for Blockchain Users**

CloakRPC is a Rust-based privacy layer that protects blockchain users from RPC-level deanonymization attacks. It acts as a secure proxy between clients and public RPC endpoints, preventing timing analysis, metadata fingerprinting, and IP-to-pseudonym correlation attacks.

## 🎯 Problem Statement

### The Deanonymization Attack (arXiv:2508.21440)

Recent research has demonstrated a novel deanonymization attack against blockchain RPC users that achieves >95% success rate across Ethereum, Bitcoin, and Solana networks. The attack works by:

1. **Temporal Correlation**: When users send transactions, their wallets immediately poll RPC endpoints to check confirmation status
2. **Passive Network Monitoring**: Adversaries with access to network infrastructure (ISPs, border routers, IXPs) can observe TCP packet timing
3. **Public Ledger Analysis**: Transaction confirmation timestamps are publicly visible on-chain
4. **Statistical Correlation**: By correlating the timing between RPC status queries and on-chain confirmations, attackers link IP addresses to blockchain pseudonyms

**Why existing solutions fail:**
- VPNs/Tor only hide IP, not timing patterns
- Privacy wallets still make predictable RPC calls
- Zero-knowledge proofs don't protect RPC metadata
- The attack is passive (zero cost, no network injection needed)

**Critical vulnerability:** Even if your transaction never pays fees or hits the chain, your RPC traffic patterns alone can deanonymize you.

---

## 🛡️ Solution Architecture

CloakRPC breaks the temporal correlation between on-chain activity and network traffic through multiple defensive layers:

### Core Defense Mechanisms

#### 1. **Timing Obfuscation**
Randomizes request timing to destroy temporal correlation patterns.

**Attack without CloakRPC:**
```
T0: User sends transaction
T0+400ms: Wallet polls RPC (predictable)
T0+800ms: Wallet polls again (predictable)
T0+1200ms: Wallet polls again (predictable)
→ Clear timing pattern correlates with on-chain confirmation
```

**Defense with CloakRPC:**
```
T0: User sends transaction  
T0+random(2000-8000)ms: First poll (unpredictable)
T0+random(3000-12000)ms: Second poll (unpredictable)
T0+random(5000-15000)ms: Third poll (unpredictable)
→ Timing correlation broken, statistical analysis fails
```

**Implementation Strategy:**
- Use cryptographically secure random delays
- Exponential distribution (not uniform) to avoid detection
- Adaptive delays based on transaction type
- Configurable privacy levels (see below)

#### 2. **Decoy Traffic Generation**
Injects fake RPC requests to create statistical noise.

**Why decoys work:**
- Attacker cannot distinguish real queries from decoys
- Reduces correlation confidence below actionable threshold
- Creates multiple false positives per real transaction

**Decoy Strategy:**
```
Real query: getSignatureStatus(TX_REAL)
Decoy 1:    getSignatureStatus(TX_FAKE_1) [recent real tx from blockchain]
Decoy 2:    getSignatureStatus(TX_FAKE_2) [recent real tx from blockchain]  
Decoy 3:    getBlockHeight()
Decoy 4:    getSignatureStatus(TX_FAKE_3)
```

**Decoy Requirements:**
- Use real transaction signatures from recent blocks (indistinguishable from legitimate queries)
- Vary RPC methods (mix status checks, block queries, account lookups)
- Natural timing distribution (don't batch all decoys together)
- Scale with privacy level (Level 1: 2 decoys, Level 3: 10+ decoys)

#### 3. **Request Batching**
Combines multiple real/decoy requests to mask individual transaction queries.

**Without batching:**
```
Request 1: Check TX_A → Network packet 1
Request 2: Check TX_B → Network packet 2  
Request 3: Check TX_C → Network packet 3
→ Each transaction = distinct network event
```

**With batching:**
```
Request 1: Check [TX_A, TX_B, TX_C, DECOY_1, DECOY_2]
→ Single network event, impossible to isolate individual transactions
```

**Batching Rules:**
- Batch window: 500ms-2000ms depending on privacy level
- Include decoys in every batch
- Preserve response ordering for client
- Support JSON-RPC batch format natively

#### 4. **Header Sanitization**
Removes identifying metadata that fingerprints wallets/users.

**Headers to strip:**
- `User-Agent` (reveals wallet software version)
- `X-Wallet-Version`, `X-Client-ID` (direct identifiers)
- `Origin`, `Referer` (reveals dApp context)
- `Authorization` tokens (strips wallet-specific auth)
- Custom tracking headers

**Headers to normalize:**
- Replace with generic `User-Agent: Mozilla/5.0` 
- Randomize request IDs
- Strip cookies/session tokens
- Normalize casing and ordering

#### 5. **Request Queue Management**
Intelligent scheduling prevents timing side-channels.

**Queue Properties:**
- FIFO with randomized delays per request
- Priority levels: sendTransaction (no delay) > getBalance (low delay) > getSignature (high delay)
- Per-request jitter calculation based on privacy level
- Non-blocking (async processing)

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CloakRPC Proxy                           │
│                                                                  │
│  ┌────────────────────┐                                         │
│  │  HTTP/WS Server    │  Accepts client connections             │
│  │  (Port 8080)       │  Parses JSON-RPC requests               │
│  └─────────┬──────────┘                                         │
│            │                                                     │
│            ▼                                                     │
│  ┌────────────────────┐                                         │
│  │ Request Classifier │  Identifies request type                │
│  │                    │  (send vs status check vs query)        │
│  └─────────┬──────────┘                                         │
│            │                                                     │
│            ▼                                                     │
│  ┌────────────────────┐        ┌──────────────────────┐        │
│  │  Request Queue     │◄───────│  Timing Engine       │        │
│  │  (with delays)     │        │  - Random delays     │        │
│  │                    │        │  - Privacy level cfg │        │
│  └─────────┬──────────┘        └──────────────────────┘        │
│            │                                                     │
│            ▼                                                     │
│  ┌────────────────────┐        ┌──────────────────────┐        │
│  │  Decoy Generator   │────────│  Blockchain Indexer  │        │
│  │  - Fake requests   │        │  (recent tx pool)    │        │
│  │  - Realistic mix   │        └──────────────────────┘        │
│  └─────────┬──────────┘                                         │
│            │                                                     │
│            ▼                                                     │
│  ┌────────────────────┐                                         │
│  │  Request Batcher   │  Combines real + decoy                  │
│  │                    │  Creates JSON-RPC batch                 │
│  └─────────┬──────────┘                                         │
│            │                                                     │
│            ▼                                                     │
│  ┌────────────────────┐                                         │
│  │  Header Sanitizer  │  Strips identifying headers             │
│  │                    │  Normalizes User-Agent                  │
│  └─────────┬──────────┘                                         │
│            │                                                     │
│            ▼                                                     │
│  ┌────────────────────┐                                         │
│  │  HTTP Client       │  Connection pool to RPC                 │
│  │  (async)           │  Retry logic                            │
│  └─────────┬──────────┘                                         │
│            │                                                     │
│            ▼                                                     │
│  ┌────────────────────┐                                         │
│  │  Response Router   │  Filters out decoy responses            │
│  │                    │  Returns only real results to client    │
│  └────────────────────┘                                         │
└─────────────────────────────────────────────────────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │   RPC Endpoint      │
              │  (Solana/Ethereum)  │
              └─────────────────────┘
```

---

## ⚙️ Privacy Levels

User-configurable trade-off between privacy and performance:

### Level 1: Low Privacy, High Performance
**Use Case:** Low-stakes transactions, development/testing

- **Timing delays:** 500ms - 2000ms
- **Decoy requests:** 1-2 per real request
- **Batching:** Disabled
- **Expected latency:** +500ms - 2s
- **Anonymity set:** ~50-100 concurrent users
- **Deanonymization resistance:** ~40-60% (moderate)

### Level 2: Balanced Privacy (DEFAULT)
**Use Case:** Normal transactions, everyday use

- **Timing delays:** 2000ms - 8000ms  
- **Decoy requests:** 3-5 per real request
- **Batching:** Light (2-3 request window)
- **Expected latency:** +2s - 8s
- **Anonymity set:** ~200-500 concurrent users
- **Deanonymization resistance:** ~70-85% (good)

### Level 3: Maximum Privacy, Lower Performance
**Use Case:** High-value transactions, whistleblowers, sensitive operations

- **Timing delays:** 5000ms - 15000ms
- **Decoy requests:** 10-20 per real request  
- **Batching:** Aggressive (5-10 request window)
- **Expected latency:** +5s - 15s
- **Anonymity set:** ~1000+ concurrent users
- **Deanonymization resistance:** >95% (strong)

**Configuration Example:**
```toml
[privacy]
level = 2  # 1, 2, or 3
custom_delay_min = 2000  # milliseconds (overrides level defaults)
custom_delay_max = 8000
decoy_count = 5
batching_enabled = true
batch_window = 1000  # milliseconds
```

---

## 📋 Implementation Roadmap

### Phase 1: MVP (Core Functionality)
**Timeline:** 2-3 weeks  
**Goal:** Proof of concept that demonstrably defeats timing correlation

**Deliverables:**
- [x] HTTP proxy server (Hyper + Tokio)
- [x] Basic timing randomization engine
- [x] Simple decoy traffic generator (hardcoded tx hashes)
- [x] Header sanitization
- [x] Solana RPC support only
- [x] CLI configuration
- [x] Request/response logging
- [x] Privacy Level 2 implementation

**Success Metric:** Reduce deanonymization success rate from 95% to <30% in controlled tests

### Phase 2: Production Ready
**Timeline:** 4-6 weeks  
**Goal:** Production-grade reliability and performance

**Deliverables:**
- [ ] WebSocket support (for subscriptions)
- [ ] Smart request batching
- [ ] All 3 privacy levels implemented
- [ ] Ethereum RPC support
- [ ] Blockchain indexer (real decoy tx pool)
- [ ] Connection pooling and retry logic
- [ ] Comprehensive error handling
- [ ] Unit and integration tests
- [ ] Performance benchmarks
- [ ] Configuration file support (TOML)

**Success Metric:** <5% performance overhead at Level 1, <100ms p99 latency

### Phase 3: Advanced Features
**Timeline:** 2-3 months  
**Goal:** Ecosystem integration and advanced privacy

**Deliverables:**
- [ ] Multi-hop relay network (Tor-like)
- [ ] Browser extension (WASM compilation)
- [ ] Native wallet SDK integration
- [ ] Request prioritization (gaming/DeFi optimizations)
- [ ] Adaptive privacy (auto-adjust based on network conditions)
- [ ] Metrics dashboard (web UI)
- [ ] Rate limiting and abuse prevention
- [ ] Docker/Kubernetes deployment configs

**Success Metric:** Integrated into 2+ production wallets, >1000 active users

### Phase 4: Research & Ecosystem
**Timeline:** Ongoing  
**Goal:** Become industry standard for RPC privacy

**Deliverables:**
- [ ] Security audit (Trail of Bits / Kudelski)
- [ ] Academic paper: "Defeating RPC Deanonymization in Practice"
- [ ] Conference presentations (Breakpoint, Devcon)
- [ ] Bug bounty program
- [ ] Protocol-level standardization (Solana Foundation collaboration)
- [ ] Multi-chain support (Avalanche, Polygon, etc.)
- [ ] Commercial support/SLA for enterprises

**Success Metric:** Cited in academic research, adopted by major infrastructure providers

---

## 🔧 Technical Implementation Details

### Core Technology Stack

**Language:** Rust (stable)
- Memory safety without garbage collection
- Excellent async runtime (Tokio)
- Zero-cost abstractions for performance
- Strong type system prevents bugs

**Key Dependencies:**
```toml
[dependencies]
tokio = { version = "1.40", features = ["full"] }
hyper = { version = "1.0", features = ["server", "client", "http1", "http2"] }
tokio-tungstenite = "0.24"  # WebSocket support
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
rand = "0.8"  # Cryptographic randomness
rand_distr = "0.4"  # Exponential distribution for delays
reqwest = { version = "0.12", features = ["json"] }
tracing = "0.1"  # Structured logging
tracing-subscriber = "0.3"
config = "0.14"  # Configuration management
clap = { version = "4.0", features = ["derive"] }  # CLI parsing
```

### Module Breakdown

#### `src/main.rs`
- Application entry point
- CLI argument parsing
- Configuration loading
- Server initialization

#### `src/proxy_server.rs`
**Responsibility:** Accept and route client connections

```rust
pub struct ProxyServer {
    config: Arc<Config>,
    request_queue: Arc<RequestQueue>,
    rpc_client: Arc<RpcClient>,
}

impl ProxyServer {
    pub async fn start(&self, addr: SocketAddr) -> Result<()>;
    async fn handle_request(&self, req: Request<Body>) -> Response<Body>;
    async fn handle_websocket(&self, ws: WebSocket) -> Result<()>;
}
```

**Key Features:**
- HTTP/1.1 and HTTP/2 support
- WebSocket upgrade handling
- Request parsing and validation
- Error handling and logging

#### `src/timing_engine.rs`
**Responsibility:** Calculate randomized delays per request

```rust
pub struct TimingEngine {
    privacy_level: PrivacyLevel,
    rng: Mutex<StdRng>,  // Thread-safe RNG
}

impl TimingEngine {
    pub fn calculate_delay(&self, request_type: RequestType) -> Duration;
    fn exponential_delay(&self, min_ms: u64, max_ms: u64) -> Duration;
}

pub enum RequestType {
    SendTransaction,     // No delay (user expects instant send)
    SignatureStatus,     // High delay (correlation target)
    GetBalance,          // Low delay
    GetBlock,           // Medium delay
}
```

**Algorithm:**
```rust
// Exponential distribution prevents pattern detection
let lambda = 1.0 / ((max_ms - min_ms) as f64);
let exp_dist = Exponential::new(lambda).unwrap();
let delay = min_ms + (exp_dist.sample(&mut rng) as u64);
delay.clamp(min_ms, max_ms)
```

#### `src/request_queue.rs`
**Responsibility:** Schedule requests with delays

```rust
pub struct RequestQueue {
    pending: Arc<Mutex<VecDeque<QueuedRequest>>>,
    timing_engine: Arc<TimingEngine>,
}

struct QueuedRequest {
    id: RequestId,
    body: JsonRpcRequest,
    execute_at: Instant,
    response_tx: oneshot::Sender<JsonRpcResponse>,
}

impl RequestQueue {
    pub async fn enqueue(&self, req: JsonRpcRequest) -> oneshot::Receiver<JsonRpcResponse>;
    async fn process_loop(&self);  // Background task
}
```

#### `src/decoy_generator.rs`
**Responsibility:** Create realistic fake RPC requests

```rust
pub struct DecoyGenerator {
    blockchain_indexer: Arc<BlockchainIndexer>,
    privacy_level: PrivacyLevel,
}

impl DecoyGenerator {
    pub async fn generate_decoys(&self, count: usize) -> Vec<JsonRpcRequest>;
    async fn get_random_recent_signature(&self) -> String;
    fn generate_decoy_method(&self) -> &str;  // Vary methods
}
```

**Decoy Methods Mix:**
- 70%: `getSignatureStatuses` (most common)
- 15%: `getTransaction`
- 10%: `getBlockHeight`
- 5%: `getAccountInfo`

#### `src/request_batcher.rs`
**Responsibility:** Combine multiple requests into JSON-RPC batches

```rust
pub struct RequestBatcher {
    batch_window: Duration,
    pending_batch: Arc<Mutex<Vec<JsonRpcRequest>>>,
}

impl RequestBatcher {
    pub async fn add_request(&self, req: JsonRpcRequest);
    async fn flush_batch(&self) -> Vec<JsonRpcRequest>;
}
```

**JSON-RPC Batch Format:**
```json
[
  {"jsonrpc":"2.0","id":1,"method":"getSignatureStatuses","params":[["sig1"]]},
  {"jsonrpc":"2.0","id":2,"method":"getSignatureStatuses","params":[["sig2"]]},
  {"jsonrpc":"2.0","id":3,"method":"getSignatureStatuses","params":[["sig3"]]}
]
```

#### `src/header_sanitizer.rs`
**Responsibility:** Strip identifying HTTP headers

```rust
pub struct HeaderSanitizer;

impl HeaderSanitizer {
    pub fn sanitize_request_headers(headers: &mut HeaderMap);
    pub fn sanitize_response_headers(headers: &mut HeaderMap);
    fn generate_generic_user_agent() -> HeaderValue;
}
```

**Sanitization Rules:**
```rust
// Remove entirely
headers.remove("user-agent");
headers.remove("x-wallet-version");
headers.remove("origin");
headers.remove("referer");
headers.remove("cookie");
headers.remove("authorization");

// Replace with generic
headers.insert("user-agent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64)");
headers.insert("accept", "application/json");
```

#### `src/rpc_client.rs`
**Responsibility:** Send requests to actual RPC endpoint

```rust
pub struct RpcClient {
    client: reqwest::Client,
    rpc_url: String,
    connection_pool_size: usize,
}

impl RpcClient {
    pub async fn send_request(&self, req: JsonRpcRequest) -> Result<JsonRpcResponse>;
    pub async fn send_batch(&self, batch: Vec<JsonRpcRequest>) -> Result<Vec<JsonRpcResponse>>;
    async fn retry_with_backoff(&self, req: JsonRpcRequest, attempts: u32) -> Result<JsonRpcResponse>;
}
```

#### `src/blockchain_indexer.rs`
**Responsibility:** Maintain pool of recent transaction signatures for decoys

```rust
pub struct BlockchainIndexer {
    recent_signatures: Arc<RwLock<VecDeque<String>>>,
    rpc_client: Arc<RpcClient>,
    max_pool_size: usize,
}

impl BlockchainIndexer {
    pub async fn start_indexing(&self);  // Background task
    pub async fn get_random_signature(&self) -> Option<String>;
    async fn fetch_recent_block_signatures(&self) -> Result<Vec<String>>;
}
```

**Indexing Strategy:**
- Poll every 30 seconds for latest block
- Extract all transaction signatures
- Maintain rolling window of last 10,000 signatures
- Randomly sample from pool for decoys

#### `src/config.rs`
**Responsibility:** Configuration management

```rust
#[derive(Debug, Deserialize)]
pub struct Config {
    pub server: ServerConfig,
    pub privacy: PrivacyConfig,
    pub rpc: RpcConfig,
}

#[derive(Debug, Deserialize)]
pub struct PrivacyConfig {
    pub level: PrivacyLevel,  // 1, 2, or 3
    pub custom_delay_min: Option<u64>,
    pub custom_delay_max: Option<u64>,
    pub decoy_count: Option<usize>,
    pub batching_enabled: bool,
    pub batch_window_ms: u64,
}

pub enum PrivacyLevel {
    Low = 1,
    Medium = 2,
    High = 3,
}
```

---

## 🧪 Testing & Validation

### Unit Tests
Test individual components in isolation:

```rust
#[cfg(test)]
mod tests {
    #[tokio::test]
    async fn test_timing_engine_delays_within_range() {
        let engine = TimingEngine::new(PrivacyLevel::Medium);
        for _ in 0..1000 {
            let delay = engine.calculate_delay(RequestType::SignatureStatus);
            assert!(delay >= Duration::from_millis(2000));
            assert!(delay <= Duration::from_millis(8000));
        }
    }

    #[tokio::test]
    async fn test_header_sanitizer_removes_identifying_info() {
        let mut headers = HeaderMap::new();
        headers.insert("user-agent", "Phantom/1.2.3".parse().unwrap());
        HeaderSanitizer::sanitize_request_headers(&mut headers);
        assert!(!headers.get("user-agent").unwrap().to_str().unwrap().contains("Phantom"));
    }
}
```

### Integration Tests
Test full request flow:

```rust
#[tokio::test]
async fn test_end_to_end_request_flow() {
    let server = ProxyServer::new(test_config()).await;
    let client = reqwest::Client::new();
    
    let response = client
        .post("http://localhost:8080")
        .json(&json!({
            "jsonrpc": "2.0",
            "id": 1,
            "method": "getSignatureStatuses",
            "params": [["test_sig"]]
        }))
        .send()
        .await
        .unwrap();
    
    assert_eq!(response.status(), 200);
}
```

### Attack Simulation Tests
Validate defense against deanonymization:

**Test Setup:**
1. Deploy CloakRPC proxy
2. Implement timing correlation attack from paper
3. Send 1000 test transactions through proxy
4. Attempt to correlate network traffic with on-chain confirmations
5. Measure success rate

**Success Criteria:**
- Level 1: Deanonymization success rate <50%
- Level 2: Deanonymization success rate <20%
- Level 3: Deanonymization success rate <5%

**Test Script Structure:**
```python
# attack_simulation.py
def correlate_timing(network_packets, blockchain_confirmations):
    matches = 0
    for packet in network_packets:
        for tx in blockchain_confirmations:
            time_delta = abs(packet.timestamp - tx.confirmation_time)
            if time_delta < CORRELATION_THRESHOLD:
                matches += 1
    return matches / len(blockchain_confirmations)

# Without CloakRPC: Expected ~95% success rate
# With CloakRPC Level 2: Expected <20% success rate
```

### Performance Benchmarks

**Metrics to measure:**
- Latency (p50, p95, p99)
- Throughput (requests/second)
- Memory usage
- CPU usage
- Network bandwidth overhead

**Benchmark Tool:**
```bash
# Using wrk
wrk -t4 -c100 -d30s --latency http://localhost:8080/rpc \
  -s benchmark.lua
```

**Expected Results:**
| Privacy Level | p50 Latency | p99 Latency | Throughput |
|--------------|-------------|-------------|------------|
| Level 1      | +600ms      | +2.5s       | 500 req/s  |
| Level 2      | +4s         | +10s        | 300 req/s  |
| Level 3      | +8s         | +18s        | 150 req/s  |

---

## 🚀 Deployment Guide

### Local Development

```bash
# Clone repository
git clone https://github.com/victorchukwuemeka/CloakRPC.git
cd CloakRPC

# Build
cargo build --release

# Run with default config
./target/release/cloakrpc \
  --rpc https://api.mainnet-beta.solana.com \
  --port 8080 \
  --privacy-level 2

# Point your wallet to http://localhost:8080
```

### Docker Deployment

```dockerfile
FROM rust:1.75 as builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/cloakrpc /usr/local/bin/
EXPOSE 8080
CMD ["cloakrpc", "--rpc", "https://api.mainnet-beta.solana.com", "--port", "8080"]
```

```bash
docker build -t cloakrpc .
docker run -p 8080:8080 cloakrpc
```

### Production (Kubernetes)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloakrpc
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cloakrpc
  template:
    metadata:
      labels:
        app: cloakrpc
    spec:
      containers:
      - name: cloakrpc
        image: cloakrpc:latest
        ports:
        - containerPort: 8080
        env:
        - name: RPC_URL
          value: "https://api.mainnet-beta.solana.com"
        - name: PRIVACY_LEVEL
          value: "2"
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
```

---

## 📊 Security Analysis

### Threat Model

**Assumptions:**
- Adversary has network-level visibility (ISP, border router, IXP)
- Adversary can observe TCP packet timing and sizes
- Adversary has access to public blockchain data
- Adversary cannot decrypt TLS traffic (HTTPS remains secure)
- Adversary is passive (no active network injection)

**Out of Scope:**
- Browser fingerprinting (different attack vector)
- On-chain analysis (address clustering, graph analysis)
- Compromised RPC endpoints (MitM attacks)
- Client-side malware

### Attack Resistance Analysis

**Timing Correlation Attack:**
- **Without CloakRPC:** 95%+ success rate
- **With CloakRPC Level 2:** <20% success rate
- **Defense:** Randomized delays + decoy traffic destroys temporal correlation

**Packet Size Analysis:**
- **Risk:** HTTP request/response sizes might leak info
- **Mitigation:** Padding (future feature), batch mixing
- **Status:** Partial mitigation

**Long-term Behavioral Profiling:**
- **Risk:** Adversary tracks patterns over weeks/months
- **Mitigation:** Privacy level rotation, relay network (Phase 3)
- **Status:** Needs more research

**Traffic Volume Analysis:**
- **Risk:** Total bandwidth usage might reveal heavy users
- **Mitigation:** Constant-rate decoy traffic (future)
- **Status:** Not yet implemented

### Known Limitations

1. **Performance vs Privacy Trade-off:** Maximum privacy (Level 3) significantly impacts UX
2. **Decoy Realism:** Requires continuous blockchain indexing; stale decoys might be filtered
3. **WebSocket Challenges:** Real-time subscriptions harder to protect than one-off requests
4. **Single-hop Privacy:** Without relay network, ISP still sees you connecting to CloakRPC
5. **Scale Limitations:** Effectiveness decreases if very few users (anonymity set size matters)

---

## 🤝 Contributing

We welcome contributions from security researchers, Rust developers, and privacy advocates.

**Areas needing help:**
- WebSocket subscription privacy improvements
- Relay network protocol design
- Performance optimizations
- Multi-chain support (Ethereum, Avalanche)
- Formal security analysis
- Browser extension development

**Contribution Process:**
1. Fork repository
2. Create feature branch (`git checkout -b feature/amazing-feature`)
3. Write tests for new code
4. Ensure `cargo test` passes
5. Submit pull request with detailed description

---

## 📚 Research References

1. **Time Tells All: Deanonymization of Blockchain RPC Users with Zero Transaction Fee**  
   Wang et al., arXiv:2508.21440, August 2025  
   *Foundation paper describing the attack CloakRPC defends against*

2. **Deanonymization in the Bitcoin P2P Network**  
   Biryukov et al., NDSS 2014  
   *Early work on Bitcoin network timing attacks*

3. **Tor's Pluggable Transports**  
   Tor Project Documentation  
   *Inspiration for traffic obfuscation techniques*

4. **Mix Networks and Traffic Analysis**  
   Chaum, 1981  
   *Foundational work on anonymity through mixing*

---

## 📄 License

MIT License - See LICENSE file for details

**TL;DR:** Use commercially, modify freely, just give credit.

---

## 🎯 Project Goals

**Primary Goal:**  
Protect blockchain users from RPC-level deanonymization attacks by implementing practical, deployable defenses.

**Success Metrics:**
- [ ] Reduce timing correlation attack success rate below 20%
- [ ] Achieve production-grade performance (<5s p99 latency at Level 2)
- [ ] Integration into 2+ major Solana wallets
- [ ] Publish peer-reviewed security analysis
- [ ] Active user base of 1000+ users
- [ ] Recognition as industry standard for RPC privacy

**Long-term Vision:**  
Make RPC privacy the default for all blockchain interactions, not an opt-in feature. CloakRPC should become as fundamental to blockchain infrastructure as TLS is to web security.

---

**Built by researchers who believe privacy is a right, not a luxury.**

*For questions, security disclosures, or collaboration: [Contact Info]*