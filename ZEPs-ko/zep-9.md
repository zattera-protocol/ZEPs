---
zep: 9
title: 출금 컨트랙트 표준 인터페이스 (ZRC-9)
description: 크로스체인 출금 시스템에서 사용되는 스마트 컨트랙트의 표준 인터페이스 정의
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Interface
created: 2025-12-31
requires: ZEP-8
---

## 초록

본 ZEP는 **ZRC-9** 표준 인터페이스를 정의합니다. ZRC-9는 증인 서명 기반 크로스체인 ZBD 출금(ZEP-8)에서 사용되는 **EVM 호환 체인**용 출금 컨트랙트의 표준 인터페이스입니다. 이 표준을 준수하는 모든 컨트랙트는 Zattera 증인 합의를 통해 승인될 수 있으며, 사용자는 표준화된 방식으로 안전하게 ZBD를 출금할 수 있습니다.

**핵심 특징**:
- EIP-712 기반 구조화된 서명 검증
- 이중 출금 방지 메커니즘
- 증인 예치금 관리
- 18 decimals 정규화 표준 (모든 금액 단위 통일)
- 표준화된 이벤트 발행
- 조회 가능한 출금 상태

## 동기

### 표준화의 필요성

ZEP-8 시스템에서 여러 출금 컨트랙트가 존재할 수 있으며, 각 컨트랙트는 다음을 보장해야 합니다:

1. **보안성**: 증인 서명 검증, 이중 출금 방지
2. **투명성**: 출금 상태 조회, 이벤트 발행
3. **상호운용성**: 표준화된 인터페이스로 증인 플러그인과 호환
4. **검증 가능성**: 증인 합의 승인 전 표준 준수 여부 확인

### 목표

- **개발자**: 표준을 따라 쉽게 컨트랙트 개발
- **증인**: 표준 준수 여부로 컨트랙트 승인/거부 결정
- **사용자**: 모든 승인된 컨트랙트에서 일관된 사용자 경험

## 명세

### 1. 필수 인터페이스

모든 ZRC-9 호환 출금 컨트랙트는 다음 인터페이스를 구현해야 합니다.

#### 1.1 IZRC9 인터페이스

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/**
 * @title IZRC9
 * @notice ZRC-9 표준 출금 컨트랙트 인터페이스
 */
interface IZRC9 {
    /**
     * @notice 출금 실행 이벤트
     * @param requestId Zattera 출금 요청 ID
     * @param recipient 스테이블코인을 받은 주소
     * @param token 전송된 토큰 주소 (USDC, DAI, USDT 등)
     * @param amount 할인 적용된 금액 (18 decimals 정규화)
     * @param discountRate 할인율 (0~10000, 예: 500 = 5%)
     * @param witness 서명한 증인 주소
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
     * @param witness 증인 주소
     * @param token 충전된 토큰 주소
     * @param amount 충전된 토큰 금액
     */
    event Deposited(address indexed witness, address indexed token, uint256 amount);

    /**
     * @notice 사용자가 증인 서명으로 출금
     * @param requestId Zattera 출금 요청 ID (고유)
     * @param recipient 스테이블코인을 받을 주소
     * @param token 출금할 토큰 주소 (USDC, DAI, USDT 등)
     * @param amount 할인 적용된 금액 (18 decimals 정규화)
     * @param discountRate 할인율 (0~10000, 예: 500 = 5%)
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
     * @param token 충전할 토큰 주소
     * @param amount 충전할 토큰 금액
     */
    function deposit(address token, uint256 amount) external;

    /**
     * @notice 출금 처리 여부 조회
     * @param requestId Zattera 출금 요청 ID
     * @return processed 처리 완료 여부
     * @return blockNumber 처리된 블록 번호 (처리되지 않은 경우 0)
     */
    function getWithdrawStatus(uint64 requestId)
        external
        view
        returns (bool processed, uint256 blockNumber);

    /**
     * @notice 증인 예치금 잔액 조회
     * @dev **중요**: 반환값은 반드시 18 decimals로 정규화되어야 함
     *      토큰의 실제 decimals와 무관하게 항상 18 decimals로 변환하여 반환
     *      예: USDC 100개 (6 decimals) → 100 * 10^18 반환
     *          DAI 100개 (18 decimals) → 100 * 10^18 반환
     * @param witness 증인 주소
     * @param token 토큰 주소
     * @return balance 예치금 잔액 (18 decimals로 정규화)
     */
    function getWitnessDeposit(address witness, address token)
        external
        view
        returns (uint256 balance);

