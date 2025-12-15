---
zep: 2
title: NFT 기반 증인 검증
description: 블록 생산 자격을 위해 증인의 ERC-721 NFT 소유권 증명 요구
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2025-11-08
requires: 3
---

## Abstract

이 ZEP는 블록 생산자가 Ethereum 호환 체인의 ERC-721 NFT 소유권을 증명하도록 요구하는 증인 인증 시스템을 제안합니다. 증인은 자신이 해당 NFT의 실제 소유자임을 암호학적으로 증명하는 메시지를 주기적으로 제출해야 합니다. 오라클 서비스는 증인이 블록 생산 자격을 얻기 전에 온체인 소유권을 검증합니다.

## Motivation

전통적인 블록체인 합의는 스테이크 기반 또는 평판 기반 증인 선택에만 의존합니다. 이는 다음과 같은 한계를 만듭니다:

1. **제한적인 시빌 저항**: 증인이 여러 신원으로 스테이크를 분할할 수 있음
2. **플루토크라시 우려**: 부의 집중이 중앙화된 블록 생산으로 이어짐
3. **외부 책임 부재**: 블록체인 신원과 실제 자산 간의 연결이 없음
4. **거버넌스 경직성**: 프로토콜에 고정된 증인 요구 사항, 조정이 어려움

NFT 기반 검증은 이러한 문제를 해결합니다:

1. **향상된 시빌 저항**: NFT 소유권이 추가적인 신원 검증 제공
2. **유연한 거버넌스**: 기존 증인이 NFT 컬렉션 요구 사항에 투표 가능
3. **커뮤니티 정렬**: 증인 자격을 커뮤니티 발행 NFT 보유와 연결
4. **크로스체인 통합**: Ethereum 생태계의 확립된 NFT 인프라 활용
5. **경제적 보안**: NFT는 생태계 성공에 대한 재정적 지분 대표

### 사용 사례

- **거버넌스 NFT**: 프로젝트가 신뢰할 수 있는 커뮤니티 구성원에게 NFT 발행
- **증인 라이선스**: 제한된 NFT 컬렉션으로 고정된 증인 세트 크기 생성
- **스테이크 표현**: NFT가 대규모 스테이크 보유자의 투표권 대표
- **크로스체인 DAO**: 여러 블록체인에 걸친 거버넌스 조정

## Specification

### 시스템 아키텍처

```
┌─────────────────────┐
│   Witness Node      │
│                     │
│  1. Generate ETH    │
│     signature       │
│     proving NFT     │
│     ownership       │
└──────────┬──────────┘
           │
           ▼
┌──────────────────────┐      ┌──────────────────────┐
│  Zattera Blockchain  │◄────►│  Oracle Service      │
│                      │      │  (Off-chain)         │
│  2. Verify           │      │                      │
│     signature        │      │  3. Query ERC-721    │
│                      │      │     ownerOf()        │
│  4. Update witness   │      │     on EVM chain     │
│     schedule         │      │                      │
└──────────────────────┘      └──────────────────────┘
```

### 핵심 구성 요소

#### 1. NFT 컬렉션 등록

증인들이 집단적으로 어떤 NFT 컬렉션이 승인될지 투표합니다.

**Operation**: `nft_collection_register_operation`

```cpp
struct nft_collection_register_operation : public base_operation
{
   account_name_type proposer;             // 컬렉션을 제안하는 증인

   string            collection_name;       // 사람이 읽을 수 있는 이름
   string            contract_address;      // 0x... 형식 (42자)
   uint32_t          chain_id;              // 1=Ethereum, 137=Polygon 등
   uint32_t          min_nft_count;         // 필요한 최소 NFT 수 (기본값: 1)
   bool              active;                // 승인될 때까지 초기에는 false

   extensions_type   extensions;

   void validate() const
   {
      FC_ASSERT(is_valid_account_name(proposer), "Invalid proposer");
      FC_ASSERT(!collection_name.empty() && collection_name.size() <= 100,
                "Collection name required (max 100 chars)");
      FC_ASSERT(contract_address.size() == 42, "Invalid contract address");
      FC_ASSERT(contract_address.substr(0, 2) == "0x",
                "Contract address must start with 0x");
      FC_ASSERT(min_nft_count > 0 && min_nft_count <= 100,
                "Min NFT count must be 1-100");
   }

   void get_required_active_authorities(flat_set<account_name_type>& a) const
   {
      a.insert(proposer);
   }
};
```

**Operation**: `nft_collection_approve_operation`

```cpp
struct nft_collection_approve_operation : public base_operation
{
   account_name_type witness_account;
   uint64_t          collection_id;
   bool              approve;  // true=승인, false=거부

   extensions_type   extensions;

   void validate() const
   {
      FC_ASSERT(is_valid_account_name(witness_account), "Invalid witness");
   }

   void get_required_active_authorities(flat_set<account_name_type>& a) const
   {
      a.insert(witness_account);
   }
};
```

**Database Object**: `nft_collection_object`

```cpp
class nft_collection_object : public object<nft_collection_object_type, nft_collection_object>
{
   nft_collection_id_type  id;

   string                  collection_name;
   string                  contract_address;
   uint32_t                chain_id;
   uint32_t                min_nft_count;
   bool                    active;

   // 증인 투표
   flat_set<account_name_type> approved_by;
   flat_set<account_name_type> rejected_by;

   time_point_sec          created;
   time_point_sec          last_updated;
};

typedef multi_index_container<
   nft_collection_object,
   indexed_by<
      ordered_unique<tag<by_id>, member<...>>,
      ordered_unique<tag<by_collection_name>, member<...>>,
      ordered_non_unique<tag<by_contract_address>,
         composite_key<nft_collection_object,
            member<..., string, &nft_collection_object::contract_address>,
            member<..., uint32_t, &nft_collection_object::chain_id>
         >
      >
   >
> nft_collection_index;
```

