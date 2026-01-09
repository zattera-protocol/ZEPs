---
zep: 8
title: 증인 서명 기반 크로스체인 ZBD 출금
description: 증인의 EIP-712 서명을 통해 사용자가 직접 외부 체인에서 ZBD를 출금하는 시스템
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2025-12-31
---

## 초록

본 ZEP는 Zattera 블록체인의 ZBD(Zattera Backed Dollars)를 외부 EVM 호환 체인의 스테이블코인(USDC, DAI, USDT 등)으로 출금할 수 있는 단방향 브리지 시스템을 제안합니다. 증인이 사용자의 출금 요청에 대해 EIP-712 서명을 생성하면, 사용자가 해당 서명을 사용하여 외부 체인의 스마트 컨트랙트에서 직접 자산을 인출합니다. 증인은 외부 체인에 스테이블코인을 예치하고, 서명 제공의 대가로 ZBD를 ZP(VESTS)로 교환받습니다 (3.5일 후, 평균 가격 적용).

**핵심 특징**:
- 증인은 서명만 생성 (가스비 부담 없음)
- 사용자가 직접 출금 실행 (완전한 셀프 커스터디)
- 교환비율 기반 경쟁 (낮은 교환비율 = 높은 선택 확률)
- 증인 보상은 ZP(VESTS)로 변환 (3.5일 지연, 평균 가격 사용)
- 가격 조작 방지 (`convert_request_object`와 동일한 메커니즘)
- 나머지 ZBD는 소각되어 네트워크 디플레이션 기여
- 이중 출금 방지 (스마트 컨트랙트 검증)
- 증인 예치금 시스템

**중요**: 본 시스템은 **출금 전용(Withdrawal-Only)**입니다. 외부 체인에서 Zattera로의 입금 기능은 제공하지 않습니다.

## 동기

### ZBD의 현재 한계

Zattera의 ZBD는 내부 스테이블코인으로 다음과 같은 제약이 있습니다:

1. **제한된 유동성**: Zattera 체인 내부에서만 사용 가능
2. **현금화 어려움**: 법정화폐나 외부 자산으로 직접 전환 불가
3. **외부 DeFi 접근 불가**: 이더리움 DeFi 프로토콜 사용 불가

### 단방향 브리지의 이점

양방향 브리지 대신 **출금 전용** 브리지를 채택함으로써:

#### 1. 보안성 대폭 향상
- 가짜 입금 공격 벡터 완전 제거
- 이중 입금 위험 없음
- 코드 복잡도 50% 감소

#### 2. 사용자 경험 개선
- 사용자가 원하는 시간에 출금 실행
- 가스 가격 낮을 때 트랜잭션 가능
- 완전한 자금 통제권

#### 3. 증인 운영 간소화
- 가스비 부담 없음
- 서명만 생성하면 됨
- 독립적 운영 가능

## 명세

### 1. 시스템 개요

#### 출금 흐름

```
1. 사용자: withdraw 100 ZBD → Ethereum (컨트랙트 0x1234... 지정)
   → Zattera: 사용자 계정에서 100 ZBD 차감 + virtual supply 100 ZBD 즉시 소각
   → 출금 요청 ID: 12345
   → 상태: queued (증인 할당 대기)
   → Plugin이 매 블록마다 할인율 기준으로 증인 선택
   → Alice (10%), Bob (30%), Carol (60%) 중 가중 랜덤
   → Alice 선택 확률: 90 / (90+70+40) = 45%
   → Bob 선택 확률: 70 / 200 = 35%
   → Carol 선택 확률: 40 / 200 = 20%
   → Alice 선택됨 (교환비율 10%)
   → 상태: queued → pending (Alice에게 할당됨)

2. Alice (5분 내):
   → EIP-712 서명 생성 (컨트랙트 0x1234... 기준)
   → Zattera에 서명 제출
   → 상태: pending → signed (서명 완료, 사용자 실행 대기)

3. 사용자 (언제든지):
   → Alice의 서명으로 0x1234... 컨트랙트 호출 (토큰: USDC)
   → 컨트랙트: EIP-712 서명 검증 (Alice가 서명했는지 확인)
   → Alice 예치금(USDC) 100 USDC 차감
   → 사용자에게 100 USDC 전송 ✅
   → 이중 출금 방지 플래그 설정

4. Alice (유효기간 내, 예: 24시간):
   → Alice의 witness plugin이 Ethereum RPC 노드를 통해 출금 실행 확인
   → 컨트랙트 조회: getWithdrawalStatus(12345) 호출
   → 반환값: (processed=true, blockNumber=18234567)
   → 해당 블록에서 Withdrawn 이벤트 필터링 → tx_hash 획득
   → Zattera에 완료 보고 제출 (tx_hash 포함)
   → Virtual supply +10 ZBD 재생성 (Alice 보상분)
   → Alice 계정에 10 ZBD 지급
   → Alice의 10 ZBD → ZP 변환 요청 생성 (3.5일 후 실행) ⏱
   → 나머지 90 ZBD는 이미 요청 시 소각됨 (영구 소각 완료)
   → 상태: completed

5. 3.5일 후 (자동):
   → Zattera 블록체인이 변환 요청 처리
   → 과거 84시간 평균 가격(median price) 사용
   → Alice의 10 ZBD 소각 → ZTR → ZP 변환
   → Alice가 ZP 획득 ✅
   → 투표권 증가
```

#### 역할 분담

| 주체 | 책임 | 보상/비용 |
|------|------|----------|
| **사용자** | - ZBD 출금 요청<br>- 외부 체인 가스비 지불<br>- 출금 트랜잭션 실행 | - ZBD 전액 차감 + virtual supply 즉시 소각<br>- 완료 시 증인 보상분만 supply 재생성<br>- 가스비 지불 (ETH) |
| **증인** | - 외부 체인 RPC 노드 운영/접근<br>- 외부 체인에 스테이블코인 예치 (USDC, DAI, USDT 등)<br>- EIP-712 서명 생성<br>- 서명 제출<br>- 외부 체인 실행 모니터링<br>- 완료 보고 제출 | - 완료 시 교환비율만큼 ZBD 수령 (supply 재생성)<br>- 자동으로 ZBD → ZP 변환 요청 생성 (3.5일 후)<br>- 평균 가격으로 공정한 변환<br>- 투표권(Voting Power) 증가<br>- RPC 노드 운영 비용 |
| **네트워크** | - Virtual supply 관리<br>- 요청 시 즉시 소각<br>- 완료 시 보상분 재생성<br>- 디플레이션 효과 | - 요청 시: -100% ZBD 즉시 소각<br>- 완료 시: +교환비율% ZBD 재생성<br>- Convert 시: 재생성된 ZBD 소각 후 ZP 생성<br>- 최종 결과: **요청 ZBD 100% 소각 보장** |
| **스마트 컨트랙트** | - 서명 검증<br>- 증인 예치금 관리<br>- 이중 출금 방지<br>- Withdrawn 이벤트 발행 | - 가스비 없음 |

### 2. 증인 설정

#### 2.1 출금 컨트랙트 화이트리스트 (증인 합의)

출금 컨트랙트는 증인 합의를 통해 승인되어야 사용 가능합니다. 이는 악의적인 컨트랙트로부터 사용자를 보호합니다.

**Approved Contract Database Object**:

```cpp
class withdraw_dollar_contract_object : public object<...>
{
    ZATTERA_STD_ALLOCATOR_CONSTRUCTOR(withdraw_dollar_contract_object)

    id_type           id;
    uint32_t          chain_id;              // EVM 체인 ID
    shared_string     contract_address;      // 승인된 출금 컨트랙트 주소
    time_point_sec    approved_at;           // 승인 시각

    // 증인 승인 추적
    flat_set<account_name_type> approving_witnesses;  // 승인한 증인 목록
    uint16_t          approval_count = 0;             // 승인 수
};

typedef multi_index_container<
    withdraw_dollar_contract_object,
    indexed_by<
        ordered_unique< tag< by_chain_contract >,
            composite_key< withdraw_dollar_contract_object,
                member< withdraw_dollar_contract_object, uint32_t, &withdraw_dollar_contract_object::chain_id >,
                member< withdraw_dollar_contract_object, shared_string, &withdraw_dollar_contract_object::contract_address >
            >
        >
    >,
    allocator< withdraw_dollar_contract_object >
> withdraw_dollar_contract_index;
```

**Contract Approval Operation**:

```cpp
struct approve_withdraw_dollar_contract_operation : public base_operation
{
    account_name_type witness;
    uint32_t          chain_id;
    string            contract_address;
    bool              approve = true;      // false = 승인 철회

    extensions_type   extensions;

    void validate() const
    {
        FC_ASSERT(is_valid_account_name(witness), "Invalid witness");
        FC_ASSERT(chain_id > 0, "Invalid chain ID");
        FC_ASSERT(contract_address.size() == 42 && contract_address.substr(0, 2) == "0x",
                  "Invalid contract address");
    }

    void get_required_active_authorities(flat_set<account_name_type>& a) const
    {
        a.insert(witness);
    }
};
```

**Approval Evaluator**:

```cpp
void approve_withdraw_dollar_contract_evaluator::do_apply(
    const approve_withdraw_dollar_contract_operation& o
)
{
    const auto& db = this->db();
    const auto& witness = db.get_witness(o.witness);

    FC_ASSERT(witness.schedule != witness_object::none,
              "Only active witnesses can approve contracts");

    const auto& idx = db.get_index<withdraw_dollar_contract_index>()
                        .indices().get<by_chain_contract>();
    auto itr = idx.find(boost::make_tuple(o.chain_id, o.contract_address));

    if (o.approve)
    {
        // 승인
        if (itr == idx.end())
        {
            // 신규 컨트랙트 승인 제안
            db.create<withdraw_dollar_contract_object>([&](auto& obj) {
                obj.chain_id = o.chain_id;
                obj.contract_address = o.contract_address;
                obj.approving_witnesses.insert(o.witness);
                obj.approval_count = 1;
                obj.approved_at = db.head_block_time();
            });
        }
        else
        {
            // 기존 컨트랙트에 승인 추가
            db.modify(*itr, [&](auto& obj) {
                if (obj.approving_witnesses.insert(o.witness).second)
                {
                    obj.approval_count += 1;
                }
            });
        }
    }
    else
    {
        // 승인 철회
        if (itr != idx.end())
        {
            db.modify(*itr, [&](auto& obj) {
                if (obj.approving_witnesses.erase(o.witness))
                {
                    obj.approval_count -= 1;
                }
            });

            // 승인이 과반 미만이면 삭제
            const auto& witness_schedule = db.get_witness_schedule_object();
            uint32_t required_approvals = witness_schedule.current_shuffled_witnesses.size() / 2 + 1;

            if (itr->approval_count < required_approvals)
            {
                db.remove(*itr);
            }
        }
    }
}
```

**승인 기준**:
- 활성 증인의 **과반수(50% + 1)** 이상 승인 필요
- 예: 21명 증인 → 최소 11명 승인 필요
- 승인이 과반 미만으로 떨어지면 자동 삭제

#### 2.2 증인별 출금 서비스 설정

승인된 컨트랙트에 대해 각 증인이 서명 주소와 교환비율을 설정합니다.

**Database Object**:

```cpp
class withdraw_dollar_provider_object : public object<...>
{
    ZATTERA_STD_ALLOCATOR_CONSTRUCTOR(withdraw_dollar_provider_object)

    id_type           id;
    account_name_type witness;
    uint32_t          chain_id;              // EVM 체인 ID (1=Ethereum, 137=Polygon)
    shared_string     contract_address;      // 출금 컨트랙트 주소 (0x...)
    shared_string     signer_address;        // EIP-712 서명에 사용할 주소 (0x...)
    uint16_t          exchange_rate;         // 교환비율 (ZATTERA_100_PERCENT = 10000)
    flat_set<shared_string> supported_tokens;  // 지원하는 ERC-20 토큰 주소 목록
    bool              is_active = true;
    time_point_sec    registered_at;
    time_point_sec    last_updated;

    // 예치금 보고 (witness => token => amount in ZBD equivalent)
    // 증인이 주기적으로 외부 체인 예치금 상태를 보고
    flat_map<shared_string, share_type> reported_deposits;  // 토큰별 예치금 (ZBD 단위, 3 decimals)
    flat_map<shared_string, share_type> reserved_deposits;  // 토큰별 예약된 예치금 (pending 요청 금액 합계)
    time_point_sec    deposits_last_reported;                // 마지막 예치금 보고 시각

    // 누적 통계
    asset             total_withdrawal_amount;    // 총 처리한 출금액 (교환비율 적용 전)
    asset             total_burned_amount;        // 총 소각된 ZBD (교환되지 않은 금액)
    uint64_t          total_withdrawal_count = 0; // 총 처리 건수
    time_point_sec    last_withdrawal_time;       // 마지막 처리 시각
};

typedef multi_index_container<
    withdraw_dollar_provider_object,
    indexed_by<
        ordered_unique< tag< by_witness_contract >,
            composite_key< withdraw_dollar_provider_object,
                member< withdraw_dollar_provider_object, account_name_type, &withdraw_dollar_provider_object::witness >,
                member< withdraw_dollar_provider_object, uint32_t, &withdraw_dollar_provider_object::chain_id >,
                member< withdraw_dollar_provider_object, shared_string, &withdraw_dollar_provider_object::contract_address >
            >
        >
    >,
    allocator< withdraw_dollar_provider_object >
> withdraw_dollar_provider_index;
```

