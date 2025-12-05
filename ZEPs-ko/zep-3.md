---
zep: 3
title: NFT 오라클 플러그인
description: EVM 호환 체인에서 ERC-721 NFT 소유권을 검증하는 오프체인 오라클 서비스
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Interface
created: 2025-11-08
---

## Abstract

이 ZEP는 Ethereum 호환 체인에서 ERC-721 NFT 소유권을 검증하는 오라클 플러그인을 명시합니다. 오라클은 증인 노드에 통합된 오프체인 서비스로 작동하며, RPC를 통해 EVM 체인을 쿼리하여 온체인에 제출된 NFT 소유권 주장을 확인합니다. 검증 결과는 블록체인에 다시 기록되어 증인 인증(ZEP-2) 및 기타 사용 사례를 위한 무신뢰 크로스체인 NFT 검증을 가능하게 합니다.

## Motivation

ZEP-2 (NFT 기반 증인 검증)는 증인이 주장하는 NFT를 실제로 소유하고 있는지 검증하는 메커니즘이 필요합니다. Zattera가 Ethereum 상태에 직접 접근할 수 없으므로 오라클 브리지가 필요합니다.

### 왜 오라클인가?

**고려된 대안 접근 방식:**

1. **크로스체인 브리지** - 너무 복잡하고, 비용이 많이 들며, 느림
2. **스냅샷 기반 검증** - 실시간이 아니고, 중앙화됨
3. **온체인 라이트 클라이언트** - 계산적으로 금지적임

**오라클 장점:**

- 표준 RPC 호출을 사용한 간단한 구현
- 빠른 검증 (브리지의 경우 몇 시간 대 몇 초)
- 최소 가스 비용 (읽기 전용 쿼리)
- 유연한 멀티체인 지원
- 증인 제어 (탈중앙화된 거버넌스)

### 신뢰 모델

오라클은 **무신뢰가 아니지만** 다음을 통해 실용적인 보안을 달성합니다:

1. **증인 운영** - 활성 증인만 오라클 실행
2. **오픈 소스** - 완전히 감사 가능한 코드
3. **다중 오라클 합의** - 여러 오라클이 동일한 주장 검증 (향후)
4. **경제적 보안** - 거짓 검증은 증인 평판 손상
5. **투명한 로그** - 모든 검증이 온체인에 기록됨

## Specification

### 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────┐
│                     Witness Node                            │
│  ┌──────────────┐         ┌───────────────────────────────┐ │
│  │   zatterad   │◄───────►│   NFT Oracle Plugin           │ │
│  │              │         │                               │ │
│  │  - Receives  │         │  - Monitors proof operations  │ │
│  │    proof ops │         │  - Queries EVM chains         │ │
│  │  - Updates   │         │  - Updates verification       │ │
│  │    status    │         │    status                     │ │
│  └──────────────┘         └───────────────────────────────┘ │
│         ▲                            │                      │
│         │                            ▼                      │
│         │                 ┌─────────────────────┐           │
│         └─────────────────│  Thread Pool        │           │
│                           │  (async workers)    │           │
│                           └─────────────────────┘           │
└─────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
                         ┌────────────────────────────┐
                         │   EVM RPC Endpoints        │
                         │                            │
                         │  - Ethereum (eth.llamarpc) │
                         │  - Polygon (polygon-rpc)   │
                         │  - BSC (bsc-dataseed)      │
                         │  - Arbitrum, Optimism, etc.│
                         └────────────────────────────┘
```

### 플러그인 인터페이스

오라클은 표준 Appbase 플러그인 인터페이스를 구현합니다.

**플러그인 선언:**

```cpp
namespace zattera { namespace plugins { namespace nft_oracle {

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

} } } // zattera::plugins::nft_oracle
```

### 구성 매개변수

**File:** `config.ini`

```ini
# NFT 오라클 플러그인 활성화
plugin = nft_oracle

