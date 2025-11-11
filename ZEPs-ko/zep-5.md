---
zep: 5
title: 계정 보상 레벨 시스템
description: 증인 합의로 관리되는 사용자 정의 가능한 보상 승수를 갖춘 4단계 계정 분류 시스템
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2025-11-11
---

## 초록

본 ZEP는 계정 분류에 따라 차등적인 보상 분배를 가능하게 하는 4단계 보상 레벨 시스템(레벨 0-3)을 제안합니다. 계정 레벨 할당과 보상 승수 모두 탈중앙화된 증인 합의 메커니즘을 통해 제어되어 투명하고 책임감 있는 보상 거버넌스를 보장합니다.

## 동기

블록체인 보상 시스템은 전통적으로 참여자의 기여 품질, 스테이크 또는 네트워크 가치와 관계없이 모든 참여자에게 균일한 보상률을 적용합니다. 이러한 획일적 접근 방식은 네트워크의 다음과 같은 능력을 제한합니다:

1. **고품질 기여 인센티브화** - 뛰어난 콘텐츠 생성자와 큐레이터에게 보상
2. **전략적 참여자 유치** - 가치 있는 생태계 구성원에게 프리미엄 보상 제공
3. **유연한 경제학 구현** - 프로토콜 변경 없이 보상 구조 조정
4. **다양한 참여 수준 인정** - 네트워크 기여의 다양한 정도 인정

증인 거버넌스를 갖춘 계층화된 보상 시스템은 다음을 가능하게 합니다:
- 능력주의 보상 분배
- 보상 할당에 대한 탈중앙화 의사결정
- 빈번한 하드포크 없는 동적 경제 정책
- 투명하고 감사 가능한 레벨 할당

## 명세

### 보상 레벨

시스템은 네 개의 개별 보상 레벨을 정의합니다:

| 레벨 | 설명 | 기본 승수 |
|-------|-------------|-------------------|
| 0 | 기본 (기본값) | 100% (1.00x) |
| 1 | 향상 | 125% (1.25x) |
| 2 | 프리미엄 | 150% (1.50x) |
| 3 | 최대 | 200% (2.00x) |

### 핵심 작업

#### 1. 보상 레벨 제안 작업

```cpp
struct reward_level_proposal_operation
{
   account_name_type proposing_witness;  // 변경을 제안하는 증인
   account_name_type target_account;     // 조정할 계정
   uint8_t           new_reward_level;   // 제안된 레벨 (0-3)
   string            justification;      // 이유 (최대 1000자)

   extensions_type   extensions;

   void validate() const
   {
      FC_ASSERT(is_valid_account_name(proposing_witness), "Invalid witness account");
      FC_ASSERT(is_valid_account_name(target_account), "Invalid target account");
      FC_ASSERT(new_reward_level <= 3, "Level must be 0-3");
      FC_ASSERT(!justification.empty() && justification.size() <= 1000,
                "Justification required (max 1000 characters)");
   }

   void get_required_active_authorities(flat_set<account_name_type>& a) const
   {
      a.insert(proposing_witness);
   }
};
```

#### 2. 보상 레벨 승인 작업

```cpp
struct reward_level_approval_operation
{
   account_name_type              witness;
   reward_level_proposal_id_type  proposal_id;
   bool                           approve;  // true=승인, false=거부

   extensions_type                extensions;

   void validate() const
   {
      FC_ASSERT(is_valid_account_name(witness), "Invalid witness account");
   }

   void get_required_active_authorities(flat_set<account_name_type>& a) const
   {
      a.insert(witness);
   }
};
```

#### 3. 보상 승수 제안 작업

```cpp
struct reward_multiplier_proposal_operation
{
   account_name_type       proposing_witness;
   array<uint16_t, 4>      new_multipliers;  // 베이시스 포인트 (10000 = 100%)
   string                  justification;

   extensions_type         extensions;

   void validate() const
   {
      FC_ASSERT(is_valid_account_name(proposing_witness), "Invalid witness");
      FC_ASSERT(!justification.empty() && justification.size() <= 1000,
                "Justification required (max 1000 characters)");

      // 승수는 [10%, 500%] 범위 내여야 함
      for (size_t i = 0; i < 4; i++)
      {
         FC_ASSERT(new_multipliers[i] >= 1000 && new_multipliers[i] <= 50000,
                   "Multipliers must be between 1000 (10%) and 50000 (500%)");
      }

      // 단조 증가: 더 높은 레벨 >= 더 낮은 레벨
      for (size_t i = 1; i < 4; i++)
      {
         FC_ASSERT(new_multipliers[i] >= new_multipliers[i-1],
                   "Higher levels must have >= multipliers");
      }
   }

   void get_required_active_authorities(flat_set<account_name_type>& a) const
   {
      a.insert(proposing_witness);
   }
};
```

