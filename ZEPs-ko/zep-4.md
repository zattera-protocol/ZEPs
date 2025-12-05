---
zep: 4
title: 네트워크 참여율 기반 보상 조정
description: 초기 단계 네트워크에서 과도한 보상을 방지하기 위해 네트워크 투표 참여율에 비례하여 콘텐츠 보상을 조정
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2025-11-11
---

## 개요

본 ZEP는 실제 네트워크 투표 참여율을 기반으로 콘텐츠 보상을 비례적으로 조정하는 메커니즘을 제안합니다. 총 투표 참여량이 설정 가능한 임계값 이하로 떨어지면 콘텐츠 보상이 비례적으로 감소하며, 나머지는 모든 이해관계자에게 분배되는 베스팅 풀에 추가됩니다. 이는 낮은 활동 기간 동안 불균형한 보상 추출을 방지하면서 유기적인 네트워크 성장을 장려합니다.

## 동기

블록체인 네트워크 출시 초기 단계에서는 소수의 사용자가 제한된 투표력으로 성숙한 네트워크를 위해 설계된 보상 풀에서 불균형하게 큰 보상을 받을 수 있습니다. 이는 다음과 같은 문제를 야기합니다:

1. **불공정한 보상 분배** - 소규모 투표 활동이 성숙한 네트워크를 위한 전체 보상을 받음
2. **경제적 불균형** - 얼리어답터가 최소한의 스테이크 참여로 과도한 가치를 추출할 수 있음
3. **네트워크 건전성** - 보상은 실제 네트워크 참여도와 사용을 반영해야 함

기존 솔루션은 이러한 문제를 적절히 해결하지 못합니다. 이 메커니즘이 없으면 초기 단계 네트워크는 보상 풀 조작과 장기적 지속 가능성을 해칠 수 있는 경제적 불균형에 취약합니다.

## 명세

### 개요

시스템은 모든 블록에서 참여율을 계산하고 네트워크 참여가 지정된 임계값 이하로 떨어질 때 콘텐츠 보상에 보상 감소율을 적용합니다.

### 주요 용어

- **총 네트워크 VESTS**: 네트워크 내 모든 베스팅 지분(스테이킹된 토큰)의 합계
- **총 투표 Rshares**: 현재 블록에서 보상받는 콘텐츠의 모든 보상 지분(투표 가중치) 합계
- **투표 임계값 비율**: 전체 보상을 위한 필수 참여율을 정의하는 체인 매개변수
- **참여율**: `total_voting_rshares / (total_network_vests × percent_network_voting_threshold)`
- **보상 감소율**: 콘텐츠 보상에 적용되는 승수 (0.0~1.0)

### 알고리즘

보상 분배 중 각 블록마다:

```
1. 참여율 계산:
   participation_ratio = total_voting_rshares / (total_network_vests × percent_network_voting_threshold)

2. 보상 감소율 결정:
   if participation_ratio >= 1.0:
       reduction_factor = 1.0  // 전체 보상, 감소 없음
   else:
       reduction_factor = participation_ratio  // 비례 감소

3. 보상 분배:
   actual_content_reward = original_reward × reduction_factor
   vesting_pool_bonus = original_reward × (1 - reduction_factor)
```

### 체인 매개변수

새로운 체인 매개변수 `percent_network_voting_threshold` 도입:

- **타입**: `uint16_t` (베이시스 포인트, 10000 = 100%)
- **기본값**: 10000 (100% - 제네시스에서 전체 네트워크 참여 필요)
- **범위**: 0~10000
- **수정 방법**: 증인 합의 또는 하드포크
- **권장값**:
  - 제네시스 출시: 5000-10000 (50-100%)
  - 초기 성장: 1000-5000 (10-50%)
  - 중기 단계: 100-1000 (1-10%)
  - 성숙한 네트워크: 10-100 (0.1-1%) 또는 0 (비활성화)

### 구현 요구사항

#### 1. 프로토콜 계층 변경

**파일**: `src/core/protocol/include/steem/protocol/chain_parameters.hpp`

체인 매개변수 추가:
```cpp
struct chain_parameters
{
   // ... 기존 매개변수 ...
   uint16_t percent_network_voting_threshold = 10000;  // 100% 기본값
};
```

#### 2. 데이터베이스 수정

**파일**: `src/core/chain/database.cpp`

`process_funds()` 함수 수정:

```cpp
void database::process_funds()
{
   // ... 기존 보상 풀 계산 ...

   // 현재 보상 기간의 총 투표 rshares 계산
   const auto& reward_idx = get_index<reward_fund_index>();
   const auto& reward_fund = reward_idx.get();

   share_type total_voting_rshares = reward_fund.recent_claims;

   // 총 네트워크 VESTS 가져오기
   const auto& dgpo = get_dynamic_global_properties();
   share_type total_vests = dgpo.total_vesting_shares.amount;

   // 임계값 매개변수 가져오기
   const auto& chain_params = get_witness_schedule_object().median_props;
   uint16_t threshold = chain_params.percent_network_voting_threshold;

   // 참여율 계산
   // participation_ratio = total_voting_rshares / (total_vests × threshold / 10000)
   fc::uint128_t required_rshares = fc::uint128_t(total_vests.value) * threshold / 10000;

   uint16_t reduction_factor = 10000; // 기본값: 감소 없음 (100%)

   if (required_rshares > 0)
   {
      fc::uint128_t ratio = (fc::uint128_t(total_voting_rshares.value) * 10000) / required_rshares;

      if (ratio < 10000)
      {
         reduction_factor = static_cast<uint16_t>(ratio.to_uint64());
      }
   }

   // 콘텐츠 보상에 감소율 적용
   asset original_content_reward = /* 계산된 보상 */;
   asset actual_content_reward = asset(
      (original_content_reward.amount.value * reduction_factor) / 10000,
      original_content_reward.symbol
   );

   asset vesting_pool_bonus = asset(
      original_content_reward.amount.value - actual_content_reward.amount.value,
      original_content_reward.symbol
   );

   // actual_content_reward를 콘텐츠 제작자에게 분배
   // ... 기존 보상 분배 로직 ...

   // vesting_pool_bonus를 베스팅 풀에 추가
   if (vesting_pool_bonus.amount > 0)
   {
      modify(dgpo, [&](dynamic_global_property_object& p) {
         p.total_vesting_fund_zattera += vesting_pool_bonus;
      });
   }
}
```

#### 3. 데이터베이스 객체

`dynamic_global_property_object`에서 임계값 통계 추적:

```cpp
class dynamic_global_property_object
{
   // ... 기존 필드 ...

   // 투표 임계값 통계
   uint32_t blocks_below_threshold = 0;
   asset total_rewards_redirected = asset(0, ZATTERA_SYMBOL);
   fc::uint128_t last_participation_ratio = 0;  // 베이시스 포인트 (10000 = 100%)
};
```

### API 확장

`database_api`에 새로운 API 메서드 추가:

```cpp
struct get_voting_threshold_info_return
{
   asset total_network_vests;
   fc::uint128_t total_voting_rshares;
   uint16_t percent_network_voting_threshold;
   uint16_t participation_ratio;          // 베이시스 포인트
   uint16_t current_reduction_factor;      // 베이시스 포인트
   uint32_t blocks_below_threshold;
   asset total_rewards_redirected;
};

get_voting_threshold_info_return get_voting_threshold_info();
```

## 근거

### 설계 결정

1. **선형 스케일링**: 보상 감소율은 단순성과 예측 가능성을 위해 참여율에 선형적으로 비례합니다. 비선형 곡선도 고려되었으나 복잡성 증가로 인해 거부되었습니다.

2. **베스팅 풀 분배**: 재분배된 보상은 소각되는 대신 베스팅 풀로 전달됩니다:
   - 베스팅 풀 가치 증가를 통해 모든 이해관계자에게 비례적으로 혜택 제공
   - 총 인플레이션율 예측 가능성 유지
   - 장기 보유 및 네트워크 헌신 인센티브화

3. **백분율로서의 임계값**: 총 VESTS의 백분율 사용은 메커니즘이 수동 재보정 없이 네트워크 성장에 따라 자연스럽게 확장되도록 합니다.

4. **참여율 상한 없음**: 100% 이상의 참여율은 보너스 보상을 제공하지 않아 인위적인 투표 활동 부풀리기를 통한 게이밍을 방지합니다.

### 고려된 대안 접근법

1. **절대 임계값**: 백분율 대신 고정 rshares 요구사항 - 네트워크 성장에 따라 확장되지 않아 거부됨

2. **시간 기반 감소**: 시간 경과에 따라 요구사항 점진적 감소 - 실제 사용을 반영하지 않아 거부됨

3. **보상 소각**: 감소된 부분을 베스팅 풀에 주는 대신 소각 - 디플레이션 압력을 생성하고 이해관계자에게 혜택을 주지 않아 거부됨

4. **이차 스케일링**: 비선형 감소 곡선 - 복잡성과 가스 비용으로 인해 거부됨

## 하위 호환성

본 제안은 합의를 깨는 변경을 도입하며 하드포크가 필요합니다.

### 마이그레이션 경로

