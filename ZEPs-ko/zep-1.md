---
zep: 1
title: ZEP 목적 및 가이드라인
description: Zattera Enhancement Proposal (ZEP) 및 Zattera Request for Comment (ZRC) 프로세스 정의
author: Zattera Core Team
discussions-to: TBD
status: Living
type: Meta
created: 2025-01-08
---

## ZEP란 무엇인가?

ZEP는 Zattera Enhancement Proposal의 약자입니다. ZEP는 Zattera 커뮤니티에 정보를 제공하거나 Zattera의 새로운 기능, 프로세스 또는 환경을 설명하는 설계 문서입니다. ZEP는 기능의 간결한 기술 사양과 해당 기능에 대한 근거를 제공해야 합니다. ZEP 작성자는 커뮤니티 내에서 합의를 구축하고 반대 의견을 문서화할 책임이 있습니다.

## ZRC란 무엇인가?

ZRC는 Zattera Request for Comment의 약자입니다. ZRC는 애플리케이션 수준의 표준 및 규약에 초점을 맞춘 ZEP의 하위 집합입니다. 여기에는 토큰 표준, 플러그인 인터페이스, 지갑 형식, API 규약 및 합의 변경이 필요하지 않은 기타 표준이 포함됩니다.

ZEP/ZRC 시스템은 Ethereum의 EIP/ERC 프로세스, Bitcoin의 BIP 프로세스, Python의 PEP 프로세스에서 영감을 받았습니다.

## ZEP 근거

ZEP는 새로운 기능을 제안하고, 이슈에 대한 커뮤니티의 기술적 의견을 수집하며, Zattera에 반영된 설계 결정을 문서화하는 주요 메커니즘으로 의도되었습니다. ZEP는 버전 관리 저장소의 텍스트 파일로 유지되므로 수정 이력이 기능 제안의 역사적 기록이 됩니다.

Zattera 구현 담당자에게 ZEP는 구현 진행 상황을 추적하는 편리한 방법입니다. 이상적으로 각 구현 유지 관리자는 자신이 구현한 ZEP 목록을 작성할 것입니다. 이를 통해 최종 사용자는 특정 구현 또는 라이브러리의 현재 상태를 편리하게 알 수 있습니다.

## ZEP 유형

ZEP에는 세 가지 유형이 있습니다:

- **Standards Track ZEP**: 대부분 또는 모든 Zattera 구현에 영향을 미치는 변경 사항을 설명합니다:
  - 네트워크 프로토콜 변경
  - 블록 또는 트랜잭션 유효성 규칙 변경
  - 제안된 애플리케이션 표준/규약
  - Zattera를 사용하는 애플리케이션의 상호 운용성에 영향을 미치는 모든 변경 또는 추가 사항

- **Meta ZEP**: Zattera 관련 프로세스를 설명하거나 프로세스의 변경 사항(또는 이벤트)을 제안합니다. Meta ZEP는 Standards Track ZEP와 유사하지만 Zattera 프로토콜 자체가 아닌 다른 영역에 적용됩니다. 예를 들어 절차, 가이드라인, 의사 결정 프로세스 변경, Zattera 개발에 사용되는 도구 또는 환경 변경 등이 포함됩니다.

- **Informational ZEP**: Zattera 설계 이슈를 설명하거나 Zattera 커뮤니티에 일반적인 가이드라인 또는 정보를 제공하지만 새로운 기능을 제안하지는 않습니다. Informational ZEP는 반드시 Zattera 커뮤니티 합의 또는 권장 사항을 나타내지는 않습니다.

## Standards Track ZEP 카테고리

Standards Track ZEP는 세 가지 카테고리로 구성됩니다:

- **Core**: 합의 포크가 필요한 개선 사항(예: 작업, 평가자, 합의 규칙, 보상 알고리즘 변경)
- **Networking**: 네트워크 프로토콜 사양 관련 개선 사항
- **Interface**: 클라이언트 API/RPC 사양 및 표준 관련 개선 사항(database_api, 플러그인 API 포함)

## ZRC vs ZEP

ZRC는 ZEP의 하위 집합입니다. 구체적으로 **ZRC는 Standards Track ZEP**이며 애플리케이션 수준의 표준에 초점을 맞춥니다. 구분은 다음과 같습니다:

- **ZEP**: 합의가 필요한 프로토콜 수준 변경(Core), 네트워크 변경(Networking) 또는 인터페이스 변경(Interface)
- **ZRC**: 프로토콜 변경이 필요하지 않은 애플리케이션 수준 표준(토큰 표준, 플러그인 인터페이스, 데이터 형식)

ZRC의 예:
- 플러그인 표준 인터페이스
- 지갑 키 유도 표준
- JSON 메타데이터 스키마
- Zattera 리소스용 URI 스키마

## ZEP 워크플로우

### 1. 아이디어 단계

ZEP를 제출하기 전에 아이디어를 철저히 검토해야 합니다. 거부될 내용에 시간을 낭비하지 않으려면 먼저 Zattera 커뮤니티에 아이디어가 독창적인지 물어보세요.

