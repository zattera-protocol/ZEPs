---
zep: 9
title: Withdrawal Contract Standard Interface (ZRC-9)
description: Standard interface definition for smart contracts used in the cross-chain withdrawal system
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Interface
created: 2025-12-31
requires: ZEP-8
---

## Abstract

This ZEP defines the **ZRC-9** standard interface. ZRC-9 is the standard withdrawal contract interface for **EVM-compatible chains** used by the witness-signature-based cross-chain ZBD withdrawal system (ZEP-8). Any contract that complies with this standard can be approved by witness consensus, and users can safely withdraw ZBD using a standardized flow.

**Key features**:
- EIP-712 structured signature verification
- Double-withdrawal prevention
- Witness deposit management
- 18-decimal normalization standard (consistent amount units)
- Standardized event emission
- Queryable withdrawal status

## Motivation

### Need for standardization

In ZEP-8, multiple withdrawal contracts may exist, and each contract must guarantee:

1. **Security**: witness signature verification, double-withdrawal prevention
2. **Transparency**: queryable withdrawal status, event emission
3. **Interoperability**: compatible with witness plugins via a standardized interface
4. **Verifiability**: witnesses can verify compliance before approval

### Goals

- **Developers**: build contracts easily by following a standard
- **Witnesses**: decide approval/rejection based on standard compliance
- **Users**: consistent UX across all approved contracts

## Specification

### 1. Required interface

All ZRC-9 compatible withdrawal contracts must implement the following interface.

#### 1.1 IZRC9 interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/**
 * @title IZRC9
 * @notice ZRC-9 standard withdrawal contract interface
 */
interface IZRC9 {
    /**
     * @notice Withdrawal execution event
     * @param requestId Zattera withdrawal request ID
     * @param recipient Address that received stablecoins
     * @param token Token address sent (USDC, DAI, USDT, etc.)
     * @param amount Discounted amount (18 decimals normalized)
     * @param discountRate Discount rate (0~10000, e.g., 500 = 5%)
     * @param witness Witness address that signed
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
     * @notice Witness deposit event
     * @param witness Witness address
     * @param token Token address deposited
     * @param amount Amount deposited
     */
    event Deposited(address indexed witness, address indexed token, uint256 amount);

    /**
     * @notice User withdrawal with witness signature
     * @param requestId Zattera withdrawal request ID (unique)
     * @param recipient Recipient address
     * @param token Token address (USDC, DAI, USDT, etc.)
     * @param amount Discounted amount (18 decimals normalized)
     * @param discountRate Discount rate (0~10000, e.g., 500 = 5%)
     * @param witnessSignature Witness EIP-712 signature
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
     * @notice Witness deposit
     * @param token Token address to deposit
     * @param amount Amount to deposit
     */
    function deposit(address token, uint256 amount) external;

    /**
     * @notice Query withdrawal status
     * @param requestId Zattera withdrawal request ID
     * @return processed Whether processed
     * @return blockNumber Processed block number (0 if not processed)
     */
    function getWithdrawStatus(uint64 requestId)
        external
        view
        returns (bool processed, uint256 blockNumber);

    /**
     * @notice Query witness deposit balance
     * @dev Important: return value must be normalized to 18 decimals
     *      regardless of token decimals
     *      Example: USDC 100 (6 decimals) -> return 100 * 10^18
     *               DAI 100 (18 decimals) -> return 100 * 10^18
     * @param witness Witness address
     * @param token Token address
     * @return balance Deposit balance (18 decimals normalized)
     */
    function getWitnessDeposit(address witness, address token)
        external
        view
        returns (uint256 balance);

    // Note: ZEP-8 does not require a separate witness auth function.
}
```

### 2. EIP-712 signature standard

#### 2.1 Domain separator

**ZRC-9 requirement**: minimal domain with `name` and `version` omitted

```solidity
/**
 * @notice ZRC-9 EIP-712 domain structure
 * @dev Omit name and version for simpler implementation
 */
struct EIP712Domain {
    uint256 chainId;            // Chain ID (1=Ethereum, 137=Polygon)
    address verifyingContract;  // Contract address
}

/**
 * @notice Domain type hash
 */
