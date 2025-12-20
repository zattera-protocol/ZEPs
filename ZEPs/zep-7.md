---
zep: 7
title: Voting Power Precision Increase
description: Change voting power field from uint16_t to uint32_t and increase precision from 10,000 to 100,000,000
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2025-12-19
---

## Abstract

This ZEP proposes a significant improvement to the voting power precision system. We propose changing the voting power from `uint16_t` type with 10,000 precision (ZATTERA_100_PERCENT) to `uint32_t` type with 100,000,000 precision. This enables precise handling of low-weight votes, fine-grained vote control for whale accounts, and enables third-party applications to utilize micro-voting.

## Motivation

### Current System Limitations

Zattera's current voting system has the following constraints:

**Current Configuration:**
- Voting power type: `uint16_t` (2 bytes)
- Precision: `ZATTERA_100_PERCENT = 10,000` (0.01% units)
- Minimum VP consumption: 1 (0.01% VP)
- Vote reserve rate: 40 (initial) → 10 (reduced)
- Vote regeneration: 5 days

**Core Issues:**

#### 1. Rounding Issues in Low-Weight Votes

At 100% VP, a 100% weight vote consumes 0.5% VP. This causes:

Vote Weight | VP Consumed (calc) | VP Consumed (actual) | Rounding Effect
------------|-------------------|---------------------|----------------
2%          | 0.01%             | 0.01%               | Accurate
1%          | 0.005%            | 0.01%               | 2x overconsumption
0.5%        | 0.0025%           | 0.01%               | 4x overconsumption

**Result**: All votes with 2% weight or less are treated the same as a 2% vote

#### 2. Daily Vote Limit

In the current system with 2% weight votes only:
- Daily maximum votes: 2,000
- Daily minimum votes: 50 (100% weight)

**Problems**:
- Curation bots and automated services hit limits
- Cannot use voting for internal flags or micro-rewards
- Limits scalability of third-party applications

#### 3. Whale Account Vote Control Issues

For accounts with large ZP (Zattera Power) holdings:

Account ZP   | Min rshares | Min Valid Vote Weight
-------------|-------------|---------------------
1,000,000    | ~0          | 2% or higher needed
10,000,000   | 1,000       | 1% possible
1,000,000,000| 99,000      | 0.01% possible but still excessive

**Problems**:
- Whales cannot give small rewards even when desired
- Fine-grained curation impossible
- Always exert "excessive influence"

#### 4. Third-Party Application Constraints

Use cases for apps wanting to leverage voting on social platforms:

**Spam Detection Bots**:
- Need differential flagging from 0.1%~5% based on suspicion level
- Current: All votes below 2% treated identically

**Forum Platforms**:
- Need fine-grained reward adjustment based on post quality
- Current: 2,000 votes/day limit prevents supporting large communities

**Curation Services**:
- Need precise reward distribution proportional to content scores
- Current: Lack of precision prevents fair distribution

### Precision Requirements Calculation

Mathematical derivation of minimum required precision:

```
Worst case: 0.01% VP × 0.5% rate × 0.01% weight
= 0.01% × 0.005 × 0.01%
= 0.00005%
= 5 / 10,000,000

Required precision: 10^7 (10,000,000)
```

**Future-proofing**:
- Vote reserve rate has reached 40 in the past
- May increase again in the future
- 0.01% VP × 2% rate × 0.01% weight = 0.0002% = 2 / 10,000,000
- **Recommended precision**: 10^8 (100,000,000)

**Type Selection**:
- `uint32_t` maximum: 4,294,967,295 (approx 10^9)
- 100,000,000 precision fits within `uint32_t` range
- Provides headroom for 10x future precision increase

## Specification

### Protocol Changes

#### 1. Constant Changes

**File**: `src/core/protocol/include/zattera/protocol/config.hpp`

```cpp
// Before
#define ZATTERA_100_PERCENT                     10000
#define ZATTERA_1_PERCENT                       (ZATTERA_100_PERCENT/100)

// After
#define ZATTERA_100_PERCENT                     100000000
#define ZATTERA_1_PERCENT                       (ZATTERA_100_PERCENT/100)
```