**승인 로직**:

```cpp
void nft_collection_approve_evaluator::do_apply(const nft_collection_approve_operation& o)
{
   const auto& witness = _db.get_witness(o.witness_account);
   FC_ASSERT(witness.signing_key != public_key_type(), "Not active witness");

   const auto& collection = _db.get<nft_collection_object>(o.collection_id);

   _db.modify(collection, [&](nft_collection_object& c)
   {
      if (o.approve)
      {
         c.approved_by.insert(o.witness_account);
         c.rejected_by.erase(o.witness_account);
      }
      else
      {
         c.rejected_by.insert(o.witness_account);
         c.approved_by.erase(o.witness_account);
      }

      c.last_updated = _db.head_block_time();

      // 상위 증인의 과반수(11/21)가 승인하면 활성화
      const auto& witness_idx = _db.get_index<witness_index>()
                                    .indices().get<by_vote_name>();
      size_t top_witness_count = 0;
      size_t approvals_from_top = 0;

      for (auto itr = witness_idx.begin();
           itr != witness_idx.end() && top_witness_count < ZATTERA_MAX_WITNESSES;
           ++itr, ++top_witness_count)
      {
         if (c.approved_by.count(itr->owner) > 0)
            approvals_from_top++;
      }

      c.active = (approvals_from_top >= (ZATTERA_MAX_WITNESSES / 2 + 1));
   });
}
```

#### 2. 증인 NFT 소유권 증명

증인이 Zattera 계정을 Ethereum 주소에 연결하는 증명을 제출합니다.

**Operation**: `witness_nft_proof_operation`

```cpp
struct witness_nft_proof_operation : public base_operation
{
   account_name_type witness_account;

   string            eth_address;           // 0x... 형식 (42자)
   string            contract_address;      // 0x... 형식 (42자)
   string            token_id;              // NFT 토큰 ID
   uint32_t          chain_id;              // 체인 식별자

   string            eth_signature;         // EIP-191 서명 (0x... 132자)
   time_point_sec    timestamp;             // 증명 타임스탬프

   extensions_type   extensions;

   void validate() const
   {
      FC_ASSERT(is_valid_account_name(witness_account), "Invalid witness");

      // Ethereum 주소 검증 (0x + 40 hex chars)
      FC_ASSERT(eth_address.size() == 42 && eth_address.substr(0, 2) == "0x",
                "Invalid Ethereum address format");

      // 컨트랙트 주소 검증
      FC_ASSERT(contract_address.size() == 42 &&
                contract_address.substr(0, 2) == "0x",
                "Invalid NFT contract address");

      // 토큰 ID 검증
      FC_ASSERT(!token_id.empty(), "Token ID required");

      // 서명 검증 (0x + 130 hex chars = 65 bytes)
      FC_ASSERT(eth_signature.size() == 132 && eth_signature.substr(0, 2) == "0x",
                "Invalid signature format");

      // 타임스탬프는 최근이어야 함 (24시간 이내)
      auto now = time_point::now();
      FC_ASSERT(timestamp <= now, "Timestamp cannot be in future");
      FC_ASSERT(now - timestamp <= fc::days(1),
                "Proof too old (must be within 24 hours)");
   }

   void get_required_active_authorities(flat_set<account_name_type>& a) const
   {
      a.insert(witness_account);
   }
};
```

**서명을 위한 메시지 형식**:

```
"zattera:<witness_account>:nft:<contract_address>:<token_id>:<timestamp>"
```

예시:
```
"zattera:alice:nft:0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D:1234:1704672000"
```

**서명 검증**:

```cpp
void witness_nft_proof_evaluator::do_apply(const witness_nft_proof_operation& o)
{
   const auto& witness = _db.get_witness(o.witness_account);
   FC_ASSERT(witness.signing_key != public_key_type(), "Witness not active");

   // NFT 컬렉션이 등록되고 활성화되었는지 확인
   const auto& collection_idx = _db.get_index<nft_collection_index>()
                                    .indices().get<by_contract_address>();
   auto collection_itr = collection_idx.find(
      boost::make_tuple(o.contract_address, o.chain_id)
   );

   FC_ASSERT(collection_itr != collection_idx.end(),
             "NFT collection not registered: ${c} on chain ${id}",
             ("c", o.contract_address)("id", o.chain_id));
   FC_ASSERT(collection_itr->active, "NFT collection not active");

   // Ethereum 서명 검증
   string message = "zattera:" + o.witness_account.operator string() +
                    ":nft:" + o.contract_address +
                    ":" + o.token_id +
                    ":" + std::to_string(o.timestamp.sec_since_epoch());

   string recovered_address = recover_eth_address(message, o.eth_signature);

   FC_ASSERT(to_lowercase(recovered_address) == to_lowercase(o.eth_address),
             "Signature verification failed");

   // 다른 증인이 이미 이 NFT를 사용 중인지 확인
   const auto& token_idx = _db.get_index<witness_nft_proof_index>()
                               .indices().get<by_token_identity>();
   auto existing_token = token_idx.find(
      boost::make_tuple(o.contract_address, o.token_id, o.chain_id)
   );

   if (existing_token != token_idx.end() &&
       existing_token->witness_account != o.witness_account)
   {
      // 다른 증인이 동일한 NFT를 요구하는 경우
      // NFT 소유권이 이전되었음을 나타냄 - 기존 증명 무효화
      _db.modify(*existing_token, [&](witness_nft_proof_object& p)
      {
         p.verification_status = witness_nft_proof_object::failed;
         p.verification_details = "NFT transferred to " + o.witness_account;
         p.verification_timestamp = _db.head_block_time();
      });
   }

   // 현재 증인의 증명 객체 생성 또는 업데이트
   const auto& proof_idx = _db.get_index<witness_nft_proof_index>()
                               .indices().get<by_witness>();
   auto proof_itr = proof_idx.find(o.witness_account);

   if (proof_itr == proof_idx.end())
   {
      _db.create<witness_nft_proof_object>([&](witness_nft_proof_object& p)
      {
         p.witness_account = o.witness_account;
         p.eth_address = o.eth_address;
         p.contract_address = o.contract_address;
         p.token_id = o.token_id;
         p.chain_id = o.chain_id;
         p.proof_timestamp = o.timestamp;
         p.verification_status = witness_nft_proof_object::pending;
         p.verification_details = "Pending oracle verification";
      });
   }
   else
   {
      _db.modify(*proof_itr, [&](witness_nft_proof_object& p)
      {
         p.eth_address = o.eth_address;
         p.contract_address = o.contract_address;
         p.token_id = o.token_id;
         p.chain_id = o.chain_id;
         p.proof_timestamp = o.timestamp;
         p.verification_status = witness_nft_proof_object::pending;
         p.verification_details = "Pending oracle verification";
      });
   }

   // 오라클이 처리하도록 신호 발생
   _db.notify_nft_proof_submitted(o);
}
```