bytes32 public constant DOMAIN_TYPEHASH = keccak256(
    "EIP712Domain(uint256 chainId,address verifyingContract)"
);
```

**Standard values**:
- `chainId`: chain ID of deployment
- `verifyingContract`: `address(this)` (contract uniqueness)

**Domain separator calculation**:
```solidity
function _buildDomainSeparator() private view returns (bytes32) {
    return keccak256(abi.encode(
        DOMAIN_TYPEHASH,
        block.chainid,
        address(this)
    ));
}
```

**Rationale**:
- `chainId` and `verifyingContract` are sufficient for uniqueness
- `name` and `version` add unnecessary complexity (contract address is already unique)
- Zattera chain does not need additional metadata

#### 2.2 Withdrawal message structure

```solidity
/**
 * @notice Withdrawal message structure
 */
struct Withdrawal {
    uint64  requestId;    // Zattera withdrawal request ID
    address recipient;    // Stablecoin recipient address
    address token;        // Token address (USDC, DAI, USDT, etc.)
    uint256 amount;       // Discounted amount (18 decimals normalized)
    uint16  discountRate; // Discount rate (0~10000)
}

/**
 * @notice Withdrawal message type hash
 */
bytes32 public constant WITHDRAWAL_TYPEHASH = keccak256(
    "Withdrawal(uint64 requestId,address recipient,address token,uint256 amount,uint16 discountRate)"
);
```

#### 3.3 Signature verification

```solidity
/**
 * @notice EIP-712 signature verification logic
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

### 3. Required security requirements

#### 3.1 Double-withdrawal prevention

```solidity
// Track processed withdrawals
mapping(uint64 => bool) public processedWithdrawals;

function withdraw(
    uint64 requestId,
    address recipient,
    address token,
    uint256 amount,
    uint16 discountRate,
    bytes calldata witnessSignature
) external {
    // 1. Prevent double withdrawal
    require(!processedWithdrawals[requestId], "Already processed");

    // ... signature verification and withdrawal logic ...

    // Mark as processed
    processedWithdrawals[requestId] = true;
}
```

#### 3.2 Deposit verification

```solidity
// Witness deposits (witness => token => balance)
mapping(address => mapping(address => uint256)) public witnessDeposits;

function withdraw(
    uint64 requestId,
    address recipient,
    address token,
    uint256 amount,        // 18 decimals normalized
    uint16 discountRate,
    bytes calldata witnessSignature
) external {
    address witness = _verifySignature(requestId, recipient, token, amount, discountRate, witnessSignature);

    // Convert 18 decimals -> token native decimals
    // Example: amount = 95 * 10^18 (18 decimals)
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

    // Check deposit sufficiency (18 decimals comparison)
    uint256 depositBalance18 = getWitnessDeposit(witness, token); // 18 decimals
    require(depositBalance18 >= amount, "Insufficient deposit");

    // Deduct deposit (native decimals)
    witnessDeposits[witness][token] -= tokenAmount;

    // ... transfer tokens ...
}
```

### 4. State query interface

#### 4.1 Withdrawal info query

```solidity
// Store withdrawal info
struct WithdrawInfo {
    bool processed;
    uint256 blockNumber;  // Processed block number
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

**Important**:
- The witness plugin uses this function to check completion and therefore it must be implemented.
- Returning `blockNumber` allows witnesses to filter the `Withdrawn` event at that exact block to find the tx hash efficiently.
- A smart contract cannot know its own transaction hash internally, so `blockNumber` is used.

### 5. Reference implementation

#### 5.1 Basic contract structure

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title ZatteraBridge
 * @notice ZEP-9 compliant withdrawal contract reference implementation
 */