# EVM RPC 엔드포인트 (형식: chain_id:url)
nft-oracle-rpc-endpoint = 1:https://eth.llamarpc.com
nft-oracle-rpc-endpoint = 137:https://polygon-rpc.com
nft-oracle-rpc-endpoint = 56:https://bsc-dataseed.binance.org
nft-oracle-rpc-endpoint = 42161:https://arb1.arbitrum.io/rpc

# 검증을 위한 워커 스레드 수
nft-oracle-threads = 4

# 결과를 신뢰하기 전에 필요한 확인 블록
nft-oracle-confirmations = 12

# RPC 요청 타임아웃 (초)
nft-oracle-timeout = 30

# 실패한 요청에 대한 재시도 횟수
nft-oracle-retry-attempts = 3

# 검증된 결과에 대한 캐시 TTL (초)
nft-oracle-cache-ttl = 3600

# 상세 로깅 활성화
nft-oracle-verbose = false
```

### RPC 엔드포인트 관리

**엔드포인트 저장:**

```cpp
struct rpc_endpoint_config
{
   uint32_t    chain_id;
   std::string url;
   uint32_t    confirmations;
   uint32_t    timeout_ms;
   uint32_t    max_retries;

   // 연결 풀링
   size_t      max_connections;

   // 헬스 체크
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

      // 헬스 모니터링
      void mark_success(uint32_t chain_id);
      void mark_failure(uint32_t chain_id);

      // 자동 페일오버
      optional<rpc_endpoint_config> get_healthy_endpoint(uint32_t chain_id);
};
```

**엔드포인트 구성 파싱:**

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

   // RPC 엔드포인트 파싱
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

   // 스레드 풀 초기화
   uint32_t num_threads = options["nft-oracle-threads"].as<uint32_t>();
   my->work = std::make_unique<boost::asio::io_service::work>(my->io_service);

   for (uint32_t i = 0; i < num_threads; ++i)
   {
      my->thread_pool.create_thread([this]() {
         my->io_service.run();
      });
   }

   // 신호 핸들러 등록
   auto& db = app().get_plugin<chain::chain_plugin>().db();
   db.on_nft_proof_submitted.connect([this](const witness_nft_proof_operation& op) {
      my->on_proof_submitted(op);
   });
}
```

### 증명 검증 플로우

**1. 신호 핸들러:**