    // 주의: ZEP-8 기준으로 별도의 증인 인증 함수는 요구되지 않습니다.
}
```

### 2. EIP-712 서명 표준

#### 2.1 도메인 분리자 (Domain Separator)

**ZRC-9 표준 요구사항**: `name`과 `version`을 생략한 최소 도메인 사용

```solidity
/**
 * @notice ZRC-9 EIP-712 도메인 구조
 * @dev name과 version을 생략하여 구현 단순화
 */
struct EIP712Domain {
    uint256 chainId;            // 체인 ID (1=Ethereum, 137=Polygon)
    address verifyingContract;  // 컨트랙트 주소
}

/**
 * @notice 도메인 타입 해시
 */
bytes32 public constant DOMAIN_TYPEHASH = keccak256(
    "EIP712Domain(uint256 chainId,address verifyingContract)"
);
```

**표준 값**:
- `chainId`: 배포된 체인의 ID
- `verifyingContract`: `address(this)` (컨트랙트 주소로 고유성 보장)

**도메인 분리자 계산**:
```solidity
function _buildDomainSeparator() private view returns (bytes32) {
    return keccak256(abi.encode(
        DOMAIN_TYPEHASH,
        block.chainid,
        address(this)
    ));
}
```

**설계 근거**:
- `chainId`와 `verifyingContract`만으로 컨트랙트 고유성 보장
- `name`과 `version`은 불필요한 복잡도 (컨트랙트 주소가 이미 고유함)
- Zattera 체인이 추가 메타데이터 관리할 필요 없음

#### 2.2 출금 메시지 구조

```solidity
/**
 * @notice 출금 메시지 구조
 */
struct Withdrawal {
    uint64  requestId;    // Zattera 출금 요청 ID
    address recipient;    // 스테이블코인 수신 주소
    address token;        // 출금할 토큰 주소 (USDC, DAI, USDT 등)
    uint256 amount;       // 할인 적용된 금액 (18 decimals 정규화)
    uint16  discountRate; // 할인율 (0~10000)
}

/**
 * @notice 출금 메시지 타입 해시
 */
bytes32 public constant WITHDRAWAL_TYPEHASH = keccak256(
    "Withdrawal(uint64 requestId,address recipient,address token,uint256 amount,uint16 discountRate)"
);
```

#### 3.3 서명 검증

```solidity
/**
 * @notice EIP-712 서명 검증 로직
 */
function _verifySignature(
    uint64 requestId,
    address recipient,
    address token,
    uint256 amount,
    uint16 discountRate,
    bytes calldata signature
) internal view returns (address signer) {
    bytes32 structHash = keccak256(abi.encode(
        WITHDRAWAL_TYPEHASH,
        requestId,
        recipient,
        token,
        amount,
        discountRate
    ));

    bytes32 digest = _hashTypedDataV4(structHash);

    return ECDSA.recover(digest, signature);
}
```

### 3. 필수 보안 요구사항

#### 3.1 이중 출금 방지

```solidity
// 처리된 출금 요청 추적
mapping(uint64 => bool) public processedWithdrawals;

function withdraw(
    uint64 requestId,
    address recipient,
    address token,
    uint256 amount,
    uint16 discountRate,
    bytes calldata witnessSignature
) external {
    // 1. 이중 출금 방지
    require(!processedWithdrawals[requestId], "Already processed");

    // ... 서명 검증 및 출금 처리 ...

    // 처리 완료 표시
    processedWithdrawals[requestId] = true;
}
```

#### 3.2 예치금 검증

```solidity
// 증인 예치금 (witness => token => balance)
mapping(address => mapping(address => uint256)) public witnessDeposits;