#### 4. 보상 승수 승인 작업

```cpp
struct reward_multiplier_approval_operation
{
   account_name_type                   witness;
   reward_multiplier_proposal_id_type  proposal_id;
   bool                                approve;

   extensions_type                     extensions;

   void validate() const
   {
      FC_ASSERT(is_valid_account_name(witness), "Invalid witness account");
   }

   void get_required_active_authorities(flat_set<account_name_type>& a) const
   {
      a.insert(witness);
   }
};
```

### 데이터베이스 객체

#### 계정 객체 확장

```cpp
// account_object 클래스 내
class account_object
{
   // ... 기존 필드 ...

   uint8_t reward_level = 0;  // 기본값: 레벨 0

   // ... 나머지 객체 ...
};
```

#### 보상 레벨 제안 객체

```cpp
class reward_level_proposal_object
{
   reward_level_proposal_id_type  id;
   account_name_type              target_account;
   uint8_t                        proposed_level;
   account_name_type              proposing_witness;
   time_point_sec                 created;
   string                         justification;
   flat_set<account_name_type>    approvals;  // 승인한 증인들
};

typedef multi_index_container<
   reward_level_proposal_object,
   indexed_by<
      ordered_unique<tag<by_id>, member<...>>,
      ordered_non_unique<tag<by_target_account>, member<...>>,
      ordered_non_unique<tag<by_created>, member<...>>
   >
> reward_level_proposal_index;
```

#### 보상 승수 객체

```cpp
class reward_multiplier_object
{
   reward_multiplier_id_type   id;
   array<uint16_t, 4>          multipliers = {
      10000,  // 레벨 0: 100%
      12500,  // 레벨 1: 125%
      15000,  // 레벨 2: 150%
      20000   // 레벨 3: 200%
   };
};
```

#### 보상 승수 제안 객체

```cpp
class reward_multiplier_proposal_object
{
   reward_multiplier_proposal_id_type  id;
   array<uint16_t, 4>                  proposed_multipliers;
   account_name_type                   proposing_witness;
   time_point_sec                      created;
   string                              justification;
   flat_set<account_name_type>         approvals;
};
```

### 합의 메커니즘

#### 증인 검증

모든 제안 및 승인 작업은 다음을 요구합니다:
1. 제안자/승인자는 활성 증인이어야 함
2. 증인은 유효한 `signing_key`를 가져야 함 (`public_key_type()`이 아님)

```cpp
const auto& witness = _db.get_witness(o.proposing_witness);
FC_ASSERT(witness.signing_key != public_key_type(), "Not an active witness");
```

#### 승인 임계값

레벨 변경과 승수 업데이트 모두 활성 증인의 **2/3 상위다수**를 요구합니다:

```cpp
// 활성 증인 수 계산 (상위 21명)
const auto& witness_idx = _db.get_index<witness_index>().indices().get<by_vote_name>();
uint32_t total_active = 0;
for (auto itr = witness_idx.begin();
     itr != witness_idx.end() && total_active < ZATTERA_MAX_WITNESSES;
     ++itr)
{
   if (itr->signing_key != public_key_type())
      total_active++;
}

const uint32_t required = (total_active * 2 / 3) + 1;  // 67% + 1

if (proposal.approvals.size() >= required)
{
   // 변경 적용 및 제안 제거
}
```

#### 자동 승인

제안하는 증인은 자동으로 자신의 제안을 승인합니다:

```cpp
_db.create<reward_level_proposal_object>([&](auto& p) {
   // ... 필드 설정 ...
   p.approvals.insert(o.proposing_witness);  // 자동 승인
});
```

### 보상 계산 통합

