---
zep: 7
title: 투표 파워 정밀도 증가
description: 투표 파워 필드를 uint16_t에서 uint32_t로 변경하고 정밀도를 10,000에서 100,000,000으로 증가
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2025-12-19
---

## 초록

본 ZEP는 투표 파워(voting power) 시스템의 정밀도를 대폭 향상시키는 제안입니다. 현재 `uint16_t` 타입과 10,000 정밀도(ZATTERA_100_PERCENT)를 사용하는 투표 파워를 `uint32_t` 타입과 100,000,000 정밀도로 변경합니다. 이를 통해 저가중치 투표의 정밀한 처리, 고래 계정의 세밀한 투표 조절, 그리고 제3자 애플리케이션의 마이크로 투표 활용을 가능하게 합니다.

## 동기

### 현재 시스템의 한계

Zattera의 현재 투표 시스템은 다음과 같은 제약이 있습니다:

**현재 설정:**
- 투표 파워 타입: `uint16_t` (2바이트)
- 정밀도: `ZATTERA_100_PERCENT = 10,000` (0.01% 단위)
- 최소 VP 소비: 1 (0.01% VP)
- Vote reserve rate: 40 (초기) → 10 (감소)
- Vote regeneration: 5일

**핵심 문제점:**

#### 1. 저가중치 투표의 반올림 문제

100% VP에서 100% 가중치 투표 시 0.5% VP를 소비합니다. 이로 인해:

투표 가중치 | VP 소비량 (계산) | VP 소비량 (실제) | 반올림 효과
-----------|----------------|----------------|------------
2%         | 0.01%          | 0.01%          | 정확
1%         | 0.005%         | 0.01%          | 2배 과다
0.5%       | 0.0025%        | 0.01%          | 4배 과다

**결과**: 2% 이하 가중치의 모든 투표가 2% 투표와 동일하게 처리됨

#### 2. 일일 투표 횟수 제한

현재 시스템에서 2% 가중치로만 투표 시:
- 하루 최대 투표: 2,000표
- 하루 최소 투표: 50표 (100% 가중치)

**문제**:
- 큐레이션 봇이나 자동화 서비스가 제한에 걸림
- 내부 플래그나 마이크로 보상용 투표 불가능
- 제3자 애플리케이션의 확장성 저해

#### 3. 고래 계정의 투표 조절 불가

대규모 ZP(Zattera Power) 보유 계정의 경우:

계정 ZP      | 최소 rshares | 최소 유효 투표 가중치
-----------|-------------|-------------------
1,000,000   | ~0          | 2% 이상 필요
10,000,000  | 1,000       | 1% 가능
1,000,000,000 | 99,000    | 0.01% 가능하나 여전히 과다

**문제**:
- 고래가 소액 보상을 주고 싶어도 최소 투표량이 너무 큼
- 세밀한 큐레이션 불가능
- 항상 "과도한 영향력" 행사

#### 4. 제3자 애플리케이션 제약

소셜 플랫폼에서 투표를 활용하려는 앱들의 사례:

**스팸 탐지 봇**:
- 의심도에 따라 0.1%~5% 차등 플래그 필요
- 현재: 2% 미만은 모두 동일하게 처리

**포럼 플랫폼**:
- 게시물 품질에 따라 세밀한 보상 조절 필요
- 현재: 하루 2,000표 제한으로 대규모 커뮤니티 지원 불가

**큐레이션 서비스**:
- 콘텐츠 점수에 비례한 정밀 보상 배분
- 현재: 정밀도 부족으로 공정한 배분 불가능

### 정밀도 요구사항 계산

필요한 최소 정밀도를 수학적으로 도출:

최악의 경우: 0.01% VP × 0.5% 비율 × 0.01% 가중치
= 0.01% × 0.005 × 0.01%
= 0.00005%
= 5 / 10,000,000

필요 정밀도: 10^7 (10,000,000)

**향후 대비**:
- 과거 vote reserve rate가 40까지 상승한 적 있음
- 향후 다시 증가 가능성 고려
- 0.01% VP × 2% 비율 × 0.01% 가중치 = 0.0002% = 2 / 10,000,000
- **권장 정밀도**: 10^8 (100,000,000)