**Ethereum 서명 복구**:

```cpp
string recover_eth_address(const string& message, const string& signature)
{
   // EIP-191 personal_sign 형식
   string prefixed = "\x19Ethereum Signed Message:\n" +
                     std::to_string(message.length()) +
                     message;

   // Keccak256 해시
   fc::sha256 hash = fc::keccak256(prefixed.data(), prefixed.size());

   // 서명 파싱: r (32 bytes) + s (32 bytes) + v (1 byte)
   auto sig_bytes = fc::from_hex(signature.substr(2));  // 0x 제거
   FC_ASSERT(sig_bytes.size() == 65, "Invalid signature length");

   // 컴포넌트 추출
   fc::ecc::compact_signature compact_sig;
   memcpy(compact_sig.data, sig_bytes.data(), 65);

   // 공개 키 복구
   fc::ecc::public_key pub_key = fc::ecc::public_key(compact_sig, hash);

   // Ethereum 주소 유도 (keccak256(pubkey)의 마지막 20 bytes)
   auto pub_data = pub_key.serialize();
   fc::sha256 pub_hash = fc::keccak256(pub_data.data() + 1, 64);  // 첫 번째 바이트 건너뛰기

   // 마지막 20 bytes 가져오기
   string eth_addr = "0x" + fc::to_hex(pub_hash.data() + 12, 20);

   return eth_addr;
}
```

**Database 신호 메커니즘**:

데이터베이스는 오라클 플러그인이 새로운 NFT 증명 제출을 감지할 수 있도록 신호를 발생시킵니다.

```cpp
// File: src/core/chain/database.hpp
class database
{
public:
   // NFT 증명 제출 신호
   boost::signals2::signal<void(const witness_nft_proof_operation&)> nft_proof_submitted;

   // 신호 발생 메서드
   void notify_nft_proof_submitted(const witness_nft_proof_operation& op)
   {
      nft_proof_submitted(op);
   }

   // ... 기타 메서드 ...
};
```

**오라클 플러그인 통합**:

오라클 플러그인은 이 신호를 구독하여 NFT 소유권 검증을 수행합니다.

```cpp
// File: src/plugins/nft_oracle/nft_oracle_plugin.cpp
void nft_oracle_plugin::plugin_initialize(const boost::program_options::variables_map& options)
{
   // 데이터베이스 신호 구독
   _db.nft_proof_submitted.connect([this](const witness_nft_proof_operation& op) {
      // 비동기 검증 작업 큐에 추가
      enqueue_verification_task(op);
   });
}

void nft_oracle_plugin::enqueue_verification_task(const witness_nft_proof_operation& op)
{
   // 백그라운드 스레드에서 EVM 체인 쿼리
   verification_queue.push([this, op]() {
      verify_nft_ownership(op);
   });
}
```

**Database Object**: `witness_nft_proof_object`

```cpp
class witness_nft_proof_object : public object<witness_nft_proof_object_type, witness_nft_proof_object>
{
   witness_nft_proof_id_type  id;

   account_name_type          witness_account;
   string                     eth_address;
   string                     contract_address;
   string                     token_id;
   uint32_t                   chain_id;

   time_point_sec             proof_timestamp;
   time_point_sec             verification_timestamp;

   enum verification_status_type {
      pending,     // 오라클 검증 대기 중
      verified,    // 오라클이 소유권 확인함
      failed,      // 오라클 검증 실패
      expired      // 증명이 너무 오래됨
   };

   uint8_t                    verification_status;
   string                     verification_details;
};

typedef multi_index_container<
   witness_nft_proof_object,
   indexed_by<
      ordered_unique<tag<by_id>, member<...>>,
      ordered_non_unique<tag<by_witness>, member<...>>,
      ordered_unique<tag<by_token_identity>,
         composite_key<witness_nft_proof_object,
            member<..., string, &witness_nft_proof_object::contract_address>,
            member<..., string, &witness_nft_proof_object::token_id>,
            member<..., uint32_t, &witness_nft_proof_object::chain_id>
         >
      >,
      ordered_non_unique<tag<by_verification_status>,
         composite_key<witness_nft_proof_object,
            member<..., uint8_t, &witness_nft_proof_object::verification_status>,
            member<..., time_point_sec, &witness_nft_proof_object::proof_timestamp>
         >
      >
   >
> witness_nft_proof_index;
```

