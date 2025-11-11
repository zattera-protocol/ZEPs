---
zep: 3
title: NFT Oracle Plugin
description: Off-chain oracle service for verifying ERC-721 NFT ownership on EVM-compatible chains
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Interface
created: 2025-11-08
---

## Abstract

This ZEP specifies an oracle plugin that verifies ERC-721 NFT ownership on Ethereum-compatible chains. The oracle operates as an off-chain service integrated into witness nodes, querying EVM chains via RPC to confirm NFT ownership claims submitted on-chain. Verification results are written back to the blockchain, enabling trustless cross-chain NFT verification for witness authorization (ZEP-2) and other use cases.

## Motivation

ZEP-2 (NFT-Based Witness Verification) requires a mechanism to verify that witnesses actually own the NFTs they claim. Since Zattera cannot directly access Ethereum state, an oracle bridge is necessary.

### Why an Oracle?

**Alternative approaches considered:**

1. **Cross-chain bridge** - Too complex, expensive, and slow
2. **Snapshot-based verification** - Not real-time, centralized
3. **On-chain light client** - Computationally prohibitive

**Oracle advantages:**

- Simple implementation using standard RPC calls
- Fast verification (seconds vs. hours for bridges)
- Minimal gas costs (read-only queries)
- Flexible multi-chain support
- Witness-controlled (decentralized governance)

### Trust Model

The oracle is **not trustless** but achieves practical security through:

1. **Witness operation** - Only active witnesses run oracles
2. **Open source** - Fully auditable code
3. **Multi-oracle consensus** - Multiple oracles verify same claim (future)
4. **Economic security** - False verification harms witness reputation
5. **Transparent logs** - All verifications recorded on-chain

## Specification

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                     Witness Node                             │
│  ┌──────────────┐         ┌───────────────────────────────┐  │
│  │   steemd     │◄───────►│   NFT Oracle Plugin           │  │
│  │              │         │                               │  │
│  │  - Receives  │         │  - Monitors proof operations  │  │
│  │    proof ops │         │  - Queries EVM chains         │  │
│  │  - Updates   │         │  - Updates verification       │  │
│  │    status    │         │    status                     │  │
│  └──────────────┘         └───────────────────────────────┘  │
│         ▲                            │                       │
│         │                            ▼                       │
│         │                 ┌─────────────────────┐            │
│         └─────────────────│  Thread Pool        │            │
│                           │  (async workers)    │            │
│                           └─────────────────────┘            │
└──────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
                         ┌─────────────────────────────┐
                         │   EVM RPC Endpoints         │
                         │                             │
                         │  - Ethereum (eth.llamarpc)  │
                         │  - Polygon (polygon-rpc)    │
                         │  - BSC (bsc-dataseed)       │
                         │  - Arbitrum, Optimism, etc. │
                         └─────────────────────────────┘
```

### Plugin Interface

The oracle implements the standard Appbase plugin interface.

**Plugin Declaration:**

```cpp
namespace steem { namespace plugins { namespace nft_oracle {

class nft_oracle_plugin : public appbase::plugin<nft_oracle_plugin>
{
   public:
      APPBASE_PLUGIN_REQUIRES((chain::chain_plugin))

      nft_oracle_plugin();
      virtual ~nft_oracle_plugin();

      static const std::string& name() {
         static std::string name = "nft_oracle";
         return name;
      }

      virtual void set_program_options(
         boost::program_options::options_description& cli,
         boost::program_options::options_description& cfg) override;

      virtual void plugin_initialize(
         const boost::program_options::variables_map& options) override;

      virtual void plugin_startup() override;
      virtual void plugin_shutdown() override;

   private:
      std::unique_ptr<detail::nft_oracle_plugin_impl> my;
};

} } } // steem::plugins::nft_oracle
```

### Configuration Parameters

**File:** `config.ini`

```ini
# Enable NFT oracle plugin
plugin = nft_oracle

# EVM RPC endpoints (format: chain_id:url)
nft-oracle-rpc-endpoint = 1:https://eth.llamarpc.com
nft-oracle-rpc-endpoint = 137:https://polygon-rpc.com
nft-oracle-rpc-endpoint = 56:https://bsc-dataseed.binance.org
nft-oracle-rpc-endpoint = 42161:https://arb1.arbitrum.io/rpc