**Affected Derived Constants**:
```cpp
// Automatically adjusted
#define ZATTERA_DEFAULT_ZBD_INTEREST_RATE       (10*ZATTERA_1_PERCENT)
#define ZATTERA_CONTENT_REWARD_PERCENT          (75*ZATTERA_1_PERCENT)
#define ZATTERA_VESTING_FUND_PERCENT            (15*ZATTERA_1_PERCENT)
#define ZATTERA_ZBD_STOP_PERCENT                (5*ZATTERA_1_PERCENT)
#define ZATTERA_ZBD_START_PERCENT               (2*ZATTERA_1_PERCENT)
#define ZATTERA_IRREVERSIBLE_THRESHOLD          (75*ZATTERA_1_PERCENT)
```

#### 2. Account Object Field Type Change

**File**: `src/core/chain/include/zattera/chain/account_object.hpp`

```cpp
class account_object : public object<account_object_type, account_object>
{
   // ... existing fields ...

   // Before
   // uint16_t voting_power = ZATTERA_100_PERCENT;

   // After
   uint32_t voting_power = ZATTERA_100_PERCENT;

   // ... remaining fields ...
};
```

**Memory Impact**:
- Increase per account: 2 bytes (uint16_t → uint32_t)
- For 1 million accounts: 2 MB additional
- Relative to total state file (80GB): 0.0025% increase

#### 3. Vote Calculation Logic Changes

**File**: `src/core/chain/zattera_evaluator.cpp`

Existing calculation logic remains the same, but internal precision improves:

```cpp
void vote_evaluator::do_apply(const vote_operation& o)
{
   // ... existing validation logic ...

   int64_t abs_weight = abs(o.weight);

   // VP consumption calculation (10,000x precision increase)
   int64_t used_power = ((current_power * abs_weight) / ZATTERA_100_PERCENT) * (60*60*24);

   int64_t max_vote_denom = dgpo.vote_power_reserve_rate * ZATTERA_VOTE_REGENERATION_SECONDS;
   FC_ASSERT(max_vote_denom > 0);

   // Rounding (now much more granular)
   used_power = (used_power + max_vote_denom - 1) / max_vote_denom;
   FC_ASSERT(used_power <= current_power, "Account does not have enough power to vote.");

   // rshares calculation (10,000x precision increase)
   int64_t abs_rshares = ((uint128_t(effective_vesting) * used_power) / ZATTERA_100_PERCENT).to_uint64();

   abs_rshares -= ZATTERA_VOTE_DUST_THRESHOLD;
   abs_rshares = std::max(int64_t(0), abs_rshares);

   // ... remaining logic ...
}
```

**Key Improvements**:
- Rounding error reduced by 10,000x
- Minimum valid vote weight: 0.0001% (1/10,000th of previous 1%)
- Whales can adjust in 0.0001% increments

#### 4. API Compatibility

**File**: `src/plugins/apis/database_api/database_api.cpp`

API responses maintain backward compatibility:

```cpp
struct api_account_object
{
   // ... existing fields ...

   // uint32_t auto-serializes to JSON
   uint32_t voting_power = 0;  // 0 ~ 100,000,000

   // ... remaining fields ...
};
```

**Client Compatibility**:
- Existing clients: Need to update logic interpreting 10,000 as 100%
- New clients: Interpret 100,000,000 as 100%
- Recommended to branch on hardfork block number

### Behavioral Specification

#### VP Regeneration Logic

**No Change**: VP regeneration operates at the same rate

```cpp
// Regeneration formula (unchanged)
int64_t regenerated_power = (ZATTERA_100_PERCENT * elapsed_seconds) / ZATTERA_VOTE_REGENERATION_SECONDS;

// Precision only increased
// Before: 10,000 regenerated per 86,400 seconds
// After: 100,000,000 regenerated per 86,400 seconds
// Rate: Identical (100%/5 days)
```

#### Vote Weight Range

```cpp
// vote_operation validation
FC_ASSERT(abs(weight) <= ZATTERA_100_PERCENT, "Weight is not a ZATTERA percentage");

// Before: -10,000 ~ +10,000
// After: -100,000,000 ~ +100,000,000
```

#### Improved Voting Scenarios

**Scenario 1: Small Account (10M ZP)**

Vote Weight | VP Consumed | rshares | Validity
------------|-------------|---------|----------
0.1%        | 2,000       | 0       | Invalid (dust)
0.5%        | 10,000      | 0       | Invalid (dust)
1%          | 20,000      | 1,000   | Valid ✓

**Scenario 2: Medium Account (100M ZP)**