contract ZatteraBridge is IZRC9, Ownable {
    using ECDSA for bytes32;

    // State variables
    uint8 public constant NORMALIZED_DECIMALS = 18;  // ZRC-9 normalization unit

    // Supported token list
    mapping(address => bool) public supportedTokens;

    // Double-withdrawal prevention
    mapping(uint64 => bool) public processedWithdrawals;

    // Withdrawal info
    mapping(uint64 => WithdrawInfo) private withdrawInfos;

    mapping(address => mapping(address => uint256)) public witnessDeposits; // witness => token => balance

    // EIP-712 type hashes
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
     * @notice EIP-712 domain separator (ZRC-9 standard)
     */
    function _domainSeparatorV4() internal view returns (bytes32) {
        return keccak256(abi.encode(
            DOMAIN_TYPEHASH,
            block.chainid,
            address(this)
        ));
    }

    /**
     * @notice EIP-712 hash (ZRC-9 standard)
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

    // IZRC9 implementation

    function withdraw(
        uint64 requestId,
        address recipient,
        address token,
        uint256 amount,        // 18 decimals normalized
        uint16 discountRate,
        bytes calldata witnessSignature
    ) external override {
        // 1. Token support check
        require(supportedTokens[token], "Token not supported");

        // 2. Double withdrawal prevention
        require(!processedWithdrawals[requestId], "Already processed");

        // 3. Signature verification
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

        // 4. Convert 18 decimals -> token native decimals
        // Example: amount = 95 * 10^18
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

        // 5. Deposit check (18 decimals) and deduct (native decimals)
        uint256 depositBalance18 = getWitnessDeposit(witness, token);
        require(depositBalance18 >= amount, "Insufficient deposit");
        witnessDeposits[witness][token] -= tokenAmount;

        // 6. Transfer tokens
        require(IERC20(token).transfer(recipient, tokenAmount), "Transfer failed");

        // 7. Mark processed
        processedWithdrawals[requestId] = true;
        withdrawInfos[requestId] = WithdrawInfo({
            processed: true,
            blockNumber: block.number,
            timestamp: block.timestamp
        });

        // 8. Emit event (18 decimals)
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
        // ZRC-9 spec: return 18 decimals normalized
        uint256 tokenBalance = witnessDeposits[witness][token];
        uint8 tokenDecimals = IERC20Metadata(token).decimals();

        // Convert token decimals -> 18 decimals
        if (tokenDecimals < NORMALIZED_DECIMALS) {
            return tokenBalance * (10 ** (NORMALIZED_DECIMALS - tokenDecimals));
        } else if (tokenDecimals > NORMALIZED_DECIMALS) {
            return tokenBalance / (10 ** (tokenDecimals - NORMALIZED_DECIMALS));
        } else {
            return tokenBalance;  // Already 18 decimals
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


    // ERC-165 support is optional.
}
```

### 6. Optional features

The following features are optional but recommended:

#### 6.1 Fee mechanism

```solidity
// Contract fee (optional)
uint256 public feePercentage = 10; // 0.1% (10/10000)
address public feeRecipient;

function withdraw(
    uint64 requestId,
    address recipient,
    address token,
    uint256 amount,        // 18 decimals normalized
    uint16 discountRate,
    bytes calldata witnessSignature
) external override {
    // ... existing validation ...

    // Convert 18 decimals -> token native decimals
    uint8 tokenDecimals = IERC20Metadata(token).decimals();
    uint256 tokenAmount;
    if (tokenDecimals >= NORMALIZED_DECIMALS) {
        tokenAmount = amount * (10 ** (tokenDecimals - NORMALIZED_DECIMALS));
    } else {
        tokenAmount = amount / (10 ** (NORMALIZED_DECIMALS - tokenDecimals));
    }
    uint256 fee = (tokenAmount * feePercentage) / 10000;
    uint256 netAmount = tokenAmount - fee;

    // Deduct deposit
    witnessDeposits[witness][token] -= tokenAmount;

    // Transfer fee
    if (fee > 0) {
        require(IERC20(token).transfer(feeRecipient, fee), "Fee transfer failed");
    }

    // Transfer net amount to user
    require(IERC20(token).transfer(recipient, netAmount), "Transfer failed");

    // ... remaining logic ...
}
```

### 7. Test requirements

Minimum test cases for ZEP-9 compliance:

#### 7.1 Required tests

1. **Interface compliance**
   - Ensure all required functions exist
   - (Optional) If ERC-165 supported, `supportsInterface(type(IZRC9).interfaceId)` returns `true`

2. **Signature verification**
   - Accept valid EIP-712 signatures
   - Reject invalid signatures
   - Reject signatures for other contracts

3. **Double withdrawal prevention**
   - Second withdrawal with same `requestId` fails

4. **Deposit management**
   - Allow withdrawal only with sufficient deposit
   - Reject withdrawal when deposit is insufficient
   - Deposit and withdrawal updates balances correctly

5. **Status query**
   - `getWithdrawStatus()` returns correct info
   - Returns `processed=false` for unprocessed requests

#### 8.2 Test example (Hardhat/Foundry)

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

        // First withdrawal succeeds
        await contract.withdraw(requestId, recipient, token, amount, discountRate, signature);

        // Second withdrawal fails
        await expect(
            contract.withdraw(requestId, recipient, token, amount, discountRate, signature)
        ).to.be.revertedWith("Already processed");
    });

    it("should verify EIP-712 signatures correctly", async function () {
        const requestId = 12345;
        const discountRate = 500;
        const invalidSignature = "0x00..."; // Invalid signature

        await expect(
            contract.withdraw(requestId, recipient, token, amount, discountRate, invalidSignature)
        ).to.be.reverted;
    });
});
```

### 8. Witness consensus approval criteria

Checklist for witnesses when approving a contract:

- [ ] Contract source code published and verified (Etherscan/Polygonscan)
- [ ] All required functions implemented
- [ ] (Optional) If ERC-165 supported, `supportsInterface(type(IZRC9).interfaceId)` returns `true`
- [ ] EIP-712 signature verification implemented correctly
- [ ] Double-withdrawal prevention exists
- [ ] Security audit completed (optional, recommended)
- [ ] Test coverage >= 80% (optional, recommended)

## Security considerations

### 1. Reentrancy protection

```solidity
// OpenZeppelin ReentrancyGuard recommended
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract ZatteraBridge is IZRC9, ReentrancyGuard {
    function withdraw(...) external override nonReentrant {
        // ...
    }
}
```

### 2. Integer overflow/underflow

Automatically prevented in Solidity 0.8.0+. For older versions, SafeMath is required.

### 3. Signature replay attack (cross-contract replay)

**Problem**: reusing the same witness signature on another contract

**Defense mechanisms**:

#### 3.1 Request ID based defense
Ensure `requestId` is globally unique:
```solidity
mapping(uint64 => bool) public processedWithdrawals;

function withdraw(...) external {
    require(!processedWithdrawals[requestId], "Already processed");
    // ...
    processedWithdrawals[requestId] = true;
}
```

#### 3.2 EIP-712 domain binding (core)

**ZRC-9 standard implementation** (minimal EIP-712 domain):
```solidity
contract ZatteraBridge is IZRC9 {
    // EIP-712 type hashes
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
     * @notice EIP-712 domain separator (ZRC-9 standard)
     */
    function _domainSeparatorV4() internal view returns (bytes32) {
        return keccak256(abi.encode(
            DOMAIN_TYPEHASH,
            block.chainid,
            address(this)  // Bind to this contract (core)
        ));
    }

    /**
     * @notice EIP-712 hash (ZRC-9 standard)
     */
    function _hashTypedDataV4(bytes32 structHash) internal view returns (bytes32) {
        return keccak256(abi.encodePacked(
            "\x19\x01",
            _domainSeparatorV4(),
            structHash
        ));
    }

    function withdraw(...) external {
        // Signature verification
        bytes32 structHash = keccak256(abi.encode(
            WITHDRAWAL_TYPEHASH,
            requestId,
            recipient,
            token,
            amount,
            discountRate
        ));

        // _hashTypedDataV4 uses address(this) internally
        bytes32 digest = _hashTypedDataV4(structHash);
        address witness = digest.recover(witnessSignature);
        // Only the selected witness on Zattera can sign a valid message
        // ...
    }
}
```

**Security guarantee**:
- `_domainSeparatorV4()` includes the contract address (`address(this)`)
- Signature for Contract A is valid only on Contract A
- Signature for Contract B is valid only on Contract B
- Reuse on another contract fails verification

**ZRC-9 EIP-712 domain structure** (minimal fields):
```solidity
// ZRC-9 standard domain (name/version omitted)
{
    chainId: block.chainid,          // Chain ID for cross-chain protection
    verifyingContract: address(this)  // Contract address for uniqueness (core)
}
```

**Witness-side signature generation** (see ZEP-8):
- Witnesses generate signatures with the user-specified contract address
- Signatures are bound to that contract
- Cross-contract replay is impossible

**Note**: See ZEP-8 section 6.5

### 4. Front-running

Front-running risk is low because the user submits the transaction.

## Backwards compatibility

This ZEP introduces a new standard, so there are no backwards compatibility issues.

It is designed to work with ZEP-8, and all contracts used in the ZEP-8 system must comply with ZEP-9.

## Test cases

Full test suite for the reference implementation is located at:
- `contracts/test/ZatteraBridge.test.ts`

## Reference implementation

- `contracts/ZatteraBridge.sol` - ZEP-9 reference implementation
- `contracts/interfaces/IZRC9.sol` - ZRC-9 interface

## Copyright

This document is placed in the public domain under CC0 1.0 Universal.
