---
zep: 6
title: 댓글 권한 제어
description: 게시물 작성자가 댓글 작성이 허용된 계정의 화이트리스트를 지정할 수 있도록 허용
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2025-11-11
---

## 초록

본 ZEP는 게시물 작성자가 자신의 콘텐츠에 댓글을 작성할 수 있는 사용자를 제한할 수 있는 권한 제어 메커니즘을 도입합니다. 작성자는 세 가지 모드 중 하나를 지정할 수 있습니다: 공개 댓글(기본값), 특정 화이트리스트 계정으로 제한, 또는 댓글 비허용. 이 기능은 프로토콜 수준에서 더 나은 콘텐츠 관리와 스팸 방지를 가능하게 합니다.

## 동기

현재 블록체인 소셜 플랫폼은 콘텐츠 제작자가 자신의 게시물에 댓글을 작성할 수 있는 사람을 제어할 수 있는 네이티브 메커니즘을 제공하지 않습니다. 이러한 제한은 여러 문제를 야기합니다:

1. **스팸 및 괴롭힘**: 작성자가 스팸 계정이나 괴롭힘으로부터 원하지 않는 댓글을 방지할 수 없음
2. **품질 제어**: 검증된 참가자나 신뢰할 수 있는 참가자로 토론을 제한할 방법이 없음
3. **비공개 토론**: 선별된 그룹 토론을 위한 게시물을 만들 수 없음
4. **콘텐츠 큐레이션**: 작성자가 댓글 품질 기준을 유지할 도구가 부족
5. **디앱 비즈니스 모델 제약**: 프리미엄 콘텐츠나 멤버십 기반 커뮤니티를 구축할 수 있는 메커니즘 부재

오프체인 솔루션(UI 필터링, 뮤트)은 다음과 같은 이유로 불충분합니다:
- 댓글이 여전히 온체인에 존재하며 리소스를 소비함
- 다른 UI가 다른 댓글 세트를 표시함(일관되지 않은 경험)
- 스팸이 여전히 평판 및 발견 알고리즘에 영향을 미침
- 스팸 댓글에 대한 경제적 억제 효과가 없음

프로토콜 수준 솔루션이 제공하는 것:
- 모든 인터페이스에서 일관된 시행
- 진정한 댓글 방지(단순 숨김이 아님)
- 스팸으로 인한 블록체인 비대화 감소
- 콘텐츠 공간에 대한 작성자 주권
- 디앱 비즈니스 모델 구현 가능

### 디앱 비즈니스 모델 활용 사례

이 기능은 다양한 비즈니스 모델을 블록체인 수준에서 구현할 수 있게 합니다:

1. **프리미엄 커뮤니티**
   - 구독자 또는 토큰 홀더만 참여 가능한 토론 공간
   - 멤버십 기반 콘텐츠 플랫폼 구축
   - 예: 교육 플랫폼에서 강의 수강생만 질문 가능

2. **검증된 사용자 전용 공간**
   - KYC 인증 완료 사용자만 댓글 허용
   - 전문가 네트워크 또는 업계별 토론 포럼
   - 예: 의료 전문가만 참여 가능한 의학 논문 토론

3. **고객 지원 및 서비스**
   - 제품 구매자만 리뷰 및 질문 작성 가능
   - 고객 전용 지원 스레드 생성
   - 예: NFT 홀더만 프로젝트 관련 질문 가능

4. **계층형 커뮤니티**
   - 스테이킹 레벨 또는 평판에 따른 차등 접근
   - DAO 멤버십 기반 거버넌스 토론
   - 예: 특정 토큰 보유자만 제안서 댓글 가능

5. **프라이빗 그룹 협업**
   - 팀 내부 프로젝트 토론
   - 클라이언트와의 비공개 커뮤니케이션
   - 예: 기업 고객 전용 업데이트 피드

이러한 기능은 디앱 개발자들이 Web2의 멤버십 모델을 블록체인에서 구현하면서도 탈중앙화와 투명성을 유지할 수 있게 합니다.