#### 승수 검색

```cpp
inline uint16_t get_reward_multiplier(const database& db, uint8_t level)
{
   FC_ASSERT(level <= 3, "Invalid reward level");
   const auto& multiplier_obj = db.get<reward_multiplier_object>();
   return multiplier_obj.multipliers[level];
}
```

#### 보상 적용

모든 보상 계산은 승수를 적용하도록 수정됩니다:

```cpp
// 저자 보상
asset calculate_author_reward(const database& db,
                               const account_object& author,
                               asset base_reward)
{
   uint16_t multiplier = get_reward_multiplier(db, author.reward_level);
   return asset((base_reward.amount.value * multiplier) / 10000,
                base_reward.symbol);
}

// 큐레이션 보상
asset calculate_curation_reward(const database& db,
                                 const account_object& curator,
                                 asset base_reward)
{
   uint16_t multiplier = get_reward_multiplier(db, curator.reward_level);
   return asset((base_reward.amount.value * multiplier) / 10000,
                base_reward.symbol);
}
```

### API 메서드

```cpp
// 계정 보상 레벨 가져오기
struct get_reward_level_return
{
   uint8_t  reward_level;
   uint16_t reward_multiplier;  // 베이시스 포인트 단위의 현재 승수
};

get_reward_level_return get_reward_level(account_name_type account);

// 현재 승수 가져오기
struct get_reward_multipliers_return
{
   array<uint16_t, 4> multipliers;
};

get_reward_multipliers_return get_reward_multipliers();

// 대기 중인 레벨 제안 가져오기
struct get_reward_level_proposals_return
{
   vector<reward_level_proposal_api_object> proposals;
};

get_reward_level_proposals_return get_reward_level_proposals(
   optional<account_name_type> account);

// 대기 중인 승수 제안 가져오기
struct get_reward_multiplier_proposals_return
{
   vector<reward_multiplier_proposal_api_object> proposals;
};

get_reward_multiplier_proposals_return get_reward_multiplier_proposals();
```

## 근거

### 증인 거버넌스

**이해관계자 투표 대신 증인 합의를 사용하는 이유는?**

1. **전문성**: 증인들은 깊은 기술적 및 경제적 이해를 가지고 있음
2. **책임성**: 증인들은 선출되며 잘못된 결정에 대해 투표 취소될 수 있음
3. **효율성**: 전체 이해관계자 투표에 비해 빠른 의사결정
4. **선례**: 기존 증인이 관리하는 매개변수(블록 크기, 수수료 등)와 일치

### 상위다수 요구사항

**단순 다수 대신 2/3 다수를 요구하는 이유는?**

1. **성급한 변경 방지**: 광범위한 합의 필요
2. **소수 보호**: 11/21 증인이 지배하는 것을 방지
3. **비잔틴 장애 허용과 일치**: 분산 합의의 표준
4. **업계 표준**: 블록체인 거버넌스에서 일반적 (Cosmos, Polkadot 등)

### 4단계 시스템

**더 세분화된 계층 대신 4단계를 사용하는 이유는?**

1. **단순성**: 이해하고 전달하기 쉬움
2. **의미 있는 차별화**: 각 계층은 중요한 차이를 나타냄
3. **가스 효율성**: 최소 저장 오버헤드 (계정당 1바이트)
4. **유연성**: 레벨 추가 없이 승수 조정 가능

### 불변 정당화

**영구적인 온체인 정당화를 요구하는 이유는?**

1. **투명성**: 모든 결정이 공개적으로 감사 가능
2. **책임성**: 증인들은 자신의 추론을 설명해야 함
3. **역사적 기록**: 거버넌스 결정의 미래 분석
4. **사기 방지**: 비공개 조정 방지

### 동적 승수

**하드포크 없이 승수 변경을 허용하는 이유는?**

1. **경제적 유연성**: 변화하는 네트워크 조건에 적응
2. **실험**: 다양한 보상 구조 테스트
3. **하드포크 빈도 감소**: 파괴적인 업그레이드 감소
4. **대응적 거버넌스**: 위기 상황에서의 빠른 조정

## 하위 호환성

본 ZIP은 하드포크가 필요한 합의 파괴 변경 사항을 도입합니다.

