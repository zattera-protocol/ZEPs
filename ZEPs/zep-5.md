---
zep: 5
title: Account Reward Level System
description: Multi-tier account classification system with customizable reward multipliers governed by witness consensus
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2025-11-11
---

## Abstract

This ZEP proposes a four-tier reward level system (Levels 0-3) that allows differential reward distribution based on account classification. Both account level assignments and reward multipliers are controlled through a decentralized witness consensus mechanism, ensuring transparent and accountable reward governance.

## Motivation

Blockchain reward systems traditionally apply uniform reward rates to all participants regardless of their contribution quality, stake, or network value. This one-size-fits-all approach limits the network's ability to:

1. **Incentivize high-quality contributions** - Reward exceptional content creators and curators
2. **Attract strategic participants** - Offer premium rewards to valuable ecosystem members
3. **Implement flexible economics** - Adjust reward structures without protocol changes
4. **Recognize different participation levels** - Acknowledge varying degrees of network contribution

A tiered reward system with witness governance enables:
- Meritocratic reward distribution
- Decentralized decision-making on reward allocation
- Dynamic economic policy without frequent hardforks
- Transparent and auditable level assignments

## Specification

### Reward Levels

The system defines four discrete reward levels:

| Level | Description | Default Multiplier |
|-------|-------------|-------------------|
| 0 | Base (default) | 100% (1.00x) |
| 1 | Enhanced | 125% (1.25x) |
| 2 | Premium | 150% (1.50x) |
| 3 | Maximum | 200% (2.00x) |

### Core Operations

#### 1. Reward Level Proposal Operation

```cpp
struct reward_level_proposal_operation
{
   account_name_type proposing_witness;  // Witness proposing the change
   account_name_type target_account;     // Account to adjust
   uint8_t           new_reward_level;   // Proposed level (0-3)
   string            justification;      // Reason (max 1000 chars)

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

#### 2. Reward Level Approval Operation

```cpp
struct reward_level_approval_operation
{
   account_name_type              witness;
   reward_level_proposal_id_type  proposal_id;
   bool                           approve;  // true=approve, false=reject

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

#### 3. Reward Multiplier Proposal Operation

```cpp
struct reward_multiplier_proposal_operation
{
   account_name_type       proposing_witness;
   array<uint16_t, 4>      new_multipliers;  // Basis points (10000 = 100%)
   string                  justification;

   extensions_type         extensions;

   void validate() const
   {
      FC_ASSERT(is_valid_account_name(proposing_witness), "Invalid witness");
      FC_ASSERT(!justification.empty() && justification.size() <= 1000,
                "Justification required (max 1000 characters)");

      // Multipliers must be in range [10%, 500%]
      for (size_t i = 0; i < 4; i++)
      {
         FC_ASSERT(new_multipliers[i] >= 1000 && new_multipliers[i] <= 50000,
                   "Multipliers must be between 1000 (10%) and 50000 (500%)");
      }

      // Monotonic increase: higher levels >= lower levels
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

#### 4. Reward Multiplier Approval Operation

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

### Database Objects

#### Account Object Extension

```cpp
// In account_object class
class account_object
{
   // ... existing fields ...

   uint8_t reward_level = 0;  // Default: Level 0