## 명세

### 댓글 권한 모드

세 가지 모드가 지원됩니다:

1. **공개(기본값)**: 누구나 댓글 작성 가능
   - 필드: `allowed_comment_accounts` 미설정(`nullopt`)
   - 동작: 기존 동작, 제한 없음

2. **화이트리스트**: 지정된 계정만 댓글 작성 가능
   - 필드: `allowed_comment_accounts` 계정 목록과 함께 설정
   - 동작: 목록에 있는 계정만 댓글 작성 가능

3. **비활성화**: 댓글 작성 불가
   - 필드: `allowed_comment_accounts` 빈 집합으로 설정
   - 동작: 모든 댓글 시도 실패

### 프로토콜 변경사항

#### 1. 댓글 오퍼레이션 확장

```cpp
struct comment_operation : public base_operation
{
   account_name_type parent_author;
   string            parent_permlink;
   account_name_type author;
   string            permlink;
   string            title;
   string            body;
   string            json_metadata;

   // 신규: 선택적 댓글 권한 제어
   optional<flat_set<account_name_type>> allowed_comment_accounts;

   extensions_type   extensions;

   void validate() const;
};
```

**검증 로직**:

```cpp
void comment_operation::validate() const
{
   // ... 기존 검증 ...

   if (allowed_comment_accounts.valid())
   {
      // 각 계정 이름 검증
      for (const auto& account : *allowed_comment_accounts)
      {
         FC_ASSERT(is_valid_account_name(account),
                   "Invalid account name: ${a}", ("a", account));
      }

      // 과도하게 큰 화이트리스트로 인한 남용 방지
      FC_ASSERT(allowed_comment_accounts->size() <= ZATTERA_MAX_COMMENT_WHITELIST_SIZE,
                "Whitelist cannot exceed ${max} accounts",
                ("max", ZATTERA_MAX_COMMENT_WHITELIST_SIZE));
   }
}
```

**프로토콜 상수**:
```cpp
#define ZATTERA_MAX_COMMENT_WHITELIST_SIZE 1000
```

#### 2. 댓글 객체 확장

```cpp
class comment_object : public object<comment_object_type, comment_object>
{
   comment_id_type   id;

   string            author;
   string            permlink;
   // ... 기존 필드 ...

   // 신규: 댓글 권한 제어
   // - nullopt = 모두에게 공개
   // - 빈 벡터 = 댓글 불가
   // - 비어있지 않은 벡터 = 허용된 계정의 화이트리스트
   optional<shared_vector<account_name_type>> allowed_comment_accounts;

   // ... 나머지 객체 ...
};
```

**참고**: chainbase가 메모리 매핑 저장소를 위해 할당자 인식 컨테이너를 요구하므로 `flat_set` 대신 `shared_vector`를 사용합니다.

#### 3. 댓글 평가자 변경사항

**파일**: `libraries/chain/zattera_evaluator.cpp`

```cpp
void comment_evaluator::do_apply(const comment_operation& o)
{
   // 댓글인 경우(루트 게시물이 아닌), 권한 확인
   if (o.parent_author != ZATTERA_ROOT_POST_PARENT)
   {
      const auto& parent = _db.get_comment(o.parent_author, o.parent_permlink);

      // 부모가 댓글 제한이 있는지 확인
      if (parent.allowed_comment_accounts.valid())
      {
         const auto& allowed = *parent.allowed_comment_accounts;

         if (allowed.empty())
         {
            // 빈 화이트리스트 = 댓글 불가
            FC_ASSERT(false,
                     "Comments are disabled for this post");
         }
         else
         {
            // 댓글 작성자가 화이트리스트에 있는지 확인
            auto it = std::find(allowed.begin(), allowed.end(), o.author);
            FC_ASSERT(it != allowed.end(),
                     "Account ${a} is not allowed to comment on this post",
                     ("a", o.author));
         }
      }
   }
   else  // 루트 게시물
   {
      // 지정된 경우 댓글 권한 설정
      if (o.allowed_comment_accounts.valid())
      {
         _db.modify(comment, [&](comment_object& c)
         {
            c.allowed_comment_accounts = shared_vector<account_name_type>(
               o.allowed_comment_accounts->begin(),
               o.allowed_comment_accounts->end(),
               c.allowed_comment_accounts.get_allocator()
            );
         });
      }
   }

   // ... 나머지 기존 댓글 생성 로직 ...
}
```