# Number of worker threads for verification
nft-oracle-threads = 4

# Confirmation blocks required before trusting result
nft-oracle-confirmations = 12

# Timeout for RPC requests (seconds)
nft-oracle-timeout = 30

# Retry attempts for failed requests
nft-oracle-retry-attempts = 3

# Cache TTL for verified results (seconds)
nft-oracle-cache-ttl = 3600

# Enable detailed logging
nft-oracle-verbose = false
```

### RPC Endpoint Management

**Endpoint Storage:**

```cpp
struct rpc_endpoint_config
{
   uint32_t    chain_id;
   std::string url;
   uint32_t    confirmations;
   uint32_t    timeout_ms;
   uint32_t    max_retries;

   // Connection pooling
   size_t      max_connections;

   // Health check
   bool        enabled;
   fc::time_point_sec last_success;
   uint32_t    consecutive_failures;
};

class rpc_endpoint_manager
{
   public:
      void add_endpoint(const rpc_endpoint_config& config);
      void remove_endpoint(uint32_t chain_id);

      optional<rpc_endpoint_config> get_endpoint(uint32_t chain_id);

      // Health monitoring
      void mark_success(uint32_t chain_id);
      void mark_failure(uint32_t chain_id);

      // Automatic failover
      optional<rpc_endpoint_config> get_healthy_endpoint(uint32_t chain_id);
};
```

**Endpoint Configuration Parsing:**

```cpp
void nft_oracle_plugin::set_program_options(
   boost::program_options::options_description& cli,
   boost::program_options::options_description& cfg)
{
   cfg.add_options()
      ("nft-oracle-rpc-endpoint",
       boost::program_options::value<std::vector<std::string>>()->composing(),
       "EVM RPC endpoint (format: chain_id:url)")
      ("nft-oracle-threads",
       boost::program_options::value<uint32_t>()->default_value(4),
       "Number of verification worker threads")
      ("nft-oracle-confirmations",
       boost::program_options::value<uint32_t>()->default_value(12),
       "Block confirmations required")
      ("nft-oracle-timeout",
       boost::program_options::value<uint32_t>()->default_value(30),
       "RPC request timeout (seconds)")
      ("nft-oracle-retry-attempts",
       boost::program_options::value<uint32_t>()->default_value(3),
       "Number of retry attempts")
      ("nft-oracle-cache-ttl",
       boost::program_options::value<uint32_t>()->default_value(3600),
       "Verification cache TTL (seconds)")
      ("nft-oracle-verbose",
       boost::program_options::value<bool>()->default_value(false),
       "Enable verbose logging");
}