   // ... rest of object ...
};
```

#### Reward Level Proposal Object

```cpp
class reward_level_proposal_object
{
   reward_level_proposal_id_type  id;
   account_name_type              target_account;
   uint8_t                        proposed_level;
   account_name_type              proposing_witness;
   time_point_sec                 created;
   string                         justification;
   flat_set<account_name_type>    approvals;  // Approving witnesses
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

#### Reward Multiplier Object

```cpp
class reward_multiplier_object
{
   reward_multiplier_id_type   id;
   array<uint16_t, 4>          multipliers = {
      10000,  // Level 0: 100%
      12500,  // Level 1: 125%
      15000,  // Level 2: 150%
      20000   // Level 3: 200%
   };
};
```

#### Reward Multiplier Proposal Object

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

### Consensus Mechanism

#### Witness Validation

All proposal and approval operations require:
1. Proposer/approver must be an active witness
2. Witness must have valid `signing_key` (not `public_key_type()`)

```cpp
const auto& witness = _db.get_witness(o.proposing_witness);
FC_ASSERT(witness.signing_key != public_key_type(), "Not an active witness");
```

#### Approval Threshold

Both level changes and multiplier updates require **2/3 supermajority** of active witnesses:

```cpp
// Count active witnesses (top 21)
const auto& witness_idx = _db.get_index<witness_index>().indices().get<by_vote_name>();
uint32_t total_active = 0;
for (auto itr = witness_idx.begin();
     itr != witness_idx.end() && total_active < STEEM_MAX_WITNESSES;
     ++itr)
{
   if (itr->signing_key != public_key_type())
      total_active++;
}

const uint32_t required = (total_active * 2 / 3) + 1;  // 67% + 1

if (proposal.approvals.size() >= required)
{
   // Apply change and remove proposal
}
```

#### Auto-Approval

The proposing witness automatically approves their own proposal:

```cpp
_db.create<reward_level_proposal_object>([&](auto& p) {
   // ... set fields ...
   p.approvals.insert(o.proposing_witness);  // Auto-approve
});
```

### Reward Calculation Integration

#### Multiplier Retrieval

```cpp
inline uint16_t get_reward_multiplier(const database& db, uint8_t level)
{
   FC_ASSERT(level <= 3, "Invalid reward level");
   const auto& multiplier_obj = db.get<reward_multiplier_object>();
   return multiplier_obj.multipliers[level];
}
```

#### Reward Application

All reward calculations are modified to apply the multiplier:

```cpp
// Author rewards
asset calculate_author_reward(const database& db,
                               const account_object& author,
                               asset base_reward)
{
   uint16_t multiplier = get_reward_multiplier(db, author.reward_level);
   return asset((base_reward.amount.value * multiplier) / 10000,
                base_reward.symbol);
}

// Curation rewards
asset calculate_curation_reward(const database& db,
                                 const account_object& curator,
                                 asset base_reward)
{
   uint16_t multiplier = get_reward_multiplier(db, curator.reward_level);
   return asset((base_reward.amount.value * multiplier) / 10000,
                base_reward.symbol);
}
```

### API Methods

```cpp
// Get account reward level
struct get_reward_level_return
{
   uint8_t  reward_level;
   uint16_t reward_multiplier;  // Current multiplier in basis points
};

get_reward_level_return get_reward_level(account_name_type account);

// Get current multipliers
struct get_reward_multipliers_return
{
   array<uint16_t, 4> multipliers;
};

get_reward_multipliers_return get_reward_multipliers();

// Get pending level proposals
struct get_reward_level_proposals_return
{
   vector<reward_level_proposal_api_object> proposals;
};

get_reward_level_proposals_return get_reward_level_proposals(
   optional<account_name_type> account);

// Get pending multiplier proposals
struct get_reward_multiplier_proposals_return
{
   vector<reward_multiplier_proposal_api_object> proposals;
};

get_reward_multiplier_proposals_return get_reward_multiplier_proposals();
```

## Rationale

### Witness Governance

**Why witness consensus instead of stakeholder voting?**

1. **Expertise**: Witnesses have deep technical and economic understanding
2. **Accountability**: Witnesses are elected and can be unvoted for poor decisions
3. **Efficiency**: Faster decision-making compared to full stakeholder votes
4. **Precedent**: Aligns with existing witness-governed parameters (block size, fees, etc.)

### Supermajority Requirement

**Why 2/3 majority instead of simple majority?**

1. **Prevents hasty changes**: Requires broad consensus
2. **Protects minority**: Prevents 11/21 witnesses from dominating
3. **Aligns with Byzantine fault tolerance**: Standard for distributed consensus
4. **Industry standard**: Common in blockchain governance (Cosmos, Polkadot, etc.)

### Four-Level System

**Why 4 levels instead of more granular tiers?**

1. **Simplicity**: Easy to understand and communicate
2. **Meaningful differentiation**: Each tier represents significant difference
3. **Gas efficiency**: Minimal storage overhead (1 byte per account)
4. **Flexibility**: Multipliers can be adjusted without adding levels

### Immutable Justifications

**Why require permanent on-chain justifications?**

1. **Transparency**: All decisions are publicly auditable
2. **Accountability**: Witnesses must explain their reasoning
3. **Historical record**: Future analysis of governance decisions
4. **Fraud prevention**: Prevents undisclosed coordination

### Dynamic Multipliers

**Why allow multiplier changes without hardforks?**

1. **Economic flexibility**: Adapt to changing network conditions
2. **Experimentation**: Test different reward structures
3. **Reduced hardfork frequency**: Fewer disruptive upgrades
4. **Responsive governance**: Quick adjustments during crises

## Backwards Compatibility

This ZIP introduces consensus-breaking changes requiring a hard fork.

### Migration Strategy

**Phase 1: Hardfork Activation**
- All accounts default to Level 0
- Default multipliers activated: [100%, 125%, 150%, 200%]
- Proposal/approval operations become valid

**Phase 2: Initial Level Assignments (Optional)**
- Genesis block can pre-assign levels to specific accounts
- Requires community discussion and pre-hardfork consensus

**Phase 3: Ongoing Governance**
- Witnesses begin proposing level changes
- Community monitors and evaluates decisions

### Breaking Changes

1. **Reward calculations**: All reward computations affected
2. **Account object schema**: New `reward_level` field added
3. **API responses**: New fields in account queries
4. **Database indices**: New proposal tracking indices

### Non-Breaking Elements

- Existing reward algorithms continue to work (with multiplier = 1.0)
- No changes to transaction format for existing operations
- Account balances and history remain intact

## Test Cases

### Test Case 1: Level Proposal Creation

```cpp
BOOST_AUTO_TEST_CASE(reward_level_proposal_create)
{
   ACTORS((alice)(witness1))

   // Make witness1 active
   witness_create("witness1", witness1_key, "url", witness1_pub, 0);

   // Verify alice starts at level 0
   const auto& alice_account = db->get_account("alice");
   BOOST_REQUIRE_EQUAL(alice_account.reward_level, 0);

   // Witness proposes level 2
   reward_level_proposal_operation op;
   op.proposing_witness = "witness1";
   op.target_account = "alice";
   op.new_reward_level = 2;
   op.justification = "Outstanding contributor";

   push_transaction(op, witness1_key);

   // Verify proposal created
   const auto& idx = db->get_index<reward_level_proposal_index>()
                         .indices().get<by_target_account>();
   auto proposal = idx.find("alice");
   BOOST_REQUIRE(proposal != idx.end());
   BOOST_REQUIRE_EQUAL(proposal->proposed_level, 2);
   BOOST_REQUIRE_EQUAL(proposal->approvals.size(), 1);  // Auto-approved
}
```

### Test Case 2: Consensus Achievement

```cpp
BOOST_AUTO_TEST_CASE(reward_level_consensus)
{
   // Create 21 witnesses
   ACTORS((alice)(w1)(w2)(w3)...(w21))

   for (auto wit : witnesses) {
      witness_create(wit, ...);
   }

   // Witness1 proposes alice -> level 3
   reward_level_proposal_operation prop;
   prop.proposing_witness = "w1";
   prop.target_account = "alice";
   prop.new_reward_level = 3;
   prop.justification = "Exceptional contributions";
   push_transaction(prop, w1_key);

   auto proposal_id = /* get proposal id */;

   // Initially, alice still at level 0
   BOOST_REQUIRE_EQUAL(db->get_account("alice").reward_level, 0);

   // 13 more witnesses approve (total 14/21 = 67%)
   for (int i = 2; i <= 14; i++) {
      reward_level_approval_operation approve;
      approve.witness = witnesses[i];
      approve.proposal_id = proposal_id;
      approve.approve = true;
      push_transaction(approve, witness_keys[i]);
   }

   // Now alice should be level 3
   BOOST_REQUIRE_EQUAL(db->get_account("alice").reward_level, 3);

   // Proposal should be removed
   const auto& idx = db->get_index<reward_level_proposal_index>()
                         .indices().get<by_target_account>();
   BOOST_REQUIRE(idx.find("alice") == idx.end());
}
```

### Test Case 3: Multiplier Update

```cpp
BOOST_AUTO_TEST_CASE(reward_multiplier_update)
{
   // Create witnesses
   ACTORS((w1)(w2)...(w21))

   // Get initial multipliers
   const auto& mult = db->get<reward_multiplier_object>();
   BOOST_REQUIRE_EQUAL(mult.multipliers[0], 10000);  // 100%
   BOOST_REQUIRE_EQUAL(mult.multipliers[3], 20000);  // 200%

   // Witness proposes new multipliers
   reward_multiplier_proposal_operation prop;
   prop.proposing_witness = "w1";
   prop.new_multipliers = {10000, 13000, 16000, 25000};
   prop.justification = "Adjusting economic incentives";
   push_transaction(prop, w1_key);

   // Get consensus (14 approvals)
   // ... approval operations ...

   // Verify multipliers updated
   BOOST_REQUIRE_EQUAL(mult.multipliers[1], 13000);  // 130%
   BOOST_REQUIRE_EQUAL(mult.multipliers[3], 25000);  // 250%
}
```

### Test Case 4: Reward Application

```cpp
BOOST_AUTO_TEST_CASE(reward_multiplier_application)
{
   ACTORS((alice)(bob))

   // Set alice to level 0, bob to level 3
   // (via consensus process)

   // Both create identical content and receive votes
   // ... content creation and voting ...

   // Calculate expected rewards
   asset base_reward = ASSET("100.000 TESTS");

   // Alice (level 0, 100% multiplier)
   asset alice_expected = base_reward;

   // Bob (level 3, 200% multiplier)
   asset bob_expected = ASSET("200.000 TESTS");

   // Verify rewards distributed correctly
   BOOST_REQUIRE_EQUAL(alice_account.reward_sbd_balance, alice_expected);
   BOOST_REQUIRE_EQUAL(bob_account.reward_sbd_balance, bob_expected);
}
```

## Reference Implementation

Reference implementation available in feature branch:

- Branch: `feature/reward-level-system`
- Key Files:
  - `libraries/protocol/include/steem/protocol/steem_operations.hpp`
  - `libraries/chain/include/steem/chain/reward_level_objects.hpp`
  - `libraries/chain/steem_evaluator.cpp`
  - `libraries/chain/database.cpp`
  - `tests/tests/reward_level_tests.cpp`

## Security Considerations

### Governance Attacks

**Witness Collusion**
- Risk: Malicious witnesses assign high levels to confederates
- Mitigation:
  - 2/3 supermajority requirement
  - Public justifications enable community oversight
  - Witnesses can be unvoted for abuse

**Sybil Attacks on Witness Voting**
- Risk: Attacker gains control of witness seats
- Mitigation: Requires massive stake to control 14+ witness positions
- Detection: Community monitors unusual witness voting patterns

### Economic Attacks

**Inflation Manipulation**
- Risk: High-level accounts inflate reward pool
- Mitigation: Multipliers have hard caps (max 500%)
- Monitoring: Track total reward distribution by level

**Level Farming**
- Risk: Users game system to obtain high levels
- Mitigation: Witness judgment + justification requirements
- Future: Automated eligibility criteria (ZIP extension)

### Technical Vulnerabilities

**Proposal ID Exhaustion**
- Risk: Proposal IDs are finite
- Mitigation: Use 64-bit IDs (2^64 proposals possible)
- Cleanup: Expired proposals can be pruned

**Multiplier Overflow**
- Risk: Large multipliers cause arithmetic overflow
- Mitigation:
  - Validation enforces max 50000 (500%)
  - Use 64-bit arithmetic for reward calculations
  - Assertions check for overflow

**Concurrent Proposals**
- Risk: Multiple proposals for same account
- Mitigation: Only one active proposal per account allowed
- Enforcement: Proposal creation checks for existing proposals

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