**Operation**:

```cpp
struct update_withdraw_dollar_provider_operation : public base_operation
{
    account_name_type      witness;
    uint32_t               chain_id;           // 1 = Ethereum, 137 = Polygon, etc.
    string                 contract_address;   // 0x1234...
    string                 signer_address;     // 0x5678... (서명에 사용할 주소)
    uint16_t               exchange_rate;      // 6000 = 60% (ZATTERA_100_PERCENT = 10000)
    vector<string>         supported_tokens;   // 지원하는 토큰 주소 목록 (0xAAA..., 0xBBB...)

    extensions_type        extensions;

    void validate() const
    {
        FC_ASSERT(is_valid_account_name(witness), "Invalid witness name");
        FC_ASSERT(chain_id > 0, "Invalid chain ID");
        FC_ASSERT(contract_address.size() == 42 && contract_address.substr(0, 2) == "0x",
                  "Invalid contract address");
        FC_ASSERT(signer_address.size() == 42 && signer_address.substr(0, 2) == "0x",
                  "Invalid signer address");
        FC_ASSERT(exchange_rate >= ZATTERA_MIN_EXCHANGE_RATE && exchange_rate <= ZATTERA_100_PERCENT,
                  "Exchange rate must be between ZATTERA_MIN_EXCHANGE_RATE and ZATTERA_100_PERCENT");
        FC_ASSERT(!supported_tokens.empty(), "Must support at least one token");
        for (const auto& token : supported_tokens) {
            FC_ASSERT(token.size() == 42 && token.substr(0, 2) == "0x",
                      "Invalid token address: ${t}", ("t", token));
        }
    }

    void get_required_active_authorities(flat_set<account_name_type>& a) const
    {
        a.insert(witness);
    }
};
```

**Evaluator**:

```cpp
void update_withdraw_dollar_provider_evaluator::do_apply(
    const update_withdraw_dollar_provider_operation& o
)
{
    const auto& db = this->db();
    const auto& witness = db.get_witness(o.witness);

    FC_ASSERT(witness.schedule != witness_object::none,
              "Only active witnesses can register");

    // 1. 컨트랙트가 승인되었는지 확인
    const auto& approved_idx = db.get_index<withdraw_dollar_contract_index>()
                                  .indices().get<by_chain_contract>();
    auto approved_itr = approved_idx.find(boost::make_tuple(o.chain_id, o.contract_address));

    FC_ASSERT(approved_itr != approved_idx.end(),
              "Contract ${c} is not approved by witness consensus",
              ("c", o.contract_address));

    // 과반수 승인 확인
    const auto& witness_schedule = db.get_witness_schedule_object();
    uint32_t required_approvals = witness_schedule.current_shuffled_witnesses.size() / 2 + 1;

    FC_ASSERT(approved_itr->approval_count >= required_approvals,
              "Contract does not have majority approval (${current}/${required})",
              ("current", approved_itr->approval_count)("required", required_approvals));

    // 2. 증인별 설정 등록/업데이트
    const auto& idx = db.get_index<withdraw_dollar_provider_index>()
                        .indices().get<by_witness_contract>();
    auto itr = idx.find(boost::make_tuple(o.witness, o.chain_id, o.contract_address));

    if (itr == idx.end())
    {
        // 신규 등록
        db.create<withdraw_dollar_provider_object>([&](auto& obj) {
            obj.witness = o.witness;
            obj.chain_id = o.chain_id;
            obj.contract_address = o.contract_address;
            obj.signer_address = o.signer_address;
            obj.exchange_rate = o.exchange_rate;

            // 지원하는 토큰 목록 저장
            obj.supported_tokens.clear();
            for (const auto& token : o.supported_tokens) {
                obj.supported_tokens.insert(token);
            }

            obj.is_active = true;
            obj.registered_at = db.head_block_time();
            obj.last_updated = db.head_block_time();

            // 누적 통계 초기화
            obj.total_withdrawal_amount = asset(0, DOLLAR_SYMBOL);
            obj.total_burned_amount = asset(0, DOLLAR_SYMBOL);
            obj.total_withdrawal_count = 0;
        });
    }
    else
    {
        // 기존 등록 업데이트
        db.modify(*itr, [&](auto& obj) {
            obj.signer_address = o.signer_address;
            obj.exchange_rate = o.exchange_rate;

            // 지원하는 토큰 목록 업데이트
            obj.supported_tokens.clear();
            for (const auto& token : o.supported_tokens) {
                obj.supported_tokens.insert(token);
            }

            obj.is_active = true;
            obj.last_updated = db.head_block_time();
        });
    }
}
```

#### 2.3 예치금 보고

증인은 주기적으로 외부 체인의 실제 예치금 상태를 Zattera 체인에 보고해야 합니다. 이를 통해 증인 선택 시 충분한 예치금이 있는 증인만 선택됩니다.

**Operation**:

```cpp
struct report_withdraw_dollar_deposits_operation : public base_operation
{
    account_name_type      witness;
    uint32_t               chain_id;
    string                 contract_address;
    map<string, asset>     deposits;  // 토큰별 예치금 (ZBD equivalent)

    extensions_type        extensions;

    void validate() const
    {
        FC_ASSERT(is_valid_account_name(witness), "Invalid witness");
        FC_ASSERT(chain_id > 0, "Invalid chain ID");
        FC_ASSERT(contract_address.size() == 42 && contract_address.substr(0, 2) == "0x",
                  "Invalid contract address");
        FC_ASSERT(!deposits.empty(), "Deposits cannot be empty");

        for (const auto& [token, amount] : deposits)
        {
            FC_ASSERT(token.size() == 42 && token.substr(0, 2) == "0x",
                      "Invalid token address: ${t}", ("t", token));
            FC_ASSERT(amount.symbol == DOLLAR_SYMBOL, "Must be ZBD asset");
            FC_ASSERT(amount.amount >= 0, "Deposit cannot be negative");
        }
    }

    void get_required_active_authorities(flat_set<account_name_type>& a) const
    {
        a.insert(witness);
    }
};
```

**Evaluator**:

```cpp
void report_withdraw_dollar_deposits_evaluator::do_apply(
    const report_withdraw_dollar_deposits_operation& o
)
{
    const auto& db = this->db();
    const auto& witness = db.get_witness(o.witness);

    FC_ASSERT(witness.schedule != witness_object::none,
              "Only active witnesses can report deposits");

    // 등록된 컨트랙트 찾기
    const auto& idx = db.get_index<withdraw_dollar_provider_index>()
                        .indices().get<by_witness_contract>();
    auto itr = idx.find(boost::make_tuple(o.witness, o.chain_id, o.contract_address));

    FC_ASSERT(itr != idx.end(),
              "Witness is not registered for this contract");

    // 예치금 보고 업데이트
    db.modify(*itr, [&](auto& obj) {
        obj.reported_deposits.clear();

        for (const auto& [token, amount] : o.deposits)
        {
            // 지원하는 토큰만 보고 가능
            FC_ASSERT(obj.supported_tokens.find(token) != obj.supported_tokens.end(),
                      "Token ${t} is not in supported tokens list", ("t", token));

            obj.reported_deposits[token] = amount.amount;
        }

        obj.deposits_last_reported = db.head_block_time();
    });
}
```

**교환비율 예시**:
- 1000 (10%) - 100 ZBD 출금 시 → 10 ZBD를 ZP로 교환, 90 ZBD 소각 (네트워크에 가장 유리)
- 6000 (60%) - 100 ZBD 출금 시 → 60 ZBD를 ZP로 교환, 40 ZBD 소각
- 10000 (100%) - 100 ZBD 출금 시 → 100 ZBD를 ZP로 교환, 소각 없음 (네트워크에 기여 없음)

**상수 정의**:
```cpp
// zattera/protocol/config.hpp
#define ZATTERA_100_PERCENT                           10000
#define ZATTERA_MIN_EXCHANGE_RATE                     1000   // 10%
#define ZATTERA_MAX_EXCHANGE_RATE                     10000  // 100%

// Withdraw Dollar Plugin 상수
#define ZATTERA_WITHDRAW_DOLLAR_MAX_ASSIGN_PER_BLOCK  100    // 블록당 최대 queued→pending 할당 개수
#define ZATTERA_WITHDRAW_DOLLAR_MAX_SIGN_PER_BLOCK    50     // 블록당 최대 서명 생성 개수
#define ZATTERA_WITHDRAW_DOLLAR_MAX_CHECK_PER_MINUTE  20     // 분당 최대 완료 확인 개수 (RPC 호출)
#define ZATTERA_WITHDRAW_DOLLAR_SIGNATURE_DEADLINE    300    // 5분 (서명 제출 마감시간, 초)
#define ZATTERA_WITHDRAW_DOLLAR_DEPOSIT_VALIDITY      3600   // 1시간 (예치금 보고 유효시간, 초)
```

**증인 선택 로직 (가중 랜덤)**:
- 사용자가 지정한 컨트랙트를 지원하는 증인 중에서 **확률적으로 선택**
- 가중치 계산: `weight = ZATTERA_100_PERCENT - exchange_rate`
- 낮은 교환비율 = 높은 가중치 = 높은 선택 확률

**예시**:
```
Alice: 1000 (10%) → 가중치 9000 → 선택 확률 45%
Bob:   3000 (30%) → 가중치 7000 → 선택 확률 35%
Carol: 6000 (60%) → 가중치 4000 → 선택 확률 20%

총 가중치: 9000 + 7000 + 4000 = 20000
```

**장점**:
- 여러 증인에게 기회 분산
- 낮은 교환비율일수록 더 많은 기회
- 네트워크 디플레이션 최대화 인센티브

#### 2.4 할인율 (Discount Rate)

사용자는 출금 요청 시 **할인율**을 설정하여 더 빠른 처리를 받을 수 있습니다.

**개념**:
- 사용자가 Zattera 체인에서 차감되는 ZBD 금액과 외부 체인에서 실제로 받을 금액의 차이
- 할인된 금액은 증인에게 인센티브로 제공되어 우선 처리 동기 부여

**작동 방식**:

```
요청 금액: 100.000 ZBD
할인율: 5% (500)

Zattera 체인 차감: -100.000 ZBD
외부 체인 수령: 95달러 상당의 토큰 (USDC, DAI 등)
할인 금액: 5.000 ZBD (증인 인센티브)
```

**우선순위 처리**:
- 증인은 할인율이 높은 요청부터 처리 (`by_witness_status` 인덱스, 할인율 우선순위 정렬)
- `on_apply_block()`: 블록당 최대 50개 처리, 할인율 높은 순
- `report_completions_periodically()`: 분당 최대 20개 확인, 할인율 높은 순

**제약 사항**:
- 최소 할인율: 0% (기본값, 할인 없음)
- 최대 할인율: 100% (10000) - `ZATTERA_100_PERCENT`
- 할인율은 operation 생성 시 설정되며 변경 불가

**예시**:

| 할인율 | Zattera 차감 | 외부 체인 수령 | 증인 인센티브 | 처리 우선순위 |
|--------|--------------|----------------|---------------|---------------|
| 0%     | 100 ZBD      | $100           | $0            | 낮음 (기본)   |
| 1%     | 100 ZBD      | $99            | $1            | ↑             |
| 5%     | 100 ZBD      | $95            | $5            | ↑             |
| 10%    | 100 ZBD      | $90            | $10           | 높음          |

**Note**: 할인율은 증인의 교환비율(exchange rate)과는 별개입니다. 교환비율은 ZBD를 ZP로 교환하는 비율이며, 할인율은 사용자가 빠른 처리를 위해 증인에게 제공하는 인센티브입니다.

### 3. 출금 프로세스

#### 3.1 사용자 출금 요청

**Database Objects**:

```cpp
enum withdraw_dollar_status
{
    queued,       // 증인 할당 대기 (할인율 기준으로 정렬됨)
    pending,      // 증인 할당됨, 서명 대기
    signed,       // 서명 완료, 사용자 실행 대기
    completed,    // 외부 체인 실행 완료
    expired       // 타임아웃
};

class withdraw_dollar_request_object : public object<...>
{
    ZATTERA_STD_ALLOCATOR_CONSTRUCTOR(withdraw_dollar_request_object)

    id_type               id;
    uint64_t              request_id;          // 고유 ID (이중 출금 방지)
    account_name_type     requester;
    asset                 amount;              // ZBD
    uint32_t              chain_id;
    shared_string         contract_address;    // 사용자 지정 출금 컨트랙트 주소
    shared_string         token;               // ERC-20 토큰 주소 (USDC, DAI, USDT 등)
    shared_string         recipient;           // 0x...
    uint16_t              discount_rate;       // 할인율 (0~10000)

    withdraw_dollar_status     status;
    account_name_type     assigned_witness;    // 선택된 증인
    shared_string         witness_signature;   // EIP-712 서명
    asset                 exchange_amount;     // 교환금액 (ZBD, amount * rate)
    uint16_t              exchange_rate;       // 적용된 교환비율

    time_point_sec        created_at;
    time_point_sec        deadline;            // 서명 제출 마감 (5분)
    time_point_sec        signed_at;
    shared_string         tx_hash;             // 외부 체인 tx
    time_point_sec        executed_at;
};

typedef multi_index_container<
    withdraw_dollar_request_object,
    indexed_by<
        ordered_unique< tag< by_request_id >,
            member< withdraw_dollar_request_object, uint64_t, &withdraw_dollar_request_object::request_id >
        >,
        // 전역 작업 큐: 상태별, 할인율 우선순위 정렬
        ordered_non_unique< tag< by_status >,
            composite_key< withdraw_dollar_request_object,
                member< withdraw_dollar_request_object, withdraw_dollar_status, &withdraw_dollar_request_object::status >,
                member< withdraw_dollar_request_object, uint16_t, &withdraw_dollar_request_object::discount_rate >,
                member< withdraw_dollar_request_object, time_point_sec, &withdraw_dollar_request_object::created_at >
            >,
            composite_key_compare<
                std::less< withdraw_dollar_status >,
                std::greater< uint16_t >,  // 할인율 우선 (높은게 먼저)
                std::less< time_point_sec >
            >
        >,
        // 증인별 작업 큐: 증인+상태별, 할인율 우선순위 정렬
        ordered_non_unique< tag< by_witness_status >,
            composite_key< withdraw_dollar_request_object,
                member< withdraw_dollar_request_object, account_name_type, &withdraw_dollar_request_object::assigned_witness >,
                member< withdraw_dollar_request_object, withdraw_dollar_status, &withdraw_dollar_request_object::status >,
                member< withdraw_dollar_request_object, uint16_t, &withdraw_dollar_request_object::discount_rate >,
                member< withdraw_dollar_request_object, time_point_sec, &withdraw_dollar_request_object::created_at >
            >,
            composite_key_compare<
                std::less< account_name_type >,
                std::less< withdraw_dollar_status >,
                std::greater< uint16_t >,  // 할인율 우선 (높은게 먼저)
                std::less< time_point_sec >
            >
        >
    >,
    allocator< withdraw_dollar_request_object >
> withdraw_dollar_request_index;
```

**Operation**:

```cpp
struct withdraw_dollar_request_operation : public base_operation
{
    account_name_type from;
    asset             amount;             // "100.000 ZBD"
    uint32_t          chain_id;           // 1 = Ethereum
    string            contract_address;   // 0x... (사용자 지정 출금 컨트랙트)
    string            token;              // 0x... (ERC-20 토큰 주소: USDC, DAI, USDT 등)
    string            recipient;          // 0x...
    uint16_t          discount_rate = 0;  // 할인율 (0~10000, 기본값 0%)

    extensions_type   extensions;

    void validate() const
    {
        FC_ASSERT(is_valid_account_name(from), "Invalid account");
        FC_ASSERT(amount.symbol == DOLLAR_SYMBOL, "Must be ZBD");
        FC_ASSERT(amount.amount > 0, "Amount must be positive");
        FC_ASSERT(chain_id > 0, "Invalid chain ID");
        FC_ASSERT(contract_address.size() == 42 && contract_address.substr(0, 2) == "0x",
                  "Invalid contract address");
        FC_ASSERT(token.size() == 42 && token.substr(0, 2) == "0x",
                  "Invalid token address");
        FC_ASSERT(recipient.size() == 42 && recipient.substr(0, 2) == "0x",
                  "Invalid recipient address");
        FC_ASSERT(discount_rate <= ZATTERA_100_PERCENT,
                  "Discount rate cannot exceed 100%");
    }

    void get_required_active_authorities(flat_set<account_name_type>& a) const
    {
        a.insert(from);
    }
};
```

**Evaluator**:

```cpp
void withdraw_dollar_request_evaluator::do_apply(
    const withdraw_dollar_request_operation& o
)
{
    const auto& db = this->db();

    // 1. ZBD 즉시 소각 (사용자 잔액 차감 + virtual supply 감소)
    // 완료 시점에 증인 보상분은 재생성되어 convert_request로 전달됨
    // 최종 결과: 증인 보상분만큼만 supply 재생성, 나머지는 영구 소각
    db.adjust_balance(o.from, -o.amount);       // 사용자 잔액: -100 ZBD
    db.adjust_supply(-o.amount);                // Virtual supply: -100 ZBD (즉시 소각)

    // 2. 출금 요청 생성 (queued 상태, 증인 미할당)
    uint64_t request_id = generate_unique_request_id(db);

    db.create<withdraw_dollar_request_object>([&](auto& obj) {
        obj.request_id = request_id;
        obj.requester = o.from;
        obj.amount = o.amount;
        obj.chain_id = o.chain_id;
        obj.contract_address = o.contract_address;
        obj.token = o.token;
        obj.recipient = o.recipient;
        obj.discount_rate = o.discount_rate;
        obj.status = withdraw_dollar_status::queued;  // queued 상태로 시작
        // assigned_witness, exchange_amount, exchange_rate, deadline은 아직 설정 안 함
        obj.created_at = db.head_block_time();
    });
}
```

#### 3.1.1 출금 요청 취소

**Operation**:

```cpp
struct cancel_withdraw_dollar_request_operation : public base_operation
{
    account_name_type from;
    uint64_t          request_id;

    extensions_type   extensions;

    void validate() const
    {
        FC_ASSERT(is_valid_account_name(from), "Invalid account");
        FC_ASSERT(request_id > 0, "Invalid request ID");
    }

    void get_required_active_authorities(flat_set<account_name_type>& a) const
    {
        a.insert(from);
    }
};
```

**Evaluator**:

```cpp
void cancel_withdraw_dollar_request_evaluator::do_apply(
    const cancel_withdraw_dollar_request_operation& o
)
{
    const auto& db = this->db();
    const auto& idx = db.get_index<withdraw_dollar_request_index>()
                        .indices().get<by_request_id>();

    auto itr = idx.find(o.request_id);
    FC_ASSERT(itr != idx.end(), "Request not found");
    FC_ASSERT(itr->requester == o.from, "Not the requester");
    FC_ASSERT(itr->status == withdraw_dollar_status::queued,
              "Can only cancel queued requests");

    // ZBD 환불 (사용자 잔액 복구 + supply 재생성)
    db.adjust_balance(o.from, itr->amount);     // 사용자 잔액: +100 ZBD
    db.adjust_supply(itr->amount);              // Virtual supply: +100 ZBD (재생성)

    // 요청 삭제
    db.remove(*itr);
}
```

#### 3.1.2 출금 요청 수정

**Operation**:

```cpp
struct update_withdraw_dollar_request_operation : public base_operation
{
    account_name_type from;
    uint64_t          request_id;

    optional<string>  new_recipient;     // 새로운 수령 주소
    optional<uint16_t> new_discount_rate; // 새로운 할인율

    extensions_type   extensions;

    void validate() const
    {
        FC_ASSERT(is_valid_account_name(from), "Invalid account");
        FC_ASSERT(request_id > 0, "Invalid request ID");

        if (new_recipient.valid())
        {
            FC_ASSERT(new_recipient->size() == 42 && new_recipient->substr(0, 2) == "0x",
                      "Invalid recipient address");
        }

        if (new_discount_rate.valid())
        {
            FC_ASSERT(*new_discount_rate <= ZATTERA_100_PERCENT,
                      "Discount rate cannot exceed 100%");
        }

        FC_ASSERT(new_recipient.valid() || new_discount_rate.valid(),
                  "Must specify at least one field to update");
    }

    void get_required_active_authorities(flat_set<account_name_type>& a) const
    {
        a.insert(from);
    }
};
```

**Evaluator**:

```cpp
void update_withdraw_dollar_request_evaluator::do_apply(
    const update_withdraw_dollar_request_operation& o
)
{
    const auto& db = this->db();
    const auto& idx = db.get_index<withdraw_dollar_request_index>()
                        .indices().get<by_request_id>();

    auto itr = idx.find(o.request_id);
    FC_ASSERT(itr != idx.end(), "Request not found");
    FC_ASSERT(itr->requester == o.from, "Not the requester");
    FC_ASSERT(itr->status == withdraw_dollar_status::queued,
              "Can only update queued requests");

    db.modify(*itr, [&](auto& obj) {
        if (o.new_recipient.valid())
            obj.recipient = *o.new_recipient;

        if (o.new_discount_rate.valid())
            obj.discount_rate = *o.new_discount_rate;
    });
}
```

**Helper Functions**:

```cpp
pair<account_name_type, uint16_t> select_withdraw_witness(
    uint32_t chain_id,
    const string& contract_address,
    const string& token,
    asset requested_amount,
    database& db
)
{
    // 예치금 보고 유효성 검사 기준
    const fc::microseconds DEPOSIT_REPORT_VALIDITY = fc::seconds(ZATTERA_WITHDRAW_DOLLAR_DEPOSIT_VALIDITY);

    // 후보 증인 목록 및 가중치 계산
    struct Candidate {
        account_name_type witness;
        uint16_t exchange_rate;
        uint64_t weight;
    };
    vector<Candidate> candidates;

    // 1. 활성 증인 목록 조회 (현재 스케줄에 있는 증인만)
    const auto& schedule = db.get_witness_schedule_object();
    const auto& witness_idx = db.get_index<witness_index>().indices().get<by_name>();
    const auto& provider_idx = db.get_index<withdraw_dollar_provider_index>()
                                  .indices().get<by_witness_contract>();

    // 스케줄에 있는 증인만 순회 (top20 + timeshare, 최대 21명)
    for (size_t i = 0; i < schedule.num_scheduled_witnesses; ++i)
    {
        const account_name_type& witness_name = schedule.current_shuffled_witnesses[i];

        // 증인 객체 조회
        auto witness_itr = witness_idx.find(witness_name);
        if (witness_itr == witness_idx.end())
            continue;

        // signing_key가 유효한지 확인 (증인이 활성화되어 있는지)
        if (witness_itr->signing_key == public_key_type())
            continue;

        // 2. 해당 증인이 지정된 컨트랙트를 지원하는지 확인
        auto provider_itr = provider_idx.find(
            boost::make_tuple(witness_name, chain_id, contract_address)
        );

        if (provider_itr == provider_idx.end())
            continue;  // 이 증인은 해당 컨트랙트를 지원하지 않음

        if (!provider_itr->is_active)
            continue;

        // **핵심 1**: 요청된 토큰을 지원하는지 확인
        if (provider_itr->supported_tokens.find(token) == provider_itr->supported_tokens.end())
            continue;

        // **핵심 2**: 예치금 보고 확인
        auto deposit_itr = provider_itr->reported_deposits.find(token);
        if (deposit_itr == provider_itr->reported_deposits.end())
            continue;  // 해당 토큰에 대한 예치금 보고 없음

        // **핵심 3**: 예치금 보고가 최근 것인지 확인 (신선도 검증)
        if (db.head_block_time() - provider_itr->deposits_last_reported > DEPOSIT_REPORT_VALIDITY)
            continue;  // 예치금 정보가 너무 오래됨

        // **핵심 4**: 실제 사용 가능한 예치금 계산 (보고된 예치금 - 예약된 예치금)
        share_type reported_deposit = deposit_itr->second;
        share_type reserved_amount = 0;

        auto reserved_itr = provider_itr->reserved_deposits.find(token);
        if (reserved_itr != provider_itr->reserved_deposits.end())
            reserved_amount = reserved_itr->second;

        share_type available_deposit = reported_deposit - reserved_amount;

        // 사용 가능한 예치금이 요청 금액보다 적으면 제외
        if (available_deposit < requested_amount.amount)
            continue;

        // 가중치 계산: 낮은 교환비율 = 높은 가중치
        uint64_t weight = ZATTERA_100_PERCENT - provider_itr->exchange_rate;

        candidates.push_back({witness_name, provider_itr->exchange_rate, weight});
    }

    if (candidates.empty())
    {
        // 사용 가능한 증인 없음 (예치금 부족 등)
        return {account_name_type(), 0};
    }

    // 블록 해시 기반 결정론적 랜덤 선택
    fc::sha256 seed = fc::sha256::hash(
        std::to_string(db.head_block_id()._hash[0]) +
        std::to_string(db.head_block_num())
    );

    // 가중치 합계
    uint64_t total_weight = 0;
    for (const auto& c : candidates)
        total_weight += c.weight;

    // 가중 랜덤 선택
    uint64_t random = seed._hash[0] % total_weight;

    uint64_t cumulative = 0;
    for (const auto& c : candidates)
    {
        cumulative += c.weight;

        if (random < cumulative)
        {
            return {c.witness, c.exchange_rate};
        }
    }

    // 폴백: 첫 번째 후보
    return {candidates[0].witness, candidates[0].exchange_rate};
}

uint64_t generate_unique_request_id(database& db)
{
    // 고유 ID 생성: block_num(32-bit) + tx_id(20-bit) + op_idx(12-bit)
    // - block_num: 블록 번호 (최대 4B 블록 = ~127년 @ 3초/블록)
    // - tx_id: 트랜잭션 ID (블록당 최대 1M tx, 충분)
    // - op_idx: operation 인덱스 (tx당 최대 4096 ops)
    // 참고: get_current_transaction_id()는 블록 내 고유 트랜잭션 ID 반환
    uint64_t block_num = db.head_block_num();
    uint32_t tx_id = db.get_current_transaction_id();
    uint16_t op_idx = db.get_current_operation_index();

    // 비트 할당: [32-bit block_num][20-bit tx_id][12-bit op_idx]
    return (static_cast<uint64_t>(block_num) << 32) |
           ((tx_id & 0xFFFFF) << 12) |
           (op_idx & 0xFFF);
}
```