**타입 선택**:
- `uint32_t` 최대값: 4,294,967,295 (약 10^9)
- 100,000,000 정밀도는 `uint32_t` 범위 내
- 향후 10배 정밀도 증가 여유 확보

## 명세

### 프로토콜 변경사항

#### 1. 상수 변경

**파일**: `src/core/protocol/include/zattera/protocol/config.hpp`

```cpp
// 변경 전
#define ZATTERA_100_PERCENT                     10000
#define ZATTERA_1_PERCENT                       (ZATTERA_100_PERCENT/100)

// 변경 후
#define ZATTERA_100_PERCENT                     100000000
#define ZATTERA_1_PERCENT                       (ZATTERA_100_PERCENT/100)
```

**영향받는 파생 상수**:
```cpp
// 자동으로 조정됨
#define ZATTERA_DEFAULT_ZBD_INTEREST_RATE       (10*ZATTERA_1_PERCENT)
#define ZATTERA_CONTENT_REWARD_PERCENT          (75*ZATTERA_1_PERCENT)
#define ZATTERA_VESTING_FUND_PERCENT            (15*ZATTERA_1_PERCENT)
#define ZATTERA_ZBD_STOP_PERCENT                (5*ZATTERA_1_PERCENT)
#define ZATTERA_ZBD_START_PERCENT               (2*ZATTERA_1_PERCENT)
#define ZATTERA_IRREVERSIBLE_THRESHOLD          (75*ZATTERA_1_PERCENT)
```

#### 2. 계정 객체 필드 타입 변경

**파일**: `src/core/chain/include/zattera/chain/account_object.hpp`

```cpp
class account_object : public object<account_object_type, account_object>
{
   // ... 기존 필드 ...

   // 변경 전
   // uint16_t voting_power = ZATTERA_100_PERCENT;

   // 변경 후
   uint32_t voting_power = ZATTERA_100_PERCENT;

   // ... 나머지 필드 ...
};
```

**메모리 영향**:
- 계정당 증가: 2바이트 (uint16_t → uint32_t)
- 100만 계정 기준: 2 MB 추가
- 전체 state 파일 (80GB) 대비: 0.0025% 증가

#### 3. 투표 계산 로직 변경

**파일**: `src/core/chain/zattera_evaluator.cpp`

기존 계산 로직은 동일하게 유지되나, 내부 정밀도가 향상됨:

```cpp
void vote_evaluator::do_apply(const vote_operation& o)
{
   // ... 기존 검증 로직 ...

   int64_t abs_weight = abs(o.weight);

   // VP 소비량 계산 (정밀도 10,000배 증가)
   int64_t used_power = ((current_power * abs_weight) / ZATTERA_100_PERCENT) * (60*60*24);

   int64_t max_vote_denom = dgpo.vote_power_reserve_rate * ZATTERA_VOTE_REGENERATION_SECONDS;
   FC_ASSERT(max_vote_denom > 0);

   // 반올림 (이제 훨씬 세밀함)
   used_power = (used_power + max_vote_denom - 1) / max_vote_denom;
   FC_ASSERT(used_power <= current_power, "Account does not have enough power to vote.");

   // rshares 계산 (정밀도 10,000배 증가)
   int64_t abs_rshares = ((uint128_t(effective_vesting) * used_power) / ZATTERA_100_PERCENT).to_uint64();

   abs_rshares -= ZATTERA_VOTE_DUST_THRESHOLD;
   abs_rshares = std::max(int64_t(0), abs_rshares);

   // ... 나머지 로직 ...
}
```

**주요 개선점**:
- 반올림 오차 10,000배 감소
- 최소 유효 투표 가중치: 0.0001% (기존 1%의 10,000분의 1)
- 고래 계정도 0.0001% 단위 조절 가능

#### 4. API 호환성

**파일**: `src/plugins/apis/database_api/database_api.cpp`

API 응답은 하위 호환성 유지:

```cpp
struct api_account_object
{
   // ... 기존 필드 ...

   // JSON으로 반환 시 uint32_t 자동 직렬화
   uint32_t voting_power = 0;  // 0 ~ 100,000,000

   // ... 나머지 필드 ...
};
```