### 동작 명세

#### 권한 상속

**규칙**: 댓글 권한 설정은 자식 댓글에 상속되지 않습니다.

**근거**:
- 각 댓글은 작성자가 독립적으로 제어
- 혼란스러운 중첩 권한 시나리오 방지
- 구현을 단순하게 유지

**예시**:
```
게시물 (Alice, 화이트리스트: [Bob, Charlie])
  ├─ 댓글 1 (Bob, 모두에게 공개)
  │    └─ 댓글 1.1 (Dan, Bob의 댓글이 공개이므로 허용됨)
  └─ 댓글 2 (Charlie, 댓글 불가)
       └─ (답글 불가능)
```

#### 권한 불변성

**규칙**: 댓글 권한은 게시물 생성 후 변경할 수 없습니다.

**근거**:
- 미끼와 전환(댓글 허용 후 잠금) 방지
- 더 간단한 구현(업데이트 로직 불필요)
- 댓글 작성자에 대한 명확한 기대

**향후 개선**: 향후 ZEP에서 별도의 업데이트 오퍼레이션 추가 가능.

#### 편집 동작

**규칙**: 게시물 본문/제목 편집은 댓글 권한에 영향을 미치지 않습니다.

**구현**: 권한 필드는 초기 생성 시에만 설정되며, 업데이트 시 무시됩니다.

```cpp
// 평가자에서
if (comment == nullptr)  // 새 게시물 생성
{
   // allowed_comment_accounts 설정
}
else  // 기존 게시물 편집
{
   // allowed_comment_accounts를 수정하지 않음
}
```

### API 확장

#### Database API

```cpp
struct get_comment_permissions_args
{
   account_name_type author;
   string            permlink;
};

struct get_comment_permissions_return
{
   bool                               comments_enabled;
   optional<vector<account_name_type>> allowed_accounts;
};

/**
 * 게시물의 댓글 권한 설정 반환
 *
 * @param author 게시물 작성자
 * @param permlink 게시물 퍼머링크
 * @returns 권한 정보
 */
get_comment_permissions_return get_comment_permissions(
   get_comment_permissions_args args);
```

**구현**:

```cpp
get_comment_permissions_return database_api::get_comment_permissions(
   get_comment_permissions_args args) const
{
   return my->_db.with_read_lock([&]()
   {
      get_comment_permissions_return result;

      const auto& comment = my->_db.get_comment(args.author, args.permlink);

      if (comment.allowed_comment_accounts.valid())
      {
         if (comment.allowed_comment_accounts->empty())
         {
            result.comments_enabled = false;
         }
         else
         {
            result.comments_enabled = true;
            result.allowed_accounts = vector<account_name_type>(
               comment.allowed_comment_accounts->begin(),
               comment.allowed_comment_accounts->end()
            );
         }
      }
      else
      {
         result.comments_enabled = true;  // 모두에게 공개
      }

      return result;
   });
}
```

## 근거

### 설계 결정사항

#### 1. 화이트리스트 vs 블랙리스트

**결정**: 블랙리스트(거부 목록) 대신 화이트리스트(허용 목록) 사용

**이유**:
- **확장성**: 화이트리스트가 일반적으로 블랙리스트보다 작음
- **프라이버시**: 블랙리스트는 갈등/괴롭힘을 공개적으로 노출
- **시빌 저항성**: 블랙리스트는 새로운 스팸 계정에 대해 효과적이지 않음
- **가스 효율성**: 저장/확인할 더 작은 데이터 구조

#### 2. 크기 제한(1000개 계정)