### 마이그레이션 전략

**1단계: 하드포크 활성화**
- 모든 계정은 레벨 0으로 기본 설정
- 기본 승수 활성화: [100%, 125%, 150%, 200%]
- 제안/승인 작업이 유효해짐

**2단계: 초기 레벨 할당 (선택 사항)**
- 제네시스 블록은 특정 계정에 레벨을 미리 할당할 수 있음
- 커뮤니티 논의 및 하드포크 전 합의 필요

**3단계: 지속적인 거버넌스**
- 증인들이 레벨 변경 제안 시작
- 커뮤니티가 결정 모니터링 및 평가

### 파괴적 변경사항

1. **보상 계산**: 모든 보상 계산이 영향을 받음
2. **계정 객체 스키마**: 새로운 `reward_level` 필드 추가
3. **API 응답**: 계정 쿼리에 새로운 필드
4. **데이터베이스 인덱스**: 새로운 제안 추적 인덱스

### 비파괴적 요소

- 기존 보상 알고리즘은 계속 작동 (승수 = 1.0)
- 기존 작업의 트랜잭션 형식 변경 없음
- 계정 잔액 및 기록 유지

## 테스트 케이스

### 테스트 케이스 1: 레벨 제안 생성

```cpp
BOOST_AUTO_TEST_CASE(reward_level_proposal_create)
{
   ACTORS((alice)(witness1))

   // witness1을 활성화
   witness_create("witness1", witness1_key, "url", witness1_pub, 0);

   // alice가 레벨 0에서 시작하는지 확인
   const auto& alice_account = db->get_account("alice");
   BOOST_REQUIRE_EQUAL(alice_account.reward_level, 0);

   // 증인이 레벨 2를 제안
   reward_level_proposal_operation op;
   op.proposing_witness = "witness1";
   op.target_account = "alice";
   op.new_reward_level = 2;
   op.justification = "Outstanding contributor";

   push_transaction(op, witness1_key);

   // 제안이 생성되었는지 확인
   const auto& idx = db->get_index<reward_level_proposal_index>()
                         .indices().get<by_target_account>();
   auto proposal = idx.find("alice");
   BOOST_REQUIRE(proposal != idx.end());
   BOOST_REQUIRE_EQUAL(proposal->proposed_level, 2);
   BOOST_REQUIRE_EQUAL(proposal->approvals.size(), 1);  // 자동 승인됨
}
```

### 테스트 케이스 2: 합의 달성

```cpp
BOOST_AUTO_TEST_CASE(reward_level_consensus)
{
   // 21명의 증인 생성
   ACTORS((alice)(w1)(w2)(w3)...(w21))

   for (auto wit : witnesses) {
      witness_create(wit, ...);
   }

   // Witness1이 alice -> 레벨 3을 제안
   reward_level_proposal_operation prop;
   prop.proposing_witness = "w1";
   prop.target_account = "alice";
   prop.new_reward_level = 3;
   prop.justification = "Exceptional contributions";
   push_transaction(prop, w1_key);

   auto proposal_id = /* 제안 ID 가져오기 */;

   // 초기에는 alice가 여전히 레벨 0
   BOOST_REQUIRE_EQUAL(db->get_account("alice").reward_level, 0);

   // 13명의 증인이 추가로 승인 (총 14/21 = 67%)
   for (int i = 2; i <= 14; i++) {
      reward_level_approval_operation approve;
      approve.witness = witnesses[i];
      approve.proposal_id = proposal_id;
      approve.approve = true;
      push_transaction(approve, witness_keys[i]);
   }

   // 이제 alice는 레벨 3이어야 함
   BOOST_REQUIRE_EQUAL(db->get_account("alice").reward_level, 3);

   // 제안은 제거되어야 함
   const auto& idx = db->get_index<reward_level_proposal_index>()
                         .indices().get<by_target_account>();
   BOOST_REQUIRE(idx.find("alice") == idx.end());
}
```

### 테스트 케이스 3: 승수 업데이트