void nft_oracle_plugin::plugin_initialize(
   const boost::program_options::variables_map& options)
{
   my = std::make_unique<detail::nft_oracle_plugin_impl>(*this);

   // Parse RPC endpoints
   if (options.count("nft-oracle-rpc-endpoint"))
   {
      auto endpoints = options["nft-oracle-rpc-endpoint"]
                          .as<std::vector<std::string>>();

      for (const auto& endpoint : endpoints)
      {
         auto colon_pos = endpoint.find(':');
         FC_ASSERT(colon_pos != std::string::npos,
                   "Invalid endpoint format: ${ep}", ("ep", endpoint));

         uint32_t chain_id = std::stoul(endpoint.substr(0, colon_pos));
         std::string url = endpoint.substr(colon_pos + 1);

         rpc_endpoint_config config;
         config.chain_id = chain_id;
         config.url = url;
         config.confirmations = options["nft-oracle-confirmations"].as<uint32_t>();
         config.timeout_ms = options["nft-oracle-timeout"].as<uint32_t>() * 1000;
         config.max_retries = options["nft-oracle-retry-attempts"].as<uint32_t>();
         config.enabled = true;

         my->endpoint_manager.add_endpoint(config);

         ilog("NFT Oracle: Configured chain ${chain}: ${url}",
              ("chain", chain_id)("url", url));
      }
   }

   // Initialize thread pool
   uint32_t num_threads = options["nft-oracle-threads"].as<uint32_t>();
   my->work = std::make_unique<boost::asio::io_service::work>(my->io_service);

   for (uint32_t i = 0; i < num_threads; ++i)
   {
      my->thread_pool.create_thread([this]() {
         my->io_service.run();
      });
   }

   // Register signal handlers
   auto& db = app().get_plugin<chain::chain_plugin>().db();
   db.on_nft_proof_submitted.connect([this](const witness_nft_proof_operation& op) {
      my->on_proof_submitted(op);
   });
}
```

### Proof Verification Flow

**1. Signal Handler:**

```cpp
void nft_oracle_plugin_impl::on_proof_submitted(
   const witness_nft_proof_operation& op)
{
   // Queue verification task
   io_service.post([this, op]() {
      try {
         verify_nft_ownership(op);
      }
      catch (const fc::exception& e) {
         elog("NFT verification error: ${e}", ("e", e.to_detail_string()));
      }
   });
}
```

**2. Ownership Verification:**

```cpp
void nft_oracle_plugin_impl::verify_nft_ownership(
   const witness_nft_proof_operation& op)
{
   auto& db = _self.chain_plugin().db();

   // Get proof object from database
   const witness_nft_proof_object* proof = nullptr;

   db.with_read_lock([&]() {
      const auto& idx = db.get_index<witness_nft_proof_index>()
                           .indices().get<by_witness>();
      auto itr = idx.find(op.witness_account);
      if (itr != idx.end()) {
         proof = &(*itr);
      }
   });

   if (!proof) {
      wlog("Proof not found for witness ${w}", ("w", op.witness_account));
      return;
   }

   // Get RPC endpoint
   auto endpoint_opt = endpoint_manager.get_healthy_endpoint(op.chain_id);
   if (!endpoint_opt) {
      elog("No healthy RPC endpoint for chain ${chain}", ("chain", op.chain_id));
      mark_verification_failed(proof->id, "No RPC endpoint available");
      return;
   }

   const auto& endpoint = *endpoint_opt;

   try {
      // Query NFT ownership
      std::string owner = query_nft_owner(
         endpoint,
         op.contract_address,
         op.token_id
      );

      // Normalize addresses to lowercase
      std::string claimed_addr = to_lowercase(op.eth_address);
      std::string actual_owner = to_lowercase(owner);

      bool verified = (claimed_addr == actual_owner);

      // Update verification status
      mark_verification_result(proof->id, verified,
         verified ? "Ownership verified" :
                   "Owner mismatch: actual=" + owner);

      endpoint_manager.mark_success(op.chain_id);

      ilog("NFT verification ${result} for ${witness}: ${contract}:${token}",
           ("result", verified ? "SUCCESS" : "FAILED")
           ("witness", op.witness_account)
           ("contract", op.contract_address)
           ("token", op.token_id));
   }
   catch (const fc::exception& e) {
      elog("Verification error: ${e}", ("e", e.to_string()));
      mark_verification_failed(proof->id, e.to_string());
      endpoint_manager.mark_failure(op.chain_id);
   }
}
```

**3. ERC-721 Query:**

```cpp
std::string nft_oracle_plugin_impl::query_nft_owner(
   const rpc_endpoint_config& endpoint,
   const std::string& contract_address,
   const std::string& token_id)
{
   // ERC-721 ownerOf(uint256 tokenId) returns (address)
   // Function selector: 0x6352211e

   std::string data = "0x6352211e" + pad_hex(token_id, 64);

   fc::variant params = fc::mutable_variant_object()
      ("to", contract_address)
      ("data", data);

   // Make eth_call
   fc::variant result = rpc_call(
      endpoint.url,
      "eth_call",
      fc::variants{params, "latest"},
      endpoint.timeout_ms
   );

   // Parse result (32-byte hex address)
   std::string owner_hex = result.as_string();

   // Convert to Ethereum address format (0x + last 40 chars)
   if (owner_hex.size() >= 42) {
      return "0x" + owner_hex.substr(owner_hex.size() - 40);
   }

   FC_THROW_EXCEPTION(fc::exception,
      "Invalid ownerOf response: ${res}", ("res", owner_hex));
}
```

**4. RPC Call Implementation:**

```cpp
fc::variant nft_oracle_plugin_impl::rpc_call(
   const std::string& url,
   const std::string& method,
   const fc::variants& params,
   uint32_t timeout_ms)
{
   // Build JSON-RPC 2.0 request
   fc::mutable_variant_object request;
   request["jsonrpc"] = "2.0";
   request["id"] = generate_request_id();
   request["method"] = method;
   request["params"] = params;

   std::string request_body = fc::json::to_string(request);

   // HTTP POST request
   fc::http::connection conn;
   conn.set_read_timeout(timeout_ms / 1000);

   fc::http::reply response = conn.request(
      "POST",
      url,
      request_body,
      fc::http::headers{{"Content-Type", "application/json"}}
   );

   if (response.status != 200) {
      FC_THROW_EXCEPTION(fc::exception,
         "RPC request failed: HTTP ${status}",
         ("status", response.status));
   }

   // Parse response
   fc::variant response_json = fc::json::from_string(response.body);

   if (response_json.get_object().contains("error")) {
      auto error = response_json["error"];
      FC_THROW_EXCEPTION(fc::exception,
         "RPC error: ${msg}",
         ("msg", error["message"].as_string()));
   }

   return response_json["result"];
}
```

**5. Database Update:**

```cpp
void nft_oracle_plugin_impl::mark_verification_result(
   witness_nft_proof_id_type proof_id,
   bool verified,
   const std::string& details)
{
   auto& db = _self.chain_plugin().db();

   db.with_write_lock([&]() {
      const auto& proof = db.get<witness_nft_proof_object>(proof_id);

      db.modify(proof, [&](witness_nft_proof_object& p) {
         p.verification_status = verified ?
            witness_nft_proof_object::verified :
            witness_nft_proof_object::failed;
         p.verification_timestamp = db.head_block_time();
         p.verification_details = details;
      });
   });
}