function withdraw(
    uint64 requestId,
    address recipient,
    address token,
    uint256 amount,        // 18 decimals 정규화
    uint16 discountRate,
    bytes calldata witnessSignature
) external {
    address witness = _verifySignature(requestId, recipient, token, amount, discountRate, witnessSignature);

    // 18 decimals → 토큰 native decimals 변환
    // 예: amount = 95 * 10^18 (18 decimals)
    //     USDC (6 decimals): 95 * 10^18 / 10^12 = 95 * 10^6
    //     DAI (18 decimals): 95 * 10^18 / 10^0 = 95 * 10^18
    //     YAM (24 decimals): 95 * 10^18 * 10^6 = 95 * 10^24
    uint8 tokenDecimals = IERC20Metadata(token).decimals();
    uint256 tokenAmount;
    if (tokenDecimals >= NORMALIZED_DECIMALS) {
        tokenAmount = amount * (10 ** (tokenDecimals - NORMALIZED_DECIMALS));
    } else {
        tokenAmount = amount / (10 ** (NORMALIZED_DECIMALS - tokenDecimals));
    }

    // 예치금 충분성 확인 (18 decimals 정규화 비교)
    uint256 depositBalance18 = getWitnessDeposit(witness, token); // 18 decimals
    require(depositBalance18 >= amount, "Insufficient deposit");

    // 예치금 차감 (native decimals)
    witnessDeposits[witness][token] -= tokenAmount;

    // ... 토큰 전송 ...
}
```

### 4. 상태 조회 인터페이스

#### 4.1 출금 정보 조회

```solidity
// 출금 정보 저장
struct WithdrawInfo {
    bool processed;
    uint256 blockNumber;  // 처리된 블록 번호
    uint256 timestamp;
}

mapping(uint64 => WithdrawInfo) private withdrawInfos;