#### 3. 증인 스케줄 검증

블록 생산 자격에는 유효한 NFT 증명이 필요합니다.

**File**: `src/core/chain/database.cpp`

```cpp
bool database::validate_witness_nft_ownership(const account_name_type& witness) const
{
   // 증인 NFT 증명 가져오기
   const auto& proof_idx = get_index<witness_nft_proof_index>()
                              .indices().get<by_witness>();
   auto proof_itr = proof_idx.find(witness);

   if (proof_itr == proof_idx.end())
      return false;  // 제출된 증명 없음

   // 검증 상태 확인
   if (proof_itr->verification_status != witness_nft_proof_object::verified)
      return false;  // 오라클에 의해 검증되지 않음

   // 만료 확인 (증명은 7일간 유효)
   auto now = head_block_time();
   if (now - proof_itr->proof_timestamp > fc::days(7))
      return false;  // 증명 만료

   return true;
}

void database::update_witness_schedule()
{
   // ... 기존 증인 스케줄 로직 ...

   // 유효한 NFT 증명이 없는 증인 필터링
   auto& schedule = get_witness_schedule_object();
   auto& active_witnesses = schedule.current_shuffled_witnesses;

   active_witnesses.erase(
      std::remove_if(active_witnesses.begin(), active_witnesses.end(),
         [this](const account_name_type& witness) {
            return !validate_witness_nft_ownership(witness);
         }
      ),
      active_witnesses.end()
   );

   FC_ASSERT(!active_witnesses.empty(),
             "No witnesses with valid NFT ownership for block production");

   // ... 나머지 스케줄링 로직 ...
}
```

#### 4. 오라클 서비스 통합

별도의 ZEP(ZEP-3)가 오라클 플러그인 사양을 정의합니다. 주요 요구 사항:

- **검증 방법**: 대상 체인에서 ERC-721 `ownerOf(tokenId)` 호출
- **RPC 엔드포인트**: 구성 가능한 Ethereum/Polygon/BSC 노드 엔드포인트
- **비동기 처리**: 백그라운드 스레드에서 비차단 검증
- **상태 업데이트**: 검증 후 `witness_nft_proof_object` 업데이트
- **오류 처리**: 상세한 오류 메시지와 함께 증명을 `failed`로 표시

**오라클 검증 플로우**:

```
1. 증인이 증명 operation 제출
2. 상태=pending로 증명 객체 생성
3. 오라클 플러그인이 알림 수신
4. 오라클이 RPC를 통해 EVM 체인 쿼리:
   - 호출: nft_contract.ownerOf(token_id)
   - 확인 대기 (예: 12블록)
5. 오라클이 반환된 소유자와 주장된 eth_address 비교
6. 오라클이 증명 객체 업데이트:
   - 일치: 상태=verified
   - 불일치: 상태=failed, details="Owner is 0x..."
   - 오류: 상태=failed, details=오류 메시지
```

### API 메서드

```cpp
// 증인 NFT 증명 상태 가져오기
struct get_witness_nft_proof_return
{
   bool                              has_proof;
   optional<witness_nft_proof_object> proof;
};

get_witness_nft_proof_return get_witness_nft_proof(account_name_type witness);

// 등록된 NFT 컬렉션 목록
struct list_nft_collections_return
{
   vector<nft_collection_object> collections;
};

list_nft_collections_return list_nft_collections(bool active_only);

// 컬렉션 승인 상태 가져오기
struct get_collection_approvals_return
{
   vector<account_name_type> approved_by;
   vector<account_name_type> rejected_by;
   bool is_active;
   uint32_t approvals_needed;
};

get_collection_approvals_return get_collection_approvals(uint64_t collection_id);
```

## Rationale

### 설계 결정

#### 1. ERC-721 표준

**결정**: 커스텀 NFT 형식 대신 ERC-721 사용

**이유**:
- 광범위한 채택을 가진 산업 표준
- 광범위한 도구 및 지갑 지원
- 검증되고 감사된 보안 및 컨트랙트
- 대규모 기존 NFT 마켓플레이스

#### 2. 멀티체인 지원

**결정**: 여러 EVM 호환 체인 지원

**이유**:
- Ethereum 가스 수수료가 과도할 수 있음
- L2 솔루션(Polygon, Arbitrum)이 더 접근 가능함
- 다른 체인은 다른 NFT 커뮤니티를 가짐
- 향후 확장의 유연성

#### 3. 오라클 아키텍처

**결정**: 온체인 브리지 대신 오프체인 오라클

**이유**:
- 더 간단한 구현 (브리지 보안 불필요)
- 더 낮은 비용 (브리지 토큰 수수료 없음)
- 더 빠른 검증 (브리지 지연 없음)
- 더 쉬운 업그레이드 및 유지 관리

**트레이드오프**: 오라클 운영자에 대한 신뢰 (증인 제어로 완화)

#### 4. 증명 만료 (7일)

**결정**: 7일마다 증명 갱신 요구

**이유**:
- NFT 전송 감지 (소유자 변경)
- NFT 판매 후 오래된 증명 방지
- 합리적인 증인 운영 부담
- 주간 유지 관리 주기가 표준임

#### 5. 컬렉션에 대한 증인 합의

**결정**: 컬렉션에 대해 증인 과반수 승인 요구

**이유**:
- 스팸 컬렉션 등록 방지
- 합법적인 NFT 프로젝트만 보장
- 탈중앙화된 거버넌스
- 기존 증인 권한과 일치

#### 6. 서명 기반 증명

**결정**: 크로스체인 메시지 대신 Ethereum 서명 사용