void nft_oracle_plugin_impl::mark_verification_failed(
   witness_nft_proof_id_type proof_id,
   const std::string& error_msg)
{
   mark_verification_result(proof_id, false, "Error: " + error_msg);
}
```

### Helper Functions

**Hex Padding:**

```cpp
std::string pad_hex(const std::string& value, size_t length)
{
   std::string hex = value;

   // Remove 0x prefix if present
   if (hex.substr(0, 2) == "0x") {
      hex = hex.substr(2);
   }

   // Pad with leading zeros
   if (hex.size() < length) {
      hex = std::string(length - hex.size(), '0') + hex;
   }

   return hex;
}
```

**Address Normalization:**

```cpp
std::string to_lowercase(const std::string& addr)
{
   std::string result = addr;
   std::transform(result.begin(), result.end(), result.begin(), ::tolower);
   return result;
}
```

**Request ID Generation:**

```cpp
uint64_t generate_request_id()
{
   static std::atomic<uint64_t> counter(0);
   return ++counter;
}
```

### Caching Layer

**Cache Implementation:**

```cpp
struct verification_cache_entry
{
   std::string contract_address;
   std::string token_id;
   uint32_t chain_id;
   std::string owner;
   fc::time_point_sec cached_at;
};

class verification_cache
{
   public:
      void put(uint32_t chain_id,
               const std::string& contract,
               const std::string& token,
               const std::string& owner,
               uint32_t ttl_seconds);

      optional<std::string> get(uint32_t chain_id,
                                 const std::string& contract,
                                 const std::string& token);

      void cleanup_expired();

   private:
      std::map<std::string, verification_cache_entry> cache_;
      std::mutex mutex_;

