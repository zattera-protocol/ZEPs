---
zep: 8
title: Witness Signature Based Cross-Chain ZBD Withdrawal
description: A system where users withdraw ZBD directly on an external chain using EIP-712 signatures from witnesses
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2025-12-31
---

## Abstract

This ZEP proposes a one-way bridge system that allows ZBD (Zattera Backed Dollars) on the Zattera blockchain to be withdrawn as stablecoins (USDC, DAI, USDT, etc.) on external EVM-compatible chains. When a witness generates an EIP-712 signature for a user's withdrawal request, the user uses that signature to withdraw assets directly from a smart contract on the external chain. The witness deposits stablecoins on the external chain and receives ZBD in exchange for providing signatures, converted into ZP (VESTS) after 3.5 days using the average price.

**Key features**:
- Witnesses only sign (no gas cost)
- Users execute withdrawals themselves (full self-custody)
- Exchange-rate-based competition (lower exchange rate = higher selection probability)
- Witness rewards are converted to ZP (VESTS) (3.5-day delay, average price)
- Price manipulation resistance (same mechanism as `convert_request_object`)
- Remaining ZBD is burned, contributing to network deflation
- Double-withdrawal prevention (smart contract verification)
- Witness deposit system

**Important**: This system is **withdrawal-only**. Deposits from external chains into Zattera are not supported.

## Motivation

### Current limitations of ZBD

ZBD is an internal stablecoin with the following constraints:

1. **Limited liquidity**: usable only within the Zattera chain
2. **Difficult cash-out**: cannot be directly converted to fiat or external assets
3. **No access to external DeFi**: cannot use Ethereum DeFi protocols

### Benefits of a one-way bridge

By adopting a **withdrawal-only** bridge instead of a two-way bridge:

#### 1. Significantly improved security
- Fake deposit attack vectors are completely removed
- No double-deposit risk
- 50% reduction in code complexity

#### 2. Better user experience
- Users withdraw at their preferred time
- Transactions can be sent when gas is cheap
- Full control of funds

#### 3. Simplified witness operations
- No gas cost burden
- Only signatures are needed
- Independent operation possible

## Specification

### 1. System overview

#### Withdrawal flow

```
1. User: withdraw 100 ZBD -> Ethereum (contract 0x1234... specified)
   -> Zattera: deduct 100 ZBD from user + immediately burn 100 ZBD from virtual supply
   -> Withdrawal request ID: 12345
   -> Status: queued (waiting for witness assignment)
   -> Plugin selects witness each block by discount rate
   -> Alice (10%), Bob (30%), Carol (60%) weighted random
   -> Alice selection probability: 90 / (90+70+40) = 45%
   -> Bob selection probability: 70 / 200 = 35%
   -> Carol selection probability: 40 / 200 = 20%
   -> Alice selected (exchange rate 10%)
   -> Status: queued -> pending (assigned to Alice)

2. Alice (within 5 minutes):
   -> Generate EIP-712 signature (for contract 0x1234...)
   -> Submit signature to Zattera
   -> Status: pending -> signed (signature done, waiting for user execution)

3. User (any time):
   -> Call contract 0x1234... with Alice's signature (token: USDC)
   -> Contract verifies EIP-712 signature (confirms Alice signed)
   -> Deduct 100 USDC from Alice's deposit
   -> Transfer 100 USDC to user
   -> Set double-withdrawal prevention flag

4. Alice (within validity window, e.g. 24 hours):
   -> Alice's witness plugin checks withdrawal execution via Ethereum RPC
   -> Contract call: getWithdrawalStatus(12345)
   -> Return: (processed=true, blockNumber=18234567)
   -> Filter Withdrawn event at that block -> get tx_hash
   -> Submit completion report to Zattera (with tx_hash)
   -> Recreate +10 ZBD to virtual supply (Alice reward)
   -> Pay 10 ZBD to Alice
   -> Create conversion request 10 ZBD -> ZP (executes after 3.5 days) ⏱
   -> Remaining 90 ZBD already burned at request time (permanent burn complete)
   -> Status: completed

5. After 3.5 days (automatic):
   -> Zattera blockchain processes conversion
   -> Uses 84-hour median price
   -> Alice's 10 ZBD burned -> ZTR -> ZP conversion
   -> Alice receives ZP
   -> Voting power increases
```

#### Roles

| Actor | Responsibilities | Rewards/Costs |
|------|------------------|--------------|
| **User** | - Request ZBD withdrawal<br>- Pay external chain gas fees<br>- Execute withdrawal transaction | - Full ZBD deducted + virtual supply burned immediately<br>- Only witness reward portion is re-minted on completion<br>- Pays gas (ETH) |
| **Witness** | - Operate/access external chain RPC<br>- Deposit stablecoins on external chain (USDC, DAI, USDT, etc.)<br>- Generate EIP-712 signatures<br>- Submit signatures<br>- Monitor external chain execution<br>- Submit completion reports | - Receive ZBD equal to exchange rate on completion (supply re-minted)<br>- Auto create ZBD -> ZP conversion request (after 3.5 days)<br>- Fair conversion at median price<br>- Voting power increase<br>- RPC operation costs |
| **Network** | - Manage virtual supply<br>- Burn immediately at request<br>- Re-mint reward on completion<br>- Deflationary effect | - At request: -100% ZBD burned immediately<br>- At completion: +exchange_rate% ZBD re-minted<br>- At convert: re-minted ZBD burned and ZP created<br>- Final result: **100% of requested ZBD burned** |
| **Smart contract** | - Verify signatures<br>- Manage witness deposits<br>- Prevent double withdrawals<br>- Emit Withdrawn events | - No gas cost |

### 2. Witness setup

#### 2.1 Withdrawal contract whitelist (witness consensus)

Withdrawal contracts must be approved by witness consensus to be used, protecting users from malicious contracts.

**Approved Contract Database Object**:

```cpp
class withdraw_dollar_contract_object : public object<...>
{
    ZATTERA_STD_ALLOCATOR_CONSTRUCTOR(withdraw_dollar_contract_object)

    id_type           id;
    uint32_t          chain_id;              // EVM chain ID
    shared_string     contract_address;      // Approved withdrawal contract address
    time_point_sec    approved_at;           // Approval time

    // Witness approval tracking
    flat_set<account_name_type> approving_witnesses;  // Approving witnesses
    uint16_t          approval_count = 0;             // Number of approvals
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
    bool              approve = true;      // false = revoke approval

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
        // Approve
        if (itr == idx.end())
        {
            // Propose approval for a new contract
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
            // Add approval to existing contract
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
        // Revoke approval
        if (itr != idx.end())
        {
            db.modify(*itr, [&](auto& obj) {
                if (obj.approving_witnesses.erase(o.witness))
                {
                    obj.approval_count -= 1;
                }
            });

            // Delete if approvals drop below majority
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

**Approval criteria**:
- Requires **majority (50% + 1)** of active witnesses
- Example: 21 witnesses -> at least 11 approvals
- If approvals drop below majority, the contract is removed automatically

#### 2.2 Per-witness withdrawal service settings

For approved contracts, each witness sets a signer address and exchange rate.

**Database Object**:

```cpp
class withdraw_dollar_provider_object : public object<...>
{
    ZATTERA_STD_ALLOCATOR_CONSTRUCTOR(withdraw_dollar_provider_object)

    id_type           id;
    account_name_type witness;
    uint32_t          chain_id;              // EVM chain ID (1=Ethereum, 137=Polygon)
    shared_string     contract_address;      // Withdrawal contract address (0x...)
    shared_string     signer_address;        // Address used for EIP-712 signatures (0x...)
    uint16_t          exchange_rate;         // Exchange rate (ZATTERA_100_PERCENT = 10000)
    flat_set<shared_string> supported_tokens;  // Supported ERC-20 token list
    bool              is_active = true;
    time_point_sec    registered_at;
    time_point_sec    last_updated;

    // Deposit reports (witness => token => amount in ZBD equivalent)
    // Witness periodically reports external chain deposit state
    flat_map<shared_string, share_type> reported_deposits;  // Per-token deposit (ZBD units, 3 decimals)
    flat_map<shared_string, share_type> reserved_deposits;  // Per-token reserved deposits (sum of pending requests)
    time_point_sec    deposits_last_reported;                // Last deposit report time

    // Cumulative stats
    asset             total_withdrawal_amount;    // Total processed withdrawal amount (before exchange rate)
    asset             total_burned_amount;        // Total burned ZBD (non-exchanged amount)
    uint64_t          total_withdrawal_count = 0; // Total processed count
    time_point_sec    last_withdrawal_time;       // Last processed time
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
    string                 signer_address;     // 0x5678... (signing address)
    uint16_t               exchange_rate;      // 6000 = 60% (ZATTERA_100_PERCENT = 10000)
    vector<string>         supported_tokens;   // Supported token addresses (0xAAA..., 0xBBB...)

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