**토론 채널**:
- Zattera 커뮤니티 포럼
- 개발자 Discord/Telegram
- GitHub Discussions

### 2. 초안 단계

아이디어가 검토되면 작성자는 다음을 수행해야 합니다:

1. **ZEP 문서 작성** - 템플릿 사용
2. **ZEP 저장소에 풀 리퀘스트 제출**
3. **편집자가 ZEP 번호 할당**
4. 상태: **Draft**

**초안 요구 사항**:
- ZEP 템플릿 형식을 따라야 함
- 모든 필수 섹션 포함 필요
- 기술적으로 타당해야 함
- 올바른 마크다운 형식이어야 함

### 3. 검토 단계

제출되면 ZEP는 커뮤니티 검토에 들어갑니다:

1. **커뮤니티 피드백** - GitHub 댓글 및 토론을 통해
2. **작성자 수정** - 피드백을 기반으로
3. **기술 검토** - 핵심 개발자에 의해
4. 더 넓은 고려를 위해 준비되면 상태가 **Review**로 변경됨

### 4. 최종 호출 단계

작성자가 ZEP가 준비되었다고 판단하면:

1. 편집자가 상태를 **Last Call**로 변경
2. **Last Call 마감일** 설정(일반적으로 14일)
3. 커뮤니티 피드백을 위한 최종 기회
4. 최종 수정 진행

### 5. 최종 단계

ZEP가 **Final**로 이동하려면:

- **Standards Track**: 참조 구현이 있어야 하며 테스트를 통과해야 함
- **Meta/Informational**: 대략적인 합의가 있어야 함
- 해결되지 않은 이의 제기가 없어야 함

상태: **Final**

### 기타 상태

- **Stagnant**: Draft 또는 Review 상태에서 6개월 이상 비활성
- **Withdrawn**: 작성자가 제안을 철회
- **Living**: 지속적으로 업데이트(예: ZEP-1)

## 성공적인 ZEP에 포함되어야 할 내용

각 ZEP에는 다음 부분이 있어야 합니다:

### 전문(Preamble)

ZEP 번호, 짧은 설명 제목(최대 44자), 설명(최대 140자), 작성자 세부 정보를 포함한 ZEP에 대한 메타데이터가 포함된 RFC 822 스타일 헤더.

### Abstract

해결되는 기술 문제에 대한 짧은(약 200단어) 설명.

### Motivation *(선택 사항)*

Motivation 섹션은 ZEP가 해결하는 문제를 다루기에 기존 프로토콜 사양이 부적절한 이유를 명확하게 설명해야 합니다. 동기가 명백한 경우 이 섹션을 생략할 수 있습니다.

### Specification

기술 사양은 새로운 기능의 구문과 의미를 설명해야 합니다. 사양은 경쟁적이고 상호 운용 가능한 구현을 허용할 만큼 충분히 상세해야 합니다.

### Rationale

Rationale은 설계를 동기 부여한 내용과 특정 설계 결정이 내려진 이유를 설명하여 사양을 구체화합니다. 고려된 대안 설계와 관련 작업을 설명해야 합니다.

### Backwards Compatibility

역호환성을 깨뜨리는 모든 ZEP는 이러한 비호환성과 그 심각성을 설명하는 섹션을 포함해야 합니다. ZEP는 작성자가 이러한 비호환성을 처리할 것을 제안하는 방법을 설명해야 합니다.

### Test Cases

합의 변경에 영향을 미치는 ZEP의 경우 구현을 위한 테스트 케이스가 필수입니다. 테스트는 ZEP에 인라인으로 데이터로 포함되거나 `../assets/zep-###/<filename>`에 포함되어야 합니다.

### Reference Implementation *(선택 사항)*

사람들이 이 사양을 이해하거나 구현하는 데 사용할 수 있는 참조/예제 구현이 포함된 선택적 섹션입니다.

### Security Considerations

모든 ZEP에는 제안된 변경과 관련된 보안 의미/고려 사항을 논의하는 섹션이 포함되어야 합니다. 이 섹션은 필수이며 잠재적인 보안 위험을 논의해야 합니다.

### Copyright

모든 ZEP는 퍼블릭 도메인에 있어야 합니다. 저작권 포기는 다음 문구를 사용해야 합니다:

```
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
```

## ZEP 형식 및 템플릿

ZEP는 마크다운 형식으로 작성되어야 합니다. 따라야 할 [템플릿](../zep-template.md)이 있습니다.

### 보조 파일

이미지, 다이어그램 및 보조 파일은 해당 ZEP의 `assets` 폴더의 하위 디렉토리에 다음과 같이 포함되어야 합니다: `assets/zep-N` (여기서 **N**은 ZEP 번호로 대체됨). ZEP에서 이미지에 링크할 때 `../assets/zep-1/image.png`와 같은 상대 링크를 사용하세요.

## ZEP 헤더 전문

각 ZEP는 YAML 형식의 RFC 822 스타일 헤더 전문으로 시작해야 합니다. 헤더는 다음 순서로 나타나야 합니다:

- `zep`: ZEP 번호
- `title`: ZEP 제목(최대 44자)
- `description`: 설명(최대 140자)
- `author`: 작성자 이름 및 연락처 정보 목록
- `discussions-to`: 공식 토론 스레드를 가리키는 URL
- `status`: Draft | Review | Last Call | Final | Stagnant | Withdrawn | Living
- `type`: Standards Track | Meta | Informational
- `category`: Core | Networking | Interface (Standards Track에만 필요)
- `created`: 작성 날짜(ISO 8601 형식: yyyy-mm-dd)
- `requires`: ZEP 번호(선택 사항)
- `withdrawal-reason`: 철회된 경우 설명(선택 사항)

### Author 헤더

`author` 헤더는 ZEP의 작성자/소유자 이름 및 연락처 정보를 나열합니다. 형식:

```
author: FirstName LastName (@GitHubUsername), FirstName LastName <email@example.com>
```

최소 한 명의 작성자는 괄호 안에 GitHub 사용자 이름을 사용하거나 꺾쇠 괄호 안에 이메일 주소를 사용해야 합니다. 이메일 주소 또는 GitHub 사용자 이름은 생략할 수 있습니다.

### discussions-to 헤더

ZEP가 초안 상태인 동안 `discussions-to` 헤더는 ZEP가 논의되는 URL을 나타냅니다. 선호하는 토론 URL은 ZEP 저장소의 GitHub Discussion입니다.

### type 헤더

`type` 헤더는 ZEP 유형을 지정합니다: Standards Track, Meta 또는 Informational.

### category 헤더

`category` 헤더는 ZEP의 카테고리를 지정합니다. 이는 Standards Track ZEP에만 필요합니다.

### created 헤더

`created` 헤더는 ZEP에 번호가 할당된 날짜를 기록합니다. ISO 8601 형식(yyyy-mm-dd)이어야 합니다.

### requires 헤더

ZEP에는 이 ZEP가 의존하는 ZEP 번호를 나타내는 `requires` 헤더가 있을 수 있습니다.

## 외부 리소스 링크

외부 리소스에 대한 링크는 **포함되어서는 안 됩니다**. 외부 리소스는 예기치 않게 사라지거나, 이동하거나, 변경될 수 있습니다.

## 다른 ZEP에 링크

다른 ZEP에 대한 참조는 `ZEP-N` 형식을 따라야 하며, 여기서 `N`은 참조하는 ZEP 번호입니다.

## ZEP 소유권 이전

때때로 ZEP 소유권을 새로운 챔피언에게 이전해야 할 필요가 있습니다. 일반적으로 우리는 원래 작성자를 이전된 ZEP의 공동 작성자로 유지하고 싶지만 이는 실제로 원래 작성자에게 달려 있습니다. 소유권을 이전하는 좋은 이유는 원래 작성자가 더 이상 업데이트하거나 ZEP 프로세스를 따를 시간이나 관심이 없거나 연락이 끊겼기 때문입니다.

## ZEP 편집자

현재 ZEP 편집자는 다음과 같습니다:

- Zattera Core Team

### ZEP 편집자 책임

제출된 각 새 ZEP에 대해 편집자는 다음을 수행합니다:

- ZEP를 읽어 준비되었고, 건전하며, 완전한지 확인
- 제목이 내용을 정확하게 설명하는지 확인
- ZEP가 형식 규칙을 따르는지 확인
- ZEP의 언어 확인(맞춤법, 문법, 문장 구조 등)
- ZEP가 준비되지 않은 경우 구체적인 지침과 함께 작성자에게 수정을 위해 다시 보냄

ZEP가 저장소에 준비되면 ZEP 편집자는 다음을 수행합니다:

- ZEP 번호 할당(일반적으로 PR 번호)
- 풀 리퀘스트 병합
- 작성자에게 다음 단계가 포함된 메시지 보내기

편집자는 ZEP에 대한 판단을 내리지 않습니다. 그들은 단지 관리 및 편집 기능을 수행할 뿐입니다.

## 스타일 가이드

### ZEP 번호

ZEP를 참조할 때 `ZEP-X` 형식으로 작성해야 하며, 여기서 `X`는 ZEP의 할당된 번호입니다.

### RFC 2119 및 RFC 8174

ZEP는 요구 사항 수준에 대한 용어로 [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.html) 및 [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174.html)를 따르는 것이 권장됩니다.

### 대문자 표기

- Zattera (고유 명사, 항상 대문자)
- ZEP/ZRC (항상 대문자)
- blockchain (소문자, 문장 시작이 아닌 경우)

## 역사

이 문서는 Ethereum의 EIP-1에서 많이 파생되었으며, EIP-1은 Amir Taaki가 작성한 Bitcoin의 BIP-0001에서 파생되었고, BIP-0001은 Python의 PEP-0001에서 파생되었습니다. 많은 부분에서 텍스트가 단순히 복사되고 수정되었습니다.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