Vote Weight | VP Consumed | rshares  | Validity
------------|-------------|----------|----------
0.01%       | 200         | 0        | Invalid (dust)
0.05%       | 1,000       | 0        | Invalid (dust)
0.1%        | 2,000       | 1,000    | Valid ✓
1%          | 20,000      | 19,000   | Valid ✓

**Scenario 3: Whale Account (1B ZP)**

Vote Weight | VP Consumed | rshares     | Validity
------------|-------------|-------------|----------
0.01%       | 200         | 1,000       | Valid ✓
0.1%        | 2,000       | 19,000      | Valid ✓
1%          | 20,000      | 199,000     | Valid ✓

**Actual Daily Maximum Votes:**

Constraints:
- VP Regeneration: 100% regenerates in 5 days
- Time Constraint: Minimum 3-second interval (ZATTERA_MIN_VOTE_INTERVAL_SEC)

Vote Weight | VP-based Max | Time-based Max | Actual Max
------------|-------------|----------------|------------
0.01%       | 500,000     | 28,800         | 28,800 (time-limited)
0.1%        | 50,000      | 28,800         | 28,800 (time-limited)
1%          | 5,000       | 28,800         | 5,000 (VP-limited)
2%          | 2,500       | 28,800         | 2,500 (VP-limited)

## Rationale

### Design Decisions

#### 1. Precision Choice: 100,000,000 (10^8)

**Considerations**:
- 10^7: Meets current requirements but no headroom
- 10^8: Prepares for future reserve rate increases
- 10^9: Risk of exceeding uint32_t range

**Choice**: 10^8
- Ensures safety margin
- Fits within uint32_t range
- Steem chose the same

#### 2. Type Choice: uint32_t

**Alternative Comparison**:

| Type     | Max Value | Memory | Precision Support | Choice |
|----------|-----------|--------|-------------------|--------|
| uint16_t | 65,535    | 2 byte | 10^4              | Current |
| uint32_t | 4.2B      | 4 byte | 10^8 ✓            | **Selected** |
| uint64_t | 18.4E     | 8 byte | 10^18             | Excessive |

**Rationale**:
- uint32_t provides sufficient precision
- Minimizes memory increase (2 bytes)
- Ensures future extensibility

#### 3. Backward Compatibility vs. Clean Transition

**Decision**: Complete transition via hardfork

**Reasons**:
- Running two systems in parallel increases complexity
- Transitioning at early stage is advantageous
- Provides clear transition point

**Alternative (Rejected)**: Gradual transition
- Increases complexity
- Bug risk
- Maintenance burden

#### 4. API Change Strategy

**Decision**: Return new precision from hardfork block

**Implementation**:
```cpp
// Client-side example
uint32_t get_voting_power_percentage(uint32_t vp, uint32_t hardfork_num)
{
   if (hardfork_num >= HARDFORK_VP_PRECISION)
      return (vp * 100) / 100000000;  // New precision
   else
      return (vp * 100) / 10000;      // Old precision
}
```

### Alternative Approaches Considered

#### A. Relax Vote Interval

**Idea**: Reduce vote interval instead of precision (3s → 1s)

**Rejected Reasons**:
- Increases blockchain load
- Doesn't solve fundamental problem (rounding)
- Whale account issues persist

#### B. Variable Precision

**Idea**: Dynamic precision based on account ZP

**Rejected Reasons**:
- High implementation complexity
- Fairness concerns
- Unpredictable behavior

#### C. Use Floating Point

**Idea**: Use float/double instead of integers

**Rejected Reasons**:
- Consensus risk (floating-point non-determinism)
- Serialization complexity
- Performance degradation

## Backward Compatibility

This ZEP introduces **consensus-breaking changes requiring a hard fork**.

### Compatibility Impact Analysis

#### Chain Level

**Database Schema**:
- ⚠️ `account_object.voting_power` type change required
- ⚠️ Shared memory file regeneration required
- ⚠️ Existing chain data migration needed

**Consensus Rules**:
- ⚠️ Vote validation logic changes
- ⚠️ Reward calculation precision changes
- ✅ Vote behavior itself remains identical

#### API Level

**Database API**:
```json
// Before
{
  "voting_power": 10000  // 100%
}

// After
{
  "voting_power": 100000000  // 100%
}
```

**Impact**:
- ⚠️ Clients need to update 100% interpretation logic
- ✅ Field name/type identical (backward compatible)
- ✅ JSON parsers unaffected