**결정**: 화이트리스트당 최대 1000개 계정

**이유**:
- **DoS 방지**: 과도하게 큰 화이트리스트 방지
- **성능**: n ≤ 1000일 때 선형 검색 O(n) 허용 가능
- **실용성**: 합법적인 사용 사례 커버(1000명 이상의 토론은 드묾)
- **가스 비용**: 트랜잭션 크기 및 검증 시간 제한

**고려된 대안**: 제한 없음 - DoS 우려로 거부됨

#### 3. 생성 후 불변

**결정**: 권한은 게시물 생성 후 변경할 수 없음

**이유**:
- **단순성**: 별도의 업데이트 오퍼레이션 불필요
- **신뢰**: 댓글 작성자는 규칙이 토론 중에 변경되지 않음을 알 수 있음
- **호환성**: 복잡한 마이그레이션 로직 회피

**트레이드오프**: 나중에 조정하려는 작성자에게 유연성 감소

**향후**: 수요가 있는 경우 나중 ZEP에서 업데이트 오퍼레이션 추가 가능

#### 4. 중첩 제한 없음

**결정**: 자식 댓글은 독립적으로 제어됨

**이유**:
- **자율성**: 댓글 작성자가 자신의 하위 스레드 제어
- **단순성**: 복잡한 권한 해결 회피
- **유연성**: 제한된 게시물에서 공개 토론 허용

**예시**: Bob(화이트리스트에 포함)은 Alice의 제한된 게시물에 대한 자신의 댓글에 누구나 답글을 달 수 있도록 허용할 수 있음

#### 5. 프로토콜 수준 vs. 애플리케이션 수준

**결정**: UI가 아닌 프로토콜 수준에서 시행

**이유**:
- **일관성**: 모든 클라이언트가 동일한 규칙 시행
- **리소스 효율성**: 스팸이 블록체인에 도달하는 것을 방지
- **경제적 인센티브**: 실패한 트랜잭션이 스패머 리소스 낭비
- **진정한 시행**: 대체 UI로 우회할 수 없음

#### 6. 선택적 필드

**결정**: 항상 존재하는 필드 대신 `optional<>` 사용

**이유**:
- **하위 호환성**: 기존 게시물 변경 없음
- **저장 효율성**: 기능 사용 시에만 비용 지불
- **기본 동작**: Null = 공개(기존 동작과 일치)

### 고려된 대안 접근 방식

#### A. 역할 기반 권한

**아이디어**: 특정 기준(평판, 스테이킹 등)이 있는 계정의 댓글 허용

**거부된 이유**:
- 더 복잡한 구현
- 기준이 게임 가능
- 세밀한 제어를 위해 여전히 화이트리스트 필요

**향후**: 이것을 기반으로 하는 별도의 ZEP 가능

#### B. 평판 임계값

**아이디어**: 댓글 작성을 위한 최소 평판 요구

**거부된 이유**:
- 새로운 합법적 사용자에게 불리
- 평판 시스템은 구현에 따라 다름
- 비공개 토론에 충분히 유연하지 않음

**향후**: 화이트리스트와 결합 가능(화이트리스트 또는 평판 > N)

#### C. 토큰 게이트 댓글

**아이디어**: 댓글 작성을 위해 특정 토큰 보유 요구

**거부된 이유**:
- 토큰 시스템 통합 필요
- 초기 구현에 너무 복잡
- SMT 기능과 중복

**향후**: 토큰 게이트 커뮤니티를 위한 확장 가능

## 하위 호환성

본 ZEP는 하드 포크가 필요한 합의 파괴 변경사항을 도입합니다.

### 호환성 분석

**기존 오퍼레이션**:
- ✅ `allowed_comment_accounts` 없는 이전 `comment_operation`은 여전히 유효
- ✅ Null/부재 필드는 "모두에게 공개"로 처리
- ✅ 기존 댓글 표시에 변경 없음