**클라이언트 호환성**:
- 기존 클라이언트: 10,000을 100%로 해석하던 로직 수정 필요
- 새 클라이언트: 100,000,000을 100%로 해석

### 동작 명세

#### VP 재생 로직

**변경 없음**: VP 재생은 동일한 비율로 작동

```cpp
// 재생 공식 (변경 없음)
int64_t regenerated_power = (ZATTERA_100_PERCENT * elapsed_seconds) / ZATTERA_VOTE_REGENERATION_SECONDS;

// 정밀도만 증가
// 이전: 86,400초당 10,000 재생
// 이후: 86,400초당 100,000,000 재생
// 비율: 동일 (100%/5일)
```

#### 투표 가중치 범위

```cpp
// vote_operation 검증
FC_ASSERT(abs(weight) <= ZATTERA_100_PERCENT, "Weight is not a ZATTERA percentage");

// 이전: -10,000 ~ +10,000
// 이후: -100,000,000 ~ +100,000,000
```

#### 개선된 투표 시나리오

**시나리오 1: 소규모 계정 (10M ZP)**

투표 가중치 | VP 소비  | rshares | 유효성
-----------|---------|---------|-------
0.1%       | 2,000   | 0       | 무효 (dust)
0.5%       | 10,000  | 0       | 무효 (dust)
1%         | 20,000  | 1,000   | 유효 ✓

**시나리오 2: 중형 계정 (100M ZP)**

투표 가중치 | VP 소비  | rshares  | 유효성
-----------|---------|----------|-------
0.01%      | 200     | 0        | 무효 (dust)
0.05%      | 1,000   | 0        | 무효 (dust)
0.1%       | 2,000   | 1,000    | 유효 ✓
1%         | 20,000  | 19,000   | 유효 ✓

**시나리오 3: 고래 계정 (1B ZP)**

투표 가중치 | VP 소비  | rshares     | 유효성
-----------|---------|-------------|-------
0.01%      | 200     | 1,000       | 유효 ✓
0.1%       | 2,000   | 19,000      | 유효 ✓
1%         | 20,000  | 199,000     | 유효 ✓

**실제 하루 최대 투표 수:**

제약 조건:
- VP 재생: 5일에 100% 재생
- 시간 제약: 최소 3초 간격 (ZATTERA_MIN_VOTE_INTERVAL_SEC)

투표 가중치 | VP 기반 최대 | 시간 기반 최대 | 실제 최대
-----------|------------|--------------|----------
0.01%      | 500,000표  | 28,800표     | 28,800표 (시간 제약)
0.1%       | 50,000표   | 28,800표     | 28,800표 (시간 제약)
1%         | 5,000표    | 28,800표     | 5,000표 (VP 제약)
2%         | 2,500표    | 28,800표     | 2,500표 (VP 제약)

## 근거

### 설계 결정사항

#### 1. 정밀도 선택: 100,000,000 (10^8)

**고려 사항**:
- 10^7: 현재 요구사항 충족하나 여유 없음
- 10^8: 향후 reserve rate 증가 대비
- 10^9: uint32_t 범위 초과 위험

**선택**: 10^8
- 안전 마진 확보
- uint32_t 범위 내

#### 2. 타입 선택: uint32_t

**대안 비교**:

| 타입      | 최대값    | 메모리 | 정밀도 지원 | 선택 |
|----------|----------|--------|------------|------|
| uint16_t | 65,535   | 2 byte | 10^4       | 기존 |
| uint32_t | 4.2B     | 4 byte | 10^8 ✓     | **채택** |
| uint64_t | 18.4E    | 8 byte | 10^18      | 과함 |

**근거**:
- uint32_t로 충분한 정밀도
- 메모리 증가 최소화 (2바이트)
- 미래 확장성 확보

#### 3. 하위 호환성 vs. 깨끗한 전환

**결정**: 하드포크로 완전 전환

**이유**:
- 두 시스템 병행 운영 복잡도 높음
- 초기 단계에서 전환이 유리
- 명확한 전환점 제공

**대안 (거부됨)**: 단계적 전환
- 복잡도 증가
- 버그 위험
- 유지보수 부담