```cpp
void nft_oracle_plugin_impl::on_proof_submitted(
   const witness_nft_proof_operation& op)
{
   // 검증 작업 큐에 추가
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

**2. 소유권 검증:**

```cpp
void nft_oracle_plugin_impl::verify_nft_ownership(
   const witness_nft_proof_operation& op)
{
   auto& db = _self.chain_plugin().db();

   // 데이터베이스에서 증명 객체 가져오기
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

   // RPC 엔드포인트 가져오기
   auto endpoint_opt = endpoint_manager.get_healthy_endpoint(op.chain_id);
   if (!endpoint_opt) {
      elog("No healthy RPC endpoint for chain ${chain}", ("chain", op.chain_id));
      mark_verification_failed(proof->id, "No RPC endpoint available");
      return;
   }

   const auto& endpoint = *endpoint_opt;

   try {
      // NFT 소유권 쿼리
      std::string owner = query_nft_owner(
         endpoint,
         op.contract_address,
         op.token_id
      );

      // 주소를 소문자로 정규화
      std::string claimed_addr = to_lowercase(op.eth_address);
      std::string actual_owner = to_lowercase(owner);

      bool verified = (claimed_addr == actual_owner);

      // 검증 상태 업데이트
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

**3. ERC-721 쿼리:**

```cpp
std::string nft_oracle_plugin_impl::query_nft_owner(
   const rpc_endpoint_config& endpoint,
   const std::string& contract_address,
   const std::string& token_id)
{
   // ERC-721 ownerOf(uint256 tokenId) returns (address)
   // 함수 선택자: 0x6352211e

   std::string data = "0x6352211e" + pad_hex(token_id, 64);

   fc::variant params = fc::mutable_variant_object()
      ("to", contract_address)
      ("data", data);

   // eth_call 수행
   fc::variant result = rpc_call(
      endpoint.url,
      "eth_call",
      fc::variants{params, "latest"},
      endpoint.timeout_ms
   );

   // 결과 파싱 (32바이트 hex 주소)
   std::string owner_hex = result.as_string();

   // Ethereum 주소 형식으로 변환 (0x + 마지막 40자)
   if (owner_hex.size() >= 42) {
      return "0x" + owner_hex.substr(owner_hex.size() - 40);
   }

   FC_THROW_EXCEPTION(fc::exception,
      "Invalid ownerOf response: ${res}", ("res", owner_hex));
}
```

**4. RPC 호출 구현:**

```cpp
fc::variant nft_oracle_plugin_impl::rpc_call(
   const std::string& url,
   const std::string& method,
   const fc::variants& params,
   uint32_t timeout_ms)
{
   // JSON-RPC 2.0 요청 구성
   fc::mutable_variant_object request;
   request["jsonrpc"] = "2.0";
   request["id"] = generate_request_id();
   request["method"] = method;
   request["params"] = params;

   std::string request_body = fc::json::to_string(request);

   // HTTP POST 요청
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

   // 응답 파싱
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

**5. 데이터베이스 업데이트:**

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

### 헬퍼 함수

**Hex 패딩:**

```cpp
std::string pad_hex(const std::string& value, size_t length)
{
   std::string hex = value;

   // 0x 접두사가 있으면 제거
   if (hex.substr(0, 2) == "0x") {
      hex = hex.substr(2);
   }

   // 앞에 0을 패딩
   if (hex.size() < length) {
      hex = std::string(length - hex.size(), '0') + hex;
   }

   return hex;
}
```

**주소 정규화:**

```cpp
std::string to_lowercase(const std::string& addr)
{
   std::string result = addr;
   std::transform(result.begin(), result.end(), result.begin(), ::tolower);
   return result;
}
```

**요청 ID 생성:**

```cpp
uint64_t generate_request_id()
{
   static std::atomic<uint64_t> counter(0);
   return ++counter;
}
```

### 캐싱 레이어

**캐시 구현:**

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

### 오류 처리

**재시도 로직:**

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
            fc::usleep(fc::milliseconds(1000 * attempts)); // 지수 백오프
         }
      }
   }

   FC_THROW_EXCEPTION(fc::exception,
      "RPC failed after ${attempts} attempts: ${err}",
      ("attempts", attempts)("err", last_error.to_string()));
}
```

## Rationale

### 설계 결정

#### 1. 플러그인 vs. 별도 서비스

**결정:** 통합 플러그인으로 구현

**이유:**
- 상태 업데이트를 위한 데이터베이스 직접 접근
- 간소화된 배포 (단일 바이너리)
- zatterad와 공유된 구성
- 체인 신호 및 이벤트에 대한 접근

**트레이드오프:** 별도 서비스보다 격리가 적음

#### 2. 비동기 아키텍처

**결정:** 비동기 워커가 있는 스레드 풀 사용

**이유:**
- 비차단 블록체인 작업
- 여러 증명의 병렬 검증
- 느린 RPC 엔드포인트를 우아하게 처리
- 구성 가능한 스레드 수로 확장 가능

#### 3. 읽기 전용 RPC 호출

**결정:** 트랜잭션 대신 `eth_call` 사용

**이유:**
- 가스 비용 없음
- 즉각적인 결과 (마이닝 지연 없음)
- 트랜잭션 서명 불필요
- 간단한 구현

#### 4. 증인 운영 오라클

**결정:** 각 증인이 자신의 오라클 실행

**이유:**
- 탈중앙화됨 (단일 제어 지점 없음)
- 경제적으로 일치함 (증인이 자신의 증명 검증)
- 간단한 조정 (오라클 네트워크 불필요)

**향후:** 추가 보안을 위한 다중 오라클 합의

#### 5. 구성 가능한 확인

**결정:** 구성 가능한 블록 확인 대기

**이유:**
- 체인 재구성에 대한 보호
- 체인당 구성 가능 (L1 vs. L2)
- 속도와 보안 간 균형

## Backwards Compatibility

이것은 역호환성 문제가 없는 새로운 플러그인입니다.

**배포:**
- 선택적 플러그인 (비증인 노드에는 필요 없음)
- 증인 노드는 NFT 검증에 참여하려면 활성화해야 함
- 오라클을 사용할 수 없는 경우 우아한 성능 저하 (증명은 pending 상태로 유지)

## Test Cases

### 테스트 케이스 1: 기본 소유권 검증

```cpp
BOOST_AUTO_TEST_CASE(oracle_verify_ownership)
{
   // Mock RPC 엔드포인트
   mock_rpc_endpoint endpoint;
   endpoint.add_response("eth_call", "0x000...742d35Cc6634C0532925a3b844Bc9e7595f0bEb");

   // 소유권 쿼리
   std::string owner = oracle.query_nft_owner(
      endpoint,
      "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D",
      "1234"
   );

   BOOST_REQUIRE_EQUAL(owner, "0x742d35cc6634c0532925a3b844bc9e7595f0beb");
}
```

### 테스트 케이스 2: RPC 타임아웃 처리

```cpp
BOOST_AUTO_TEST_CASE(oracle_rpc_timeout)
{
   mock_rpc_endpoint endpoint;
   endpoint.set_timeout(100); // 100ms 타임아웃
   endpoint.set_delay(200);   // 200ms 지연

   BOOST_REQUIRE_THROW(
      oracle.query_nft_owner(endpoint, "0x...", "1"),
      fc::timeout_exception
   );
}
```

### 테스트 케이스 3: 재시도 로직

```cpp
BOOST_AUTO_TEST_CASE(oracle_retry_on_failure)
{
   mock_rpc_endpoint endpoint;
   endpoint.fail_count = 2; // 처음 2번 시도 실패
   endpoint.max_retries = 3;

   // 3번째 시도에서 성공해야 함
   std::string owner = oracle.query_nft_owner(endpoint, "0x...", "1");

   BOOST_REQUIRE_EQUAL(endpoint.attempt_count, 3);
}
```

### 테스트 케이스 4: 캐시 기능

```cpp
BOOST_AUTO_TEST_CASE(oracle_cache)
{
   verification_cache cache;

   cache.put(1, "0xABC...", "123", "0x742...", 3600);

   auto cached = cache.get(1, "0xABC...", "123");
   BOOST_REQUIRE(cached.has_value());
   BOOST_REQUIRE_EQUAL(*cached, "0x742...");

   // 다른 토큰은 캐시 미스되어야 함
   auto miss = cache.get(1, "0xABC...", "456");
   BOOST_REQUIRE(!miss.has_value());
}
```

### 테스트 케이스 5: 멀티체인 지원

```cpp
BOOST_AUTO_TEST_CASE(oracle_multi_chain)
{
   oracle_plugin oracle;

   oracle.add_endpoint(1, "https://eth.llamarpc.com");
   oracle.add_endpoint(137, "https://polygon-rpc.com");
   oracle.add_endpoint(56, "https://bsc-dataseed.binance.org");

   // Ethereum에서 검증
   auto eth_owner = oracle.query_nft_owner(1, "0x...", "1");

   // Polygon에서 검증
   auto poly_owner = oracle.query_nft_owner(137, "0x...", "1");

   // 다른 체인, 잠재적으로 다른 소유자
   BOOST_REQUIRE(eth_owner != poly_owner);
}
```

## Reference Implementation

기능 브랜치에서 사용 가능:

- Branch: `feature/nft-oracle-plugin`
- 주요 파일:
  - `src/plugins/nft_oracle/nft_oracle_plugin.hpp`
  - `src/plugins/nft_oracle/nft_oracle_plugin.cpp`
  - `src/plugins/nft_oracle/rpc_client.hpp`
  - `src/plugins/nft_oracle/rpc_client.cpp`
  - `tests/plugin_tests/nft_oracle_tests.cpp`

## Security Considerations

### 오라클 신뢰 모델

**단일 오라클 위험:**
- 현재 설계: 각 증인이 자신의 오라클 실행
- 위험: 악의적인 증인이 가짜 증명 승인
- 완화: 경제적 억제 (평판 손실, 투표 취소)

**향후 개선:** 다중 오라클 합의
- N개 중 M개의 오라클이 동의하도록 요구
- 여러 RPC 제공자와 교차 검증
- 부정직한 오라클 감지 및 처벌

### RPC 엔드포인트 보안

**악의적인 RPC 제공자:**
- 위험: 가짜 RPC가 거짓 소유권 데이터 반환
- 완화:
  - 신뢰할 수 있는 RPC 제공자 사용 (Infura, Alchemy, QuickNode)
  - 여러 폴백 엔드포인트 구성
  - 제공자 간 결과 비교
  - 이상 응답 모니터링

**RPC 엔드포인트 손상:**
- 위험: 공격자가 RPC 엔드포인트 제어
- 완화:
  - HTTPS만 사용 (MITM 방지)
  - SSL 인증서 검증
  - 엔드포인트 평판 모니터링

### 서비스 거부

**RPC 플러딩:**
- 위험: 스팸 증명 제출로 오라클 과부하
- 완화:
  - 증인당 속도 제한
  - 증명 제출에 대한 operation 수수료
  - 큐 크기 제한

**느린 RPC 공격:**
- 위험: 의도적으로 느린 RPC 응답
- 완화:
  - 구성 가능한 타임아웃
  - 자동 엔드포인트 헬스 체크
  - 백업 엔드포인트로 페일오버

### 데이터 무결성

**응답 검증:**
- 응답 형식이 ERC-721 표준과 일치하는지 확인
- 제로 주소 확인 (유효하지 않은 토큰)
- hex 인코딩 검증
- 응답 크기 건전성 확인

**체인 재구성:**
- 구성 가능한 확인 대기 (기본값: 12)
- 재구성이 감지되면 재검증
- 시간 제한 후 증명을 만료로 표시

### 프라이버시 고려 사항

**공개 NFT 소유권:**
- 모든 검증은 블록체인에서 공개됨
- 증인의 Ethereum 주소 공개
- Zattera 신원을 Ethereum 신원과 연결

**완화:** 이것은 증인 인증 사용 사례에 대해 의도적이고 허용 가능함

## Future Enhancements

### 1. 다중 오라클 합의

여러 오라클이 동일한 증명을 검증하도록 요구:

```cpp
struct multi_oracle_config
{
   uint32_t required_confirmations; // 예: 5개 중 3개
   vector<account_name_type> oracle_witnesses;
};
```

### 2. 이벤트 로그 검증

이벤트 로그를 통해 NFT 전송 검증:

```cpp
// Transfer 이벤트 감시
event Transfer(address indexed from, address indexed to, uint256 indexed tokenId)
```

### 3. 배치 검증

단일 RPC 호출로 여러 증명 검증:

```cpp
// eth_batchCall 또는 multicall 컨트랙트 사용
vector<verification_result> batch_verify(vector<proof> proofs);
```

### 4. 라이트 클라이언트 통합

무신뢰 검증을 위한 Ethereum 라이트 클라이언트 실행:

```cpp
// 외부 RPC 불필요
class eth_light_client_plugin;
```

### 5. 크로스체인 브리지 통합

검증을 위해 네이티브 브리지(IBC, LayerZero) 사용:

```cpp
struct bridge_config
{
   string bridge_contract;
   uint32_t verification_delay;
};
```

### 6. 자동 엔드포인트 검색

RPC 엔드포인트를 자동으로 검색 및 순위 지정:

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