**데이터베이스 스키마**:
- ⚠️ 선택적 필드를 추가하기 위해 데이터베이스 마이그레이션 필요
- ⚠️ 공유 메모리 파일 형식 변경(재생 필요)

**API**:
- ✅ 기존 API 호출 계속 작동
- ✅ 응답에 새 필드 추가(하위 호환)
- ✅ 새 필드를 무시하는 클라이언트는 기본 동작 확인

### 마이그레이션 경로

**하드포크 전**:
1. 모든 게시물은 댓글에 공개
2. `allowed_comment_accounts` 필드 존재하지 않음

**하드포크 활성화**:
1. 데이터베이스 스키마 업데이트
2. 기존 게시물: `allowed_comment_accounts = nullopt`(공개)
3. 새 오퍼레이션은 권한 필드 포함 가능

**하드포크 후**:
1. 작성자는 제한된 게시물 생성 가능
2. 이전 게시물은 재생성하지 않는 한 공개 유지
3. API는 권한 정보 반환

**다음에 대한 중단 변경 없음**:
- 기능을 사용하지 않는 사용자
- 기존 게시물 및 댓글
- 읽기 전용 API 소비자
- 필드를 무시하는 지갑

## 테스트 케이스

### 테스트 케이스 1: 공개 댓글(기본값)

```cpp
BOOST_AUTO_TEST_CASE(comment_permission_open_default)
{
   ACTORS((alice)(bob))

   // Alice가 권한 필드 없이 게시물 생성
   comment_operation post;
   post.parent_author = "";
   post.author = "alice";
   post.permlink = "test-post";
   post.title = "Test";
   post.body = "Content";
   // allowed_comment_accounts 미설정

   push_transaction(post, alice_private_key);

   // Bob이 댓글 작성 가능
   comment_operation reply;
   reply.parent_author = "alice";
   reply.parent_permlink = "test-post";
   reply.author = "bob";
   reply.permlink = "bobs-reply";
   reply.body = "Nice post!";

   // 성공해야 함
   push_transaction(reply, bob_private_key);

   const auto& comment = db->get_comment("bob", "bobs-reply");
   BOOST_REQUIRE(comment.parent_author == "alice");
}
```

### 테스트 케이스 2: 화이트리스트 - 허용된 계정

```cpp
BOOST_AUTO_TEST_CASE(comment_permission_whitelist_allowed)
{
   ACTORS((alice)(bob)(charlie))

   // Alice가 화이트리스트로 게시물 생성
   comment_operation post;
   post.parent_author = "";
   post.author = "alice";
   post.permlink = "restricted-post";
   post.title = "Restricted";
   post.body = "Only bob and charlie can comment";
   post.allowed_comment_accounts = flat_set<account_name_type>{"bob", "charlie"};

   push_transaction(post, alice_private_key);

   // Bob(화이트리스트에 포함)이 댓글 작성 가능
   comment_operation bob_reply;
   bob_reply.parent_author = "alice";
   bob_reply.parent_permlink = "restricted-post";
   bob_reply.author = "bob";
   bob_reply.permlink = "bobs-reply";
   bob_reply.body = "Thanks for including me!";

   // 성공해야 함
   push_transaction(bob_reply, bob_private_key);

   const auto& comment = db->get_comment("bob", "bobs-reply");
   BOOST_REQUIRE(comment.id._id > 0);
}
```

### 테스트 케이스 3: 화이트리스트 - 거부된 계정

```cpp
BOOST_AUTO_TEST_CASE(comment_permission_whitelist_rejected)
{
   ACTORS((alice)(bob)(dan))

   // Alice가 화이트리스트 [bob]로 게시물 생성
   comment_operation post;
   post.parent_author = "";
   post.author = "alice";
   post.permlink = "restricted-post";
   post.allowed_comment_accounts = flat_set<account_name_type>{"bob"};

   push_transaction(post, alice_private_key);

   // Dan(화이트리스트에 없음)이 댓글 작성 시도
   comment_operation dan_reply;
   dan_reply.parent_author = "alice";
   dan_reply.parent_permlink = "restricted-post";
   dan_reply.author = "dan";
   dan_reply.permlink = "dans-reply";
   dan_reply.body = "Can I comment?";

   // 실패해야 함
   ZATTERA_REQUIRE_THROW(
      push_transaction(dan_reply, dan_private_key),
      fc::exception
   );
}
```