#### 3.2 증인 할당 (할인율 기반 부하 분산)

**Plugin Implementation**:

```cpp
void withdraw_dollar_plugin::assign_withdraw_requests()
{
    // 매 블록마다 실행 (on_apply_block에서 호출)
    auto& db = database();
    const auto& queued_idx = db.get_index<withdraw_dollar_request_index>()
                                .indices().get<by_status>();

    // 1. queued 상태의 요청을 할인율 내림차순으로 조회
    auto queued_range = queued_idx.equal_range(withdraw_dollar_status::queued);

    if (queued_range.first == queued_range.second)
        return;  // queued 요청 없음

    // 2. 할인율 높은 요청부터 증인에게 할당
    // 중요: status를 변경하면 by_status 인덱스 키가 변경되어 iterator 무효화됨
    //       따라서 먼저 수집 후 처리
    vector<withdraw_dollar_request_id_type> requests_to_assign;
    size_t count = 0;

    for (auto itr = queued_range.first;
         itr != queued_range.second && count < ZATTERA_WITHDRAW_DOLLAR_MAX_ASSIGN_PER_BLOCK;
         ++itr, ++count)
    {
        requests_to_assign.push_back(itr->request_id);
    }

    // 3. 수집된 요청들을 처리 (iterator 안전)
    const auto& by_id_idx = db.get_index<withdraw_dollar_request_index>()
                               .indices().get<by_request_id>();

    size_t assigned_count = 0;  // 할당된 요청 수 추적

    for (const auto& req_id : requests_to_assign)
    {
        auto itr = by_id_idx.find(req_id);
        if (itr == by_id_idx.end() || itr->status != withdraw_dollar_status::queued)
            continue;  // 이미 처리되었거나 삭제됨

        // 증인 선택 (예치금 사용 가능 여부 고려)
        auto [selected_witness, exchange_rate] = select_withdraw_witness(
            itr->chain_id,
            itr->contract_address,
            itr->token,
            itr->amount,
            db
        );

        if (selected_witness == account_name_type())
            continue;  // 사용 가능한 증인이 없음

        // 교환금액 계산
        asset exchange_amount = itr->amount * exchange_rate / ZATTERA_100_PERCENT;

        // queued → pending 상태로 전환 (안전하게 수정)
        db.modify(*itr, [&](auto& obj) {
            obj.status = withdraw_dollar_status::pending;
            obj.assigned_witness = selected_witness;
            obj.exchange_amount = exchange_amount;
            obj.exchange_rate = exchange_rate;
            obj.deadline = db.head_block_time() + fc::seconds(ZATTERA_WITHDRAW_DOLLAR_SIGNATURE_DEADLINE);
        });

        // provider의 reserved_deposits 증가
        const auto& provider_idx = db.get_index<withdraw_dollar_provider_index>()
                                      .indices().get<by_witness_contract>();
        auto provider_itr = provider_idx.find(
            boost::make_tuple(selected_witness, itr->chain_id, itr->contract_address)
        );

        if (provider_itr != provider_idx.end())
        {
            db.modify(*provider_itr, [&](auto& obj) {
                obj.reserved_deposits[itr->token] += itr->amount.amount;
            });
        }

        assigned_count++;
    }

    if (assigned_count > 0)
    {
        ilog("Assigned ${count} queued requests to witnesses", ("count", assigned_count));
    }
}

#### 3.3 증인 서명 생성

**Plugin Implementation**:

```cpp
void withdraw_dollar_plugin::on_apply_block(const signed_block& block)
{
    auto& db = database();

    // 0. queued 요청을 할인율 기준으로 증인에게 할당
    assign_withdraw_requests();

    // 1. pending 요청 처리 (서명 생성)
    const auto& idx = db.get_index<withdraw_dollar_request_index>()
                        .indices().get<by_witness_status>();

    const account_name_type my_witness = get_my_witness_name();

    // 2. pending 요청 처리 (서명 생성)
    // equal_range는 witness와 status가 일치하는 모든 요청을 반환
    // 인덱스가 by_witness_status이므로 할인율 내림차순으로 정렬됨
    auto pending_range = idx.equal_range(
        boost::make_tuple(my_witness, withdraw_dollar_status::pending)
    );

    size_t pending_count = 0;
    for (auto itr = pending_range.first;
         itr != pending_range.second && pending_count < ZATTERA_WITHDRAW_DOLLAR_MAX_SIGN_PER_BLOCK;
         ++itr, ++pending_count)
    {
        if (db.head_block_time() > itr->deadline)
        {
            wlog("Request ${id} expired", ("id", itr->request_id));
            continue;
        }

        // EIP-712 서명 생성
        try {
            string signature = generate_withdraw_signature(
                itr->chain_id,
                itr->contract_address,
                itr->request_id,
                itr->recipient,
                itr->token,
                itr->amount,
                itr->discount_rate
            );

            // Zattera 체인에 서명 제출
            broadcast_operation(withdraw_dollar_approve_operation{
                .witness = my_witness,
                .request_id = itr->request_id,
                .signature = signature
            });
        }
        catch (const std::exception& e) {
            elog("Failed to sign withdrawal ${id}: ${e}",
                 ("id", itr->request_id)("e", e.what()));
        }
    }

    // 2. signed 요청은 백그라운드에서 처리 (완료 보고는 report_completions_periodically에서)
}

void withdraw_dollar_plugin::report_completions_periodically()
{
    // 주기적으로 호출 (예: 1분마다)
    auto& db = database();
    // 할인율 높은 순서로 정렬된 인덱스 사용
    const auto& idx = db.get_index<withdraw_dollar_request_index>()
                        .indices().get<by_witness_status>();

    const account_name_type my_witness = get_my_witness_name();

    // signed 상태인 요청만 조회 (외부 체인 완료 확인)
    // equal_range는 witness와 status가 일치하는 모든 요청을 반환
    // 인덱스가 by_witness_status이므로 할인율 내림차순으로 정렬됨
    auto signed_range = idx.equal_range(
        boost::make_tuple(my_witness, withdraw_dollar_status::signed)
    );

    size_t checked_count = 0;
    for (auto itr = signed_range.first;
         itr != signed_range.second && checked_count < ZATTERA_WITHDRAW_DOLLAR_MAX_CHECK_PER_MINUTE;
         ++itr, ++checked_count)
    {
        try {
            // 외부 체인에서 처리 상태 확인
            auto tx_hash = query_withdraw_transaction(
                itr->chain_id,
                itr->contract_address,
                itr->request_id
            );

            if (!tx_hash.empty())
            {
                // 완료 보고 제출
                broadcast_operation(complete_withdraw_dollar_operation{
                    .witness = my_witness,
                    .request_id = itr->request_id,
                    .tx_hash = tx_hash
                });

                ilog("Withdrawal ${id} completed: ${tx}",
                     ("id", itr->request_id)("tx", tx_hash));
            }
        }
        catch (const std::exception& e) {
            elog("Failed to check withdrawal ${id}: ${e}",
                 ("id", itr->request_id)("e", e.what()));
        }
    }
}

void withdraw_dollar_plugin::report_deposits_periodically()
{
    // 주기적으로 호출 (예: 5분마다)
    auto& db = database();

    // 등록된 모든 컨트랙트에 대해 예치금 보고
    const auto& idx = db.get_index<withdraw_dollar_provider_index>()
                        .indices().get<by_witness_contract>();

    auto range = idx.equal_range(get_my_witness_name());

    for (auto itr = range.first; itr != range.second; ++itr)
    {
        if (!itr->is_active)
            continue;

        try {
            // 외부 체인에서 각 토큰별 예치금 조회
            map<string, asset> deposits;

            for (const auto& token : itr->supported_tokens)
            {
                // ZRC-9 표준: getWitnessDeposit(address witness, address token)
                // **중요**: ZRC-9 스펙에 따라 반환값은 항상 18 decimals로 정규화됨
                //          (토큰의 실제 decimals와 무관하게 컨트랙트에서 18로 변환)
                //          예: USDC 100개 (6 decimals) → 100000000000000000000 (18 decimals)
                //              DAI 100개 (18 decimals) → 100000000000000000000 (18 decimals)
                uint256_t token_balance = query_deposit_balance(
                    itr->chain_id,
                    itr->contract_address,
                    itr->signer_address,  // 증인의 서명 주소
                    token
                );

                // 18 decimals → 3 decimals (ZBD) 변환
                asset dollar_amount = token_amount_to_dollar(token_balance);

                deposits[token] = dollar_amount;
            }

            // Zattera 체인에 예치금 보고
            broadcast_operation(report_withdraw_dollar_deposits_operation{
                .witness = get_my_witness_name(),
                .chain_id = itr->chain_id,
                .contract_address = itr->contract_address,
                .deposits = deposits
            });

            ilog("Reported deposits for contract ${c} on chain ${id}",
                 ("c", itr->contract_address)("id", itr->chain_id));
        }
        catch (const std::exception& e) {
            elog("Failed to report deposits for contract ${c}: ${e}",
                 ("c", itr->contract_address)("e", e.what()));
        }
    }
}

uint256_t query_deposit_balance(
    uint32_t chain_id,
    const string& contract_address,
    const string& witness_address,
    const string& token
)
{
    auto& web3_client = get_web3_client(chain_id);

    // ZRC-9 표준 함수 호출: getWitnessDeposit(address witness, address token)
    auto result = web3_client.call({
        .to = contract_address,
        .data = web3::encode_function_call(
            "getWitnessDeposit(address,address)",
            witness_address,
            token
        )
    });

    // uint256 반환값 디코딩
    return web3::decode_result<uint256_t>(result);
}

// ZRC-9에서 조회한 예치금을 ZBD equivalent로 변환
// **중요**: ZRC-9 스펙에 따라 input은 항상 18 decimals로 정규화되어 있음
//          (토큰의 실제 decimals와 무관 - 컨트랙트가 18로 변환 후 반환)
asset token_amount_to_dollar(uint256_t amount)
{
    const uint8_t dollar_decimals = DOLLAR_SYMBOL.decimals();  // 3 decimals
    const uint8_t ZRC9_DECIMALS = 18;  // ZRC-9 표준 인터페이스 스펙

    // 18 decimals → 3 decimals (ZBD) 변환
    // 예: 100000000000000000000 (100, 18 decimals) → 100000 (100.000 ZBD, 3 decimals)
    uint256_t dollar_raw = amount / pow(10, ZRC9_DECIMALS - dollar_decimals);

    return asset(dollar_raw, DOLLAR_SYMBOL);
}

// EIP-712 타입 정의
namespace eip712 {
    struct Withdrawal {
        uint64_t requestId;
        address recipient;
        address token;
        uint256 amount;        // 할인 적용된 금액 (외부 체인에서 실제 받을 금액)
        uint16_t discountRate; // 할인율 (0~10000)
    };

    // EIP-712 타입 해시
    const string WITHDRAWAL_TYPEHASH =
        "Withdrawal(uint64 requestId,address recipient,address token,uint256 amount,uint16 discountRate)";
}