    // 1. Check contract approval
    const auto& approved_idx = db.get_index<withdraw_dollar_contract_index>()
                                  .indices().get<by_chain_contract>();
    auto approved_itr = approved_idx.find(boost::make_tuple(o.chain_id, o.contract_address));

    FC_ASSERT(approved_itr != approved_idx.end(),
              "Contract ${c} is not approved by witness consensus",
              ("c", o.contract_address));

    // Check majority approval
    const auto& witness_schedule = db.get_witness_schedule_object();
    uint32_t required_approvals = witness_schedule.current_shuffled_witnesses.size() / 2 + 1;

    FC_ASSERT(approved_itr->approval_count >= required_approvals,
              "Contract does not have majority approval (${current}/${required})",
              ("current", approved_itr->approval_count)("required", required_approvals));

    // 2. Register/update per-witness settings
    const auto& idx = db.get_index<withdraw_dollar_provider_index>()
                        .indices().get<by_witness_contract>();
    auto itr = idx.find(boost::make_tuple(o.witness, o.chain_id, o.contract_address));

    if (itr == idx.end())
    {
        // New registration
        db.create<withdraw_dollar_provider_object>([&](auto& obj) {
            obj.witness = o.witness;
            obj.chain_id = o.chain_id;
            obj.contract_address = o.contract_address;
            obj.signer_address = o.signer_address;
            obj.exchange_rate = o.exchange_rate;

            // Store supported token list
            obj.supported_tokens.clear();
            for (const auto& token : o.supported_tokens) {
                obj.supported_tokens.insert(token);
            }

            obj.is_active = true;
            obj.registered_at = db.head_block_time();
            obj.last_updated = db.head_block_time();

            // Initialize stats
            obj.total_withdrawal_amount = asset(0, DOLLAR_SYMBOL);
            obj.total_burned_amount = asset(0, DOLLAR_SYMBOL);
            obj.total_withdrawal_count = 0;
        });
    }
    else
    {
        // Update existing registration
        db.modify(*itr, [&](auto& obj) {
            obj.signer_address = o.signer_address;
            obj.exchange_rate = o.exchange_rate;

            // Update supported token list
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

#### 2.3 Deposit reporting

Witnesses must periodically report actual external-chain deposits to Zattera. This ensures only witnesses with sufficient deposits are selected.

**Operation**:

```cpp
struct report_withdraw_dollar_deposits_operation : public base_operation
{
    account_name_type      witness;
    uint32_t               chain_id;
    string                 contract_address;
    map<string, asset>     deposits;  // Per-token deposits (ZBD equivalent)

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

    // Find registered contract
    const auto& idx = db.get_index<withdraw_dollar_provider_index>()
                        .indices().get<by_witness_contract>();
    auto itr = idx.find(boost::make_tuple(o.witness, o.chain_id, o.contract_address));

    FC_ASSERT(itr != idx.end(),
              "Witness is not registered for this contract");

    // Update deposit report
    db.modify(*itr, [&](auto& obj) {
        obj.reported_deposits.clear();

        for (const auto& [token, amount] : o.deposits)
        {
            // Only supported tokens can be reported
            FC_ASSERT(obj.supported_tokens.find(token) != obj.supported_tokens.end(),
                      "Token ${t} is not in supported tokens list", ("t", token));

            obj.reported_deposits[token] = amount.amount;
        }

        obj.deposits_last_reported = db.head_block_time();
    });
}
```

**Exchange rate examples**:
- 1000 (10%) - withdraw 100 ZBD -> 10 ZBD exchanged to ZP, 90 ZBD burned (best for network)
- 6000 (60%) - withdraw 100 ZBD -> 60 ZBD exchanged to ZP, 40 ZBD burned
- 10000 (100%) - withdraw 100 ZBD -> 100 ZBD exchanged to ZP, no burn (no network benefit)

**Constants**:
```cpp
// zattera/protocol/config.hpp
#define ZATTERA_100_PERCENT                           10000
#define ZATTERA_MIN_EXCHANGE_RATE                     1000   // 10%
#define ZATTERA_MAX_EXCHANGE_RATE                     10000  // 100%

// Withdraw Dollar Plugin constants
#define ZATTERA_WITHDRAW_DOLLAR_MAX_ASSIGN_PER_BLOCK  100    // Max queued->pending assignments per block
#define ZATTERA_WITHDRAW_DOLLAR_MAX_SIGN_PER_BLOCK    50     // Max signatures per block
#define ZATTERA_WITHDRAW_DOLLAR_MAX_CHECK_PER_MINUTE  20     // Max completion checks per minute (RPC calls)
#define ZATTERA_WITHDRAW_DOLLAR_SIGNATURE_DEADLINE    300    // 5 minutes (signature deadline, seconds)
#define ZATTERA_WITHDRAW_DOLLAR_DEPOSIT_VALIDITY      3600   // 1 hour (deposit report validity, seconds)
```

**Witness selection logic (weighted random)**:
- Select probabilistically among witnesses who support the specified contract
- Weight calculation: `weight = ZATTERA_100_PERCENT - exchange_rate`
- Lower exchange rate = higher weight = higher selection chance

**Example**:
```
Alice: 1000 (10%) -> weight 9000 -> selection probability 45%
Bob:   3000 (30%) -> weight 7000 -> selection probability 35%
Carol: 6000 (60%) -> weight 4000 -> selection probability 20%

Total weight: 9000 + 7000 + 4000 = 20000
```

**Benefits**:
- Opportunity distributed among multiple witnesses
- Lower exchange rate gives more chances
- Incentive to maximize network deflation

#### 2.4 Discount rate

Users can set a **discount rate** when requesting withdrawal to get faster processing.

**Concept**:
- The difference between the ZBD deducted on Zattera and the actual amount received on the external chain
- The discounted amount is provided to witnesses as an incentive for prioritization

**How it works**:

```
Requested amount: 100.000 ZBD
Discount rate: 5% (500)

Zattera deduction: -100.000 ZBD
External receipt: 95 USD worth of tokens (USDC, DAI, etc.)
Discount amount: 5.000 ZBD (witness incentive)
```

**Priority processing**:
- Witnesses process requests with higher discount rates first (`by_witness_status` index, discount rate priority)
- `on_apply_block()`: up to 50 per block, higher discount first
- `report_completions_periodically()`: up to 20 per minute, higher discount first

**Constraints**:
- Minimum discount rate: 0% (default, no discount)
- Maximum discount rate: 100% (10000) - `ZATTERA_100_PERCENT`
- Discount rate is set at operation creation and cannot be changed

**Example**:

| Discount rate | Zattera deduction | External receipt | Witness incentive | Priority |
|--------------|-------------------|------------------|-------------------|----------|
| 0%           | 100 ZBD           | $100             | $0                | Low (default) |
| 1%           | 100 ZBD           | $99              | $1                | Higher |
| 5%           | 100 ZBD           | $95              | $5                | Higher |
| 10%          | 100 ZBD           | $90              | $10               | High |

**Note**: Discount rate is independent of the witness exchange rate. The exchange rate determines how much ZBD is converted to ZP, while the discount rate is a user-provided incentive for faster processing.

### 3. Withdrawal process

#### 3.1 User withdrawal request

**Database Objects**:

```cpp
enum withdraw_dollar_status
{
    queued,       // Waiting for witness assignment (sorted by discount rate)
    pending,      // Witness assigned, waiting for signature
    signed,       // Signed, waiting for user execution
    completed,    // Executed on external chain
    expired       // Timed out
};

class withdraw_dollar_request_object : public object<...>
{
    ZATTERA_STD_ALLOCATOR_CONSTRUCTOR(withdraw_dollar_request_object)

    id_type               id;
    uint64_t              request_id;          // Unique ID (double withdrawal prevention)
    account_name_type     requester;
    asset                 amount;              // ZBD
    uint32_t              chain_id;
    shared_string         contract_address;    // User-specified withdrawal contract
    shared_string         token;               // ERC-20 token (USDC, DAI, USDT, etc.)
    shared_string         recipient;           // 0x...
    uint16_t              discount_rate;       // Discount rate (0~10000)

    withdraw_dollar_status     status;
    account_name_type     assigned_witness;    // Selected witness
    shared_string         witness_signature;   // EIP-712 signature
    asset                 exchange_amount;     // Exchange amount (ZBD, amount * rate)
    uint16_t              exchange_rate;       // Applied exchange rate

    time_point_sec        created_at;
    time_point_sec        deadline;            // Signature deadline (5 minutes)
    time_point_sec        signed_at;
    shared_string         tx_hash;             // External chain tx
    time_point_sec        executed_at;
};

typedef multi_index_container<
    withdraw_dollar_request_object,
    indexed_by<
        ordered_unique< tag< by_request_id >,
            member< withdraw_dollar_request_object, uint64_t, &withdraw_dollar_request_object::request_id >
        >,
        // Global work queue: status + discount rate priority
        ordered_non_unique< tag< by_status >,
            composite_key< withdraw_dollar_request_object,
                member< withdraw_dollar_request_object, withdraw_dollar_status, &withdraw_dollar_request_object::status >,
                member< withdraw_dollar_request_object, uint16_t, &withdraw_dollar_request_object::discount_rate >,
                member< withdraw_dollar_request_object, time_point_sec, &withdraw_dollar_request_object::created_at >
            >,
            composite_key_compare<
                std::less< withdraw_dollar_status >,
                std::greater< uint16_t >,  // Higher discount first
                std::less< time_point_sec >
            >
        >,
        // Per-witness work queue: witness + status + discount rate priority
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
                std::greater< uint16_t >,  // Higher discount first
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
    string            contract_address;   // 0x... (user-specified withdrawal contract)
    string            token;              // 0x... (ERC-20 token: USDC, DAI, USDT, etc.)
    string            recipient;          // 0x...
    uint16_t          discount_rate = 0;  // Discount rate (0~10000, default 0%)

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

    // 1. Immediately burn ZBD (deduct user balance + reduce virtual supply)
    // Witness reward portion is re-minted at completion and sent to convert_request
    // Final result: only witness reward is re-minted; the rest is permanently burned
    db.adjust_balance(o.from, -o.amount);       // User balance: -100 ZBD
    db.adjust_supply(-o.amount);                // Virtual supply: -100 ZBD (burned immediately)

    // 2. Create withdrawal request (queued, no witness assigned)
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
        obj.status = withdraw_dollar_status::queued;  // Start in queued
        // assigned_witness, exchange_amount, exchange_rate, deadline not set yet
        obj.created_at = db.head_block_time();
    });
}
```

#### 3.1.1 Cancel withdrawal request

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

    // Refund ZBD (restore user balance + re-mint supply)
    db.adjust_balance(o.from, itr->amount);     // User balance: +100 ZBD
    db.adjust_supply(itr->amount);              // Virtual supply: +100 ZBD (re-minted)

    // Remove request
    db.remove(*itr);
}
```

#### 3.1.2 Update withdrawal request

**Operation**:

```cpp
struct update_withdraw_dollar_request_operation : public base_operation
{
    account_name_type from;
    uint64_t          request_id;

    optional<string>  new_recipient;     // New recipient address
    optional<uint16_t> new_discount_rate; // New discount rate

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
    // Deposit report validity window
    const fc::microseconds DEPOSIT_REPORT_VALIDITY = fc::seconds(ZATTERA_WITHDRAW_DOLLAR_DEPOSIT_VALIDITY);

    // Candidate list and weights
    struct Candidate {
        account_name_type witness;
        uint16_t exchange_rate;
        uint64_t weight;
    };
    vector<Candidate> candidates;

    // 1. Active witnesses (only scheduled witnesses)
    const auto& schedule = db.get_witness_schedule_object();
    const auto& witness_idx = db.get_index<witness_index>().indices().get<by_name>();
    const auto& provider_idx = db.get_index<withdraw_dollar_provider_index>()
                                  .indices().get<by_witness_contract>();

    // Iterate scheduled witnesses only (top20 + timeshare, up to 21)
    for (size_t i = 0; i < schedule.num_scheduled_witnesses; ++i)
    {
        const account_name_type& witness_name = schedule.current_shuffled_witnesses[i];

        // Get witness object
        auto witness_itr = witness_idx.find(witness_name);
        if (witness_itr == witness_idx.end())
            continue;

        // Check signing_key (active witness)
        if (witness_itr->signing_key == public_key_type())
            continue;

        // 2. Check if witness supports the specified contract
        auto provider_itr = provider_idx.find(
            boost::make_tuple(witness_name, chain_id, contract_address)
        );

        if (provider_itr == provider_idx.end())
            continue;  // Witness does not support this contract

        if (!provider_itr->is_active)
            continue;

        // Key check 1: token support
        if (provider_itr->supported_tokens.find(token) == provider_itr->supported_tokens.end())
            continue;

        // Key check 2: deposit report exists
        auto deposit_itr = provider_itr->reported_deposits.find(token);
        if (deposit_itr == provider_itr->reported_deposits.end())
            continue;

        // Key check 3: deposit report freshness
        if (db.head_block_time() - provider_itr->deposits_last_reported > DEPOSIT_REPORT_VALIDITY)
            continue;

        // Key check 4: available deposit = reported - reserved
        share_type reported_deposit = deposit_itr->second;
        share_type reserved_amount = 0;

        auto reserved_itr = provider_itr->reserved_deposits.find(token);
        if (reserved_itr != provider_itr->reserved_deposits.end())
            reserved_amount = reserved_itr->second;

        share_type available_deposit = reported_deposit - reserved_amount;

        // Skip if insufficient available deposit
        if (available_deposit < requested_amount.amount)
            continue;

        // Weight: lower exchange rate -> higher weight
        uint64_t weight = ZATTERA_100_PERCENT - provider_itr->exchange_rate;

        candidates.push_back({witness_name, provider_itr->exchange_rate, weight});
    }

    if (candidates.empty())
    {
        // No available witness (insufficient deposits, etc.)
        return {account_name_type(), 0};
    }

    // Deterministic random selection based on block hash
    fc::sha256 seed = fc::sha256::hash(
        std::to_string(db.head_block_id()._hash[0]) +
        std::to_string(db.head_block_num())
    );

    // Total weight
    uint64_t total_weight = 0;
    for (const auto& c : candidates)
        total_weight += c.weight;

    // Weighted random selection
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

    // Fallback: first candidate
    return {candidates[0].witness, candidates[0].exchange_rate};
}

uint64_t generate_unique_request_id(database& db)
{
    // Unique ID: block_num (32-bit) + tx_id (20-bit) + op_idx (12-bit)
    // - block_num: block number (up to 4B blocks = ~127 years @ 3s blocks)
    // - tx_id: transaction ID within block
    // - op_idx: operation index within transaction
    // Note: get_current_transaction_id() returns a unique tx ID within a block
    uint64_t block_num = db.head_block_num();
    uint32_t tx_id = db.get_current_transaction_id();
    uint16_t op_idx = db.get_current_operation_index();

    // Bit allocation: [32-bit block_num][20-bit tx_id][12-bit op_idx]
    return (static_cast<uint64_t>(block_num) << 32) |
           ((tx_id & 0xFFFFF) << 12) |
           (op_idx & 0xFFF);
}
```

#### 3.2 Witness assignment (discount-rate-based load distribution)

**Plugin Implementation**:

```cpp
void withdraw_dollar_plugin::assign_withdraw_requests()
{
    // Run every block (called from on_apply_block)
    auto& db = database();
    const auto& queued_idx = db.get_index<withdraw_dollar_request_index>()
                                .indices().get<by_status>();

    // 1. Query queued requests, sorted by discount rate descending
    auto queued_range = queued_idx.equal_range(withdraw_dollar_status::queued);

    if (queued_range.first == queued_range.second)
        return;  // No queued requests

    // 2. Assign higher-discount requests first
    // Note: changing status changes by_status index keys and invalidates iterators
    //       collect first, then process
    vector<withdraw_dollar_request_id_type> requests_to_assign;
    size_t count = 0;

    for (auto itr = queued_range.first;
         itr != queued_range.second && count < ZATTERA_WITHDRAW_DOLLAR_MAX_ASSIGN_PER_BLOCK;
         ++itr, ++count)
    {
        requests_to_assign.push_back(itr->request_id);
    }

    // 3. Process collected requests safely
    const auto& by_id_idx = db.get_index<withdraw_dollar_request_index>()
                               .indices().get<by_request_id>();

    size_t assigned_count = 0;

    for (const auto& req_id : requests_to_assign)
    {
        auto itr = by_id_idx.find(req_id);
        if (itr == by_id_idx.end() || itr->status != withdraw_dollar_status::queued)
            continue;  // Already processed or removed

        // Select witness (considering available deposit)
        auto [selected_witness, exchange_rate] = select_withdraw_witness(
            itr->chain_id,
            itr->contract_address,
            itr->token,
            itr->amount,
            db
        );

        if (selected_witness == account_name_type())
            continue;  // No available witness

        // Compute exchange amount
        asset exchange_amount = itr->amount * exchange_rate / ZATTERA_100_PERCENT;

        // queued -> pending
        db.modify(*itr, [&](auto& obj) {
            obj.status = withdraw_dollar_status::pending;
            obj.assigned_witness = selected_witness;
            obj.exchange_amount = exchange_amount;
            obj.exchange_rate = exchange_rate;
            obj.deadline = db.head_block_time() + fc::seconds(ZATTERA_WITHDRAW_DOLLAR_SIGNATURE_DEADLINE);
        });

        // Increase provider reserved_deposits
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

#### 3.3 Witness signature generation

**Plugin Implementation**:

```cpp
void withdraw_dollar_plugin::on_apply_block(const signed_block& block)
{
    auto& db = database();

    // 0. Assign queued requests by discount rate
    assign_withdraw_requests();

    // 1. Process pending requests (signature generation)
    const auto& idx = db.get_index<withdraw_dollar_request_index>()
                        .indices().get<by_witness_status>();

    const account_name_type my_witness = get_my_witness_name();

    // 2. Process pending requests (signature generation)
    // equal_range returns all requests matching witness and status
    // by_witness_status index is sorted by discount rate descending
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

        // Generate EIP-712 signature
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

            // Submit signature to Zattera
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

    // 2. Signed requests are handled in the background (completion reported by report_completions_periodically)
}

void withdraw_dollar_plugin::report_completions_periodically()
{
    // Called periodically (e.g., every minute)
    auto& db = database();
    // Use index sorted by discount rate descending
    const auto& idx = db.get_index<withdraw_dollar_request_index>()
                        .indices().get<by_witness_status>();

    const account_name_type my_witness = get_my_witness_name();

    // Only signed requests (check external chain completion)
    auto signed_range = idx.equal_range(
        boost::make_tuple(my_witness, withdraw_dollar_status::signed)
    );

    size_t checked_count = 0;
    for (auto itr = signed_range.first;
         itr != signed_range.second && checked_count < ZATTERA_WITHDRAW_DOLLAR_MAX_CHECK_PER_MINUTE;
         ++itr, ++checked_count)
    {
        try {
            // Check status on external chain
            auto tx_hash = query_withdraw_transaction(
                itr->chain_id,
                itr->contract_address,
                itr->request_id
            );

            if (!tx_hash.empty())
            {
                // Submit completion report
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
    // Called periodically (e.g., every 5 minutes)
    auto& db = database();

    // Report deposits for all registered contracts
    const auto& idx = db.get_index<withdraw_dollar_provider_index>()
                        .indices().get<by_witness_contract>();

    auto range = idx.equal_range(get_my_witness_name());

    for (auto itr = range.first; itr != range.second; ++itr)
    {
        if (!itr->is_active)
            continue;

        try {
            // Query deposits per token from external chain
            map<string, asset> deposits;

            for (const auto& token : itr->supported_tokens)
            {
                // ZRC-9 standard: getWitnessDeposit(address witness, address token)
                // Important: ZRC-9 spec always normalizes return value to 18 decimals
                //            (regardless of token decimals)
                //            Example: USDC 100 (6 decimals) -> 100000000000000000000 (18 decimals)
                //                     DAI 100 (18 decimals) -> 100000000000000000000 (18 decimals)
                uint256_t token_balance = query_deposit_balance(
                    itr->chain_id,
                    itr->contract_address,
                    itr->signer_address,  // witness signer address
                    token
                );

                // Convert 18 decimals -> 3 decimals (ZBD)
                asset dollar_amount = token_amount_to_dollar(token_balance);

                deposits[token] = dollar_amount;
            }

            // Report deposits to Zattera chain
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

    // ZRC-9 function: getWitnessDeposit(address witness, address token)
    auto result = web3_client.call({
        .to = contract_address,
        .data = web3::encode_function_call(
            "getWitnessDeposit(address,address)",
            witness_address,
            token
        )
    });

    // Decode uint256 return
    return web3::decode_result<uint256_t>(result);
}

// Convert ZRC-9 deposit to ZBD equivalent
// Important: ZRC-9 input is always normalized to 18 decimals
//            (regardless of token decimals - contract returns 18 decimals)
asset token_amount_to_dollar(uint256_t amount)
{
    const uint8_t dollar_decimals = DOLLAR_SYMBOL.decimals();  // 3 decimals
    const uint8_t ZRC9_DECIMALS = 18;  // ZRC-9 interface spec

    // 18 decimals -> 3 decimals (ZBD)
    // Example: 100000000000000000000 (100, 18 decimals) -> 100000 (100.000 ZBD, 3 decimals)
    uint256_t dollar_raw = amount / pow(10, ZRC9_DECIMALS - dollar_decimals);

    return asset(dollar_raw, DOLLAR_SYMBOL);
}

// EIP-712 type definitions
namespace eip712 {
    struct Withdrawal {
        uint64_t requestId;
        address recipient;
        address token;
        uint256 amount;        // Discounted amount (actual amount on external chain)
        uint16_t discountRate; // Discount rate (0~10000)
    };

    // EIP-712 type hash
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
    // EIP-712 domain (ZRC-9 standard: omit name/version)
    // verifyingContract uses the user-specified withdrawal contract (contract binding)
    eip712::Domain domain = {
        .chainId = chain_id,
        .verifyingContract = contract_address
    };

    // Compute discounted amount and normalize to 18 decimals
    // Example: 100 ZBD (raw: 100000, 3 decimals), 5% discount
    //     -> 95000 (ZBD raw) * 10^15 = 95 * 10^18
    share_type discounted_zbd = amount.amount.value * (ZATTERA_100_PERCENT - discount_rate) / ZATTERA_100_PERCENT;
    uint256_t discounted_amount_18 = uint256_t(discounted_zbd) * 1000000000000000;  // * 10^15

    // Message
    eip712::Withdrawal message = {
        .requestId = request_id,
        .recipient = recipient,
        .token = token,
        .amount = discounted_amount_18,  // 18-decimal normalized
        .discountRate = discount_rate
    };

    // Sign with witness external chain private key
    return eip712::sign(domain, message, get_my_eth_private_key());
}

string query_withdraw_transaction(
    uint32_t chain_id,
    const string& contract_address,
    uint64_t request_id
)
{
    // Witness accesses external chain RPC
    auto& web3_client = get_web3_client(chain_id);

    // 1. Call contract getWithdrawalStatus
    // function getWithdrawalStatus(uint64 requestId)
    //     external view returns (bool processed, uint256 blockNumber)

    auto result = web3_client.call({
        .to = contract_address,
        .data = web3::encode_function_call(
            "getWithdrawalStatus(uint64)",
            request_id
        )
    });

    // Decode result: (bool processed, uint256 blockNumber)
    auto [processed, block_number] = web3::decode_result<bool, uint256>(result);

    if (!processed || block_number == 0)
    {
        return "";  // Not processed yet
    }

    // 2. Filter Withdrawn event at that block
    // event Withdrawn(uint64 indexed requestId, address indexed recipient,
    //                 address indexed token, uint256 amount, uint16 discountRate, address witness)
    auto event_signature = web3::keccak256("Withdrawn(uint64,address,address,uint256,uint16,address)");
    auto request_id_topic = web3::encode_uint64(request_id);

    auto logs = web3_client.get_logs({
        .address = contract_address,
        .topics = {event_signature, request_id_topic},
        .fromBlock = block_number,
        .toBlock = block_number  // Query only that block
    });

    if (logs.empty())
    {
        // Suspicious but retryable; log warning only
        wlog("Withdrawal marked processed at block ${b} but event not found for request ${r}",
             ("b", block_number)("r", request_id));
        return "";  // Retry next cycle
    }

    // Return tx hash
    return logs[0].transactionHash;
}
```

**Operation**:

```cpp
struct withdraw_dollar_approve_operation : public base_operation
{
    account_name_type witness;
    uint64_t          request_id;
    string            signature;     // EIP-712 signature (hex)

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

    // 1. Check withdrawal request
    const auto& idx = db.get_index<withdraw_dollar_request_index>()
                        .indices().get<by_request_id>();
    auto itr = idx.find(o.request_id);
    FC_ASSERT(itr != idx.end(), "Request not found");
    FC_ASSERT(itr->status == withdraw_dollar_status::pending, "Not pending");
    FC_ASSERT(itr->assigned_witness == o.witness, "Wrong witness");
    FC_ASSERT(db.head_block_time() <= itr->deadline, "Deadline passed");

    // 2. Verify signature
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

    // 3. Save signature (status: signed)
    db.modify(*itr, [&](auto& obj) {
        obj.status = withdraw_dollar_status::signed;
        obj.witness_signature = o.signature;
        obj.signed_at = db.head_block_time();
    });

    // Reward is paid at completion report
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
    // Build EIP-712 message hash (ZRC-9 standard: omit name/version)
    eip712::Domain domain = {
        .chainId = chain_id,
        .verifyingContract = contract_address  // User-specified contract
    };

    // Compute discounted amount and normalize to 18 decimals (same as generation)
    share_type discounted_zbd = amount.amount.value * (ZATTERA_100_PERCENT - discount_rate) / ZATTERA_100_PERCENT;
    uint256_t discounted_amount_18 = uint256_t(discounted_zbd) * 1000000000000000;  // * 10^15

    eip712::Withdrawal message = {
        .requestId = request_id,
        .recipient = recipient,
        .token = token,
        .amount = discounted_amount_18,
        .discountRate = discount_rate
    };

    bytes32 digest = eip712::hash_typed_data(domain, message);

    // Recover address from signature
    string recovered_address = ecdsa::recover(digest, signature);

    return (recovered_address == signer_address);
}
```

#### 3.3 User withdrawal execution

**User side (JavaScript)**:

```javascript
// 1. Query withdrawal request from Zattera API
const request = await zatteraApi.getWithdrawalRequest(requestId);

if (request.status === 'signed') {
    // 2. Parse ZBD amount and normalize to 18 decimals
    // CLI response: "100.000 ZBD" -> ZBD raw (3 decimals): 100000 -> 18 decimals: 100 * 10^18
    const amountZBD = parseFloat(request.amount.split(' ')[0]);  // 100.000
    const amountRaw = amountZBD * 1000;  // 100000 (ZBD 3 decimals)

    // Apply discount
    const discountedZBD = Math.floor(amountRaw * (10000 - request.discount_rate) / 10000);  // 95000

    // Normalize to 18 decimals (ZRC-9 standard)
    const amount18 = ethers.BigNumber.from(discountedZBD).mul(ethers.BigNumber.from(10).pow(15));
    // 95000 * 10^15 = 95 * 10^18

    // 3. Call external chain contract
    const bridgeContract = new ethers.Contract(
        request.contract_address,  // User-specified contract
        BRIDGE_ABI,
        userWallet
    );

    // 4. Withdraw with witness signature
    const tx = await bridgeContract.withdraw(
        request.request_id,        // uint64: Zattera withdrawal request ID
        request.recipient,         // address: token recipient
        request.token,             // address: ERC-20 token (USDC, DAI, USDT, etc.)
        amount18,                  // uint256: discounted amount (18 decimals)
        request.discount_rate,     // uint16: discount rate (0~10000)
        request.witness_signature  // bytes: witness EIP-712 signature
    );

    await tx.wait();

    // Token amount is converted from 18 decimals to token native decimals inside the contract
    // Example: 95 * 10^18 (18 decimals)
    //   - USDC (6 decimals): 95 * 10^18 / 10^12 = 95 * 10^6 (95 USDC)
    //   - DAI (18 decimals): 95 * 10^18 / 10^0 = 95 * 10^18 (95 DAI)
    console.log(`Withdrawn ${amountZBD * (10000 - request.discount_rate) / 10000} ZBD worth of tokens (${request.discount_rate / 100}% discount)`);
    console.log(`Token: ${request.token}`);
    console.log(`Tx hash: ${tx.hash}`);

    // 5. Done
    // Note: witness will automatically detect completion and report to Zattera
    // User needs no further action
}
```

#### 3.4 External chain smart contract

ZEP-8 uses a bridge contract implementing the **ZRC-9 (ZEP-9)** standard interface.

**ZRC-9 interface requirements** (see ZEP-9 for full implementation):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/**
 * @title IZRC9
 * @notice Zattera cross-chain bridge standard interface
 * @dev See ZEP-9 for full spec and implementation
 */
interface IZRC9 {
    /**
     * @notice Withdrawal completion event
     * @param requestId Zattera withdrawal request ID
     * @param recipient Address receiving tokens
     * @param token ERC-20 token address
     * @param amount Discounted amount (18 decimals normalized)
     * @param discountRate Discount rate (0~10000 basis points)
     * @param witness Witness address that processed withdrawal
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
     */
    event Deposited(address indexed witness, address indexed token, uint256 amount);

    /**
     * @notice User executes withdrawal with witness signature
     * @param requestId Zattera withdrawal request ID
     * @param recipient Token recipient
     * @param token ERC-20 token address
     * @param amount Discounted amount (18 decimals normalized)
     * @param discountRate Discount rate (0~10000 basis points)
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
     * @notice Deposit witness collateral
     * @param token ERC-20 token address
     * @param amount Amount to deposit
     */
    function deposit(address token, uint256 amount) external;

    /**
     * @notice Query withdrawal status
     * @param requestId Zattera withdrawal request ID
     * @return processed Whether processed
     * @return blockNumber Processed block number (0 if not processed)
     */
    function getWithdrawalStatus(uint64 requestId)
        external
        view
        returns (bool processed, uint256 blockNumber);

    /**
     * @notice Query witness deposit balance
     * @param witness Witness address
     * @param token Token address
     * @return balance Deposit balance (18 decimals normalized)
     */
    function getWitnessDeposit(address witness, address token)
        external
        view
        returns (uint256 balance);
}
```

**ZEP-8 specific requirements**:

1. **EIP-712 signature structure**:
```solidity
// EIP-712 domain (ZRC-9 standard: omit name/version)
EIP712Domain(uint256 chainId, address verifyingContract)

// Withdrawal type
Withdrawal(uint64 requestId, address recipient, address token, uint256 amount, uint16 discountRate)
```

2. **Discount rate and amount normalization**:
   - `amount`: discounted amount (18 decimals normalized)
   - `discountRate`: 0~10000 basis points (e.g., 500 = 5%)
   - Contract converts 18 decimals to token native decimals

3. **Deposit management**:
   - Witnesses deposit via `deposit()`
   - Deposits are non-withdrawable (for withdrawal service)
   - `getWitnessDeposit()` returns 18-decimal normalized balance

4. **Withdrawal processing verification**:
   - Double-withdrawal prevention (by requestId)
   - Witness verification via EIP-712 signature
   - Check deposit balance before deducting

**Note**: See **ZEP-9** for full ZRC-9 contract implementation.

#### 3.5 Completion report

After a witness confirms external-chain execution, they report completion on the Zattera chain to receive the ZBD -> ZP reward.

**Operation**:

```cpp
struct complete_withdraw_dollar_operation : public base_operation
{
    account_name_type witness;       // Submitted by witness
    uint64_t          request_id;
    string            tx_hash;       // External chain tx

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

    // 1. Check withdrawal request
    const auto& idx = db.get_index<withdraw_dollar_request_index>()
                        .indices().get<by_request_id>();
    auto itr = idx.find(o.request_id);
    FC_ASSERT(itr != idx.end(), "Request not found");
    FC_ASSERT(itr->assigned_witness == o.witness, "Wrong witness");
    FC_ASSERT(itr->status == withdraw_dollar_status::signed, "Not signed");

    // 2. Verify external chain withdrawal transaction
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

    // 3. Mark complete
    db.modify(*itr, [&](auto& obj) {
        obj.status = withdraw_dollar_status::completed;
        obj.tx_hash = o.tx_hash;
        obj.executed_at = db.head_block_time();
    });

    // 4. Pay witness reward and create ZBD -> ZP conversion request
    // 4-1. Re-mint reward supply and pay witness
    db.adjust_supply(itr->exchange_amount);              // Virtual supply: +60 ZBD (re-minted)
    db.adjust_balance(o.witness, itr->exchange_amount);  // Witness balance: +60 ZBD

    // 4-2. Convert witness ZBD to ZP (ZATTERA_CONVERSION_DELAY = 3.5 days)
    // Reuse convert_request_object to avoid price manipulation
    db.create<convert_request_object>([&](auto& obj) {
        obj.owner = o.witness;
        obj.requestid = db.get_convert_request_id(o.witness);
        obj.amount = itr->exchange_amount;  // ZBD amount after exchange rate
        obj.conversion_date = db.head_block_time() + ZATTERA_CONVERSION_DELAY;  // 3.5 days later
    });
    // convert_request is processed by database::process_conversions(), which burns ZBD and creates ZP

    // 4-3. Remaining ZBD already burned at request time
    // At completion: -100 (request) +60 (completion) = net -40 ZBD (temporary)
    // Final (after convert): -100 (request) +60 (completion) -60 (convert) = net -100 ZBD

    // 5. Update witness stats
    const auto& contract_idx = db.get_index<withdraw_dollar_provider_index>()
                                  .indices().get<by_witness_contract>();
    auto contract_itr = contract_idx.find(
        boost::make_tuple(o.witness, itr->chain_id, itr->contract_address)
    );

    // Permanently burned amount = request - witness reward
    asset permanently_burned = itr->amount - itr->exchange_amount;

    db.modify(*contract_itr, [&](auto& obj) {
        obj.total_withdrawal_amount += itr->amount;           // Principal before exchange rate
        obj.total_burned_amount += permanently_burned;        // Burned amount
        obj.total_withdrawal_count += 1;
        obj.last_withdrawal_time = db.head_block_time();

        // Reduce reserved_deposits (release deposit for completed request)
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

    // Check Withdrawn event
    // event Withdrawn(uint64 indexed requestId, address indexed recipient,
    //                 address indexed token, uint256 amount, uint16 discountRate, address witness)
    // amount is 18-decimal normalized
    for (const auto& log : receipt.logs)
    {
        if (log.topics[0] == keccak256("Withdrawn(uint64,address,address,uint256,uint16,address)"))
        {
            // Indexed parameters (topics)
            uint64_t logged_id = decode_uint64(log.topics[1]);
            string logged_recipient = decode_address(log.topics[2]);
            string logged_token = decode_address(log.topics[3]);

            // Non-indexed parameters (data) in ABI order
            // logged_amount is 18-decimal normalized
            auto [logged_amount, logged_discount_rate, logged_witness] =
                decode_log_data<uint256, uint16, address>(log.data);

            // Convert ZBD 3 decimals -> 18 decimals for verification
            // Example: 100.000 ZBD (raw: 100000) * 95% = 95000
            //     -> 18 decimals: 95000 * 10^15 = 95 * 10^18
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

### 4. Economic model and witness reward processing

#### 4.1 Overall economic flow

**ZBD lifecycle** (exchange rate 60% example):

```
[At request]
User balance: -100 ZBD
Virtual supply: -100 ZBD (burned immediately)

[At completion]
Virtual supply: +60 ZBD (re-minted for witness reward)
Witness balance: +60 ZBD
  -> convert_request created (convert to ZP after 3.5 days)

[After 3.5 days - convert]
Witness balance: -60 ZBD (burned at conversion)
Virtual supply: -60 ZBD
Witness ZP: +XXX VESTS (ZBD -> ZTR -> ZP)

[Final]
Virtual supply change: -100 (request) +60 (completion) -60 (convert) = -100 ZBD
Result: 100% of requested ZBD burned
```

**Key principles**:
1. **At request**: immediate burn (deduct user balance + reduce supply)
2. **At completion**: re-mint only witness reward + pay witness
3. **At conversion**: burn witness ZBD and mint ZP
4. **Final result**: 100% of requested ZBD burned
5. **Accounting transparency**: supply changes are clearly traceable at every stage

#### 4.2 Witness reward conversion

Witness rewards (ZBD) are converted to ZP using the existing `convert_request_object` mechanism.

**Reusing existing system**:

```cpp
// Re-mint reward supply and pay witness at completion
 db.adjust_supply(exchange_amount);                   // Supply: +60 ZBD (re-minted)
 db.adjust_balance(witness_account, exchange_amount); // Witness: +60 ZBD

// Convert witness ZBD to ZP immediately (existing convert mechanism)
 db.create<convert_request_object>([&](auto& obj) {
     obj.owner = witness_account;
     obj.requestid = db.get_convert_request_id(witness_account);
     obj.amount = exchange_amount;  // ZBD after exchange rate
     obj.conversion_date = db.head_block_time() + ZATTERA_CONVERSION_DELAY;
 });

// Remaining ZBD already burned at request time
// At completion: request -100 ZBD, completion +60 ZBD -> net -40 ZBD (temporary)
// At conversion: re-minted 60 ZBD burned as well

// After 3.5 days, database::process_conversions() automatically:
// 1. Deduct ZBD from witness balance: -60 ZBD
// 2. Burn virtual supply: -60 ZBD
// 3. Convert ZBD -> ZTR (84-hour median price)
// 4. Convert ZTR -> ZP (current vesting price)
// 5. Pay ZP to witness
// Final: -100 (request) +60 (completion) -60 (convert) = net -100 ZBD burned
```

**Key points**:

1. **3.5-day delay**: `ZATTERA_CONVERSION_DELAY` (84 hours)
2. **Median price used**: `feed_history.current_median_history`
   - Median price over the past 84 hours
   - Prevents short-term price manipulation
3. **Two-step conversion**:
   - Step 1: ZBD -> ZTR (median price)
   - Step 2: ZTR -> ZP (vesting share price)
4. **Fairness**: same rules for all ZBD -> ZP conversions
5. **Code reuse**: proven existing conversion logic

### 5. Timeouts and reassignment

If a witness does not submit a signature within 5 minutes, the request is reassigned to another witness.

**Plugin Implementation**:

```cpp
void withdraw_dollar_plugin::on_apply_block(const signed_block& block)
{
    auto& db = database();

    // Check timed-out requests
    const auto& idx = db.get_index<withdraw_dollar_request_index>()
                        .indices().get<by_status>();
    auto itr = idx.lower_bound(withdraw_dollar_status::pending);
    auto end = idx.upper_bound(withdraw_dollar_status::pending);

    for (; itr != end; ++itr)
    {
        if (db.head_block_time() > itr->deadline)
        {
            // Attempt reassignment
            try {
                auto [new_witness, new_rate] = select_withdraw_witness(
                    itr->chain_id,
                    itr->contract_address,
                    itr->token,
                    itr->amount,  // Include requested amount
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
                // No available witness -> mark expired
                db.modify(*itr, [&](auto& obj) {
                    obj.status = withdraw_dollar_status::expired;
                });

                // Reduce provider reserved_deposits (release deposit)
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

                // Refund ZBD to user (restore balance + re-mint supply)
                db.adjust_balance(itr->requester, itr->amount);  // User balance: +100 ZBD
                db.adjust_supply(itr->amount);                   // Virtual supply: +100 ZBD (re-minted)

                elog("Request ${id} expired, refunded: ${e}",
                     ("id", itr->request_id)("e", e.to_detail_string()));
            }
        }
    }
}
```

### 6. Security considerations

#### 6.1 Price manipulation resistance (ZBD -> ZP conversion)

**Mechanism**: same protection as `convert_request_object`

**3.5-day delay + median price**:
```cpp
// When creating conversion request
obj.conversion_date = db.head_block_time() + ZATTERA_CONVERSION_DELAY;  // 3.5 days later

// When executing conversion
asset liquid_amount = itr->amount * fhistory.current_median_history;  // 84-hour median
```

**Protection effect**:
```
Scenario: malicious witness attempts price manipulation
-> Manipulate ZTR/ZBD price for a short window (e.g., 30 minutes)
-> Submit withdrawal completion
-> Conversion executes 3.5 days later
-> 84-hour median price used at conversion
-> Short-term spike has negligible impact
-> Fair price conversion
```

**Key points**:
- **No immediate conversion**: all ZBD -> ZP conversions wait 3.5 days
- **Median price enforced**: short-term spikes are neutralized
- **Consistency**: same rules as existing `convert` operation
- **Fairness**: same price mechanism for all participants

#### 6.2 Contract whitelist (witness consensus)

**Purpose**: protect users from malicious contracts

**Mechanism**:
- Withdrawal contracts require **majority witness approval** (50% + 1)
- Unapproved contracts cannot be registered/used
- Witnesses can revoke approval at any time
- If approvals drop below majority, contract is removed automatically

**Protection effect**:
```
Scenario: malicious witness proposes a fake contract
-> Majority of witnesses reject
-> Contract not approved
-> Users cannot use the contract
-> User funds protected
```

**Verification logic**:
```cpp
// During witness service registration
const auto& idx = db.get_index<withdraw_dollar_contract_index>()
                    .indices().get<by_chain_contract>();
auto itr = idx.find(boost::make_tuple(chain_id, contract_address));

FC_ASSERT(itr != idx.end(),
          "Contract is not approved by witness consensus");

FC_ASSERT(itr->approval_count >= required_approvals,
          "Contract does not have majority approval");
```

#### 6.3 Deposit tracking and validation

**Purpose**: select only witnesses with sufficient deposits to avoid execution failures

**Mechanism**:
1. **Periodic reporting**: witnesses report external-chain deposit state to Zattera every 5 minutes
2. **On-chain query**: query actual deposits via ZRC-9 `getWitnessDeposit(witness, token)`
3. **Selection checks**: witness selection verifies 4 conditions
   - Token support
   - Deposit report exists
   - Deposit sufficiency (>= requested amount)
   - Report freshness (within 1 hour)

**Protection effect**:
```
Scenario 1: insufficient deposit
-> Witness reports USDC 1000 ZBD equivalent
-> User requests withdrawal 2000 ZBD
-> Witness excluded from candidates
-> Another witness with enough deposit selected

Scenario 2: stale information
-> Witness reported deposits 2 hours ago
-> Exceeds 1-hour validity
-> Witness excluded
-> Only witnesses with fresh reports selected

Scenario 3: report mismatch
-> Witness reports 5000 ZBD but actual is 1000 ZBD
-> User execution fails with "Insufficient deposit"
-> User can retry with another signature
-> Misreporting witness loses reputation
```

**Witness incentives**:
- Accurate reporting -> more assignments
- Inaccurate reporting -> execution failures -> reputation loss -> lower selection probability

**Plugin implementation**:
```cpp
void withdraw_dollar_plugin::report_deposits_periodically()
{
    // Runs every 5 minutes
    for (each registered contract) {
        for (each supported token) {
            // 1. Query actual external-chain deposit
            uint256_t balance = query_deposit_balance(
                chain_id, contract, signer_address, token
            );

            // 2. Convert to dollars
            deposits[token] = token_amount_to_dollar(balance);
        }

        // 3. Report to Zattera chain
        broadcast(report_withdraw_dollar_deposits_operation);
    }
}
```

#### 6.4 Double withdrawal prevention

**Smart contract**:
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

#### 6.5 Signature validity verification

Zattera verifies EIP-712 signatures to block invalid signatures:

```cpp
bool valid = verify_eip712_signature(
    itr->chain_id,
    itr->contract_address,
    o.request_id,
    itr->recipient,
    itr->token,              // token address
    itr->amount,
    itr->discount_rate,      // discount rate
    o.signature,
    contract_itr->signer_address
);
FC_ASSERT(valid, "Invalid signature");
```

**Verification includes**:
- EIP-712 signature structure (domain + message)
- Signer address matches registered signer_address
- Request ID, recipient, token, amount, discount rate included in signature

#### 6.6 Signature replay attack prevention (cross-contract replay)

**Problem**: reusing the same witness signature on another contract

**Attack scenario**:
```
1. User creates withdrawal request for Contract A (0xAAAA...)
2. Witness generates EIP-712 signature and submits
3. Malicious user tries to reuse the signature on Contract B (0xBBBB...)
4. Contract B uses the same witness, so signature appears valid
5. Double withdrawal possible
```

**Solution**: include `verifyingContract` in the EIP-712 domain

**Witness signature generation**:
```cpp
string generate_withdraw_signature(
    uint32_t chain_id,
    const string& contract_address,  // User-requested contract address
    uint64_t request_id,
    const string& recipient,
    const string& token,
    asset amount,
    uint16_t discount_rate
)
{
    // EIP-712 domain (ZRC-9 standard: omit name/version)
    eip712::Domain domain = {
        .chainId = chain_id,
        .verifyingContract = contract_address  // Bind to specific contract
    };

    // Compute discounted amount and normalize to 18 decimals
    share_type discounted_zbd = amount.amount.value * (ZATTERA_100_PERCENT - discount_rate) / ZATTERA_100_PERCENT;
    uint256_t discounted_amount_18 = uint256_t(discounted_zbd) * 1000000000000000;  // * 10^15

    eip712::Withdrawal message = {
        .requestId = request_id,
        .recipient = recipient,
        .token = token,                       // Include token address
        .amount = discounted_amount_18,       // 18-decimal normalized
        .discountRate = discount_rate         // Include discount rate
    };

    // Signature valid only for this contract
    return eip712::sign(domain, message, get_my_eth_private_key());
}
```

**Signature verification**:
```cpp
bool verify_eip712_signature(
    uint32_t chain_id,
    const string& contract_address,  // Withdrawal request contract
    uint64_t request_id,
    const string& recipient,
    const string& token,
    asset amount,
    uint16_t discount_rate,
    const string& signature,
    const string& signer_address
)
{
    // Build domain with same contract address (ZRC-9 standard)
    eip712::Domain domain = {
        .chainId = chain_id,
        .verifyingContract = contract_address
    };

    // Compute discounted amount and normalize to 18 decimals (same as generation)
    share_type discounted_zbd = amount.amount.value * (ZATTERA_100_PERCENT - discount_rate) / ZATTERA_100_PERCENT;
    uint256_t discounted_amount_18 = uint256_t(discounted_zbd) * 1000000000000000;  // * 10^15

    eip712::Withdrawal message = {
        .requestId = request_id,
        .recipient = recipient,
        .token = token,
        .amount = discounted_amount_18,
        .discountRate = discount_rate
    };

    bytes32 digest = eip712::hash_typed_data(domain, message);
    string recovered_address = ecdsa::recover(digest, signature);

    return (recovered_address == signer_address);
}
```

**Security guarantee**:
- Signatures for Contract A are valid only on Contract A
- Signatures for Contract B are valid only on Contract B
- Different `verifyingContract` -> different signature hash
- **Cross-contract replay prevented**

**Note**: See ZEP-9 section 7.1 for smart contract implementation

### 7. Economic incentives

#### 7.1 Witness reward mechanism

When a witness provides a withdrawal signature and reports completion, they receive **ZBD equal to the exchange rate, converted to ZP (VESTS)** after 3.5 days. The remaining ZBD is **permanently burned**, contributing to deflation.

**Example**:
```
User withdrawal request: 100 ZBD
Witness exchange rate: 60% (6000 BPS)

1. At request:
   - User account: 100 ZBD deducted
   - Virtual supply: 100 ZBD burned immediately

2. At completion report:
   - Virtual supply: 60 ZBD re-minted (witness reward)
   - Witness account: 60 ZBD paid
   - Witness 60 ZBD -> ZP conversion scheduled (executes after 3.5 days)
   - Remaining 40 ZBD already burned at request

3. After 3.5 days (automatic):
   - Witness account: 60 ZBD deducted
   - Virtual supply: 60 ZBD burned
   - 60 ZBD -> ZTR (84-hour median price)
   - ZTR -> ZP (current vesting share price)
   - Witness receives ZP -> voting power increases

Result: all 100 ZBD requested is burned
      (request -100, completion +60, convert -60 = net -100 ZBD)
```

**ZP conversion formula** (two-step):
```cpp
// Step 1: ZBD -> ZTR (after 3.5 days, using median price)
asset exchange_amount = withdrawal_amount * exchange_rate / ZATTERA_100_PERCENT;
asset liquid_amount = exchange_amount * feed_history.current_median_history;

// Step 2: ZTR -> ZP (immediately, current vesting price)
asset zp_amount = liquid_amount * gpo.get_vesting_share_price();
```

**Why 3.5-day delay?**
- **Price manipulation resistance**: neutralizes short-term spikes
- **Fairness**: same rules for all ZBD conversions
- **Consistency**: same as existing `convert` operation

#### 7.2 Exchange rate competition

Among witnesses who support the specified contract, selection is **weighted random**.

**Selection logic (probabilistic)**:
```
Witnesses supporting the same contract:
- Alice: 1000 (10%) -> weight 9000 -> selection probability 45% -> 90% ZBD burned
- Bob:   3000 (30%) -> weight 7000 -> selection probability 35% -> 70% ZBD burned
- Carol: 6000 (60%) -> weight 4000 -> selection probability 20% -> 40% ZBD burned

Total weight: 20000
Weight = ZATTERA_100_PERCENT - exchange_rate
```

**Competitive dynamics**:
- Witnesses compete with **lower exchange rates** -> higher selection probability -> more withdrawals
- Lower exchange rate = more ZBD burned = greater deflation
- Too low -> lower witness rewards -> reduced profitability
- Market equilibrium determines optimal exchange rates
- **Fairness**: multiple witnesses get opportunities, no monopoly
- Witnesses can set different exchange rates for different contracts

#### 7.3 Long-term incentive alignment

**Benefits of ZP accumulation**:

1. **Increased voting power**: more ZP -> higher chance of being elected witness
2. **Curation rewards**: ZP earns rewards when curating content
3. **Compounding effect**: ZP continues to increase via inflation
4. **Long-term commitment**: 13-week power-down encourages long-term participation

**From the witness perspective**:
- Lower exchange rate -> more assignments -> more processing opportunities
- More ZP accumulation -> stronger voting power -> long-term witness status
- Register across multiple contracts for broader market access
- Optimize exchange rates per contract

**From the network perspective**:
- Automatic preference for low exchange rates -> more ZBD burned
- Deflation effect -> higher ZBD value
- Free competition among witnesses finds market equilibrium

#### 7.4 Transparency and trust

**Cumulative stats tracking**:
- `total_withdrawal_amount`: total processed amount (before exchange rate)
- `total_burned_amount`: total burned ZBD (network contribution)
- `total_withdrawal_count`: total processed count
- `last_withdrawal_time`: last processed time

**Usage**:
- Users can verify witness performance and trustworthiness
- Active witnesses = higher trust + stable service
- Network tracks witness contributions transparently
- **Burn contribution**: `total_burned_amount` measures deflation contribution

### 8. CLI wallet integration

```bash
# Step 1: Witnesses approve contract (majority required)
unlocked >>> approve_withdrawal_contract alice 1 \
    0x1234567890abcdef1234567890abcdef12345678
# Params: witness, chain_id, contract_address
# Meaning: Alice approves Ethereum (chain_id=1) contract 0x1234...

# Other witnesses approve similarly
unlocked >>> approve_withdrawal_contract bob 1 \
    0x1234567890abcdef1234567890abcdef12345678

unlocked >>> approve_withdrawal_contract carol 1 \
    0x1234567890abcdef1234567890abcdef12345678

# ... (until majority)

# List approved contracts
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

# Step 2: Witness registers service for approved contract
unlocked >>> register_withdrawal_contract alice 1 \
    0x1234567890abcdef1234567890abcdef12345678 \
    0x742d35Cc6634C0532925a3b844Bc454e4438f44e \
    3000 \
    ["0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48","0x6B175474E89094C44Da98b954EedeAC495271d0F"]
# Params: witness, chain_id, contract_address, signer_address, exchange_rate, supported_tokens
# Meaning: Alice offers service for approved 0x1234... contract at 30% exchange rate
#         Signer address is 0x742d... (exchange_rate: 3000 = 30%)
#         Supported tokens: USDC (0xA0b8...), DAI (0x6B17...)

# User: request ZBD withdrawal (specify contract and token)
unlocked >>> withdraw_dollar bob "100.000 ZBD" 1 \
    0x1234567890abcdef1234567890abcdef12345678 \
    0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 \
    0x9876543210abcdef9876543210abcdef98765432
# Params: from, amount, chain_id, contract_address, token, recipient
# Meaning: withdraw 100 ZBD to USDC on Ethereum via 0x1234..., send to 0x9876...

# Query withdrawal request
unlocked >>> get_withdraw_dollar_request 12345
{
  "request_id": 12345,
  "requester": "bob",
  "amount": "100.000 ZBD",           // CLI returns string, JS needs parsing
  "chain_id": 1,
  "contract_address": "0x1234567890abcdef1234567890abcdef12345678",
  "token": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",  // USDC
  "recipient": "0x9876543210abcdef9876543210abcdef98765432",
  "status": "signed",
  "assigned_witness": "alice",
  "witness_signature": "0x1234...",
  "exchange_amount": "30.000 ZBD",   // CLI returns string
  "exchange_rate": 3000,
  "created_at": "2025-12-31T12:00:00",
  "signed_at": "2025-12-31T12:02:30"
}
# Note: JSON-RPC API may return amount as raw value (number)
#       - CLI: "100.000 ZBD" (string)
#       - JSON-RPC: 100000 (number, 3 decimals raw value)

# Query witness contract registration
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
    "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48": "5000.000 ZBD",  // USDC deposit
    "0x6B175474E89094C44Da98b954EedeAC495271d0F": "3000.000 ZBD"   // DAI deposit
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

# Witness: report deposits (runs periodically)
unlocked >>> report_withdraw_dollar_deposits alice 1 \
    0x1234567890abcdef1234567890abcdef12345678 \
    {"0xA0b8...": "5000.000 ZBD", "0x6B17...": "3000.000 ZBD"}
# Params: witness, chain_id, contract_address, deposits (per-token ZBD equivalent)

# Witness: report completion (after external chain execution confirmed)
unlocked >>> report_withdrawal_complete alice 12345 0xabcd1234567890...

# List all witness stats for a contract
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

## Backwards compatibility

This ZEP adds the following:

**New Operations** (hardfork required):
- `approve_withdraw_dollar_contract_operation` - contract approval/revocation
- `update_withdraw_dollar_provider_operation` - witness service registration/update
- `report_withdraw_dollar_deposits_operation` - deposit reporting (periodic)
- `withdraw_dollar_request_operation` - withdrawal request
- `withdraw_dollar_approve_operation` - withdrawal approval (signature submission)
- `complete_withdraw_dollar_operation` - withdrawal completion report

**New Database Objects**:
- `withdraw_dollar_contract_object` - approved contract whitelist
- `withdraw_dollar_provider_object` - per-witness service settings
- `withdraw_dollar_request_object` - withdrawal request tracking

**Reused Object**:
- `convert_request_object` - witness ZBD -> ZP conversion (same as existing user conversion)

**New Plugin**:
- `withdraw_dollar_plugin`

**Security model**:
- Withdrawal contracts require majority (50% + 1) witness approval
- Unapproved contracts cannot be used
- Protects users from malicious contracts

Existing witness systems are unaffected, and participation in withdrawals is optional.

## Implementation notes

### `withdraw_dollar_request_object` memory management (`CLEAR_WITHDRAW_DOLLAR_REQUESTS`)

Memory management for completed/expired withdrawal requests follows the `CLEAR_VOTES` pattern.

**CMake option**:
```cmake
OPTION( CLEAR_WITHDRAW_DOLLAR_REQUESTS "Clear completed/expired withdrawal requests from memory" ON )
```

**Node-type behavior**:

| Node type | Setting | Behavior | Purpose |
|----------|---------|----------|---------|
| Witness/Seed | `ON` (default) | Immediately delete `completed`/`expired` | Memory efficiency, consensus |
| Account History | `OFF` | Preserve all requests | Full withdrawal history |

**Implementation example**:

```cpp
// complete_withdraw_dollar_evaluator.cpp
void complete_withdraw_dollar_evaluator::do_apply(const complete_withdraw_dollar_operation& o)
{
    // ... witness reward and conversion ...

    db.modify(*itr, [&](auto& obj) {
        obj.status = withdraw_dollar_status::completed;
        obj.tx_hash = o.tx_hash;
        obj.executed_at = db.head_block_time();
    });

    // ... stats update ...

#ifdef CLEAR_WITHDRAW_DOLLAR_REQUESTS
    db.remove(*itr);  // Consensus nodes: delete immediately
#endif
}

// withdraw_dollar_plugin.cpp - timeout handling
catch (const fc::exception& e)
{
    // Refund ZBD to user
    db.adjust_balance(itr->requester, itr->amount);
    db.adjust_supply(itr->amount);

    // Reduce provider reserved_deposits
    // ...

    elog("Request ${id} expired, refunded", ("id", itr->request_id));

#ifdef CLEAR_WITHDRAW_DOLLAR_REQUESTS
    db.remove(*itr);  // Consensus nodes: delete immediately
#else
    db.modify(*itr, [&](auto& obj) {
        obj.status = withdraw_dollar_status::expired;
    });
#endif
}
```

**History query API** (Account History nodes only):
```cpp
vector<withdraw_dollar_request_object> get_withdrawal_history(
    account_name_type account,
    uint32_t limit = 100
)
{
#ifndef CLEAR_WITHDRAW_DOLLAR_REQUESTS
    // Full history including completed/expired
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

**Memory savings**:
- Consensus nodes: keep only active requests (`queued`, `pending`, `signed`)
  - Estimated: 300 requests x 400 bytes = 120KB
- Account History nodes: keep full history
  - Estimated: ~300MB after 1 year

**Relation to EVM contract**:
- EVM `Withdrawn` event and `WithdrawInfo` mapping provide permanent history
- Zattera chain tracks only active requests to maximize memory efficiency
- Users can view full withdrawal history on Etherscan/Polygonscan

## Reference implementation

Reference implementation is located at:

- `src/plugins/witness_withdrawal/` - Witness withdrawal plugin
- `src/core/chain/withdrawal_evaluator.cpp` - Withdrawal evaluators
- `src/core/chain/include/zattera/chain/withdrawal_objects.hpp` - Database objects
- `contracts/ZatteraBridge.sol` - Ethereum smart contract

## Copyright

This document is placed in the public domain under CC0 1.0 Universal.