1. **하드포크 이전**: `percent_network_voting_threshold = 0`으로 설정하여 메커니즘 비활성화
2. **하드포크 활성화**: 보수적인 임계값으로 활성화 (예: 10000 = 100%)
3. **하드포크 이후**: 네트워크 성숙에 따라 증인 합의를 통해 점진적으로 임계값 감소

### 기존 기능에 미치는 영향

- **보상 계산**: 모든 콘텐츠 보상 계산이 영향을 받음
- **베스팅 풀**: 베스팅 풀은 낮은 활동 기간 동안 추가 변동 보상을 받음
- **API 응답**: 보상 관련 API 호출이 조정된 금액을 표시함
- **지갑 디스플레이**: UI는 변동 보상율을 고려해야 함

## 테스트 케이스

### 테스트 케이스 1: 임계값 미만 (30% 참여)

```cpp
BOOST_AUTO_TEST_CASE( reward_reduction_below_threshold )
{
   // 설정
   total_network_vests = 1,000,000 VESTS
   percent_network_voting_threshold = 10000 (100%)
   total_voting_rshares = 300,000 (총 VESTS의 30%)
   original_reward = 100 ZATTERA

   // 예상
   participation_ratio = 300,000 / 1,000,000 = 0.3
   reduction_factor = 0.3
   actual_content_reward = 100 * 0.3 = 30 ZATTERA
   vesting_pool_bonus = 100 * 0.7 = 70 ZATTERA
}
```

### 테스트 케이스 2: 임계값 초과 (150% 참여)

```cpp
BOOST_AUTO_TEST_CASE( no_reduction_above_threshold )
{
   // 설정
   total_network_vests = 100,000,000 VESTS
   percent_network_voting_threshold = 1000 (10%)
   required_voting = 10,000,000 VESTS
   total_voting_rshares = 15,000,000 (필요량의 150%)
   original_reward = 100 ZATTERA

   // 예상
   participation_ratio = 15,000,000 / 10,000,000 = 1.5 (1.0으로 제한)
   reduction_factor = 1.0
   actual_content_reward = 100 ZATTERA (감소 없음)
   vesting_pool_bonus = 0 ZATTERA
}
```

### 테스트 케이스 3: 정확히 임계값

```cpp
BOOST_AUTO_TEST_CASE( exact_threshold_boundary )
{
   // 설정
   total_network_vests = 1,000,000 VESTS
   percent_network_voting_threshold = 5000 (50%)
   required_voting = 500,000 VESTS
   total_voting_rshares = 500,000 (정확히 필요량의 100%)
   original_reward = 100 ZATTERA

   // 예상
   participation_ratio = 500,000 / 500,000 = 1.0
   reduction_factor = 1.0
   actual_content_reward = 100 ZATTERA
   vesting_pool_bonus = 0 ZATTERA
}
```

## 참조 구현

완전한 참조 구현은 기능 브랜치에서 사용 가능합니다:

- 브랜치: `feature/voting-threshold-reward-reduction`
- 주요 파일:
  - `src/core/protocol/include/steem/protocol/chain_parameters.hpp`
  - `src/core/chain/database.cpp` (process_funds 함수)
  - `src/core/chain/include/steem/chain/global_property_object.hpp`
  - `tests/tests/reward_tests.cpp`

## 보안 고려사항

### 잠재적 공격 벡터

1. **임계값 게이밍**: 악의적인 행위자들이 참여를 임계값 바로 아래로 유지하도록 조율할 수 있음
   - 완화: 참여율에 의미 있는 영향을 주려면 상당한 스테이크가 필요함
   - 모니터링: 참여율의 급격한 하락 추적

2. **임계값 조작**: 악의적인 행위자들이 임계값 설정을 조작하려 시도할 수 있음
   - 완화: 임계값 조정은 증인 합의가 필요함
   - 투명성: 모든 매개변수 변경은 온체인에서 공개적으로 확인 가능함

3. **보상 계산 오버플로**: 큰 VESTS 값이 비율 계산에서 오버플로를 발생시킬 수 있음
   - 완화: 중간 계산에 `fc::uint128_t` 사용
   - 검증: 중요 경로에 오버플로 검사 추가

### 경제적 보안

1. **인플레이션 일관성**: 총 보상 분배(콘텐츠 + 베스팅 풀 보너스)는 일정하게 유지되어 예측 가능한 인플레이션 유지
2. **소각 없음**: 모든 보상이 참여자에게 분배되어 디플레이션 압력 방지
3. **점진적 전환**: 권장되는 임계값 감소 일정이 경제적 충격 방지

## 저작권

저작권 및 관련 권리는 [CC0](https://creativecommons.org/publicdomain/zero/1.0/)를 통해 포기됩니다.