#### Client Level

**Wallets/Apps**:
```javascript
// Needs modification
function getVotingPowerPercent(account) {
  // Before
  // return account.voting_power / 100;

  // After (with hardfork detection)
  if (isAfterHardfork(account.head_block_num)) {
    return account.voting_power / 1000000;
  } else {
    return account.voting_power / 100;
  }
}
```

**Required Actions**:
- Update display logic
- Increase vote weight slider precision
- Add hardfork detection logic

### Migration Strategy

#### Phase 1: Hardfork Preparation (HF-2 months)

1. **Code Deployment**:
   - Release new node software
   - Testnet validation
   - Documentation updates

2. **Client Notification**:
   - Announce API changes
   - Provide migration guide
   - Deploy sample code

#### Phase 2: Hardfork Activation (HF Day)

**Automatic Migration**:
```cpp
void database::process_hardfork_vp_precision()
{
   if (head_block_num() == HARDFORK_VP_PRECISION_BLOCK)
   {
      // Scale all account VP
      const auto& accounts = get_index<account_index>().indices();
      for (const auto& account : accounts)
      {
         modify(account, [&](account_object& a)
         {
            // Scale 10,000 → 100,000,000
            a.voting_power = (a.voting_power * 100000000) / 10000;
         });
      }
   }
}
```

**Validation**:
- Verify total VP sum invariance
- Confirm ratio preservation
- Log records

#### Phase 3: Post-Hardfork (HF+)

1. **Monitoring**:
   - Analyze voting patterns
   - Track performance metrics
   - Detect unexpected behavior

2. **Client Support**:
   - Respond to bug reports
   - Resolve migration issues

### Rollback Plan

**Worst-case scenario**: Critical bug discovered

**Response**:
1. Prepare emergency hardfork
2. Revert to previous precision
3. Regenerate chain

**Prevention**:
- Extensive testing
- Bug bounty program
- Long-term testnet operation

## Security Considerations

### Potential Vulnerabilities

#### 1. Integer Overflow

**Risk**: Overflow during large number calculations

**Affected Code**:
```cpp
int64_t used_power = ((current_power * abs_weight) / ZATTERA_100_PERCENT) * (60*60*24);
```

**Mitigation**:
```cpp
// Prevent overflow using uint128_t
int64_t used_power = ((uint128_t(current_power) * abs_weight) / ZATTERA_100_PERCENT).to_uint64() * (60*60*24);
```

**Verification**:
- Boundary value testing
- Fuzz testing
- Static analysis

#### 2. Rounding Attacks

**Risk**: Exploiting rounding errors for infinite VP usage

**Example**:
```
Attack: Repeat 0.0000001% votes
Intent: Gain rshares without VP consumption
```

**Mitigation**:
```cpp
// Blocked by ZATTERA_VOTE_DUST_THRESHOLD
abs_rshares -= ZATTERA_VOTE_DUST_THRESHOLD;  // 1000
abs_rshares = std::max(int64_t(0), abs_rshares);
```

**Effect**: Invalid if below minimum rshares

#### 3. VP Regeneration Inconsistency

**Risk**: VP regeneration errors at precision change boundary

**Scenario**:
```
Just before hardfork: VP = 5,000 (old precision)
Just after hardfork: VP = 50,000,000 (new precision)
Regeneration calculation: Inconsistency if calculated with different precisions
```

**Mitigation**:
```cpp
// Batch scaling at hardfork point
// All regeneration thereafter uses new precision
```

#### 4. API Parsing Errors

**Risk**: Clients interpreting with wrong precision

**Impact**:
- Incorrect VP display to users
- Vote weight miscalculation

**Mitigation**:
- Add API version field
- Documentation
- Provide example code

### Economic Impact

#### Reward Distribution

**Change**: Increased accuracy of micro-vote rewards

**Simulation**:
```
Before: 0.1% vote = 2% vote (same rshares)
After: 0.1% vote = exactly 5% rshares

Effect: Precise distribution even for small rewards
```

**Monitoring**:
- Track reward distribution changes
- Analyze curation patterns

#### Spam Prevention

**Concern**: Increased micro-vote spam

**Defense Mechanisms**:
1. Maintain `ZATTERA_MIN_VOTE_INTERVAL_SEC = 3`
2. Maintain `ZATTERA_VOTE_DUST_THRESHOLD`
3. Can adjust RC (Resource Credit) costs