#### 4. API 변경 전략

**결정**: 하드포크 블록부터 새 정밀도 반환

**구현**:
```cpp
// 클라이언트 측 예시
uint32_t get_voting_power_percentage(uint32_t vp, uint32_t hardfork_num)
{
   if (hardfork_num >= HARDFORK_VP_PRECISION)
      return (vp * 100) / 100000000;  // 새 정밀도
   else
      return (vp * 100) / 10000;      // 구 정밀도
}
```

### 고려된 대안

#### A. 투표 간격 완화

**아이디어**: 정밀도 대신 투표 간격 단축 (3초 → 1초)

**거부 이유**:
- 블록체인 부하 증가
- 근본 문제 미해결 (반올림)
- 고래 계정 문제 여전

#### B. 가변 정밀도

**아이디어**: 계정 ZP에 따라 동적 정밀도

**거부 이유**:
- 구현 복잡도 높음
- 공정성 문제
- 예측 불가능

#### C. 부동소수점 사용

**아이디어**: 정수 대신 float/double

**거부 이유**:
- 합의 위험 (부동소수점 비결정성)
- 직렬화 복잡도
- 성능 저하

## 하위 호환성

본 ZEP는 **하드포크가 필요한 합의 파괴 변경**입니다.

### 호환성 영향 분석

#### 체인 레벨

**데이터베이스 스키마**:
- ⚠️ `account_object.voting_power` 타입 변경 필요
- ⚠️ 공유 메모리 파일 재생성 필요
- ⚠️ 기존 체인 데이터 마이그레이션

**합의 규칙**:
- ⚠️ 투표 검증 로직 변경
- ⚠️ 보상 계산 정밀도 변경
- ✅ 투표 동작 자체는 동일

#### API 레벨

**Database API**:
```json
// 변경 전
{
  "voting_power": 10000  // 100%
}

// 변경 후
{
  "voting_power": 100000000  // 100%
}
```

**영향**:
- ⚠️ 클라이언트가 100% 해석 로직 수정 필요
- ✅ 필드명/타입 동일 (하위 호환)
- ✅ JSON 파서는 문제없음

#### 클라이언트 레벨

**지갑/앱**:
```javascript
// 수정 필요
function getVotingPowerPercent(account) {
  // 이전
  // return account.voting_power / 100;

  // 이후 (하드포크 감지)
  if (isAfterHardfork(account.head_block_num)) {
    return account.voting_power / 1000000;
  } else {
    return account.voting_power / 100;
  }
}
```

**필요 조치**:
- 표시 로직 업데이트
- 투표 가중치 슬라이더 정밀도 증가
- 하드포크 감지 로직 추가

### 마이그레이션 전략

#### Phase 1: 하드포크 준비

1. **코드 배포**:
   - 새 노드 소프트웨어 릴리스
   - 테스트넷 검증
   - 문서 업데이트

2. **클라이언트 통지**:
   - API 변경 공지
   - 마이그레이션 가이드 제공
   - 샘플 코드 배포

#### Phase 2: 하드포크 활성화 (HF Day)

**자동 마이그레이션**:
```cpp
void database::process_hardfork_vp_precision()
{
   if (head_block_num() == HARDFORK_VP_PRECISION_BLOCK)
   {
      // 모든 계정 VP 스케일링
      const auto& accounts = get_index<account_index>().indices();
      for (const auto& account : accounts)
      {
         modify(account, [&](account_object& a)
         {
            // 10,000 → 100,000,000 스케일링
            a.voting_power = (a.voting_power * 100000000) / 10000;
         });
      }
   }
}
```

**검증**:
- 총 VP 합계 불변성 확인
- 비율 유지 확인
- 로그 기록

#### Phase 3: 하드포크 후 (HF+)

1. **모니터링**:
   - 투표 패턴 분석
   - 성능 메트릭 추적
   - 예상치 못한 동작 감지

2. **클라이언트 지원**:
   - 버그 리포트 대응
   - 마이그레이션 이슈 해결

### 롤백 계획

**최악의 시나리오**: 치명적 버그 발견

**대응**:
1. 긴급 하드포크 준비
2. 이전 정밀도로 복귀
3. 체인 재생성