### 테스트 케이스 4: 댓글 불가

```cpp
BOOST_AUTO_TEST_CASE(comment_permission_disabled)
{
   ACTORS((alice)(bob))

   // Alice가 빈 화이트리스트로 게시물 생성
   comment_operation post;
   post.parent_author = "";
   post.author = "alice";
   post.permlink = "no-comments";
   post.allowed_comment_accounts = flat_set<account_name_type>();  // 빈 집합

   push_transaction(post, alice_private_key);

   // Bob이 댓글 작성 시도
   comment_operation bob_reply;
   bob_reply.parent_author = "alice";
   bob_reply.parent_permlink = "no-comments";
   bob_reply.author = "bob";
   bob_reply.permlink = "bobs-reply";
   bob_reply.body = "Testing...";

   // 실패해야 함
   ZATTERA_REQUIRE_THROW(
      push_transaction(bob_reply, bob_private_key),
      fc::exception
   );
}
```

### 테스트 케이스 5: 화이트리스트 크기 제한

```cpp
BOOST_AUTO_TEST_CASE(comment_permission_size_limit)
{
   ACTORS((alice))

   // 1001개 계정으로 화이트리스트 생성(제한 초과)
   flat_set<account_name_type> large_whitelist;
   for (int i = 0; i < 1001; i++)
   {
      large_whitelist.insert("user" + std::to_string(i));
   }

   comment_operation post;
   post.parent_author = "";
   post.author = "alice";
   post.permlink = "test";
   post.allowed_comment_accounts = large_whitelist;

   // 검증 실패해야 함
   BOOST_REQUIRE_THROW(
      post.validate(),
      fc::exception
   );
}
```

### 테스트 케이스 6: 유효하지 않은 계정 이름

```cpp
BOOST_AUTO_TEST_CASE(comment_permission_invalid_account)
{
   comment_operation post;
   post.parent_author = "";
   post.author = "alice";
   post.permlink = "test";
   post.allowed_comment_accounts = flat_set<account_name_type>{"invalid.name!"};

   // 검증 실패해야 함
   BOOST_REQUIRE_THROW(
      post.validate(),
      fc::exception
   );
}
```

### 테스트 케이스 7: 자식 댓글 독립성

```cpp
BOOST_AUTO_TEST_CASE(comment_permission_child_independence)
{
   ACTORS((alice)(bob)(charlie))

   // Alice가 제한된 게시물 생성 [bob만]
   comment_operation post;
   post.author = "alice";
   post.allowed_comment_accounts = flat_set<account_name_type>{"bob"};
   push_transaction(post, alice_private_key);

   // Bob이 공개 권한으로 댓글 작성(허용됨)
   comment_operation bob_reply;
   bob_reply.parent_author = "alice";
   bob_reply.author = "bob";
   // allowed_comment_accounts 없음 = 모두에게 공개
   push_transaction(bob_reply, bob_private_key);

   // Charlie가 Bob의 댓글에 답글(성공해야 함)
   comment_operation charlie_reply;
   charlie_reply.parent_author = "bob";
   charlie_reply.author = "charlie";
   push_transaction(charlie_reply, charlie_private_key);

   // Bob의 댓글이 공개이므로 성공해야 함
   const auto& comment = db->get_comment("charlie", charlie_reply.permlink);
   BOOST_REQUIRE(comment.parent_author == "bob");
}
```

## 참조 구현

기능 브랜치에서 사용 가능:

- 브랜치: `feature/comment-permission-control`
- 주요 파일:
  - `libraries/protocol/include/zattera/protocol/zattera_operations.hpp`
  - `libraries/chain/include/zattera/chain/comment_object.hpp`
  - `libraries/chain/zattera_evaluator.cpp`
  - `libraries/plugins/apis/database_api/database_api.cpp`
  - `tests/tests/comment_permission_tests.cpp`