string generate_withdraw_signature(
    uint32_t chain_id,
    const string& contract_address,
    uint64_t request_id,
    const string& recipient,
    const string& token,
    asset amount,
    uint16_t discount_rate
)
{
    // EIP-712 도메인 (ZRC-9 표준: name/version 생략)
    // verifyingContract는 사용자가 지정한 출금 컨트랙트 주소 사용 (컨트랙트 바인딩)
    eip712::Domain domain = {
        .chainId = chain_id,
        .verifyingContract = contract_address
    };

    // 할인 적용된 금액 계산 및 18 decimals로 정규화
    // 예: 100 ZBD (raw: 100000, 3 decimals), 5% 할인
    //     → 95000 (ZBD raw) * 10^15 = 95 * 10^18
    share_type discounted_zbd = amount.amount.value * (ZATTERA_100_PERCENT - discount_rate) / ZATTERA_100_PERCENT;
    uint256_t discounted_amount_18 = uint256_t(discounted_zbd) * 1000000000000000;  // * 10^15

    // 메시지
    eip712::Withdrawal message = {
        .requestId = request_id,
        .recipient = recipient,
        .token = token,
        .amount = discounted_amount_18,  // 18 decimals 정규화
        .discountRate = discount_rate
    };

    // 증인의 외부 체인 private key로 서명
    return eip712::sign(domain, message, get_my_eth_private_key());
}

string query_withdraw_transaction(
    uint32_t chain_id,
    const string& contract_address,
    uint64_t request_id
)
{
    // 증인은 외부 체인 RPC 노드에 접근
    auto& web3_client = get_web3_client(chain_id);

    // 1. 컨트랙트의 getWithdrawalStatus 함수 호출
    // function getWithdrawalStatus(uint64 requestId)
    //     external view returns (bool processed, uint256 blockNumber)

    auto result = web3_client.call({
        .to = contract_address,
        .data = web3::encode_function_call(
            "getWithdrawalStatus(uint64)",
            request_id
        )
    });

    // 결과 디코딩: (bool processed, uint256 blockNumber)
    auto [processed, block_number] = web3::decode_result<bool, uint256>(result);

    if (!processed || block_number == 0)
    {
        return "";  // 아직 처리 안됨
    }

    // 2. 해당 블록에서 Withdrawn 이벤트 필터링
    // event Withdrawn(uint64 indexed requestId, address indexed recipient,
    //                 address indexed token, uint256 amount, uint16 discountRate, address witness)
    auto event_signature = web3::keccak256("Withdrawn(uint64,address,address,uint256,uint16,address)");
    auto request_id_topic = web3::encode_uint64(request_id);

    auto logs = web3_client.get_logs({
        .address = contract_address,
        .topics = {event_signature, request_id_topic},
        .fromBlock = block_number,
        .toBlock = block_number  // 정확히 해당 블록만 조회
    });

    if (logs.empty())
    {
        // 의심스러운 상황이지만 재시도 가능하므로 경고만 출력
        wlog("Withdrawal marked processed at block ${b} but event not found for request ${r}",
             ("b", block_number)("r", request_id));
        return "";  // 다음 주기에 재시도
    }

    // 트랜잭션 해시 반환
    return logs[0].transactionHash;
}
```

**Operation**:

```cpp
struct withdraw_dollar_approve_operation : public base_operation
{
    account_name_type witness;
    uint64_t          request_id;
    string            signature;     // EIP-712 서명 (hex)

    extensions_type   extensions;

    void validate() const
    {
        FC_ASSERT(is_valid_account_name(witness), "Invalid witness");
        FC_ASSERT(request_id > 0, "Invalid request ID");
        FC_ASSERT(signature.size() > 0, "Empty signature");
    }

    void get_required_active_authorities(flat_set<account_name_type>& a) const
    {
        a.insert(witness);
    }
};
```

**Evaluator**:

```cpp
void withdraw_dollar_approve_evaluator::do_apply(
    const withdraw_dollar_approve_operation& o
)
{
    const auto& db = this->db();

    // 1. 출금 요청 확인
    const auto& idx = db.get_index<withdraw_dollar_request_index>()
                        .indices().get<by_request_id>();
    auto itr = idx.find(o.request_id);
    FC_ASSERT(itr != idx.end(), "Request not found");
    FC_ASSERT(itr->status == withdraw_dollar_status::pending, "Not pending");
    FC_ASSERT(itr->assigned_witness == o.witness, "Wrong witness");
    FC_ASSERT(db.head_block_time() <= itr->deadline, "Deadline passed");

    // 2. 서명 검증
    const auto& contract_idx = db.get_index<withdraw_dollar_provider_index>()
                                  .indices().get<by_witness_contract>();
    auto contract_itr = contract_idx.find(
        boost::make_tuple(o.witness, itr->chain_id, itr->contract_address)
    );
    FC_ASSERT(contract_itr != contract_idx.end(), "Contract not registered");

    bool valid = verify_eip712_signature(
        itr->chain_id,
        itr->contract_address,
        o.request_id,
        itr->recipient,
        itr->token,
        itr->amount,
        itr->discount_rate,
        o.signature,
        contract_itr->signer_address
    );
    FC_ASSERT(valid, "Invalid signature");

    // 3. 서명 저장 (상태: signed)
    db.modify(*itr, [&](auto& obj) {
        obj.status = withdraw_dollar_status::signed;
        obj.witness_signature = o.signature;
        obj.signed_at = db.head_block_time();
    });

    // 보상은 완료 보고 시 지급
}

bool verify_eip712_signature(
    uint32_t chain_id,
    const string& contract_address,
    uint64_t request_id,
    const string& recipient,
    const string& token,
    asset amount,
    uint16_t discount_rate,
    const string& signature,
    const string& signer_address
)
{
    // EIP-712 메시지 해시 생성 (ZRC-9 표준: name/version 생략)
    eip712::Domain domain = {
        .chainId = chain_id,
        .verifyingContract = contract_address  // 사용자 지정 컨트랙트 주소
    };

    // 할인 적용된 금액 계산 및 18 decimals로 정규화 (생성 로직과 동일)
    share_type discounted_zbd = amount.amount.value * (ZATTERA_100_PERCENT - discount_rate) / ZATTERA_100_PERCENT;
    uint256_t discounted_amount_18 = uint256_t(discounted_zbd) * 1000000000000000;  // * 10^15

    eip712::Withdrawal message = {
        .requestId = request_id,
        .recipient = recipient,
        .token = token,
        .amount = discounted_amount_18,  // 18 decimals 정규화
        .discountRate = discount_rate
    };

    bytes32 digest = eip712::hash_typed_data(domain, message);

    // 서명에서 주소 복구
    string recovered_address = ecdsa::recover(digest, signature);

    return (recovered_address == signer_address);
}
```

#### 3.3 사용자 출금 실행

**사용자 측 (JavaScript)**:

```javascript
// 1. Zattera API에서 출금 요청 조회
const request = await zatteraApi.getWithdrawalRequest(requestId);

if (request.status === 'signed') {
    // 2. ZBD amount 파싱 및 18 decimals로 정규화
    // CLI 응답: "100.000 ZBD" → ZBD raw (3 decimals): 100000 → 18 decimals: 100 * 10^18
    const amountZBD = parseFloat(request.amount.split(' ')[0]);  // 100.000
    const amountRaw = amountZBD * 1000;  // 100000 (ZBD 3 decimals)

    // 할인 적용
    const discountedZBD = Math.floor(amountRaw * (10000 - request.discount_rate) / 10000);  // 95000

    // 18 decimals로 정규화 (ZRC-9 표준)
    const amount18 = ethers.BigNumber.from(discountedZBD).mul(ethers.BigNumber.from(10).pow(15));
    // 95000 * 10^15 = 95 * 10^18

    // 3. 외부 체인 컨트랙트 호출
    const bridgeContract = new ethers.Contract(
        request.contract_address,  // 사용자가 지정한 컨트랙트 주소
        BRIDGE_ABI,
        userWallet
    );

    // 4. 증인 서명으로 출금
    const tx = await bridgeContract.withdraw(
        request.request_id,        // uint64: Zattera 출금 요청 ID
        request.recipient,         // address: 토큰 수령 주소
        request.token,             // address: ERC-20 토큰 주소 (USDC, DAI, USDT 등)
        amount18,                  // uint256: 할인 적용된 금액 (18 decimals 정규화)
        request.discount_rate,     // uint16: 할인율 (0~10000)
        request.witness_signature  // bytes: 증인의 EIP-712 서명
    );

    await tx.wait();

    // 토큰 amount는 컨트랙트 내부에서 18 decimals → 토큰 native decimals로 자동 변환됨
    // 예: 95 * 10^18 (18 decimals)
    //   - USDC (6 decimals): 95 * 10^18 / 10^12 = 95 * 10^6 (95 USDC)
    //   - DAI (18 decimals): 95 * 10^18 / 10^0 = 95 * 10^18 (95 DAI)
    console.log(`Withdrawn ${amountZBD * (10000 - request.discount_rate) / 10000} ZBD worth of tokens (${request.discount_rate / 100}% discount)`);
    console.log(`Token: ${request.token}`);
    console.log(`Tx hash: ${tx.hash}`);

    // 5. 완료!
    // 참고: 증인이 자동으로 완료를 감지하고 Zattera 체인에 보고합니다
    // 사용자는 추가 액션이 필요 없습니다
}
```

#### 3.4 외부 체인 스마트 컨트랙트

ZEP-8은 **ZRC-9 (ZEP-9)** 표준 인터페이스를 구현하는 브리지 컨트랙트를 사용합니다.

**ZRC-9 인터페이스 요구사항** (상세 구현은 ZEP-9 참조):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/**
 * @title IZRC9
 * @notice Zattera 크로스체인 브리지 표준 인터페이스
 * @dev 상세 스펙 및 구현은 ZEP-9 참조
 */
interface IZRC9 {
    /**
     * @notice 출금 완료 이벤트
     * @param requestId Zattera 체인 출금 요청 ID
     * @param recipient 토큰을 받은 주소
     * @param token ERC-20 토큰 주소
     * @param amount 할인 적용된 금액 (18 decimals 정규화)
     * @param discountRate 할인율 (0~10000 basis points)
     * @param witness 출금을 처리한 증인 주소
     */
    event Withdrawn(
        uint64 indexed requestId,
        address indexed recipient,
        address indexed token,
        uint256 amount,
        uint16 discountRate,
        address witness
    );

    /**
     * @notice 증인 예치금 충전 이벤트
     */
    event Deposited(address indexed witness, address indexed token, uint256 amount);

    /**
     * @notice 사용자가 증인 서명으로 출금 실행
     * @param requestId Zattera 출금 요청 ID
     * @param recipient 토큰을 받을 주소
     * @param token ERC-20 토큰 주소
     * @param amount 할인 적용된 금액 (18 decimals 정규화)
     * @param discountRate 할인율 (0~10000 basis points)
     * @param witnessSignature 증인의 EIP-712 서명
     */
    function withdraw(
        uint64 requestId,
        address recipient,
        address token,
        uint256 amount,
        uint16 discountRate,
        bytes calldata witnessSignature
    ) external;

    /**
     * @notice 증인 예치금 충전
     * @param token ERC-20 토큰 주소
     * @param amount 예치할 토큰 수량
     */
    function deposit(address token, uint256 amount) external;

    /**
     * @notice 출금 처리 상태 조회
     * @param requestId Zattera 출금 요청 ID
     * @return processed 처리 여부
     * @return blockNumber 처리된 블록 번호 (미처리 시 0)
     */
    function getWithdrawalStatus(uint64 requestId)
        external
        view
        returns (bool processed, uint256 blockNumber);

    /**
     * @notice 증인 예치금 잔액 조회
     * @param witness 증인 주소
     * @param token 토큰 주소
     * @return balance 예치금 잔액 (18 decimals 정규화)
     */
    function getWitnessDeposit(address witness, address token)
        external
        view
        returns (uint256 balance);
}
```

**ZEP-8 특화 요구사항**:

1. **EIP-712 서명 구조**:
```solidity
// EIP-712 도메인 (ZRC-9 표준: name/version 생략)
EIP712Domain(uint256 chainId, address verifyingContract)

// Withdrawal 타입
Withdrawal(uint64 requestId, address recipient, address token, uint256 amount, uint16 discountRate)
```

2. **할인율 및 금액 정규화**:
   - `amount`: 할인 적용된 금액 (18 decimals 정규화)
   - `discountRate`: 0~10000 basis points (예: 500 = 5%)
   - 컨트랙트는 18 decimals를 토큰 native decimals로 자동 변환

3. **예치금 관리**:
   - 증인은 `deposit()`으로 예치금 충전
   - 예치금은 인출 불가 (출금 서비스 제공용)
   - `getWitnessDeposit()`는 18 decimals로 정규화하여 반환

4. **출금 처리 검증**:
   - 이중 출금 방지 (requestId 기반)
   - EIP-712 서명으로 증인 검증
   - 예치금 잔액 확인 후 차감

**참고**: ZRC-9 컨트랙트의 전체 구현 코드는 **ZEP-9** 문서를 참조하세요.

#### 3.5 출금 완료 보고

증인이 외부 체인에서 출금 실행을 확인한 후, Zattera 체인에 완료를 보고합니다. 이 보고를 통해 증인은 ZBD → ZP 변환 보상을 받습니다.

**Operation**:

```cpp
struct complete_withdraw_dollar_operation : public base_operation
{
    account_name_type witness;       // 증인이 제출
    uint64_t          request_id;
    string            tx_hash;       // 외부 체인 tx

    extensions_type   extensions;

    void validate() const
    {
        FC_ASSERT(is_valid_account_name(witness), "Invalid witness");
        FC_ASSERT(request_id > 0, "Invalid request ID");
        FC_ASSERT(tx_hash.size() == 66 && tx_hash.substr(0, 2) == "0x",
                  "Invalid tx hash");
    }

    void get_required_active_authorities(flat_set<account_name_type>& a) const
    {
        a.insert(witness);
    }
};
```

**Evaluator**:

```cpp
void complete_withdraw_dollar_evaluator::do_apply(
    const complete_withdraw_dollar_operation& o
)
{
    const auto& db = this->db();

    // 1. 출금 요청 확인
    const auto& idx = db.get_index<withdraw_dollar_request_index>()
                        .indices().get<by_request_id>();
    auto itr = idx.find(o.request_id);
    FC_ASSERT(itr != idx.end(), "Request not found");
    FC_ASSERT(itr->assigned_witness == o.witness, "Wrong witness");
    FC_ASSERT(itr->status == withdraw_dollar_status::signed, "Not signed");

    // 2. 외부 체인에서 출금 트랜잭션 검증
    bool verified = verify_withdraw_transaction(
        itr->chain_id,
        o.tx_hash,
        o.request_id,
        itr->recipient,
        itr->token,
        itr->amount,
        itr->discount_rate
    );
    FC_ASSERT(verified, "Withdrawal transaction verification failed");

    // 3. 완료 표시
    db.modify(*itr, [&](auto& obj) {
        obj.status = withdraw_dollar_status::completed;
        obj.tx_hash = o.tx_hash;
        obj.executed_at = db.head_block_time();
    });

    // 4. 증인에게 보상 지급 및 ZBD → ZP 변환 요청 생성
    // 4-1. 증인 보상분 supply 재생성 및 증인에게 지급
    db.adjust_supply(itr->exchange_amount);              // Virtual supply: +60 ZBD (재생성)
    db.adjust_balance(o.witness, itr->exchange_amount);  // 증인 잔액: +60 ZBD

    // 4-2. 증인이 받은 ZBD를 즉시 ZP 변환 요청 (ZATTERA_CONVERSION_DELAY = 3.5일)
    // 기존 convert_request_object를 재사용 - 가격 조작 방지 메커니즘
    db.create<convert_request_object>([&](auto& obj) {
        obj.owner = o.witness;
        obj.requestid = db.get_convert_request_id(o.witness);
        obj.amount = itr->exchange_amount;  // 교환비율 적용된 ZBD 금액
        obj.conversion_date = db.head_block_time() + ZATTERA_CONVERSION_DELAY;  // 3.5일 후
    });
    // convert_request 실행 시 database::process_conversions()가 이 ZBD를 소각하고 ZP 생성

    // 4-3. 나머지 ZBD는 이미 요청 시 소각됨
    // 완료 시점 기준: -100 (요청) +60 (완료) = net -40 ZBD (일시적)
    // 최종 결과 (convert 후): -100 (요청) +60 (완료) -60 (convert) = net -100 ZBD

    // 5. 증인 누적 통계 업데이트
    const auto& contract_idx = db.get_index<withdraw_dollar_provider_index>()
                                  .indices().get<by_witness_contract>();
    auto contract_itr = contract_idx.find(
        boost::make_tuple(o.witness, itr->chain_id, itr->contract_address)
    );

    // 영구 소각된 금액 계산 (요청 금액 - 증인 보상)
    asset permanently_burned = itr->amount - itr->exchange_amount;

    db.modify(*contract_itr, [&](auto& obj) {
        obj.total_withdrawal_amount += itr->amount;           // 교환비율 적용 전 원금
        obj.total_burned_amount += permanently_burned;        // 영구 소각된 금액 누적
        obj.total_withdrawal_count += 1;
        obj.last_withdrawal_time = db.head_block_time();

        // reserved_deposits 감소 (완료된 요청의 예치금 해제)
        auto reserved_itr = obj.reserved_deposits.find(itr->token);
        if (reserved_itr != obj.reserved_deposits.end())
        {
            reserved_itr->second -= itr->amount.amount;
            if (reserved_itr->second <= 0)
                obj.reserved_deposits.erase(reserved_itr);
        }
    });
}

bool verify_withdraw_transaction(
    uint32_t chain_id,
    const string& tx_hash,
    uint64_t request_id,
    const string& recipient,
    const string& token,
    asset amount,
    uint16_t discount_rate
)
{
    auto receipt = web3_client.get_transaction_receipt(chain_id, tx_hash);

    // Withdrawn 이벤트 확인
    // event Withdrawn(uint64 indexed requestId, address indexed recipient,
    //                 address indexed token, uint256 amount, uint16 discountRate, address witness)
    // amount는 18 decimals 정규화
    for (const auto& log : receipt.logs)
    {
        if (log.topics[0] == keccak256("Withdrawn(uint64,address,address,uint256,uint16,address)"))
        {
            // Indexed 파라미터 (topics)
            uint64_t logged_id = decode_uint64(log.topics[1]);
            string logged_recipient = decode_address(log.topics[2]);
            string logged_token = decode_address(log.topics[3]);

            // Non-indexed 파라미터 (data) - ABI 인코딩된 순서대로
            // logged_amount는 18 decimals 정규화된 값
            auto [logged_amount, logged_discount_rate, logged_witness] =
                decode_log_data<uint256, uint16, address>(log.data);

            // ZBD 3 decimals → 18 decimals 변환 후 검증
            // 예: 100.000 ZBD (raw: 100000) * 95% = 95000
            //     → 18 decimals: 95000 * 10^15 = 95 * 10^18
            share_type discounted_zbd = amount.amount.value * (ZATTERA_100_PERCENT - discount_rate) / ZATTERA_100_PERCENT;
            uint256_t expected_amount_18 = uint256_t(discounted_zbd) * 1000000000000000;  // * 10^15

            return (logged_id == request_id &&
                    logged_recipient == recipient &&
                    logged_token == token &&
                    logged_amount == expected_amount_18 &&
                    logged_discount_rate == discount_rate);
        }
    }

    return false;
}
```

### 4. 경제 모델 및 증인 보상 처리

#### 4.1 전체 경제 흐름

**ZBD 라이프사이클** (교환비율 60%의 경우):

```
[요청 시점]
사용자 잔액: -100 ZBD
Virtual supply: -100 ZBD (즉시 소각)

[완료 시점]
Virtual supply: +60 ZBD (증인 보상분 재생성)
증인 잔액: +60 ZBD
  └─ convert_request 생성 (3.5일 후 ZP로 변환)

[3.5일 후 - Convert 실행]
증인 잔액: -60 ZBD (convert 실행 시 소각)
Virtual supply: -60 ZBD
증인 ZP: +XXX VESTS (ZBD → ZTR → ZP 변환)

[최종 결과]
Virtual supply 변화: -100 (요청) +60 (완료) -60 (convert) = -100 ZBD
결과: 요청된 100 ZBD 전부 소각 완료
```

**핵심 원칙**:
1. ✅ **요청 시**: 즉시 소각 (사용자 잔액 차감 + supply 감소)
2. ✅ **완료 시**: 증인 보상분만 supply 재생성 + 증인에게 지급
3. ✅ **변환 시**: 증인의 ZBD 소각하고 ZP 생성
4. ✅ **최종 결과**: 요청 ZBD 100% 소각 보장
5. ✅ **회계 투명성**: 모든 시점에서 supply 변화가 명확히 추적 가능

#### 4.2 증인 보상 변환

증인 보상(ZBD)을 ZP로 변환하는 과정은 기존 `convert_request_object`를 재사용합니다.

**기존 시스템 재사용**:

```cpp
// 완료 시점에 증인 보상분 supply 재생성 및 지급
db.adjust_supply(exchange_amount);                   // Supply: +60 ZBD (재생성)
db.adjust_balance(witness_account, exchange_amount); // 증인: +60 ZBD

// 증인의 ZBD를 즉시 ZP 변환 요청 (기존 convert 메커니즘)
db.create<convert_request_object>([&](auto& obj) {
    obj.owner = witness_account;
    obj.requestid = db.get_convert_request_id(witness_account);
    obj.amount = exchange_amount;  // 교환비율 적용된 ZBD
    obj.conversion_date = db.head_block_time() + ZATTERA_CONVERSION_DELAY;
});

// 나머지 ZBD는 이미 요청 시 소각됨
// 완료 시점 기준: 요청 시 -100 ZBD, 완료 시 +60 ZBD → net -40 ZBD (일시적)
// 하지만 convert 실행 시 재생성된 60 ZBD도 소각됨

// 3.5일 후 database::process_conversions()가 자동으로:
// 1. 증인 잔액에서 ZBD 차감: -60 ZBD
// 2. Virtual supply 소각: -60 ZBD
// 3. ZBD → ZTR 변환 (84시간 평균 가격)
// 4. ZTR → ZP 변환 (현재 vesting price)
// 5. 증인에게 ZP 지급
// 최종 결과: -100 (요청) +60 (완료) -60 (convert) = net -100 ZBD 영구 소각
```

**핵심 포인트**:

1. **3.5일 지연**: `ZATTERA_CONVERSION_DELAY` (84시간) 적용
2. **평균 가격 사용**: `feed_history.current_median_history` 사용
   - 과거 84시간 동안의 median price
   - 단기 가격 조작 방지
3. **2단계 변환**:
   - 1단계: ZBD → ZTR (median price)
   - 2단계: ZTR → ZP (vesting share price)
4. **공정성**: 모든 ZBD → ZP 변환이 동일한 규칙 적용
5. **코드 재사용**: 검증된 기존 변환 로직 사용

### 5. 타임아웃 및 재선택

증인이 5분 내에 서명을 제출하지 않으면, 다른 증인에게 재선택됩니다.

**Plugin Implementation**:

```cpp
void withdraw_dollar_plugin::on_apply_block(const signed_block& block)
{
    auto& db = database();

    // 타임아웃된 요청 확인
    const auto& idx = db.get_index<withdraw_dollar_request_index>()
                        .indices().get<by_status>();
    auto itr = idx.lower_bound(withdraw_dollar_status::pending);
    auto end = idx.upper_bound(withdraw_dollar_status::pending);

    for (; itr != end; ++itr)
    {
        if (db.head_block_time() > itr->deadline)
        {
            // 재선택 시도
            try {
                auto [new_witness, new_rate] = select_withdraw_witness(
                    itr->chain_id,
                    itr->contract_address,
                    itr->token,
                    itr->amount,  // 요청 금액 추가
                    db
                );

                db.modify(*itr, [&](auto& obj) {
                    obj.assigned_witness = new_witness;
                    obj.exchange_rate = new_rate;
                    obj.exchange_amount = obj.amount * new_rate / ZATTERA_100_PERCENT;
                    obj.deadline = db.head_block_time() + fc::seconds(300);
                });

                ilog("Request ${id} reassigned to ${witness}",
                     ("id", itr->request_id)("witness", new_witness));
            }
            catch (const fc::exception& e)
            {
                // 사용 가능한 증인 없음 → 만료 처리
                db.modify(*itr, [&](auto& obj) {
                    obj.status = withdraw_dollar_status::expired;
                });

                // provider의 reserved_deposits 감소 (만료된 요청의 예치금 해제)
                const auto& provider_idx = db.get_index<withdraw_dollar_provider_index>()
                                              .indices().get<by_witness_contract>();
                auto provider_itr = provider_idx.find(
                    boost::make_tuple(itr->assigned_witness, itr->chain_id, itr->contract_address)
                );

                if (provider_itr != provider_idx.end())
                {
                    db.modify(*provider_itr, [&](auto& obj) {
                        auto reserved_itr = obj.reserved_deposits.find(itr->token);
                        if (reserved_itr != obj.reserved_deposits.end())
                        {
                            reserved_itr->second -= itr->amount.amount;
                            if (reserved_itr->second <= 0)
                                obj.reserved_deposits.erase(reserved_itr);
                        }
                    });
                }

                // 사용자에게 ZBD 환불 (잔액 복구 + supply 재생성)
                db.adjust_balance(itr->requester, itr->amount);  // 사용자 잔액: +100 ZBD
                db.adjust_supply(itr->amount);                   // Virtual supply: +100 ZBD (재생성)

                elog("Request ${id} expired, refunded: ${e}",
                     ("id", itr->request_id)("e", e.to_detail_string()));
            }
        }
    }
}
```

### 6. 보안 고려사항

#### 6.1 가격 조작 방지 (ZBD → ZP 변환)

**메커니즘**: `convert_request_object`와 동일한 보호

**3.5일 지연 + 평균 가격**:
```cpp
// 변환 요청 생성 시
obj.conversion_date = db.head_block_time() + ZATTERA_CONVERSION_DELAY;  // 3.5일 후

// 변환 실행 시
asset liquid_amount = itr->amount * fhistory.current_median_history;  // 84시간 평균
```