**예방**:
- 광범위한 테스트
- 버그 바운티 프로그램
- 테스트넷 장기 운영

## 보안 고려사항

### 잠재적 취약점

#### 1. 정수 오버플로

**위험**: 큰 숫자 계산 시 오버플로

**영향 받는 코드**:
```cpp
int64_t used_power = ((current_power * abs_weight) / ZATTERA_100_PERCENT) * (60*60*24);
```

**완화**:
```cpp
// uint128_t 사용으로 오버플로 방지
int64_t used_power = ((uint128_t(current_power) * abs_weight) / ZATTERA_100_PERCENT).to_uint64() * (60*60*24);
```

**검증**:
- 경계값 테스트
- 퍼즈 테스트
- 정적 분석

#### 2. 반올림 공격

**위험**: 반올림 오차 악용하여 VP 무한 사용

**예시**:
```
공격: 0.0000001% 투표 반복
의도: VP 소비 없이 rshares 획득
```

**완화**:
```cpp
// ZATTERA_VOTE_DUST_THRESHOLD로 차단
abs_rshares -= ZATTERA_VOTE_DUST_THRESHOLD;  // 1000
abs_rshares = std::max(int64_t(0), abs_rshares);
```

**효과**: 최소 rshares 미달 시 무효

#### 3. VP 재생 불일치

**위험**: 정밀도 변경 경계에서 VP 재생 오류

**시나리오**:
```
하드포크 직전: VP = 5,000 (구 정밀도)
하드포크 직후: VP = 50,000,000 (새 정밀도)
재생 계산: 다른 정밀도로 계산 시 불일치
```

**완화**:
```cpp
// 하드포크 시점에 일괄 스케일링
// 이후 재생은 새 정밀도로 통일
```

#### 4. API 파싱 오류

**위험**: 클라이언트가 잘못된 정밀도로 해석

**영향**:
- 사용자에게 잘못된 VP 표시
- 투표 가중치 오계산

**완화**:
- API 버전 필드 추가
- 문서화
- 예제 코드 제공

### 경제적 영향

#### 보상 분배

**변화**: 미세 투표의 보상 정확도 증가

**시뮬레이션**:
```
이전: 0.1% 투표 = 2% 투표 (동일 rshares)
이후: 0.1% 투표 = 정확히 5% rshares

효과: 작은 보상도 정밀 분배
```

**모니터링**:
- 보상 분포 변화 추적
- 큐레이션 패턴 분석

#### 스팸 방지

**우려**: 마이크로 투표 스팸 증가

**방어 메커니즘**:
1. `ZATTERA_MIN_VOTE_INTERVAL_SEC = 3` 유지
2. `ZATTERA_VOTE_DUST_THRESHOLD` 유지
3. RC (Resource Credit) 비용 조정 가능

**예상**: 큰 문제 없음 (기존 제한 충분)

## 성능 영향

### 메모리

**계정 객체 크기**:
```
변경 전: ~140 bytes
변경 후: ~142 bytes (+2 bytes)
```

**전체 영향** (100만 계정):
```
추가 메모리: 2 MB
State 파일: 80 GB → 80.002 GB
증가율: 0.0025%
```

**결론**: 무시 가능

### CPU

**투표 검증**:
- 정수 연산만 사용 (부동소수점 없음)
- 계산 복잡도 동일
- 추가 비용 없음

**벤치마크** (예상):
```
투표 처리: ~0.5ms (변경 전후 동일)
블록 검증: ~10ms (변경 전후 동일)
```

### 네트워크

**트랜잭션 크기**:
- `vote_operation` 크기 불변
- 투표 가중치는 int16_t (변경 없음)
- 네트워크 부하 동일

### 데이터베이스

**인덱스**:
- voting_power로 정렬하는 인덱스 없음
- 재인덱싱 불필요

**쿼리 성능**:
- 영향 없음

## 테스트 케이스

### 테스트 케이스 1: 정밀도 스케일링 검증