function getWithdrawStatus(uint64 requestId)
    external
    view
    returns (bool processed, uint256 blockNumber)
{
    WithdrawInfo memory info = withdrawInfos[requestId];
    return (info.processed, info.blockNumber);
}
```

**중요**:
- Zattera 증인 플러그인은 이 함수를 사용하여 출금 처리 여부를 확인하므로, 반드시 구현해야 합니다.
- `blockNumber`를 반환함으로써 증인은 해당 블록에서만 `Withdrawn` 이벤트를 필터링하여 효율적으로 트랜잭션 해시를 찾을 수 있습니다.
- 스마트 컨트랙트는 자신의 트랜잭션 해시(`tx.hash`)를 내부에서 알 수 없으므로 `blockNumber`를 사용합니다.

### 5. 참조 구현

#### 5.1 기본 컨트랙트 구조

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title ZatteraBridge
 * @notice ZEP-9 호환 출금 컨트랙트 참조 구현
 */
contract ZatteraBridge is IZRC9, Ownable {
    using ECDSA for bytes32;

    // 상태 변수
    uint8 public constant NORMALIZED_DECIMALS = 18;  // ZRC-9 표준 정규화 단위

    // 지원하는 토큰 목록
    mapping(address => bool) public supportedTokens;

    // 이중 출금 방지
    mapping(uint64 => bool) public processedWithdrawals;

    // 출금 정보
    mapping(uint64 => WithdrawInfo) private withdrawInfos;

    mapping(address => mapping(address => uint256)) public witnessDeposits; // witness => token => balance

    // EIP-712 타입 해시
    bytes32 public constant DOMAIN_TYPEHASH = keccak256(
        "EIP712Domain(uint256 chainId,address verifyingContract)"
    );

    bytes32 public constant WITHDRAWAL_TYPEHASH = keccak256(
        "Withdrawal(uint64 requestId,address recipient,address token,uint256 amount,uint16 discountRate)"
    );

    struct WithdrawInfo {
        bool processed;
        uint256 blockNumber;
        uint256 timestamp;
    }

    constructor(address[] memory _supportedTokens) {
        for (uint i = 0; i < _supportedTokens.length; i++) {
            supportedTokens[_supportedTokens[i]] = true;
        }
    }

    /**
     * @notice EIP-712 도메인 분리자 계산 (ZRC-9 표준)
     */
    function _domainSeparatorV4() internal view returns (bytes32) {
        return keccak256(abi.encode(
            DOMAIN_TYPEHASH,
            block.chainid,
            address(this)
        ));
    }

    /**
     * @notice EIP-712 해시 생성 (ZRC-9 표준)
     */
    function _hashTypedDataV4(bytes32 structHash) internal view returns (bytes32) {
        return keccak256(abi.encodePacked(
            "\x19\x01",
            _domainSeparatorV4(),
            structHash
        ));
    }

    function addSupportedToken(address token) external onlyOwner {
        supportedTokens[token] = true;
    }

    function removeSupportedToken(address token) external onlyOwner {
        supportedTokens[token] = false;
    }

    // IZRC9 구현

    function withdraw(
        uint64 requestId,
        address recipient,
        address token,
        uint256 amount,        // 18 decimals 정규화
        uint16 discountRate,
        bytes calldata witnessSignature
    ) external override {
        // 1. 토큰 지원 확인
        require(supportedTokens[token], "Token not supported");

        // 2. 이중 출금 방지
        require(!processedWithdrawals[requestId], "Already processed");

        // 3. 서명 검증
        bytes32 structHash = keccak256(abi.encode(
            WITHDRAWAL_TYPEHASH,
            requestId,
            recipient,
            token,
            amount,
            discountRate
        ));
        bytes32 digest = _hashTypedDataV4(structHash);
        address witness = digest.recover(witnessSignature);

        // 4. 18 decimals → 토큰 native decimals 변환
        // 예: amount = 95 * 10^18
        //     USDC (6 decimals): 95 * 10^18 / 10^12 = 95000000
        //     DAI (18 decimals): 95 * 10^18 / 10^0 = 95 * 10^18
        //     YAM (24 decimals): 95 * 10^18 * 10^6 = 95000000000000000000000000
        uint8 tokenDecimals = IERC20Metadata(token).decimals();
        uint256 tokenAmount;
        if (tokenDecimals >= NORMALIZED_DECIMALS) {
            tokenAmount = amount * (10 ** (tokenDecimals - NORMALIZED_DECIMALS));
        } else {
            tokenAmount = amount / (10 ** (NORMALIZED_DECIMALS - tokenDecimals));
        }

        // 5. 예치금 확인 (18 decimals 비교) 및 차감 (native decimals)
        uint256 depositBalance18 = getWitnessDeposit(witness, token);
        require(depositBalance18 >= amount, "Insufficient deposit");
        witnessDeposits[witness][token] -= tokenAmount;

        // 6. 토큰 전송
        require(IERC20(token).transfer(recipient, tokenAmount), "Transfer failed");

        // 7. 처리 완료 표시
        processedWithdrawals[requestId] = true;
        withdrawInfos[requestId] = WithdrawInfo({
            processed: true,
            blockNumber: block.number,
            timestamp: block.timestamp
        });

        // 8. 이벤트 emit (18 decimals)
        emit Withdrawn(requestId, recipient, token, amount, discountRate, witness);
    }

    function deposit(address token, uint256 amount) external override {
        require(supportedTokens[token], "Token not supported");
        require(IERC20(token).transferFrom(msg.sender, address(this), amount), "Transfer failed");

        witnessDeposits[msg.sender][token] += amount;
        emit Deposited(msg.sender, token, amount);
    }

    function getWitnessDeposit(address witness, address token)
        external
        view
        override
        returns (uint256 balance)
    {
        // ZRC-9 스펙: 18 decimals로 정규화하여 반환
        uint256 tokenBalance = witnessDeposits[witness][token];
        uint8 tokenDecimals = IERC20Metadata(token).decimals();

        // 토큰 decimals → 18 decimals 변환
        if (tokenDecimals < NORMALIZED_DECIMALS) {
            return tokenBalance * (10 ** (NORMALIZED_DECIMALS - tokenDecimals));
        } else if (tokenDecimals > NORMALIZED_DECIMALS) {
            return tokenBalance / (10 ** (tokenDecimals - NORMALIZED_DECIMALS));
        } else {
            return tokenBalance;  // 이미 18 decimals
        }
    }

    function getWithdrawStatus(uint64 requestId)
        external
        view
        override
        returns (bool processed, uint256 blockNumber)
    {
        WithdrawInfo memory info = withdrawInfos[requestId];
        return (info.processed, info.blockNumber);
    }


    // ERC-165 지원 여부는 선택 사항입니다.
}
```

### 6. 선택적 기능

다음 기능들은 선택적이지만 권장됩니다:

#### 6.1 수수료 메커니즘