      std::string make_key(uint32_t chain_id,
                           const std::string& contract,
                           const std::string& token);
};
```

### Error Handling

**Retry Logic:**

```cpp
fc::variant rpc_call_with_retry(
   const rpc_endpoint_config& endpoint,
   const std::string& method,
   const fc::variants& params)
{
   uint32_t attempts = 0;
   fc::exception last_error;

   while (attempts < endpoint.max_retries)
   {
      try {
         return rpc_call(endpoint.url, method, params, endpoint.timeout_ms);
      }
      catch (const fc::exception& e) {
         last_error = e;
         attempts++;

         if (attempts < endpoint.max_retries) {
            wlog("RPC attempt ${attempt} failed, retrying...",
                 ("attempt", attempts));
            fc::usleep(fc::milliseconds(1000 * attempts)); // Exponential backoff
         }
      }
   }

   FC_THROW_EXCEPTION(fc::exception,
      "RPC failed after ${attempts} attempts: ${err}",
      ("attempts", attempts)("err", last_error.to_string()));
}
```

## Rationale

### Design Decisions

#### 1. Plugin vs. Separate Service

**Decision:** Implement as integrated plugin

**Reasons:**
- Direct access to database for status updates
- Simplified deployment (single binary)
- Shared configuration with steemd
- Access to chain signals and events

**Trade-off:** Less isolation than separate service

#### 2. Asynchronous Architecture

**Decision:** Use thread pool with async workers

**Reasons:**
- Non-blocking blockchain operations
- Parallel verification of multiple proofs
- Handles slow RPC endpoints gracefully
- Scalable with configurable thread count

#### 3. Read-Only RPC Calls

**Decision:** Use `eth_call` instead of transactions

**Reasons:**
- No gas costs
- Instant results (no mining delay)
- No transaction signing required
- Simple implementation

#### 4. Witness-Operated Oracles

**Decision:** Each witness runs their own oracle

**Reasons:**
- Decentralized (no single point of control)
- Economically aligned (witnesses verify their own proofs)
- Simple coordination (no oracle network needed)

**Future:** Multi-oracle consensus for additional security

#### 5. Configurable Confirmations

**Decision:** Wait for configurable block confirmations

**Reasons:**
- Protection against chain reorganizations
- Configurable per chain (L1 vs. L2)
- Balance between speed and security

## Backwards Compatibility

This is a new plugin with no backwards compatibility concerns.

**Deployment:**
- Optional plugin (not required for non-witness nodes)
- Witness nodes must enable to participate in NFT verification
- Graceful degradation if oracle unavailable (proofs stay pending)

## Test Cases

### Test Case 1: Basic Ownership Verification

```cpp
BOOST_AUTO_TEST_CASE(oracle_verify_ownership)
{
   // Mock RPC endpoint
   mock_rpc_endpoint endpoint;
   endpoint.add_response("eth_call", "0x000...742d35Cc6634C0532925a3b844Bc9e7595f0bEb");

   // Query ownership
   std::string owner = oracle.query_nft_owner(
      endpoint,
      "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D",
      "1234"
   );

   BOOST_REQUIRE_EQUAL(owner, "0x742d35cc6634c0532925a3b844bc9e7595f0beb");
}
```

### Test Case 2: RPC Timeout Handling

```cpp
BOOST_AUTO_TEST_CASE(oracle_rpc_timeout)
{
   mock_rpc_endpoint endpoint;
   endpoint.set_timeout(100); // 100ms timeout
   endpoint.set_delay(200);   // 200ms delay

   BOOST_REQUIRE_THROW(
      oracle.query_nft_owner(endpoint, "0x...", "1"),
      fc::timeout_exception
   );
}
```

### Test Case 3: Retry Logic

```cpp
BOOST_AUTO_TEST_CASE(oracle_retry_on_failure)
{
   mock_rpc_endpoint endpoint;
   endpoint.fail_count = 2; // Fail first 2 attempts
   endpoint.max_retries = 3;

   // Should succeed on 3rd attempt
   std::string owner = oracle.query_nft_owner(endpoint, "0x...", "1");

   BOOST_REQUIRE_EQUAL(endpoint.attempt_count, 3);
}
```

### Test Case 4: Cache Functionality

```cpp
BOOST_AUTO_TEST_CASE(oracle_cache)
{
   verification_cache cache;

   cache.put(1, "0xABC...", "123", "0x742...", 3600);

   auto cached = cache.get(1, "0xABC...", "123");
   BOOST_REQUIRE(cached.has_value());
   BOOST_REQUIRE_EQUAL(*cached, "0x742...");

   // Different token should miss cache
   auto miss = cache.get(1, "0xABC...", "456");
   BOOST_REQUIRE(!miss.has_value());
}
```

### Test Case 5: Multi-Chain Support

```cpp
BOOST_AUTO_TEST_CASE(oracle_multi_chain)
{
   oracle_plugin oracle;

   oracle.add_endpoint(1, "https://eth.llamarpc.com");
   oracle.add_endpoint(137, "https://polygon-rpc.com");
   oracle.add_endpoint(56, "https://bsc-dataseed.binance.org");

   // Verify on Ethereum
   auto eth_owner = oracle.query_nft_owner(1, "0x...", "1");

   // Verify on Polygon
   auto poly_owner = oracle.query_nft_owner(137, "0x...", "1");

   // Different chains, potentially different owners
   BOOST_REQUIRE(eth_owner != poly_owner);
}
```

## Reference Implementation

Available in feature branch:

- Branch: `feature/nft-oracle-plugin`
- Key Files:
  - `libraries/plugins/nft_oracle/nft_oracle_plugin.hpp`
  - `libraries/plugins/nft_oracle/nft_oracle_plugin.cpp`
  - `libraries/plugins/nft_oracle/rpc_client.hpp`
  - `libraries/plugins/nft_oracle/rpc_client.cpp`
  - `tests/plugin_tests/nft_oracle_tests.cpp`

## Security Considerations

### Oracle Trust Model

**Single Oracle Risk:**
- Current design: Each witness runs own oracle
- Risk: Malicious witness approves fake proofs
- Mitigation: Economic disincentive (reputation loss, unvoting)

**Future Enhancement:** Multi-oracle consensus
- Require N out of M oracles to agree
- Cross-verify with multiple RPC providers
- Detect and penalize dishonest oracles

### RPC Endpoint Security

**Malicious RPC Provider:**
- Risk: Fake RPC returns false ownership data
- Mitigation:
  - Use trusted RPC providers (Infura, Alchemy, QuickNode)
  - Configure multiple fallback endpoints
  - Compare results across providers
  - Monitor for anomalous responses

**RPC Endpoint Compromise:**
- Risk: Attacker controls RPC endpoint
- Mitigation:
  - HTTPS only (prevent MITM)
  - Validate SSL certificates
  - Endpoint reputation monitoring

### Denial of Service

**RPC Flooding:**
- Risk: Spam proof submissions to overload oracle
- Mitigation:
  - Rate limiting per witness
  - Operation fees for proof submission
  - Queue size limits

**Slow RPC Attacks:**
- Risk: Deliberately slow RPC responses
- Mitigation:
  - Configurable timeouts
  - Automatic endpoint health checks
  - Failover to backup endpoints

### Data Integrity

**Response Validation:**
- Verify response format matches ERC-721 standard
- Check for zero address (invalid token)
- Validate hex encoding
- Sanity check response size

**Chain Reorganizations:**
- Wait for configurable confirmations (default: 12)
- Re-verify if reorg detected
- Mark proofs as expired after time limit

### Privacy Considerations

**Public NFT Ownership:**
- All verifications are public on blockchain
- Reveals witness's Ethereum addresses
- Links Steem identity to Ethereum identity

**Mitigation:** This is intentional and acceptable for witness authorization use case

## Future Enhancements

### 1. Multi-Oracle Consensus

Require multiple oracles to verify same proof:

```cpp
struct multi_oracle_config
{
   uint32_t required_confirmations; // e.g., 3 out of 5
   vector<account_name_type> oracle_witnesses;
};
```

### 2. Event Log Verification

Verify NFT transfers via event logs:

```cpp
// Watch Transfer events
event Transfer(address indexed from, address indexed to, uint256 indexed tokenId)
```

### 3. Batch Verification

Verify multiple proofs in single RPC call:

```cpp
// Use eth_batchCall or multicall contract
vector<verification_result> batch_verify(vector<proof> proofs);
```

### 4. Light Client Integration

Run Ethereum light client for trustless verification:

```cpp
// No external RPC needed
class eth_light_client_plugin;
```

### 5. Cross-Chain Bridge Integration

Use native bridges (IBC, LayerZero) for verification:

```cpp
struct bridge_config
{
   string bridge_contract;
   uint32_t verification_delay;
};
```

### 6. Automated Endpoint Discovery

Discover and rank RPC endpoints automatically:

```cpp
class endpoint_discovery_service
{
   void discover_endpoints(uint32_t chain_id);
   void benchmark_endpoints();
   void auto_failover();
};
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