## 보안 고려사항

### 잠재적 취약점

#### 1. 큰 화이트리스트를 통한 DoS

**위험**: 공격자가 최대 크기의 화이트리스트로 게시물 생성

**영향**:
- 트랜잭션 크기 증가
- 검증 속도 저하(O(n) 조회)
- 저장 비용 증가

**완화**:
- 하드 제한: 최대 1000개 계정
- n ≤ 1000일 때 선형 검색 허용 가능
- 필요 시 이진 검색 최적화 고려

**모니터링**: 평균 화이트리스트 크기 분포 추적

#### 2. 스팸 계정 생성

**위험**: 스패머가 블랙리스트를 우회하기 위해 새 계정 생성

**영향**: 화이트리스트 접근 방식이 이미 이를 해결

**완화**: 화이트리스트 모델은 본질적으로 시빌 공격에 저항력이 있음

#### 3. 권한 검증 우회

**위험**: 평가자의 버그로 인해 무단 댓글 허용

**영향**: 기능 완전히 손상

**완화**:
- 포괄적인 테스트 커버리지
- 보안 전문가의 감사
- 버그 바운티 프로그램

**테스트**: 다양한 계정 조합으로 퍼즈 테스트

#### 4. API 정보 유출

**위험**: 공개적으로 표시되는 화이트리스트로 인한 프라이버시 우려

**영향**: 작성자의 소셜 그래프 노출

**완화**: 이것은 의도적이며 불가피함
- 화이트리스트는 블록체인의 특성상 공개
- 사용자는 프라이버시 영향을 인식해야 함
- 향후: 암호화된 화이트리스트(고급 기능)

### 경제적 고려사항

#### 저장 비용

- 작은 영향: 대부분의 게시물은 기능을 사용하지 않음
- 화이트리스트 사용자는 비례 RC 비용 지불
- 선택적 필드 = 사용하지 않을 때 비용 없음

#### 네트워크 효과

- 긍정적: 스팸 감소, 품질 향상
- 부정적: 참여 지표 감소 가능
- 중립적: 사용자 선택(선택적 기능)

## 향후 개선사항

향후 ZEP를 위한 잠재적 확장:

### 1. 권한 업데이트

작성자가 생성 후 권한을 수정할 수 있도록 허용:

```cpp
struct update_comment_permissions_operation
{
   account_name_type author;
   string            permlink;
   optional<flat_set<account_name_type>> new_allowed_accounts;
};
```

### 2. 역할 기반 접근

화이트리스트와 기준 결합:

```cpp
struct comment_permission_criteria
{
   optional<flat_set<account_name_type>> whitelist;
   optional<uint32_t> min_reputation;
   optional<asset> min_stake;
};
```

### 3. 위임

중재자가 권한을 관리할 수 있도록 허용:

```cpp
struct delegate_comment_moderation_operation
{
   account_name_type author;
   string            permlink;
   account_name_type moderator;
   time_point_sec    expiration;
};
```

### 4. 블랙리스트 모드

화이트리스트를 거부 목록으로 보완:

```cpp
struct comment_permission_v2
{
   optional<flat_set<account_name_type>> allow_list;
   optional<flat_set<account_name_type>> deny_list;
   permission_mode mode;  // whitelist, blacklist, open, closed
};
```

### 5. 시간 기반 제한

기간 후 댓글 잠금:

```cpp
struct comment_operation
{
   // ... 기존 필드 ...
   optional<time_point_sec> comment_lock_time;
};
```

### 6. NFT/토큰 게이팅

토큰 소유권 요구:

```cpp
struct token_gated_comments
{
   string token_symbol;
   asset  minimum_balance;
};
```

## 저작권

저작권 및 관련 권리는 [CC0](https://creativecommons.org/publicdomain/zero/1.0/)를 통해 포기됩니다.