**이유**:
- 브리지 인프라 불필요
- 모든 지갑과 작동 (MetaMask 등)
- Zattera 계정을 ETH 주소에 증명 가능하게 연결
- 비용 효율적 (주당 한 번의 서명)

#### 7. NFT 양도 처리

**결정**: 새로운 증인이 동일한 NFT를 요구할 때 이전 증인의 증명을 자동으로 무효화

**동작 방식**:
증인 B가 증인 A가 현재 보유한 NFT에 대한 증명을 제출할 때:
1. 증인 A의 증명이 즉시 `failed`로 표시됨
2. 검증 세부 정보 업데이트: "NFT transferred to [witness_B]"
3. 증인 A가 블록 생성 자격 상실
4. 증인 B의 증명이 오라클 검증을 위해 `pending` 상태로 진입
5. 오라클이 소유권을 확인한 후에만 증인 B가 자격 획득

**이유**:
- **즉각적인 보호**: 증인 A가 양도된 NFT로 블록 생성을 계속하는 것을 방지
- **무신뢰 전환**: 수동 개입 불필요
- **명확한 소유권**: 하나의 NFT는 한 번에 하나의 증인만 인증 가능
- **감사 추적**: 실패한 증명이 소유권 이전 기록을 남김
- **DoS 방지**: 오라클이 여전히 자격 부여 전에 새 요구를 검증

**트레이드오프**:
- **공격 벡터**: 악의적인 증인이 다른 증인의 NFT를 허위로 요구할 수 있음
  - 완화책: 오라클 검증 실패, 실제 피해 없음
  - 공격자의 오라클 검증까지 원래 증인은 자격 유지
- **경쟁 조건**: 오라클 검증 기간 중 NFT가 판매되는 경우
  - 완화책: 7일 재검증 주기로 소유권 변경 감지
  - 두 증인 모두 오라클 확인까지 일시적으로 자격 상실 가능

### 고려된 대안 접근 방식

#### A. 크로스체인 브리지

**아이디어**: 검증을 위해 IBC 또는 네이티브 브리지 사용

**거부된 이유**:
- 복잡한 인프라
- 높은 운영 비용
- 상당한 공격 표면
- 느린 검증 시간

#### B. 스냅샷 기반 검증

**아이디어**: NFT 보유자의 주기적인 스냅샷

**거부된 이유**:
- 실시간이 아님 (오래된 데이터)
- 중앙화된 스냅샷 권한
- 타이밍 조정이 어려움
- 지속적인 소유권 증명 없음

#### C. NFT 스테이킹

**아이디어**: 스마트 컨트랙트에 NFT 잠금 요구

**거부된 이유**:
- 각 체인에 커스텀 컨트랙트 필요
- 증인의 유동성 우려
- 스마트 컨트랙트 위험
- 멀티체인 스테이킹의 복잡성

**향후**: 향상으로 추가될 수 있음

## Backwards Compatibility

이 ZEP는 하드 포크가 필요한 합의 위반 변경을 도입합니다.

### 호환성 분석

**증인 운영**:
- ⚠️ 기존 증인은 하드포크까지 계속 운영
- ⚠️ 포스트 하드포크: 모든 증인은 NFT 증명 제출 필수
- ⚠️ 유예 기간 권장 (예: 30일)

**데이터베이스 스키마**:
- ⚠️ 새로운 객체 타입 추가
- ⚠️ 증인 스케줄 로직 수정
- ⚠️ 공유 메모리 파일 형식 변경

**API**:
- ✅ 새로운 API 추가 (비위반)
- ✅ 기존 증인 쿼리 변경 없음

### 마이그레이션 경로

**Phase 1: 프리 하드포크 준비 (T-30일)**
1. 모든 증인 노드에 오라클 플러그인 배포
2. EVM RPC 엔드포인트 구성
3. 증인이 초기 NFT 컬렉션 등록 및 승인
4. 테스트넷에서 검증 테스트

**Phase 2: 소프트 활성화 (T-0)**
1. 하드포크로 새로운 operation 활성화
2. 증인이 자발적으로 증명 제출
3. 증명이 검증되지만 강제되지 않음
4. 검증 성공률 모니터링

**Phase 3: 하드 강제 (T+30일)**
1. >90% 증인이 검증된 증명을 가진 경우:
   - 증인 스케줄 필터링 활성화
   - 증명이 없는 증인은 블록에서 제외됨
2. <90%인 경우:
   - 유예 기간 연장
   - 문제 조사

**긴급 롤백**:
- 긴급 하드포크를 통해 NFT 요구 사항 비활성화
- 전통적인 증인 스케줄링으로 되돌림
- 문제 조사 및 수정
- 향후 하드포크에서 재활성화

## Test Cases

### 테스트 케이스 1: 컬렉션 등록

```cpp
BOOST_AUTO_TEST_CASE(nft_collection_registration)
{
   ACTORS((witness1)(witness2))

   witness_create("witness1", ...);
   witness_create("witness2", ...);

   // Witness1이 컬렉션 제안
   nft_collection_register_operation op;
   op.proposer = "witness1";
   op.collection_name = "Test NFT Collection";
   op.contract_address = "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D";
   op.chain_id = 1;  // Ethereum 메인넷
   op.min_nft_count = 1;

   push_transaction(op, witness1_key);

   // 컬렉션이 생성되었지만 활성화되지 않았는지 확인
   const auto& idx = db->get_index<nft_collection_index>()
                         .indices().get<by_collection_name>();
   auto coll = idx.find("Test NFT Collection");
   BOOST_REQUIRE(coll != idx.end());
   BOOST_REQUIRE(!coll->active);
   BOOST_REQUIRE_EQUAL(coll->approved_by.size(), 1);  // 제안자가 자동 승인
}
```