```solidity
// 컨트랙트 수수료 (선택)
uint256 public feePercentage = 10; // 0.1% (10/10000)
address public feeRecipient;

function withdraw(
    uint64 requestId,
    address recipient,
    address token,
    uint256 amount,        // 18 decimals 정규화
    uint16 discountRate,
    bytes calldata witnessSignature
) external override {
    // ... 기존 검증 로직 ...

    // 18 decimals → 토큰 native decimals 변환
    uint8 tokenDecimals = IERC20Metadata(token).decimals();
    uint256 tokenAmount;
    if (tokenDecimals >= NORMALIZED_DECIMALS) {
        tokenAmount = amount * (10 ** (tokenDecimals - NORMALIZED_DECIMALS));
    } else {
        tokenAmount = amount / (10 ** (NORMALIZED_DECIMALS - tokenDecimals));
    }
    uint256 fee = (tokenAmount * feePercentage) / 10000;
    uint256 netAmount = tokenAmount - fee;

    // 예치금 차감
    witnessDeposits[witness][token] -= tokenAmount;

    // 수수료 전송
    if (fee > 0) {
        require(IERC20(token).transfer(feeRecipient, fee), "Fee transfer failed");
    }

    // 사용자에게 순액 전송
    require(IERC20(token).transfer(recipient, netAmount), "Transfer failed");

    // ... 나머지 로직 ...
}
```

### 7. 테스트 요구사항

ZEP-9 준수를 검증하기 위한 최소 테스트 케이스:

#### 7.1 필수 테스트

1. **인터페이스 준수**
   - 모든 필수 함수 존재 확인
   - (선택) ERC-165 지원 시 `supportsInterface(type(IZRC9).interfaceId)` 호출 시 `true` 반환

2. **서명 검증**
   - 유효한 EIP-712 서명 허용
   - 잘못된 서명 거부
   - 다른 컨트랙트의 서명 거부

3. **이중 출금 방지**
   - 동일한 `requestId`로 두 번째 출금 시도 시 실패

4. **예치금 관리**
   - 충분한 예치금이 있는 경우만 출금 허용
   - 예치금 부족 시 출금 거부
   - 예치금 충전 및 출금 시 차감 정상 작동

5. **상태 조회**
   - `getWithdrawStatus()` 정확한 정보 반환
   - 처리되지 않은 요청에 대해 `processed=false` 반환

#### 8.2 테스트 예제 (Hardhat/Foundry)

```javascript
describe("ZEP-9 Compliance", function () {
    it("should support ZRC-9 interface (ERC-165, optional)", async function () {
        const interfaceId = "0x????????"; // IZRC9 interface ID
        expect(await contract.supportsInterface(interfaceId)).to.be.true;
    });

    it("should prevent double withdrawal", async function () {
        const requestId = 12345;
        const discountRate = 500;
        const signature = await createSignature(requestId, recipient, token, amount, discountRate);

        // 첫 번째 출금 성공
        await contract.withdraw(requestId, recipient, token, amount, discountRate, signature);

        // 두 번째 출금 실패
        await expect(
            contract.withdraw(requestId, recipient, token, amount, discountRate, signature)
        ).to.be.revertedWith("Already processed");
    });

    it("should verify EIP-712 signatures correctly", async function () {
        const requestId = 12345;
        const discountRate = 500;
        const invalidSignature = "0x00..."; // 잘못된 서명

        await expect(
            contract.withdraw(requestId, recipient, token, amount, discountRate, invalidSignature)
        ).to.be.reverted;
    });
});
```

### 8. 증인 합의 승인 기준

Zattera 증인들이 컨트랙트 승인 시 확인해야 할 체크리스트:

- [ ] 컨트랙트 소스 코드 공개 및 검증됨 (Etherscan/Polygonscan)
- [ ] 모든 필수 함수 구현됨
- [ ] (선택) ERC-165 지원 시 `supportsInterface(type(IZRC9).interfaceId)` returns `true`
- [ ] EIP-712 서명 검증 올바르게 구현됨
- [ ] 이중 출금 방지 메커니즘 존재
- [ ] 보안 감사 완료 (선택, 권장)
- [ ] 테스트 커버리지 80% 이상 (선택, 권장)

## 보안 고려사항

### 1. 재진입 공격 방지

```solidity
// OpenZeppelin ReentrancyGuard 사용 권장
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract ZatteraBridge is IZRC9, ReentrancyGuard {
    function withdraw(...) external override nonReentrant {
        // ...
    }
}
```

### 2. 정수 오버플로우/언더플로우

Solidity 0.8.0+ 사용 시 자동 방지됨. 하위 버전 사용 시 SafeMath 필수.

### 3. 서명 재사용 공격 (Cross-Contract Replay Attack)