```cpp
BOOST_AUTO_TEST_CASE(voting_power_precision_scaling)
{
   ACTORS((alice))

   // 하드포크 전: 10,000 = 100%
   BOOST_REQUIRE(alice.voting_power == 10000);

   // 하드포크 활성화
   generate_blocks(HARDFORK_VP_PRECISION_BLOCK);

   // 하드포크 후: 100,000,000 = 100%
   const auto& new_alice = db->get_account("alice");
   BOOST_REQUIRE(new_alice.voting_power == 100000000);

   // 비율 유지 확인
   BOOST_REQUIRE(new_alice.voting_power / 100000000.0 == 1.0);
}
```

### 테스트 케이스 2: 저가중치 투표 정밀도

```cpp
BOOST_AUTO_TEST_CASE(low_weight_vote_precision)
{
   ACTORS((alice)(bob))
   fund("alice", ASSET("1000.000 ZTR"));
   vest("alice", ASSET("1000.000 ZTR"));
   generate_blocks(HARDFORK_VP_PRECISION_BLOCK);

   comment_operation post;
   post.author = "bob";
   post.permlink = "test";
   post.body = "Test";
   push_transaction(post, bob_private_key);

   // 0.1% 투표
   vote_operation vote;
   vote.voter = "alice";
   vote.author = "bob";
   vote.permlink = "test";
   vote.weight = 100000;  // 0.1% (100,000 / 100,000,000)

   auto old_vp = db->get_account("alice").voting_power;
   push_transaction(vote, alice_private_key);
   auto new_vp = db->get_account("alice").voting_power;

   // VP 소비 확인 (이전에는 2%와 동일했음)
   int64_t used_vp = old_vp - new_vp;
   BOOST_REQUIRE(used_vp < old_vp / 200);  // 0.5% 미만
   BOOST_REQUIRE(used_vp > old_vp / 2000); // 0.05% 초과
}
```

### 테스트 케이스 3: 고래 계정 미세 투표

```cpp
BOOST_AUTO_TEST_CASE(whale_micro_voting)
{
   ACTORS((whale)(bob))
   fund("whale", ASSET("1000000000.000 ZTR"));  // 10억 ZTR
   vest("whale", ASSET("1000000000.000 ZTR"));
   generate_blocks(HARDFORK_VP_PRECISION_BLOCK);

   comment_operation post;
   post.author = "bob";
   post.permlink = "test";
   push_transaction(post, bob_private_key);

   // 0.01% 투표 (이전에는 불가능)
   vote_operation vote;
   vote.voter = "whale";
   vote.author = "bob";
   vote.permlink = "test";
   vote.weight = 10000;  // 0.01% (10,000 / 100,000,000)

   push_transaction(vote, whale_private_key);

   const auto& comment = db->get_comment("bob", "test");

   // rshares 생성 확인
   BOOST_REQUIRE(comment.net_rshares > 0);

   // 예상 범위 확인
   int64_t whale_vests_balance = db->get_account("whale").vesting_share_balance.amount.value;
   int64_t expected_rshares = whale_vests_balance / 10000;  // 0.01%

   BOOST_REQUIRE(comment.net_rshares < expected_rshares * 1.1);  // 10% 마진
   BOOST_REQUIRE(comment.net_rshares > expected_rshares * 0.9);
}
```

### 테스트 케이스 4: 대량 투표 성능

```cpp
BOOST_AUTO_TEST_CASE(mass_voting_performance)
{
   ACTORS((curator))
   fund("curator", ASSET("10000.000 ZTR"));
   vest("curator", ASSET("10000.000 ZTR"));
   generate_blocks(HARDFORK_VP_PRECISION_BLOCK);

   // 100개 포스트 생성
   std::vector<std::string> permlinks;
   for (int i = 0; i < 100; i++) {
      comment_operation post;
      post.author = "curator";
      post.permlink = "post-" + std::to_string(i);
      post.body = "Content";
      push_transaction(post, curator_private_key);
      permlinks.push_back(post.permlink);
   }

   // 0.1% 투표 × 100회
   auto start_time = fc::time_point::now();

   for (const auto& permlink : permlinks) {
      vote_operation vote;
      vote.voter = "curator";
      vote.author = "curator";
      vote.permlink = permlink;
      vote.weight = 100000;  // 0.1%

      generate_blocks(1);  // 3초 간격 준수
      push_transaction(vote, curator_private_key);
   }

   auto end_time = fc::time_point::now();
   auto elapsed = (end_time - start_time).to_seconds();

   // VP 소비 확인
   const auto& curator_acc = db->get_account("curator");
   BOOST_REQUIRE(curator_acc.voting_power > 90000000);  // 90% 이상 남음

   // 성능 확인 (100개 투표가 합리적 시간 내)
   BOOST_REQUIRE(elapsed < 500);  // 500초 미만
}
```