**보호 효과**:
```
시나리오: 악의적 증인이 가격 조작 시도
→ 단기간 ZTR/ZBD 가격 조작 (예: 30분)
→ 출금 완료 보고 제출
→ 변환은 3.5일 후 실행
→ 변환 시각의 84시간 평균 가격 사용
→ 단기 조작은 평균에 미미한 영향
→ 공정한 가격으로 변환 ✅
```

**핵심**:
- **즉시 변환 금지**: 모든 ZBD → ZP 변환은 3.5일 대기
- **평균 가격 강제**: 단기 가격 스파이크 무력화
- **일관성**: 기존 `convert` operation과 동일한 규칙
- **공정성**: 모든 참여자에게 동일한 가격 메커니즘 적용

#### 6.2 컨트랙트 화이트리스트 (증인 합의)

**목적**: 악의적인 컨트랙트로부터 사용자 보호

**메커니즘**:
- 출금 컨트랙트는 증인 **과반수 승인** 필요 (50% + 1)
- 승인되지 않은 컨트랙트는 등록/사용 불가
- 증인은 언제든지 승인 철회 가능
- 승인이 과반 미만으로 떨어지면 자동 삭제

**보호 효과**:
```
시나리오: 악의적 증인이 가짜 컨트랙트 제안
→ 과반수 증인이 거부
→ 컨트랙트 승인 불가
→ 사용자가 해당 컨트랙트 사용 불가
→ 사용자 자금 보호 ✅
```

**검증 로직**:
```cpp
// 증인 서비스 등록 시
const auto& idx = db.get_index<withdraw_dollar_contract_index>()
                    .indices().get<by_chain_contract>();
auto itr = idx.find(boost::make_tuple(chain_id, contract_address));

FC_ASSERT(itr != idx.end(),
          "Contract is not approved by witness consensus");

FC_ASSERT(itr->approval_count >= required_approvals,
          "Contract does not have majority approval");
```

#### 6.3 예치금 추적 및 검증

**목적**: 충분한 예치금이 있는 증인만 선택하여 실행 실패 방지

**메커니즘**:
1. **주기적 보고**: 증인이 5분마다 외부 체인 예치금 상태를 Zattera 체인에 보고
2. **온체인 조회**: ZRC-9 `getWitnessDeposit(witness, token)` 함수로 실제 예치금 조회
3. **선택 시 검증**: 증인 선택 시 4가지 조건 확인
   - 토큰 지원 여부
   - 예치금 보고 존재
   - 예치금 충분성 (요청 금액 이상)
   - 보고 신선도 (1시간 이내)

**보호 효과**:
```
시나리오 1: 예치금 부족
→ 증인이 USDC 1000 ZBD equivalent 보고
→ 사용자가 2000 ZBD 출금 요청
→ 해당 증인은 선택 대상에서 제외
→ 충분한 예치금이 있는 다른 증인 선택 ✅

시나리오 2: 오래된 정보
→ 증인이 2시간 전에 예치금 보고
→ 1시간 유효기간 초과
→ 해당 증인은 선택 대상에서 제외
→ 최신 정보를 보고한 증인만 선택 ✅

시나리오 3: 보고와 실제 불일치
→ 증인이 5000 ZBD 보고했으나 실제는 1000 ZBD
→ 사용자 실행 시 외부 체인 컨트랙트에서 검증
→ "Insufficient deposit" 에러로 트랜잭션 실패
→ 사용자는 다른 서명으로 재시도 가능
→ 잘못된 정보 제공 증인은 평판 하락 ⚠️
```

**증인 인센티브**:
- 정확한 정보 제공 → 더 많은 출금 요청 할당
- 부정확한 정보 → 사용자 실행 실패 → 평판 하락 → 선택 확률 감소

**Plugin 구현**:
```cpp
void withdraw_dollar_plugin::report_deposits_periodically()
{
    // 5분마다 자동 실행
    for (각 등록된 컨트랙트) {
        for (각 지원 토큰) {
            // 1. 외부 체인에서 실제 예치금 조회
            uint256_t balance = query_deposit_balance(
                chain_id, contract, signer_address, token
            );

            // 2. Dollar로 계산
            deposits[token] = token_amount_to_dollar(balance);
        }

        // 3. Zattera 체인에 보고
        broadcast(report_withdraw_dollar_deposits_operation);
    }
}
```

#### 6.4 이중 출금 방지

**스마트 컨트랙트**:
```solidity
struct WithdrawalInfo {
    bool processed;
    uint256 blockNumber;
}
mapping(uint64 => WithdrawalInfo) public withdrawalInfo;

function withdraw(...) external {
    require(!withdrawalInfo[requestId].processed, "Already processed");
    // ...
    withdrawalInfo[requestId] = WithdrawalInfo({
        processed: true,
        blockNumber: block.number
    });
}
```

#### 6.5 서명 유효성 검증

Zattera 체인에서 EIP-712 서명을 검증하여 잘못된 서명 차단:

```cpp
bool valid = verify_eip712_signature(
    itr->chain_id,
    itr->contract_address,
    o.request_id,
    itr->recipient,
    itr->token,              // 토큰 주소
    itr->amount,
    itr->discount_rate,      // 할인율
    o.signature,
    contract_itr->signer_address
);
FC_ASSERT(valid, "Invalid signature");
```

**검증 내용**:
- EIP-712 서명 구조 검증 (domain + message)
- 서명자 주소가 등록된 signer_address와 일치하는지 확인
- 요청 ID, 수령자, 토큰, 금액, 할인율이 모두 서명에 포함됨

#### 6.6 서명 재사용 공격 방지 (Cross-Contract Replay Attack)

**문제**: 동일한 증인 서명을 다른 컨트랙트에서 재사용하는 공격

**공격 시나리오**:
```
1. 사용자가 Contract A (0xAAAA...)에 대한 출금 요청 생성
2. 증인이 EIP-712 서명 생성 및 제출
3. 악의적 사용자가 동일한 서명을 Contract B (0xBBBB...)에서 재사용 시도
4. Contract B도 동일한 증인을 사용하므로 서명이 유효해 보임
5. 이중 출금 발생 가능성 🚨
```

**해결 메커니즘**: EIP-712 도메인에 `verifyingContract` 필드 포함

**증인 서명 생성 시**:
```cpp
string generate_withdraw_signature(
    uint32_t chain_id,
    const string& contract_address,  // ✅ 사용자 요청의 컨트랙트 주소
    uint64_t request_id,
    const string& recipient,
    const string& token,              // ✅ 토큰 주소
    asset amount,
    uint16_t discount_rate            // ✅ 할인율
)
{
    // EIP-712 도메인 (ZRC-9 표준: name/version 생략)
    eip712::Domain domain = {
        .chainId = chain_id,
        .verifyingContract = contract_address  // ✅ 특정 컨트랙트에 바인딩
    };

    // 할인 적용된 금액 계산 및 18 decimals로 정규화
    share_type discounted_zbd = amount.amount.value * (ZATTERA_100_PERCENT - discount_rate) / ZATTERA_100_PERCENT;
    uint256_t discounted_amount_18 = uint256_t(discounted_zbd) * 1000000000000000;  // * 10^15

    eip712::Withdrawal message = {
        .requestId = request_id,
        .recipient = recipient,
        .token = token,                       // ✅ 토큰 주소 포함
        .amount = discounted_amount_18,       // ✅ 18 decimals 정규화
        .discountRate = discount_rate         // ✅ 할인율 포함
    };

    // 서명은 이 컨트랙트에서만 유효
    return eip712::sign(domain, message, get_my_eth_private_key());
}
```

**서명 검증 시**:
```cpp
bool verify_eip712_signature(
    uint32_t chain_id,
    const string& contract_address,  // ✅ 출금 요청의 컨트랙트 주소
    uint64_t request_id,
    const string& recipient,
    const string& token,              // ✅ 토큰 주소
    asset amount,
    uint16_t discount_rate,           // ✅ 할인율
    const string& signature,
    const string& signer_address
)
{
    // 동일한 컨트랙트 주소로 도메인 생성 (ZRC-9 표준: name/version 생략)
    eip712::Domain domain = {
        .chainId = chain_id,
        .verifyingContract = contract_address  // ✅ 동일한 주소로 검증
    };

    // 할인 적용된 금액 계산 및 18 decimals로 정규화 (생성 로직과 동일)
    share_type discounted_zbd = amount.amount.value * (ZATTERA_100_PERCENT - discount_rate) / ZATTERA_100_PERCENT;
    uint256_t discounted_amount_18 = uint256_t(discounted_zbd) * 1000000000000000;  // * 10^15

    eip712::Withdrawal message = {
        .requestId = request_id,
        .recipient = recipient,
        .token = token,                       // ✅ 토큰 주소 포함
        .amount = discounted_amount_18,       // ✅ 18 decimals 정규화
        .discountRate = discount_rate         // ✅ 할인율 포함
    };

    bytes32 digest = eip712::hash_typed_data(domain, message);
    string recovered_address = ecdsa::recover(digest, signature);

    return (recovered_address == signer_address);
}
```

**보안 보장**:
- Contract A용 서명은 Contract A에서만 유효
- Contract B용 서명은 Contract B에서만 유효
- 동일한 `requestId`라도 `verifyingContract`가 다르면 서명 해시가 달라짐
- **크로스 컨트랙트 재사용 불가능** ✅

**참고**: ZEP-9 섹션 7.1에서 스마트 컨트랙트 측 구현 참조

### 7. 경제적 인센티브

#### 7.1 증인 보상 메커니즘

증인이 출금 서명을 제공하고 완료 보고하면, **교환비율만큼 ZBD를 ZP(VESTS)로 변환**받습니다 (3.5일 후).
나머지 ZBD는 **영구 소각**되어 네트워크 디플레이션에 기여합니다.

**예시**:
```
사용자 출금 요청: 100 ZBD
증인 교환비율: 60% (6000 BPS)

1. 출금 요청 시:
   - 사용자 계정: 100 ZBD 차감
   - Virtual supply: 100 ZBD 즉시 소각

2. 증인 완료 보고 제출 시:
   - Virtual supply: 60 ZBD 재생성 (증인 보상분)
   - 증인 계정: 60 ZBD 지급
   - 증인의 60 ZBD → ZP 변환 예약 (3.5일 후 실행)
   - 나머지 40 ZBD는 이미 요청 시 영구 소각됨

3. 3.5일 후 (자동):
   - 증인 계정: 60 ZBD 차감
   - Virtual supply: 60 ZBD 소각
   - 60 ZBD → ZTR (84시간 평균 가격 사용)
   - ZTR → ZP (현재 vesting share price 사용)
   - 증인이 ZP 획득 → 투표권 증가

결과: 요청된 100 ZBD 전부 소각됨
      (요청 시 -100, 완료 시 +60, convert 시 -60 = net -100 ZBD)
```

**ZP 변환 공식** (2단계):
```cpp
// 1단계: ZBD → ZTR (3.5일 후, 평균 가격 사용)
asset exchange_amount = withdrawal_amount * exchange_rate / ZATTERA_100_PERCENT;
asset liquid_amount = exchange_amount * feed_history.current_median_history;

// 2단계: ZTR → ZP (즉시, 현재 가격 사용)
asset zp_amount = liquid_amount * gpo.get_vesting_share_price();
```

**왜 3.5일 지연?**
- **가격 조작 방지**: 단기 가격 스파이크 무력화
- **공정성**: 모든 ZBD 변환이 동일한 규칙 적용
- **일관성**: 기존 `convert` operation과 동일

#### 7.2 교환비율 경쟁

사용자가 지정한 출금 컨트랙트를 지원하는 증인 중 **가중 랜덤 선택**됩니다.

**선택 로직 (확률적)**:
```
같은 컨트랙트를 지원하는 증인들:
- Alice: 1000 (10%) → 가중치 9000 → 선택 확률 45% → 90% ZBD 소각
- Bob:   3000 (30%) → 가중치 7000 → 선택 확률 35% → 70% ZBD 소각
- Carol: 6000 (60%) → 가중치 4000 → 선택 확률 20% → 40% ZBD 소각

총 가중치: 20000
가중치 = ZATTERA_100_PERCENT - exchange_rate
```

**경쟁 역학**:
- 증인들은 **낮은 교환비율**로 경쟁 → 더 높은 선택 확률 → 더 많은 출금 처리
- 낮은 교환비율 = 더 많은 ZBD 소각 = 네트워크 디플레이션 기여
- 하지만 너무 낮으면 → 증인 보상 감소 → 수익성 저하
- 시장 균형점에서 최적 교환비율 형성
- **공정성**: 여러 증인에게 기회 분산, 독점 방지
- 증인은 여러 컨트랙트에 다양한 교환비율 설정 가능

#### 7.3 장기 인센티브 정렬

**ZP 획득의 장점**:

1. **투표권 증가**: 더 많은 ZP = 더 높은 증인 선출 가능성
2. **큐레이션 보상**: ZP로 콘텐츠 큐레이션 시 보상
3. **복리 효과**: ZP는 인플레이션으로 계속 증가
4. **장기 커밋먼트**: 13주 파워다운 → 장기 참여 유도

**증인 입장**:
- 낮은 교환비율 제시 → 더 많은 출금 요청 선택 → 거래 처리 기회 증가
- 더 많은 ZP 축적 → 투표권 강화 → 장기적 증인 지위 유지
- 여러 컨트랙트에 등록하여 다양한 시장 접근 가능
- 컨트랙트별로 최적 교환비율 전략 수립 가능

**네트워크 입장**:
- 낮은 교환비율 선호 자동화 → 더 많은 ZBD 소각
- 디플레이션 효과 → ZBD 가치 상승
- 증인들의 자유 경쟁으로 시장 균형 형성

#### 7.4 투명성과 신뢰

**누적 통계 추적**:
- 각 증인의 `total_withdrawal_amount`: 처리한 총 출금액 (교환비율 적용 전)
- `total_burned_amount`: 총 소각된 ZBD 금액 (네트워크 기여도)
- `total_withdrawal_count`: 총 처리 건수
- `last_withdrawal_time`: 마지막 처리 시각

**활용**:
- 사용자는 증인의 실적을 확인하고 신뢰도 판단
- 활발한 증인 = 높은 신뢰도 + 안정적 서비스
- 네트워크는 증인의 기여도를 투명하게 추적
- **소각 기여도**: `total_burned_amount`로 네트워크 디플레이션 기여 측정

### 8. CLI 월렛 통합

```bash
# 1단계: 증인들이 컨트랙트 승인 (과반수 필요)
unlocked >>> approve_withdrawal_contract alice 1 \
    0x1234567890abcdef1234567890abcdef12345678
# 파라미터: witness, chain_id, contract_address
# 의미: Alice가 Ethereum(chain_id=1)의 0x1234... 컨트랙트를 승인

# 다른 증인들도 동일하게 승인
unlocked >>> approve_withdrawal_contract bob 1 \
    0x1234567890abcdef1234567890abcdef12345678

unlocked >>> approve_withdrawal_contract carol 1 \
    0x1234567890abcdef1234567890abcdef12345678

# ... (과반수까지 승인)

# 승인된 컨트랙트 목록 조회
unlocked >>> list_approved_contracts 1
[
  {
    "chain_id": 1,
    "contract_address": "0x1234567890abcdef1234567890abcdef12345678",
    "approval_count": 15,
    "required_approvals": 11,
    "approving_witnesses": ["alice", "bob", "carol", ...],
    "approved_at": "2025-12-31T10:00:00"
  }
]

# 2단계: 승인된 컨트랙트에 증인이 서비스 등록
unlocked >>> register_withdrawal_contract alice 1 \
    0x1234567890abcdef1234567890abcdef12345678 \
    0x742d35Cc6634C0532925a3b844Bc454e4438f44e \
    3000 \
    ["0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48","0x6B175474E89094C44Da98b954EedeAC495271d0F"]
# 파라미터: witness, chain_id, contract_address, signer_address, exchange_rate, supported_tokens
# 의미: Alice는 승인된 0x1234... 컨트랙트에 대해 30% 교환비율로 서비스 제공
#       서명 주소는 0x742d... 사용 (exchange_rate: 3000 = 30%)
#       지원하는 토큰: USDC (0xA0b8...), DAI (0x6B17...)

# 사용자: ZBD 출금 요청 (컨트랙트 및 토큰 지정)
unlocked >>> withdraw_dollar bob "100.000 ZBD" 1 \
    0x1234567890abcdef1234567890abcdef12345678 \
    0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 \
    0x9876543210abcdef9876543210abcdef98765432
# 파라미터: from, amount, chain_id, contract_address, token, recipient
# 의미: 100 ZBD를 Ethereum의 0x1234... 컨트랙트를 통해 USDC로 0x9876...로 출금

# 출금 요청 조회
unlocked >>> get_withdraw_dollar_request 12345
{
  "request_id": 12345,
  "requester": "bob",
  "amount": "100.000 ZBD",           // CLI는 문자열 반환, JS에서는 파싱 필요
  "chain_id": 1,
  "contract_address": "0x1234567890abcdef1234567890abcdef12345678",
  "token": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",  // USDC
  "recipient": "0x9876543210abcdef9876543210abcdef98765432",
  "status": "signed",
  "assigned_witness": "alice",
  "witness_signature": "0x1234...",
  "exchange_amount": "30.000 ZBD",   // CLI는 문자열 반환
  "exchange_rate": 3000,
  "created_at": "2025-12-31T12:00:00",
  "signed_at": "2025-12-31T12:02:30"
}
# 참고: JSON-RPC API는 amount를 raw value (숫자)로 반환할 수 있음
#       - CLI: "100.000 ZBD" (문자열)
#       - JSON-RPC: 100000 (숫자, 3 decimals raw value)

# 증인의 컨트랙트 등록 상태 조회
unlocked >>> get_witness_withdrawal_contract alice 1 0x1234567890abcdef1234567890abcdef12345678
{
  "witness": "alice",
  "chain_id": 1,
  "contract_address": "0x1234567890abcdef1234567890abcdef12345678",
  "signer_address": "0x742d35Cc6634C0532925a3b844Bc454e4438f44e",
  "exchange_rate": 3000,
  "supported_tokens": [
    "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",  // USDC
    "0x6B175474E89094C44Da98b954EedeAC495271d0F"   // DAI
  ],
  "reported_deposits": {
    "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48": "5000.000 ZBD",  // USDC 예치금
    "0x6B175474E89094C44Da98b954EedeAC495271d0F": "3000.000 ZBD"   // DAI 예치금
  },
  "deposits_last_reported": "2025-12-31T12:00:00",
  "is_active": true,
  "registered_at": "2025-12-31T10:00:00",
  "total_withdrawal_amount": "15000.000 ZBD",
  "total_burned_amount": "10500.000 ZBD",
  "total_withdrawal_count": 150,
  "last_withdrawal_time": "2025-12-31T12:02:30",
  "burn_rate": "70.00%"
}

# 증인: 예치금 보고 (주기적으로 자동 실행)
unlocked >>> report_withdraw_dollar_deposits alice 1 \
    0x1234567890abcdef1234567890abcdef12345678 \
    {"0xA0b8...": "5000.000 ZBD", "0x6B17...": "3000.000 ZBD"}
# 파라미터: witness, chain_id, contract_address, deposits (토큰별 예치금, ZBD equivalent)

# 증인: 출금 완료 보고 (외부 체인에서 실행 확인 후)
unlocked >>> report_withdrawal_complete alice 12345 0xabcd1234567890...

# 특정 컨트랙트의 모든 증인 통계 조회
unlocked >>> list_contract_witnesses 1 0x1234567890abcdef1234567890abcdef12345678
[
  {
    "witness": "alice",
    "exchange_rate": 3000,
    "total_withdrawal_amount": "15000.000 ZBD",
    "total_burned_amount": "10500.000 ZBD",
    "total_withdrawal_count": 150,
    "average_per_withdrawal": "100.000 ZBD",
    "burn_rate": "70.00%",
    "last_withdrawal_time": "2025-12-31T12:02:30"
  },
  {
    "witness": "bob",
    "exchange_rate": 5000,
    "total_withdrawal_amount": "8000.000 ZBD",
    "total_burned_amount": "4000.000 ZBD",
    "total_withdrawal_count": 80,
    "average_per_withdrawal": "100.000 ZBD",
    "burn_rate": "50.00%",
    "last_withdrawal_time": "2025-12-31T11:50:00"
  }
]
```

## 하위 호환성

본 ZEP는 다음을 추가합니다:

**새로운 Operations** (하드포크 필요):
- `approve_withdraw_dollar_contract_operation` - 컨트랙트 승인/철회
- `update_withdraw_dollar_provider_operation` - 증인 서비스 등록/업데이트
- `report_withdraw_dollar_deposits_operation` - 예치금 보고 (주기적)
- `withdraw_dollar_request_operation` - 출금 요청
- `withdraw_dollar_approve_operation` - 출금 승인 (서명 제출)
- `complete_withdraw_dollar_operation` - 출금 완료 보고

**새로운 Database Objects**:
- `withdraw_dollar_contract_object` - 승인된 컨트랙트 화이트리스트
- `withdraw_dollar_provider_object` - 증인별 서비스 설정
- `withdraw_dollar_request_object` - 출금 요청 추적

**기존 Object 재사용**:
- `convert_request_object` - 증인 ZBD → ZP 변환 (기존 사용자 변환과 동일)

**새로운 Plugin**:
- `withdraw_dollar_plugin`

**보안 모델**:
- 출금 컨트랙트는 증인 과반수(50% + 1) 승인 필요
- 승인되지 않은 컨트랙트는 사용 불가
- 악의적 컨트랙트로부터 사용자 보호

기존 증인 시스템에는 영향을 주지 않으며, 출금 기능 참여는 선택 사항입니다.

## 구현 참고사항

### `withdraw_dollar_request_object` 메모리 관리 (`CLEAR_WITHDRAW_DOLLAR_REQUESTS`)

완료/만료된 출금 요청의 메모리 관리는 `CLEAR_VOTES` 패턴을 따릅니다.

**CMake 옵션**:
```cmake
OPTION( CLEAR_WITHDRAW_DOLLAR_REQUESTS "Clear completed/expired withdrawal requests from memory" ON )
```

**노드 타입별 동작**:

| 노드 타입 | 설정 | 동작 | 목적 |
|----------|------|------|------|
| Witness/Seed | `ON` (기본) | `completed`/`expired` 즉시 삭제 | 메모리 효율, 컨센서스 |
| Account History | `OFF` | 모든 요청 보존 | 완전한 출금 히스토리 제공 |

**구현 예시**:

```cpp
// complete_withdraw_dollar_evaluator.cpp
void complete_withdraw_dollar_evaluator::do_apply(const complete_withdraw_dollar_operation& o)
{
    // ... 증인 보상 및 변환 처리 ...

    db.modify(*itr, [&](auto& obj) {
        obj.status = withdraw_dollar_status::completed;
        obj.tx_hash = o.tx_hash;
        obj.executed_at = db.head_block_time();
    });

    // ... 통계 업데이트 ...

#ifdef CLEAR_WITHDRAW_DOLLAR_REQUESTS
    db.remove(*itr);  // 컨센서스 노드: 즉시 삭제
#endif
}

// withdraw_dollar_plugin.cpp - 타임아웃 처리
catch (const fc::exception& e)
{
    // 사용자에게 ZBD 환불
    db.adjust_balance(itr->requester, itr->amount);
    db.adjust_supply(itr->amount);

    // provider의 reserved_deposits 감소
    // ...

    elog("Request ${id} expired, refunded", ("id", itr->request_id));

#ifdef CLEAR_WITHDRAW_DOLLAR_REQUESTS
    db.remove(*itr);  // 컨센서스 노드: 즉시 삭제
#else
    db.modify(*itr, [&](auto& obj) {
        obj.status = withdraw_dollar_status::expired;
    });
#endif
}
```

**히스토리 조회 API** (Account History 노드 전용):
```cpp
vector<withdraw_dollar_request_object> get_withdrawal_history(
    account_name_type account,
    uint32_t limit = 100
)
{
#ifndef CLEAR_WITHDRAW_DOLLAR_REQUESTS
    // completed/expired 포함 전체 조회 가능
    const auto& idx = db.get_index<withdraw_dollar_request_index>()
                        .indices().get<by_requester>();
    // ...
#else
    FC_THROW_EXCEPTION(fc::assert_exception,
        "Withdrawal history not available on this node. "
        "Connect to an account history node.");
#endif
}
```

**메모리 절감 효과**:
- 컨센서스 노드: 활성 요청만 유지 (`queued`, `pending`, `signed`)
  - 예상: 300개 × 400 bytes = 120KB
- Account History 노드: 전체 히스토리 보존
  - 예상: 1년 누적 시 ~300MB

**EVM 컨트랙트와의 관계**:
- EVM의 `Withdrawn` 이벤트와 `WithdrawInfo` 매핑이 영구 히스토리 제공
- Zattera 체인은 활성 요청만 추적하여 메모리 효율 극대화
- 사용자는 Etherscan/Polygonscan에서 완전한 출금 이력 조회 가능

## 참조 구현

참조 구현은 다음 위치에 있습니다:

- `src/plugins/witness_withdrawal/` - Witness withdrawal plugin
- `src/core/chain/withdrawal_evaluator.cpp` - Withdrawal evaluators
- `src/core/chain/include/zattera/chain/withdrawal_objects.hpp` - Database objects
- `contracts/ZatteraBridge.sol` - Ethereum smart contract

## 저작권

본 문서는 CC0 1.0 Universal 라이선스에 따라 공개 도메인에 배치됩니다.