### 테스트 케이스 2: 컬렉션 승인 합의

```cpp
BOOST_AUTO_TEST_CASE(nft_collection_approval)
{
   // 21명의 증인 생성
   ACTORS((w1)(w2)...(w21))

   for (auto w : witnesses)
      witness_create(w, ...);

   // 컬렉션 등록
   nft_collection_register_operation reg_op;
   reg_op.proposer = "w1";
   reg_op.collection_name = "BAYC";
   reg_op.contract_address = "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D";
   reg_op.chain_id = 1;
   push_transaction(reg_op, w1_key);

   auto coll_id = /* get collection id */;

   // 초기에는 활성화되지 않음
   BOOST_REQUIRE(!get_collection(coll_id).active);

   // 10명의 증인이 추가로 승인 (총 11/21)
   for (int i = 2; i <= 11; i++)
   {
      nft_collection_approve_operation approve_op;
      approve_op.witness_account = witnesses[i];
      approve_op.collection_id = coll_id;
      approve_op.approve = true;
      push_transaction(approve_op, witness_keys[i]);
   }

   // 이제 활성화됨 (11/21 >= 과반수)
   BOOST_REQUIRE(get_collection(coll_id).active);
}
```

### 테스트 케이스 3: NFT 증명 제출

```cpp
BOOST_AUTO_TEST_CASE(witness_nft_proof_submission)
{
   ACTORS((alice))
   witness_create("alice", ...);

   // 먼저 컬렉션 등록
   // ...

   // Alice가 Ethereum 서명 생성
   string message = "zattera:alice:nft:0xBC4C...:1234:1704672000";
   string signature = sign_ethereum_message(alice_eth_key, message);

   // 증명 제출
   witness_nft_proof_operation proof_op;
   proof_op.witness_account = "alice";
   proof_op.eth_address = "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb";
   proof_op.contract_address = "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D";
   proof_op.token_id = "1234";
   proof_op.chain_id = 1;
   proof_op.eth_signature = signature;
   proof_op.timestamp = fc::time_point_sec(1704672000);

   push_transaction(proof_op, alice_key);

   // pending 상태로 증명이 생성되었는지 확인
   const auto& proof_idx = db->get_index<witness_nft_proof_index>()
                               .indices().get<by_witness>();
   auto proof = proof_idx.find("alice");
   BOOST_REQUIRE(proof != proof_idx.end());
   BOOST_REQUIRE_EQUAL(proof->verification_status,
                      witness_nft_proof_object::pending);
}
```

### 테스트 케이스 4: 서명 검증

```cpp
BOOST_AUTO_TEST_CASE(ethereum_signature_recovery)
{
   string message = "zattera:alice:nft:0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D:1234:1704672000";

   // 알려진 Ethereum 개인 키로 서명
   string eth_privkey = "0x1234...";
   string expected_addr = "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb";

   string signature = sign_with_ethereum_key(eth_privkey, message);

   // 서명에서 주소 복구
   string recovered = recover_eth_address(message, signature);

   BOOST_REQUIRE_EQUAL(to_lowercase(recovered),
                      to_lowercase(expected_addr));
}
```

### 테스트 케이스 5: 증인 스케줄 필터링

```cpp
BOOST_AUTO_TEST_CASE(witness_schedule_nft_filtering)
{
   ACTORS((alice)(bob)(charlie))

   // 증인 생성
   witness_create("alice", ...);
   witness_create("bob", ...);
   witness_create("charlie", ...);

   // Alice는 검증된 NFT 증명 있음
   create_verified_nft_proof("alice", ...);

   // Bob은 만료된 증명 있음
   create_expired_nft_proof("bob", ...);

   // Charlie는 증명 없음

   // 증인 스케줄 업데이트
   db->update_witness_schedule();

   const auto& schedule = db->get_witness_schedule_object();
   const auto& active = schedule.current_shuffled_witnesses;

   // Alice만 스케줄에 있어야 함
   BOOST_REQUIRE(std::find(active.begin(), active.end(), "alice") != active.end());
   BOOST_REQUIRE(std::find(active.begin(), active.end(), "bob") == active.end());
   BOOST_REQUIRE(std::find(active.begin(), active.end(), "charlie") == active.end());
}
```

### 테스트 케이스 6: 증명 만료

```cpp
BOOST_AUTO_TEST_CASE(nft_proof_expiration)
{
   ACTORS((alice))
   witness_create("alice", ...);

   // 검증된 증명 생성
   create_verified_nft_proof("alice", ...);

   // 초기에는 유효함
   BOOST_REQUIRE(db->validate_witness_nft_ownership("alice"));

   // 8일 빨리 감기
   generate_blocks(db->head_block_time() + fc::days(8));

   // 이제 만료됨 (7일 제한)
   BOOST_REQUIRE(!db->validate_witness_nft_ownership("alice"));
}
```

### 테스트 케이스 7: 증인 간 NFT 양도