### 테스트 케이스 5: VP 재생 정확도

```cpp
BOOST_AUTO_TEST_CASE(vp_regeneration_accuracy)
{
   ACTORS((alice))
   generate_blocks(HARDFORK_VP_PRECISION_BLOCK);

   // VP 50% 소진
   modify(db->get_account("alice"), [&](account_object& a) {
      a.voting_power = 50000000;  // 50%
      a.last_vote_time = db->head_block_time();
   });

   // 1일 경과
   generate_blocks(db->head_block_time() + fc::days(1));

   const auto& alice = db->get_account("alice");

   // 예상 재생: 20% (100% / 5일)
   int64_t expected_vp = 50000000 + 20000000;  // 70%

   // 실제 재생 확인 (오차 범위 0.1%)
   BOOST_REQUIRE(abs(alice.voting_power - expected_vp) < 100000);
}
```

### 테스트 케이스 6: 정수 오버플로 방지

```cpp
BOOST_AUTO_TEST_CASE(integer_overflow_prevention)
{
   ACTORS((alice))
   generate_blocks(HARDFORK_VP_PRECISION_BLOCK);

   // 최대 VP로 설정
   modify(db->get_account("alice"), [&](account_object& a) {
      a.voting_power = 100000000;  // 100%
   });

   // 최대 가중치 투표
   vote_operation vote;
   vote.voter = "alice";
   vote.weight = 100000000;  // 100%

   // 오버플로 없이 성공해야 함
   BOOST_REQUIRE_NO_THROW(push_transaction(vote, alice_private_key));

   // 음수 가중치도 테스트
   vote.weight = -100000000;
   BOOST_REQUIRE_NO_THROW(push_transaction(vote, alice_private_key));
}
```

## 참조 구현

구현 코드는 다음 브랜치에서 확인 가능:

**브랜치**: `feature/vp-precision-increase`

**주요 변경 파일**:
- `src/core/protocol/include/zattera/protocol/config.hpp` - 상수 변경
- `src/core/chain/include/zattera/chain/account_object.hpp` - 타입 변경
- `src/core/chain/zattera_evaluator.cpp` - 계산 로직
- `src/core/chain/database.cpp` - 하드포크 마이그레이션
- `tests/tests/voting_precision_tests.cpp` - 테스트 케이스

## 향후 개선사항

### 1. 적응형 정밀도

VP에 따라 동적으로 정밀도 조정:

```cpp
uint32_t get_effective_precision(share_type vesting_shares)
{
   if (vesting_shares < 1000000)  // 1M VESTS
      return 100000;               // 10^5 정밀도
   else if (vesting_shares < 1000000000)  // 1B VESTS
      return 10000000;             // 10^7 정밀도
   else
      return 100000000;            // 10^8 정밀도
}
```

**장점**: 메모리 절약
**단점**: 복잡도 증가

### 2. VP 배수 시스템

특정 조건에서 VP 재생 가속:

```cpp
struct vp_multiplier_extension
{
   uint16_t multiplier = 100;  // 100 = 1.0x
   time_point_sec expiration;
};
```

**사용 사례**:
- 충성도 보상
- 이벤트 기간 부스트

### 3. 동적 Reserve Rate

네트워크 활동에 따라 자동 조정:

```cpp
uint32_t calculate_dynamic_reserve_rate()
{
   uint64_t daily_votes = get_daily_vote_count();

   if (daily_votes > 10000000)  // 과부하
      return 20;  // VP 재생 빠르게
   else
      return 10;  // 정상
}
```

## 저작권

저작권 및 관련 권리는 [CC0](https://creativecommons.org/publicdomain/zero/1.0/)를 통해 포기됩니다.