**Expectation**: No major issues (existing limits sufficient)

## Performance Impact

### Memory

**Account Object Size**:
```
Before: ~140 bytes
After: ~142 bytes (+2 bytes)
```

**Total Impact** (1 million accounts):
```
Additional memory: 2 MB
State file: 80 GB → 80.002 GB
Growth rate: 0.0025%
```

**Conclusion**: Negligible

### CPU

**Vote Validation**:
- Only integer operations used (no floating-point)
- Identical computational complexity
- No additional cost

**Benchmark** (estimated):
```
Vote processing: ~0.5ms (unchanged before/after)
Block validation: ~10ms (unchanged before/after)
```

### Network

**Transaction Size**:
- `vote_operation` size unchanged
- Vote weight remains int16_t (no change)
- Network load identical

### Database

**Indexes**:
- No indexes sorting by voting_power
- No reindexing needed

**Query Performance**:
- No impact

## Test Cases

### Test Case 1: Precision Scaling Validation

```cpp
BOOST_AUTO_TEST_CASE(voting_power_precision_scaling)
{
   ACTORS((alice))

   // Before hardfork: 10,000 = 100%
   BOOST_REQUIRE(alice.voting_power == 10000);

   // Activate hardfork
   generate_blocks(HARDFORK_VP_PRECISION_BLOCK);

   // After hardfork: 100,000,000 = 100%
   const auto& new_alice = db->get_account("alice");
   BOOST_REQUIRE(new_alice.voting_power == 100000000);

   // Verify ratio preservation
   BOOST_REQUIRE(new_alice.voting_power / 100000000.0 == 1.0);
}
```

