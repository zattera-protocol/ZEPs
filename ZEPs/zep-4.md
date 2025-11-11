---
zep: 4
title: Network Participation-Based Reward Adjustment
description: Scale content rewards proportionally to network voting participation to prevent excessive rewards in early-stage networks
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2025-11-11
---

## Abstract

This ZEP proposes a mechanism to scale content rewards proportionally based on actual network voting participation. When total voting power falls below a configurable threshold, content rewards are reduced proportionally, with the remainder distributed to block producers (witnesses). This prevents disproportionate reward extraction during low-activity periods while incentivizing organic network growth.

## Motivation

In the early stages of a blockchain network launch, a small number of users with limited voting power could potentially claim disproportionately large rewards from the reward pool designed for a mature network. This creates several problems:

1. **Unfair reward distribution** - Small voting activity receives full rewards designed for a mature network
2. **Economic imbalance** - Early adopters can extract excessive value with minimal stake participation
3. **Network health** - Rewards should reflect actual network engagement and usage
4. **Witness incentivization** - Block producers need additional incentives during bootstrap phase

Existing solutions do not adequately address these concerns. Without this mechanism, early-stage networks are vulnerable to reward pool manipulation and economic imbalance that can harm long-term sustainability.

## Specification

### Overview

The system calculates a participation ratio on every block and applies a reduction factor to content rewards when network participation falls below a specified threshold.

### Key Terms

- **Total Network VESTS**: Sum of all vesting shares (staked tokens) in the network
- **Total Voting Rshares**: Sum of all reward shares (voting weight) for content being rewarded in the current block
- **Percent Voting Threshold**: Chain parameter defining required participation rate for full rewards
- **Participation Ratio**: `total_voting_rshares / (total_network_vests × percent_network_voting_threshold)`
- **Reduction Factor**: Multiplier applied to content rewards (0.0 to 1.0)

### Algorithm

For each block during reward distribution:

```
1. Calculate participation ratio:
   participation_ratio = total_voting_rshares / (total_network_vests × percent_network_voting_threshold)

2. Determine reduction factor:
   if participation_ratio >= 1.0:
       reduction_factor = 1.0  // Full rewards, no reduction
   else:
       reduction_factor = participation_ratio  // Proportional reduction

3. Distribute rewards:
   actual_content_reward = original_reward × reduction_factor
   witness_bonus = original_reward × (1 - reduction_factor)
```

### Chain Parameters

A new chain parameter `percent_network_voting_threshold` is introduced:

- **Type**: `uint16_t` (basis points, where 10000 = 100%)
- **Default**: 10000 (100% - full network participation required at genesis)
- **Range**: 0 to 10000
- **Modifiable via**: Witness consensus or hardfork
- **Recommended values**:
  - Genesis launch: 5000-10000 (50-100%)
  - Early growth: 1000-5000 (10-50%)
  - Mid-stage: 100-1000 (1-10%)
  - Mature network: 10-100 (0.1-1%) or 0 (disabled)

### Implementation Requirements

#### 1. Protocol Layer Changes

**File**: `libraries/protocol/include/zattera/protocol/chain_parameters.hpp`

Add chain parameter:
```cpp
struct chain_parameters
{
   // ... existing parameters ...
   uint16_t percent_network_voting_threshold = 10000;  // 100% default
};
```

#### 2. Database Modifications

**File**: `libraries/chain/database.cpp`

Modify `process_funds()` function:

```cpp
void database::process_funds()
{
   // ... existing reward pool calculations ...

   // Calculate total voting rshares for current reward period
   const auto& reward_idx = get_index<reward_fund_index>();
   const auto& reward_fund = reward_idx.get();

   share_type total_voting_rshares = reward_fund.recent_claims;

   // Get total network VESTS
   const auto& dgpo = get_dynamic_global_properties();
   share_type total_vests = dgpo.total_vesting_shares.amount;

   // Get threshold parameter
   const auto& chain_params = get_witness_schedule_object().median_props;
   uint16_t threshold = chain_params.percent_network_voting_threshold;

   // Calculate participation ratio
   // participation_ratio = total_voting_rshares / (total_vests × threshold / 10000)
   fc::uint128_t required_rshares = fc::uint128_t(total_vests.value) * threshold / 10000;

   uint16_t reduction_factor = 10000; // Default: no reduction (100%)

   if (required_rshares > 0)
   {
      fc::uint128_t ratio = (fc::uint128_t(total_voting_rshares.value) * 10000) / required_rshares;

      if (ratio < 10000)
      {
         reduction_factor = static_cast<uint16_t>(ratio.to_uint64());
      }
   }

   // Apply reduction to content rewards
   asset original_content_reward = /* calculated reward */;
   asset actual_content_reward = asset(
      (original_content_reward.amount.value * reduction_factor) / 10000,
      original_content_reward.symbol
   );

   asset witness_bonus = asset(
      original_content_reward.amount.value - actual_content_reward.amount.value,
      original_content_reward.symbol
   );

   // Distribute actual_content_reward to content creators
   // ... existing reward distribution logic ...

   // Add witness_bonus to current block witness
   if (witness_bonus.amount > 0)
   {
      const auto& cwit = get_witness(dgpo.current_witness);
      modify(get_account(cwpo.owner), [&](account_object& a) {
         a.balance += witness_bonus;
      });
   }
}
```

#### 3. Database Objects

Track threshold statistics in `dynamic_global_property_object`:

```cpp
class dynamic_global_property_object
{
   // ... existing fields ...

   // Voting threshold statistics
   uint32_t blocks_below_threshold = 0;
   asset total_rewards_redirected = asset(0, ZATTERA_SYMBOL);
   fc::uint128_t last_participation_ratio = 0;  // Basis points (10000 = 100%)
};
```