```cpp
BOOST_AUTO_TEST_CASE(reward_multiplier_update)
{
   // 증인 생성
   ACTORS((w1)(w2)...(w21))

   // 초기 승수 가져오기
   const auto& mult = db->get<reward_multiplier_object>();
   BOOST_REQUIRE_EQUAL(mult.multipliers[0], 10000);  // 100%
   BOOST_REQUIRE_EQUAL(mult.multipliers[3], 20000);  // 200%

   // 증인이 새로운 승수 제안
   reward_multiplier_proposal_operation prop;
   prop.proposing_witness = "w1";
   prop.new_multipliers = {10000, 13000, 16000, 25000};
   prop.justification = "Adjusting economic incentives";
   push_transaction(prop, w1_key);

   // 합의 도달 (14개 승인)
   // ... 승인 작업 ...

   // 승수가 업데이트되었는지 확인
   BOOST_REQUIRE_EQUAL(mult.multipliers[1], 13000);  // 130%
   BOOST_REQUIRE_EQUAL(mult.multipliers[3], 25000);  // 250%
}
```

### 테스트 케이스 4: 보상 적용

```cpp
BOOST_AUTO_TEST_CASE(reward_multiplier_application)
{
   ACTORS((alice)(bob))

   // alice를 레벨 0으로, bob을 레벨 3으로 설정
   // (합의 프로세스를 통해)

   // 둘 다 동일한 콘텐츠를 생성하고 투표를 받음
   // ... 콘텐츠 생성 및 투표 ...

   // 예상 보상 계산
   asset base_reward = ASSET("100.000 TESTS");

   // Alice (레벨 0, 100% 승수)
   asset alice_expected = base_reward;

   // Bob (레벨 3, 200% 승수)
   asset bob_expected = ASSET("200.000 TESTS");

   // 보상이 올바르게 분배되었는지 확인
   BOOST_REQUIRE_EQUAL(alice_account.reward_sbd_balance, alice_expected);
   BOOST_REQUIRE_EQUAL(bob_account.reward_sbd_balance, bob_expected);
}
```

## 참조 구현

참조 구현은 기능 브랜치에서 사용 가능합니다:

- 브랜치: `feature/reward-level-system`
- 주요 파일:
  - `libraries/protocol/include/zattera/protocol/zattera_operations.hpp`
  - `libraries/chain/include/zattera/chain/reward_level_objects.hpp`
  - `libraries/chain/zattera_evaluator.cpp`
  - `libraries/chain/database.cpp`
  - `tests/tests/reward_level_tests.cpp`

## 보안 고려사항

### 거버넌스 공격

**증인 담합**
- 위험: 악의적인 증인들이 공모자에게 높은 레벨 할당
- 완화:
  - 2/3 상위다수 요구사항
  - 공개 정당화로 커뮤니티 감시 가능
  - 증인은 남용 시 투표 취소 가능

**증인 투표에 대한 시빌 공격**
- 위험: 공격자가 증인 의석 통제권 획득
- 완화: 14개 이상의 증인 위치를 통제하려면 막대한 스테이크 필요
- 탐지: 커뮤니티가 비정상적인 증인 투표 패턴 모니터링

### 경제적 공격

**인플레이션 조작**
- 위험: 높은 레벨 계정이 보상 풀을 부풀림
- 완화: 승수는 하드 캡 보유 (최대 500%)
- 모니터링: 레벨별 총 보상 분배 추적

**레벨 파밍**
- 위험: 사용자가 높은 레벨을 얻기 위해 시스템 악용
- 완화: 증인 판단 + 정당화 요구사항
- 미래: 자동화된 자격 기준 (ZIP 확장)

### 기술적 취약점

**제안 ID 고갈**
- 위험: 제안 ID가 유한함
- 완화: 64비트 ID 사용 (2^64개의 제안 가능)
- 정리: 만료된 제안 정리 가능

**승수 오버플로우**
- 위험: 큰 승수가 산술 오버플로우 발생
- 완화:
  - 검증이 최대 50000 (500%) 강제
  - 보상 계산에 64비트 산술 사용
  - 오버플로우 확인 어설션

**동시 제안**
- 위험: 동일한 계정에 대한 여러 제안
- 완화: 계정당 하나의 활성 제안만 허용
- 강제: 제안 생성 시 기존 제안 확인

## 저작권

저작권 및 관련 권리는 [CC0](https://creativecommons.org/publicdomain/zero/1.0/)를 통해 포기되었습니다.