**문제**: 동일한 증인 서명을 다른 컨트랙트에서 재사용하는 공격

**방어 메커니즘**:

#### 3.1 Request ID 기반 방어
`requestId`가 전역적으로 고유함을 보장:
```solidity
mapping(uint64 => bool) public processedWithdrawals;

function withdraw(...) external {
    require(!processedWithdrawals[requestId], "Already processed");
    // ...
    processedWithdrawals[requestId] = true;
}
```

#### 3.2 EIP-712 도메인 바인딩 (핵심)

**ZRC-9 표준 구현** (minimal EIP-712 domain):
```solidity
contract ZatteraBridge is IZRC9 {
    // EIP-712 타입 해시
    bytes32 public constant DOMAIN_TYPEHASH = keccak256(
        "EIP712Domain(uint256 chainId,address verifyingContract)"
    );

    bytes32 public constant WITHDRAWAL_TYPEHASH = keccak256(
        "Withdrawal(uint64 requestId,address recipient,address token,uint256 amount,uint16 discountRate)"
    );

    constructor(address[] memory _supportedTokens) {
        for (uint i = 0; i < _supportedTokens.length; i++) {
            supportedTokens[_supportedTokens[i]] = true;
        }
    }

    /**
     * @notice EIP-712 도메인 분리자 계산 (ZRC-9 표준)
     */
    function _domainSeparatorV4() internal view returns (bytes32) {
        return keccak256(abi.encode(
            DOMAIN_TYPEHASH,
            block.chainid,
            address(this)  // 이 컨트랙트 주소로 바인딩 (핵심!)
        ));
    }

    /**
     * @notice EIP-712 해시 생성 (ZRC-9 표준)
     */
    function _hashTypedDataV4(bytes32 structHash) internal view returns (bytes32) {
        return keccak256(abi.encodePacked(
            "\x19\x01",
            _domainSeparatorV4(),
            structHash
        ));
    }

    function withdraw(...) external {
        // 서명 검증
        bytes32 structHash = keccak256(abi.encode(
            WITHDRAWAL_TYPEHASH,
            requestId,
            recipient,
            token,
            amount,
            discountRate
        ));

        // _hashTypedDataV4는 내부적으로 address(this)를 사용
        bytes32 digest = _hashTypedDataV4(structHash);
        address witness = digest.recover(witnessSignature);
        // Zattera 체인에서 선택된 증인만 유효한 서명 생성 가능
        // ...
    }
}
```

**보안 보장**:
- `_domainSeparatorV4()`는 컨트랙트 주소(`address(this)`)를 EIP-712 도메인에 포함
- Contract A의 서명은 Contract A에서만 유효
- Contract B의 서명은 Contract B에서만 유효
- 다른 컨트랙트에서 서명 재사용 시 자동으로 검증 실패 ✅

**ZRC-9 EIP-712 도메인 구조** (최소 필드):
```solidity
// ZRC-9 표준 도메인 (name/version 생략)
{
    chainId: block.chainid,          // 체인 ID로 크로스체인 보호
    verifyingContract: address(this)  // 컨트랙트 주소로 고유성 보장 (핵심!)
}
```

**증인 측 서명 생성** (ZEP-8 참조):
- 증인은 사용자가 지정한 컨트랙트 주소로 EIP-712 서명 생성
- 서명은 해당 컨트랙트 주소에 바인딩됨
- 크로스 컨트랙트 재사용 불가능

**참고**: ZEP-8 섹션 6.5 참조

### 4. 프론트러닝

사용자가 트랜잭션을 제출하므로 프론트러닝 위험 낮음.

## 하위 호환성

본 ZEP는 새로운 표준이므로 하위 호환성 문제가 없습니다.

ZEP-8과 함께 사용되도록 설계되었으며, ZEP-8 시스템에서 사용되는 모든 컨트랙트는 ZEP-9를 준수해야 합니다.

## 테스트 케이스

참조 구현의 전체 테스트 스위트는 다음 위치에 있습니다:
- `contracts/test/ZatteraBridge.test.ts`

## 참조 구현

- `contracts/ZatteraBridge.sol` - ZEP-9 표준 참조 구현
- `contracts/interfaces/IZRC9.sol` - ZRC-9 인터페이스

## 저작권

본 문서는 CC0 1.0 Universal 라이선스에 따라 공개 도메인에 배치됩니다.