### API Extensions

Add new API methods to `database_api`:

```cpp
struct get_voting_threshold_info_return
{
   asset total_network_vests;
   fc::uint128_t total_voting_rshares;
   uint16_t percent_network_voting_threshold;
   uint16_t participation_ratio;          // Basis points
   uint16_t current_reduction_factor;      // Basis points
   uint32_t blocks_below_threshold;
   asset total_rewards_redirected;
};

get_voting_threshold_info_return get_voting_threshold_info();
```

## Rationale

### Design Decisions

1. **Linear Scaling**: The reduction factor scales linearly with participation ratio for simplicity and predictability. Alternative non-linear curves were considered but rejected due to increased complexity.

2. **Witness Bonus Distribution**: Redirected rewards go to block producers rather than being burned because:
   - Incentivizes witnesses during critical bootstrap phase
   - Maintains total inflation rate predictability
   - Aligns witness interests with network growth

3. **Threshold as Percentage**: Using percentage of total VESTS allows the mechanism to scale naturally as the network grows without manual recalibration.

4. **No Upper Bound on Participation**: Participation ratios above 100% do not provide bonus rewards to prevent gaming via artificial inflation of voting activity.

### Alternative Approaches Considered

1. **Absolute Threshold**: Fixed rshares requirement instead of percentage - rejected because it doesn't scale with network growth

2. **Time-based Decay**: Gradually reduce requirements over time - rejected because it doesn't reflect actual usage

3. **Burned Rewards**: Burn reduced portion instead of giving to witnesses - rejected because it doesn't incentivize anyone to bootstrap the network

4. **Quadratic Scaling**: Non-linear reduction curve - rejected due to complexity and gas costs

## Backwards Compatibility

This proposal introduces a consensus-breaking change and requires a hard fork.

### Migration Path

1. **Pre-Hardfork**: Set `percent_network_voting_threshold = 0` to disable the mechanism
2. **Hardfork Activation**: Enable with conservative threshold (e.g., 10000 = 100%)
3. **Post-Hardfork**: Gradually reduce threshold via witness consensus as network matures

### Impact on Existing Functionality

- **Reward Calculations**: All content reward calculations are affected
- **Witness Rewards**: Witnesses receive additional variable rewards during low-activity periods
- **API Responses**: Reward-related API calls will show adjusted amounts
- **Wallet Displays**: UIs must account for variable reward rates

## Test Cases

### Test Case 1: Below Threshold (30% Participation)

```cpp
BOOST_AUTO_TEST_CASE( reward_reduction_below_threshold )
{
   // Setup
   total_network_vests = 1,000,000 VESTS
   percent_network_voting_threshold = 10000 (100%)
   total_voting_rshares = 300,000 (30% of total VESTS)
   original_reward = 100 ZATTERA

   // Expected
   participation_ratio = 300,000 / 1,000,000 = 0.3
   reduction_factor = 0.3
   actual_content_reward = 100 * 0.3 = 30 ZATTERA
   witness_bonus = 100 * 0.7 = 70 ZATTERA
}
```

### Test Case 2: Above Threshold (150% Participation)

```cpp
BOOST_AUTO_TEST_CASE( no_reduction_above_threshold )
{
   // Setup
   total_network_vests = 100,000,000 VESTS
   percent_network_voting_threshold = 1000 (10%)
   required_voting = 10,000,000 VESTS
   total_voting_rshares = 15,000,000 (150% of required)
   original_reward = 100 ZATTERA

   // Expected
   participation_ratio = 15,000,000 / 10,000,000 = 1.5 (capped at 1.0)
   reduction_factor = 1.0
   actual_content_reward = 100 ZATTERA (no reduction)
   witness_bonus = 0 ZATTERA
}
```

### Test Case 3: Exactly At Threshold

```cpp
BOOST_AUTO_TEST_CASE( exact_threshold_boundary )
{
   // Setup
   total_network_vests = 1,000,000 VESTS
   percent_network_voting_threshold = 5000 (50%)
   required_voting = 500,000 VESTS
   total_voting_rshares = 500,000 (exactly 100% of required)
   original_reward = 100 ZATTERA

   // Expected
   participation_ratio = 500,000 / 500,000 = 1.0
   reduction_factor = 1.0
   actual_content_reward = 100 ZATTERA
   witness_bonus = 0 ZATTERA
}
```

## Reference Implementation

A complete reference implementation is available in the feature branch:

- Branch: `feature/voting-threshold-reward-reduction`
- Key Files:
  - `libraries/protocol/include/zattera/protocol/chain_parameters.hpp`
  - `libraries/chain/database.cpp` (process_funds function)
  - `libraries/chain/include/zattera/chain/global_property_object.hpp`
  - `tests/tests/reward_tests.cpp`

## Security Considerations

### Potential Attack Vectors

1. **Threshold Gaming**: Malicious actors could coordinate to keep participation just below threshold to maximize witness bonuses
   - Mitigation: Requires significant stake to meaningfully affect participation ratio
   - Monitoring: Track sudden drops in participation rates

2. **Witness Collusion**: Witnesses could benefit from keeping threshold artificially high
   - Mitigation: Threshold adjustment requires witness consensus
   - Transparency: All parameter changes are publicly visible on-chain

3. **Reward Calculation Overflow**: Large VESTS values could cause overflow in ratio calculation
   - Mitigation: Use `fc::uint128_t` for intermediate calculations
   - Validation: Add overflow checks in critical paths

### Economic Security

1. **Inflation Consistency**: Total reward distribution (content + witness bonus) remains constant, maintaining predictable inflation
2. **No Burning**: All rewards are distributed to participants, preventing deflationary pressure
3. **Gradual Transition**: Recommended threshold reduction schedule prevents economic shocks

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