```cpp
BOOST_AUTO_TEST_CASE(nft_transfer_between_witnesses)
{
   ACTORS((alice)(bob))
   witness_create("alice", ...);
   witness_create("bob", ...);

   // 컬렉션 등록
   auto collection_id = register_and_approve_collection("BAYC", "0xBC4C...", 1);

   // Alice가 NFT #1234에 대한 증명 제출
   string alice_message = "zattera:alice:nft:0xBC4C...:1234:1704672000";
   string alice_signature = sign_ethereum_message(alice_eth_key, alice_message);

   witness_nft_proof_operation alice_proof;
   alice_proof.witness_account = "alice";
   alice_proof.eth_address = "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb";
   alice_proof.contract_address = "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D";
   alice_proof.token_id = "1234";
   alice_proof.chain_id = 1;
   alice_proof.eth_signature = alice_signature;
   alice_proof.timestamp = fc::time_point_sec(1704672000);

   push_transaction(alice_proof, alice_key);

   // Alice의 증명 생성 확인
   const auto& proof_idx = db->get_index<witness_nft_proof_index>()
                               .indices().get<by_witness>();
   auto alice_proof_obj = proof_idx.find("alice");
   BOOST_REQUIRE(alice_proof_obj != proof_idx.end());
   BOOST_REQUIRE_EQUAL(alice_proof_obj->verification_status,
                      witness_nft_proof_object::pending);

   // Alice에 대한 오라클 검증 시뮬레이션
   db->modify(*alice_proof_obj, [&](witness_nft_proof_object& p) {
      p.verification_status = witness_nft_proof_object::verified;
      p.verification_details = "Oracle confirmed";
   });

   // Alice는 이제 블록 생성 자격 있음
   BOOST_REQUIRE(db->validate_witness_nft_ownership("alice"));

   // NFT 양도: Bob이 동일한 NFT에 대한 증명 제출
   string bob_message = "zattera:bob:nft:0xBC4C...:1234:1704672100";
   string bob_signature = sign_ethereum_message(bob_eth_key, bob_message);

   witness_nft_proof_operation bob_proof;
   bob_proof.witness_account = "bob";
   bob_proof.eth_address = "0x9876543210fedcba9876543210fedcba98765432";
   bob_proof.contract_address = "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D";
   bob_proof.token_id = "1234";  // 동일한 NFT
   bob_proof.chain_id = 1;
   bob_proof.eth_signature = bob_signature;
   bob_proof.timestamp = fc::time_point_sec(1704672100);

   push_transaction(bob_proof, bob_key);

   // Alice의 증명 무효화 확인
   alice_proof_obj = proof_idx.find("alice");
   BOOST_REQUIRE(alice_proof_obj != proof_idx.end());
   BOOST_REQUIRE_EQUAL(alice_proof_obj->verification_status,
                      witness_nft_proof_object::failed);
   BOOST_REQUIRE(alice_proof_obj->verification_details.find("transferred to bob")
                 != string::npos);

   // Alice는 더 이상 자격 없음
   BOOST_REQUIRE(!db->validate_witness_nft_ownership("alice"));

   // Bob의 증명은 대기 중
   auto bob_proof_obj = proof_idx.find("bob");
   BOOST_REQUIRE(bob_proof_obj != proof_idx.end());
   BOOST_REQUIRE_EQUAL(bob_proof_obj->verification_status,
                      witness_nft_proof_object::pending);

   // Bob은 아직 자격 없음 (오라클 대기 중)
   BOOST_REQUIRE(!db->validate_witness_nft_ownership("bob"));

   // Bob에 대한 오라클 검증 시뮬레이션
   db->modify(*bob_proof_obj, [&](witness_nft_proof_object& p) {
      p.verification_status = witness_nft_proof_object::verified;
      p.verification_details = "Oracle confirmed";
   });

   // Bob은 이제 자격 있음
   BOOST_REQUIRE(db->validate_witness_nft_ownership("bob"));
}
```

### 테스트 케이스 8: 허위 NFT 요구 감지

```cpp
BOOST_AUTO_TEST_CASE(false_nft_claim)
{
   ACTORS((alice)(malicious))
   witness_create("alice", ...);
   witness_create("malicious", ...);

   // Alice는 검증된 증명 있음
   auto collection_id = register_and_approve_collection("BAYC", "0xBC4C...", 1);
   create_verified_nft_proof("alice", "0x742d35Cc...", "0xBC4C...", "1234", 1);

   BOOST_REQUIRE(db->validate_witness_nft_ownership("alice"));

   // 악의적인 증인이 Alice의 NFT를 허위로 요구
   string fake_message = "zattera:malicious:nft:0xBC4C...:1234:1704672200";
   string fake_signature = sign_ethereum_message(malicious_eth_key, fake_message);

   witness_nft_proof_operation fake_proof;
   fake_proof.witness_account = "malicious";
   fake_proof.eth_address = "0xbadbadbadbadbadbadbadbadbadbadbadbadbad";
   fake_proof.contract_address = "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D";
   fake_proof.token_id = "1234";  // Alice의 NFT
   fake_proof.chain_id = 1;
   fake_proof.eth_signature = fake_signature;
   fake_proof.timestamp = fc::time_point_sec(1704672200);

   push_transaction(fake_proof, malicious_key);

   // Alice의 증명 무효화됨 (일시적으로)
   const auto& proof_idx = db->get_index<witness_nft_proof_index>()
                               .indices().get<by_witness>();
   auto alice_proof = proof_idx.find("alice");
   BOOST_REQUIRE_EQUAL(alice_proof->verification_status,
                      witness_nft_proof_object::failed);

   // 악의적인 증명은 대기 중
   auto mal_proof = proof_idx.find("malicious");
   BOOST_REQUIRE_EQUAL(mal_proof->verification_status,
                      witness_nft_proof_object::pending);

   // 오라클 검증 시뮬레이션 - 허위 요구 감지
   db->modify(*mal_proof, [&](witness_nft_proof_object& p) {
      p.verification_status = witness_nft_proof_object::failed;
      p.verification_details = "Oracle: actual owner is 0x742d35Cc...";
   });

   // 악의적인 증인은 자격 없음
   BOOST_REQUIRE(!db->validate_witness_nft_ownership("malicious"));

   // Alice는 자격을 되찾기 위해 증명을 재제출해야 함
   BOOST_REQUIRE(!db->validate_witness_nft_ownership("alice"));
}
```