### Test Case 2: Low-Weight Vote Precision

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

   // 0.1% vote
   vote_operation vote;
   vote.voter = "alice";
   vote.author = "bob";
   vote.permlink = "test";
   vote.weight = 100000;  // 0.1% (100,000 / 100,000,000)

   auto old_vp = db->get_account("alice").voting_power;
   push_transaction(vote, alice_private_key);
   auto new_vp = db->get_account("alice").voting_power;

   // Verify VP consumption (previously same as 2%)
   int64_t used_vp = old_vp - new_vp;
   BOOST_REQUIRE(used_vp < old_vp / 200);  // Less than 0.5%
   BOOST_REQUIRE(used_vp > old_vp / 2000); // Greater than 0.05%
}
```

### Test Case 3: Whale Account Micro-Voting

```cpp
BOOST_AUTO_TEST_CASE(whale_micro_voting)
{
   ACTORS((whale)(bob))
   fund("whale", ASSET("1000000000.000 ZTR"));  // 1B ZTR
   vest("whale", ASSET("1000000000.000 ZTR"));
   generate_blocks(HARDFORK_VP_PRECISION_BLOCK);

   comment_operation post;
   post.author = "bob";
   post.permlink = "test";
   push_transaction(post, bob_private_key);

   // 0.01% vote (previously impossible)
   vote_operation vote;
   vote.voter = "whale";
   vote.author = "bob";
   vote.permlink = "test";
   vote.weight = 10000;  // 0.01% (10,000 / 100,000,000)

   push_transaction(vote, whale_private_key);

   const auto& comment = db->get_comment("bob", "test");

   // Verify rshares generation
   BOOST_REQUIRE(comment.net_rshares > 0);

   // Verify expected range
   int64_t whale_vests_balance = db->get_account("whale").vesting_share_balance.amount.value;
   int64_t expected_rshares = whale_vests_balance / 10000;  // 0.01%

   BOOST_REQUIRE(comment.net_rshares < expected_rshares * 1.1);  // 10% margin
   BOOST_REQUIRE(comment.net_rshares > expected_rshares * 0.9);
}
```

### Test Case 4: Mass Voting Performance

```cpp
BOOST_AUTO_TEST_CASE(mass_voting_performance)
{
   ACTORS((curator))
   fund("curator", ASSET("10000.000 ZTR"));
   vest("curator", ASSET("10000.000 ZTR"));
   generate_blocks(HARDFORK_VP_PRECISION_BLOCK);

   // Create 100 posts
   std::vector<std::string> permlinks;
   for (int i = 0; i < 100; i++) {
      comment_operation post;
      post.author = "curator";
      post.permlink = "post-" + std::to_string(i);
      post.body = "Content";
      push_transaction(post, curator_private_key);
      permlinks.push_back(post.permlink);
   }

   // Vote 0.1% × 100 times
   auto start_time = fc::time_point::now();

   for (const auto& permlink : permlinks) {
      vote_operation vote;
      vote.voter = "curator";
      vote.author = "curator";
      vote.permlink = permlink;
      vote.weight = 100000;  // 0.1%

      generate_blocks(1);  // Respect 3-second interval
      push_transaction(vote, curator_private_key);
   }

   auto end_time = fc::time_point::now();
   auto elapsed = (end_time - start_time).to_seconds();

   // Verify VP consumption
   const auto& curator_acc = db->get_account("curator");
   BOOST_REQUIRE(curator_acc.voting_power > 90000000);  // >90% remaining

   // Verify performance (100 votes in reasonable time)
   BOOST_REQUIRE(elapsed < 500);  // Less than 500 seconds
}
```

### Test Case 5: VP Regeneration Accuracy

```cpp
BOOST_AUTO_TEST_CASE(vp_regeneration_accuracy)
{
   ACTORS((alice))
   generate_blocks(HARDFORK_VP_PRECISION_BLOCK);

   // Deplete VP to 50%
   modify(db->get_account("alice"), [&](account_object& a) {
      a.voting_power = 50000000;  // 50%
      a.last_vote_time = db->head_block_time();
   });

   // Elapse 1 day
   generate_blocks(db->head_block_time() + fc::days(1));

   const auto& alice = db->get_account("alice");

   // Expected regeneration: 20% (100% / 5 days)
   int64_t expected_vp = 50000000 + 20000000;  // 70%

   // Verify actual regeneration (within 0.1% margin)
   BOOST_REQUIRE(abs(alice.voting_power - expected_vp) < 100000);
}
```

### Test Case 6: Integer Overflow Prevention

```cpp
BOOST_AUTO_TEST_CASE(integer_overflow_prevention)
{
   ACTORS((alice))
   generate_blocks(HARDFORK_VP_PRECISION_BLOCK);

   // Set to maximum VP
   modify(db->get_account("alice"), [&](account_object& a) {
      a.voting_power = 100000000;  // 100%
   });

   // Maximum weight vote
   vote_operation vote;
   vote.voter = "alice";
   vote.weight = 100000000;  // 100%

   // Should succeed without overflow
   BOOST_REQUIRE_NO_THROW(push_transaction(vote, alice_private_key));

   // Test negative weight too
   vote.weight = -100000000;
   BOOST_REQUIRE_NO_THROW(push_transaction(vote, alice_private_key));
}
```

## Reference Implementation

Implementation code available in branch:

**Branch**: `feature/vp-precision-increase`

**Key Changed Files**:
- `src/core/protocol/include/zattera/protocol/config.hpp` - Constant changes
- `src/core/chain/include/zattera/chain/account_object.hpp` - Type changes
- `src/core/chain/zattera_evaluator.cpp` - Calculation logic
- `src/core/chain/database.cpp` - Hardfork migration
- `tests/tests/voting_precision_tests.cpp` - Test cases

## Future Improvements

Potential extensions for future ZEPs:

### 1. Adaptive Precision

Dynamically adjust precision based on VP:

```cpp
uint32_t get_effective_precision(share_type vesting_shares)
{
   if (vesting_shares < 1000000)  // 1M VESTS
      return 100000;               // 10^5 precision
   else if (vesting_shares < 1000000000)  // 1B VESTS
      return 10000000;             // 10^7 precision
   else
      return 100000000;            // 10^8 precision
}
```

**Pros**: Memory savings
**Cons**: Increased complexity

### 2. VP Multiplier System

Accelerate VP regeneration under certain conditions:

```cpp
struct vp_multiplier_extension
{
   uint16_t multiplier = 100;  // 100 = 1.0x
   time_point_sec expiration;
};
```

**Use Cases**:
- Loyalty rewards
- Event period boosts

### 3. Dynamic Reserve Rate

Auto-adjust based on network activity:

```cpp
uint32_t calculate_dynamic_reserve_rate()
{
   uint64_t daily_votes = get_daily_vote_count();

   if (daily_votes > 10000000)  // Overload
      return 20;  // Faster VP regen
   else
      return 10;  // Normal
}
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