## Reference Implementation

기능 브랜치에서 사용 가능:

- Branch: `feature/nft-witness-verification`
- 주요 파일:
  - `src/core/protocol/include/steem/protocol/steem_operations.hpp`
  - `src/core/chain/include/steem/chain/nft_verification_objects.hpp`
  - `src/core/chain/nft_evaluator.cpp`
  - `src/core/chain/database.cpp` (증인 스케줄)
  - `src/plugins/nft_oracle/` (ZEP-3 참조)
  - `tests/tests/nft_witness_tests.cpp`

## Security Considerations

### 암호화 보안

**서명 검증**
- 위험: 서명 위조
- 완화: 검증된 secp256k1 복구 사용
- 라이브러리: fc::ecc, libsecp256k1

**재생 공격**
- 위험: 이전 서명 재사용
- 완화: 서명된 메시지에 타임스탬프 포함
- 검증: 24시간 이상 된 증명 거부

### 오라클 보안

**오라클 손상**
- 위험: 악의적인 오라클이 가짜 증명 승인
- 완화:
  - 증인이 제어하는 오라클
  - 오픈 소스 오라클 코드
  - 다중 오라클 검증 (향후)
  - 이상 승인 모니터링

**RPC 엔드포인트 조작**
- 위험: 가짜 RPC가 거짓 소유권 데이터 반환
- 완화:
  - 신뢰할 수 있는 RPC 제공자 사용
  - 다중 RPC 엔드포인트
  - 제공자 간 결과 비교
  - 블록 익스플로러와 교차 검증

**체인 재구성**
- 위험: 재구성에서 NFT 소유권 변경
- 완화: 확인 깊이 대기 (예: 12블록)

### 경제적 보안

**NFT 전송 공격**
- 위험: 잠깐 NFT 소유, 증명 제출, 전송
- 완화:
  - 7일 증명 유효성으로 지속적인 소유권 필요
  - 빈번한 재검증으로 전송 감지
  - NFT 스테이킹 (향후 개선)

**증인 간 NFT 양도**
- 위험: 증인이 제한을 우회하기 위해 다른 증인에게 NFT 판매
- 동작: 이전 증인의 증명 자동 무효화
- 영향:
  - 이전 증인은 새로운 요구 시 즉시 자격 상실
  - 새 증인은 오라클 검증 후에만 자격 획득
  - 해당 NFT로 두 증인 모두 블록을 생성할 수 없는 일시적인 공백 발생
- 보호: 오라클 검증이 합법적인 소유권 이전 보장

**허위 NFT 요구 공격**
- 위험: 악의적인 증인이 다른 증인의 NFT를 소유하지 않고 요구
- 완화:
  - 유효하지 않은 증명은 오라클 검증 실패
  - 원래 증인의 증명은 서명 검증 후에만 무효화됨
  - 공격자는 아무 이득 없음 (오라클이 가짜 요구 거부)
  - 실패한 증명 시도가 모니터링을 위해 기록됨
- 트레이드오프: 원래 증인 증명이 실패로 표시되는 짧은 기간
  - 기간: 오라클 검증 완료까지 (~1-2분)
  - 영향: 블록 생성 스케줄에 대한 최소한의 중단

**증인 담합**
- 위험: 악의적인 증인이 가짜 컬렉션 승인
- 완화:
  - 11/21 과반수 필요
  - 공개 정당화 및 투표
  - 커뮤니티 감독
  - 제거를 위한 거버넌스 메커니즘

**DoS 공격**
- 위험: 스팸 증명 제출
- 완화:
  - 증인당 속도 제한
  - Operation 수수료
  - 활성 증인만 제출 가능

### 운영 보안

**키 관리**
- 위험: Ethereum 개인 키 손상
- 완화:
  - NFT 소유권과 블록 서명을 위한 별도 키
  - 하드웨어 지갑 지원
  - 키 회전 절차

**RPC 가용성**
- 위험: RPC 엔드포인트 오프라인
- 완화:
  - 폴백 엔드포인트
  - 로컬 아카이브 노드
  - 우아한 성능 저하

## Future Enhancements

### 1. 다중 NFT 요구 사항

증인이 여러 컬렉션의 NFT를 보유하도록 요구:

```cpp
struct multi_nft_requirement
{
   vector<nft_collection_id_type> required_collections;
   uint32_t min_collections;  // "Y개 중 X개" 로직
};
```

### 2. NFT 스테이킹

향상된 보안을 위해 스마트 컨트랙트에 NFT 잠금:

```cpp
struct nft_staking_contract
{
   string contract_address;
   uint32_t min_stake_duration;  // 최소 잠금 기간
};
```

### 3. 크로스체인 브리지

오라클 대신 네이티브 브리지 사용:

```cpp
struct bridge_verification
{
   string bridge_contract;
   uint32_t confirmation_blocks;
};
```

### 4. 동적 NFT 값

희귀도, 가격 등에 따라 NFT에 다르게 가중치 부여:

```cpp
struct nft_weight_criteria
{
   asset min_floor_price;
   map<string, uint32_t> trait_weights;
};
```

### 5. 위임된 소유권

NFT 소유자가 증인 권한을 위임할 수 있도록 허용:

```cpp
struct nft_delegation_operation
{
   string nft_owner_eth_address;
   account_name_type delegated_witness;
   time_point_sec expiration;
   string eth_signature;
};
```

### 6. 다중 오라클 합의

여러 오라클이 동의하도록 요구:

```cpp
struct oracle_consensus_config
{
   uint32_t min_oracle_agreements;  // 예: 5개 중 3개
   vector<account_name_type> trusted_oracles;
};
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
