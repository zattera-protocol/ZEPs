---
zep: 2
title: NFT-Based Witness Verification
description: Require witnesses to prove ownership of ERC-721 NFTs before block production eligibility
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2025-11-08
requires: 3
---

## Abstract

This ZEP proposes a witness authorization system requiring block producers to prove ownership of ERC-721 NFTs on Ethereum-compatible chains. Witnesses must periodically submit cryptographic proofs demonstrating that they are the actual owners of the relevant NFTs. An oracle service verifies on-chain ownership before witnesses become eligible for block production.

## Motivation

Traditional blockchain consensus relies solely on stake-based or reputation-based witness selection. This creates limitations:

1. **Limited Sybil resistance**: Witnesses can split stake across multiple identities
2. **Plutocracy concerns**: Wealth concentration leads to centralized block production
3. **No external accountability**: No link between blockchain identity and real-world assets
4. **Governance rigidity**: Witness requirements fixed in protocol, difficult to adjust

NFT-based verification addresses these issues:

1. **Enhanced Sybil resistance**: NFT ownership provides additional identity verification
2. **Flexible governance**: NFT collection requirements can be voted on by existing witnesses
3. **Community alignment**: Link witness eligibility to holding community-issued NFTs
4. **Cross-chain integration**: Leverage Ethereum ecosystem's established NFT infrastructure
5. **Economic security**: NFTs represent financial stake in ecosystem success

### Use Cases

- **Governance NFTs**: Project issues NFTs to trusted community members
- **Witness Licenses**: Limited NFT collection creates fixed witness set size
- **Stake Representation**: NFTs represent large stake holders' voting power
- **Cross-chain DAOs**: Coordinate governance across multiple blockchains

## Specification

### System Architecture

```
┌─────────────────────┐
│   Witness Node      │
│                     │
│  1. Generate ETH    │
│     signature       │
│     proving NFT     │
│     ownership       │
└──────────┬──────────┘
           │
           ▼
┌──────────────────────┐      ┌──────────────────────┐
│  Zattera Blockchain  │◄────►│  Oracle Service      │
│                      │      │  (Off-chain)         │
│  2. Verify           │      │                      │
│     signature        │      │  3. Query ERC-721    │
│                      │      │     ownerOf()        │
│  4. Update witness   │      │     on EVM chain     │
│     schedule         │      │                      │
└──────────────────────┘      └──────────────────────┘
```

### Core Components

#### 1. NFT Collection Registration

Witnesses collectively vote on which NFT collections are accepted.

**Operation**: `nft_collection_register_operation`

```cpp
struct nft_collection_register_operation : public base_operation
{
   account_name_type proposer;             // Witness proposing collection

   string            collection_name;       // Human-readable name
   string            contract_address;      // 0x... format (42 chars)
   uint32_t          chain_id;              // 1=Ethereum, 137=Polygon, etc.
   uint32_t          min_nft_count;         // Minimum NFTs required (default: 1)
   bool              active;                // Initially false until approved

   extensions_type   extensions;

   void validate() const
   {
      FC_ASSERT(is_valid_account_name(proposer), "Invalid proposer");
      FC_ASSERT(!collection_name.empty() && collection_name.size() <= 100,
                "Collection name required (max 100 chars)");
      FC_ASSERT(contract_address.size() == 42, "Invalid contract address");
      FC_ASSERT(contract_address.substr(0, 2) == "0x",
                "Contract address must start with 0x");
      FC_ASSERT(min_nft_count > 0 && min_nft_count <= 100,
                "Min NFT count must be 1-100");
   }

   void get_required_active_authorities(flat_set<account_name_type>& a) const
   {
      a.insert(proposer);
   }
};
```

**Operation**: `nft_collection_approve_operation`

```cpp
struct nft_collection_approve_operation : public base_operation
{
   account_name_type witness_account;
   uint64_t          collection_id;
   bool              approve;  // true=approve, false=reject

   extensions_type   extensions;

   void validate() const
   {
      FC_ASSERT(is_valid_account_name(witness_account), "Invalid witness");
   }

   void get_required_active_authorities(flat_set<account_name_type>& a) const
   {
      a.insert(witness_account);
   }
};
```

**Database Object**: `nft_collection_object`

```cpp
class nft_collection_object : public object<nft_collection_object_type, nft_collection_object>
{
   nft_collection_id_type  id;

   string                  collection_name;
   string                  contract_address;
   uint32_t                chain_id;
   uint32_t                min_nft_count;
   bool                    active;

   // Witness voting
   flat_set<account_name_type> approved_by;
   flat_set<account_name_type> rejected_by;

   time_point_sec          created;
   time_point_sec          last_updated;
};

typedef multi_index_container<
   nft_collection_object,
   indexed_by<
      ordered_unique<tag<by_id>, member<...>>,
      ordered_unique<tag<by_collection_name>, member<...>>,
      ordered_non_unique<tag<by_contract_address>,
         composite_key<nft_collection_object,
            member<..., string, &nft_collection_object::contract_address>,
            member<..., uint32_t, &nft_collection_object::chain_id>
         >
      >
   >
> nft_collection_index;
```

**Approval Logic**:

```cpp
void nft_collection_approve_evaluator::do_apply(const nft_collection_approve_operation& o)
{
   const auto& witness = _db.get_witness(o.witness_account);
   FC_ASSERT(witness.signing_key != public_key_type(), "Not active witness");

   const auto& collection = _db.get<nft_collection_object>(o.collection_id);

   _db.modify(collection, [&](nft_collection_object& c)
   {
      if (o.approve)
      {
         c.approved_by.insert(o.witness_account);
         c.rejected_by.erase(o.witness_account);
      }
      else
      {
         c.rejected_by.insert(o.witness_account);
         c.approved_by.erase(o.witness_account);
      }

      c.last_updated = _db.head_block_time();

      // Activate if majority (11/21) of top witnesses approve
      const auto& witness_idx = _db.get_index<witness_index>()
                                    .indices().get<by_vote_name>();
      size_t top_witness_count = 0;
      size_t approvals_from_top = 0;

      for (auto itr = witness_idx.begin();
           itr != witness_idx.end() && top_witness_count < ZATTERA_MAX_WITNESSES;
           ++itr, ++top_witness_count)
      {
         if (c.approved_by.count(itr->owner) > 0)
            approvals_from_top++;
      }

      c.active = (approvals_from_top >= (ZATTERA_MAX_WITNESSES / 2 + 1));
   });
}
```

#### 2. Witness NFT Ownership Proof

Witnesses submit proofs linking their Zattera account to Ethereum address.

**Operation**: `witness_nft_proof_operation`

```cpp
struct witness_nft_proof_operation : public base_operation
{
   account_name_type witness_account;

   string            eth_address;           // 0x... format (42 chars)
   string            contract_address;      // 0x... format (42 chars)
   string            token_id;              // NFT token ID
   uint32_t          chain_id;              // Chain identifier

   string            eth_signature;         // EIP-191 signature (0x... 132 chars)
   time_point_sec    timestamp;             // Proof timestamp

   extensions_type   extensions;

   void validate() const
   {
      FC_ASSERT(is_valid_account_name(witness_account), "Invalid witness");

      // Validate Ethereum address (0x + 40 hex chars)
      FC_ASSERT(eth_address.size() == 42 && eth_address.substr(0, 2) == "0x",
                "Invalid Ethereum address format");

      // Validate contract address
      FC_ASSERT(contract_address.size() == 42 &&
                contract_address.substr(0, 2) == "0x",
                "Invalid NFT contract address");

      // Validate token ID
      FC_ASSERT(!token_id.empty(), "Token ID required");

      // Validate signature (0x + 130 hex chars = 65 bytes)
      FC_ASSERT(eth_signature.size() == 132 && eth_signature.substr(0, 2) == "0x",
                "Invalid signature format");

      // Timestamp must be recent (within 24 hours)
      auto now = time_point::now();
      FC_ASSERT(timestamp <= now, "Timestamp cannot be in future");
      FC_ASSERT(now - timestamp <= fc::days(1),
                "Proof too old (must be within 24 hours)");
   }

   void get_required_active_authorities(flat_set<account_name_type>& a) const
   {
      a.insert(witness_account);
   }
};
```

**Message Format for Signature**:

```
"zattera:<witness_account>:nft:<contract_address>:<token_id>:<timestamp>"
```

Example:
```
"zattera:alice:nft:0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D:1234:1704672000"
```

**Signature Verification**:

```cpp
void witness_nft_proof_evaluator::do_apply(const witness_nft_proof_operation& o)
{
   const auto& witness = _db.get_witness(o.witness_account);
   FC_ASSERT(witness.signing_key != public_key_type(), "Witness not active");

   // Check if NFT collection is registered and active
   const auto& collection_idx = _db.get_index<nft_collection_index>()
                                    .indices().get<by_contract_address>();
   auto collection_itr = collection_idx.find(
      boost::make_tuple(o.contract_address, o.chain_id)
   );

   FC_ASSERT(collection_itr != collection_idx.end(),
             "NFT collection not registered: ${c} on chain ${id}",
             ("c", o.contract_address)("id", o.chain_id));
   FC_ASSERT(collection_itr->active, "NFT collection not active");

   // Verify Ethereum signature
   string message = "zattera:" + o.witness_account.operator string() +
                    ":nft:" + o.contract_address +
                    ":" + o.token_id +
                    ":" + std::to_string(o.timestamp.sec_since_epoch());

   string recovered_address = recover_eth_address(message, o.eth_signature);

   FC_ASSERT(to_lowercase(recovered_address) == to_lowercase(o.eth_address),
             "Signature verification failed");

   // Create or update proof object
   const auto& proof_idx = _db.get_index<witness_nft_proof_index>()
                               .indices().get<by_witness>();
   auto proof_itr = proof_idx.find(o.witness_account);

   if (proof_itr == proof_idx.end())
   {
      _db.create<witness_nft_proof_object>([&](witness_nft_proof_object& p)
      {
         p.witness_account = o.witness_account;
         p.eth_address = o.eth_address;
         p.contract_address = o.contract_address;
         p.token_id = o.token_id;
         p.chain_id = o.chain_id;
         p.proof_timestamp = o.timestamp;
         p.verification_status = witness_nft_proof_object::pending;
         p.verification_details = "Pending oracle verification";
      });
   }
   else
   {
      _db.modify(*proof_itr, [&](witness_nft_proof_object& p)
      {
         p.eth_address = o.eth_address;
         p.contract_address = o.contract_address;
         p.token_id = o.token_id;
         p.chain_id = o.chain_id;
         p.proof_timestamp = o.timestamp;
         p.verification_status = witness_nft_proof_object::pending;
         p.verification_details = "Pending oracle verification";
      });
   }

   // Emit signal for oracle to pick up
   _db.notify_nft_proof_submitted(o);
}
```

**Ethereum Signature Recovery**:

```cpp
string recover_eth_address(const string& message, const string& signature)
{
   // EIP-191 personal_sign format
   string prefixed = "\x19Ethereum Signed Message:\n" +
                     std::to_string(message.length()) +
                     message;

   // Keccak256 hash
   fc::sha256 hash = fc::keccak256(prefixed.data(), prefixed.size());

   // Parse signature: r (32 bytes) + s (32 bytes) + v (1 byte)
   auto sig_bytes = fc::from_hex(signature.substr(2));  // Remove 0x
   FC_ASSERT(sig_bytes.size() == 65, "Invalid signature length");

   // Extract components
   fc::ecc::compact_signature compact_sig;
   memcpy(compact_sig.data, sig_bytes.data(), 65);

   // Recover public key
   fc::ecc::public_key pub_key = fc::ecc::public_key(compact_sig, hash);

   // Derive Ethereum address (last 20 bytes of keccak256(pubkey))
   auto pub_data = pub_key.serialize();
   fc::sha256 pub_hash = fc::keccak256(pub_data.data() + 1, 64);  // Skip first byte

   // Take last 20 bytes
   string eth_addr = "0x" + fc::to_hex(pub_hash.data() + 12, 20);

   return eth_addr;
}
```

**Database Object**: `witness_nft_proof_object`

```cpp
class witness_nft_proof_object : public object<witness_nft_proof_object_type, witness_nft_proof_object>
{
   witness_nft_proof_id_type  id;

   account_name_type          witness_account;
   string                     eth_address;
   string                     contract_address;
   string                     token_id;
   uint32_t                   chain_id;

   time_point_sec             proof_timestamp;
   time_point_sec             verification_timestamp;

   enum verification_status_type {
      pending,     // Awaiting oracle verification
      verified,    // Oracle confirmed ownership
      failed,      // Oracle verification failed
      expired      // Proof too old
   };

   uint8_t                    verification_status;
   string                     verification_details;
};

typedef multi_index_container<
   witness_nft_proof_object,
   indexed_by<
      ordered_unique<tag<by_id>, member<...>>,
      ordered_non_unique<tag<by_witness>, member<...>>,
      ordered_unique<tag<by_token_identity>,
         composite_key<witness_nft_proof_object,
            member<..., string, &witness_nft_proof_object::contract_address>,
            member<..., string, &witness_nft_proof_object::token_id>,
            member<..., uint32_t, &witness_nft_proof_object::chain_id>
         >
      >,
      ordered_non_unique<tag<by_verification_status>,
         composite_key<witness_nft_proof_object,
            member<..., uint8_t, &witness_nft_proof_object::verification_status>,
            member<..., time_point_sec, &witness_nft_proof_object::proof_timestamp>
         >
      >
   >
> witness_nft_proof_index;
```

#### 3. Witness Schedule Validation

Block production eligibility requires valid NFT proof.

**File**: `libraries/chain/database.cpp`

```cpp
bool database::validate_witness_nft_ownership(const account_name_type& witness) const
{
   // Get witness NFT proof
   const auto& proof_idx = get_index<witness_nft_proof_index>()
                              .indices().get<by_witness>();
   auto proof_itr = proof_idx.find(witness);

   if (proof_itr == proof_idx.end())
      return false;  // No proof submitted

   // Check verification status
   if (proof_itr->verification_status != witness_nft_proof_object::verified)
      return false;  // Not verified by oracle

   // Check expiration (proofs valid for 7 days)
   auto now = head_block_time();
   if (now - proof_itr->proof_timestamp > fc::days(7))
      return false;  // Proof expired

   return true;
}

void database::update_witness_schedule()
{
   // ... existing witness schedule logic ...

   // Filter witnesses without valid NFT proofs
   auto& schedule = get_witness_schedule_object();
   auto& active_witnesses = schedule.current_shuffled_witnesses;

   active_witnesses.erase(
      std::remove_if(active_witnesses.begin(), active_witnesses.end(),
         [this](const account_name_type& witness) {
            return !validate_witness_nft_ownership(witness);
         }
      ),
      active_witnesses.end()
   );

   FC_ASSERT(!active_witnesses.empty(),
             "No witnesses with valid NFT ownership for block production");

   // ... rest of scheduling logic ...
}
```

#### 4. Oracle Service Integration

Separate ZEP (ZEP-3) defines oracle plugin specification. Key requirements:

- **Verification Method**: Call ERC-721 `ownerOf(tokenId)` on target chain
- **RPC Endpoints**: Configurable Ethereum/Polygon/BSC node endpoints
- **Async Processing**: Non-blocking verification in background threads
- **Status Updates**: Update `witness_nft_proof_object` after verification
- **Error Handling**: Mark proofs as `failed` with detailed error messages

**Oracle Verification Flow**:

```
1. Witness submits proof operation
2. Proof object created with status=pending
3. Oracle plugin receives notification
4. Oracle queries EVM chain via RPC:
   - Call: nft_contract.ownerOf(token_id)
   - Wait for confirmations (e.g., 12 blocks)
5. Oracle compares returned owner with claimed eth_address
6. Oracle updates proof object:
   - Match: status=verified
   - Mismatch: status=failed, details="Owner is 0x..."
   - Error: status=failed, details=error message
```

### API Methods

```cpp
// Get witness NFT proof status
struct get_witness_nft_proof_return
{
   bool                              has_proof;
   optional<witness_nft_proof_object> proof;
};

get_witness_nft_proof_return get_witness_nft_proof(account_name_type witness);

// List registered NFT collections
struct list_nft_collections_return
{
   vector<nft_collection_object> collections;
};

list_nft_collections_return list_nft_collections(bool active_only);

// Get collection approval status
struct get_collection_approvals_return
{
   vector<account_name_type> approved_by;
   vector<account_name_type> rejected_by;
   bool is_active;
   uint32_t approvals_needed;
};

get_collection_approvals_return get_collection_approvals(uint64_t collection_id);
```

## Rationale

### Design Decisions

#### 1. ERC-721 Standard

**Decision**: Use ERC-721 instead of custom NFT format

**Reasons**:
- Industry standard with broad adoption
- Extensive tooling and wallet support
- Proven security and audited contracts
- Large existing NFT marketplaces

#### 2. Multi-Chain Support

**Decision**: Support multiple EVM-compatible chains

**Reasons**:
- Ethereum gas fees may be prohibitive
- L2 solutions (Polygon, Arbitrum) more accessible
- Different chains have different NFT communities
- Flexibility for future expansion

#### 3. Oracle Architecture

**Decision**: Off-chain oracle instead of on-chain bridge

**Reasons**:
- Simpler implementation (no bridge security)
- Lower costs (no bridge token fees)
- Faster verification (no bridge latency)
- Easier to upgrade and maintain

**Trade-off**: Trust in oracle operators (mitigated by witness control)

#### 4. Proof Expiration (7 Days)

**Decision**: Require proof renewal every 7 days

**Reasons**:
- Detects NFT transfers (owner changes)
- Prevents stale proofs after NFT sale
- Reasonable witness operational burden
- Weekly maintenance cycle is standard

#### 5. Witness Consensus for Collections

**Decision**: Require majority witness approval for collections

**Reasons**:
- Prevents spam collection registrations
- Ensures legitimate NFT projects only
- Decentralized governance
- Aligns with existing witness authority

#### 6. Signature-Based Proof

**Decision**: Use Ethereum signatures instead of cross-chain messages

**Reasons**:
- No bridge infrastructure required
- Works with any wallet (MetaMask, etc.)
- Provably links Zattera account to ETH address
- Cost-effective (one signature per week)

### Alternative Approaches Considered

#### A. Cross-Chain Bridge

**Idea**: Use IBC or native bridge for verification

**Rejected because**:
- Complex infrastructure
- High operational costs
- Significant attack surface
- Slow verification times

#### B. Snapshot-Based Verification

**Idea**: Periodic snapshots of NFT holders

**Rejected because**:
- Not real-time (stale data)
- Centralized snapshot authority
- Difficult to coordinate timing
- No proof of continuous ownership

#### C. NFT Staking

**Idea**: Require locking NFTs in smart contract

**Rejected because**:
- Requires custom contracts on each chain
- Liquidity concerns for witnesses
- Smart contract risks
- Complexity of multi-chain staking

**Future**: Could be added as enhancement

## Backwards Compatibility

This ZEP introduces consensus-breaking changes requiring a hard fork.

### Compatibility Analysis

**Witness Operations**:
- ⚠️ Existing witnesses continue operating until hardfork
- ⚠️ Post-hardfork: All witnesses must submit NFT proofs
- ⚠️ Grace period recommended (e.g., 30 days)

**Database Schema**:
- ⚠️ New object types added
- ⚠️ Witness schedule logic modified
- ⚠️ Shared memory file format changes

**APIs**:
- ✅ New APIs added (non-breaking)
- ✅ Existing witness queries unchanged

### Migration Path

**Phase 1: Pre-Hardfork Preparation (T-30 days)**
1. Deploy oracle plugin to all witness nodes
2. Configure EVM RPC endpoints
3. Witnesses register and approve initial NFT collections
4. Test verification on testnet

**Phase 2: Soft Activation (T-0)**
1. Hardfork enables new operations
2. Witnesses voluntarily submit proofs
3. Proofs are verified but NOT enforced
4. Monitor verification success rates

**Phase 3: Hard Enforcement (T+30 days)**
1. If >90% witnesses have verified proofs:
   - Enable witness schedule filtering
   - Witnesses without proofs excluded from blocks
2. If <90%:
   - Extend grace period
   - Investigate issues

**Emergency Rollback**:
- Disable NFT requirement via emergency hardfork
- Revert to traditional witness scheduling
- Investigate and fix issues
- Re-enable in future hardfork

## Test Cases

### Test Case 1: Collection Registration

```cpp
BOOST_AUTO_TEST_CASE(nft_collection_registration)
{
   ACTORS((witness1)(witness2))

   witness_create("witness1", ...);
   witness_create("witness2", ...);

   // Witness1 proposes collection
   nft_collection_register_operation op;
   op.proposer = "witness1";
   op.collection_name = "Test NFT Collection";
   op.contract_address = "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D";
   op.chain_id = 1;  // Ethereum mainnet
   op.min_nft_count = 1;

   push_transaction(op, witness1_key);

   // Verify collection created but not active
   const auto& idx = db->get_index<nft_collection_index>()
                         .indices().get<by_collection_name>();
   auto coll = idx.find("Test NFT Collection");
   BOOST_REQUIRE(coll != idx.end());
   BOOST_REQUIRE(!coll->active);
   BOOST_REQUIRE_EQUAL(coll->approved_by.size(), 1);  // Auto-approved by proposer
}
```

### Test Case 2: Collection Approval Consensus

```cpp
BOOST_AUTO_TEST_CASE(nft_collection_approval)
{
   // Create 21 witnesses
   ACTORS((w1)(w2)...(w21))

   for (auto w : witnesses)
      witness_create(w, ...);

   // Register collection
   nft_collection_register_operation reg_op;
   reg_op.proposer = "w1";
   reg_op.collection_name = "BAYC";
   reg_op.contract_address = "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D";
   reg_op.chain_id = 1;
   push_transaction(reg_op, w1_key);

   auto coll_id = /* get collection id */;

   // Initially not active
   BOOST_REQUIRE(!get_collection(coll_id).active);

   // 10 more witnesses approve (total 11/21)
   for (int i = 2; i <= 11; i++)
   {
      nft_collection_approve_operation approve_op;
      approve_op.witness_account = witnesses[i];
      approve_op.collection_id = coll_id;
      approve_op.approve = true;
      push_transaction(approve_op, witness_keys[i]);
   }

   // Now active (11/21 >= majority)
   BOOST_REQUIRE(get_collection(coll_id).active);
}
```

### Test Case 3: NFT Proof Submission

```cpp
BOOST_AUTO_TEST_CASE(witness_nft_proof_submission)
{
   ACTORS((alice))
   witness_create("alice", ...);

   // Register collection first
   // ...

   // Alice generates Ethereum signature
   string message = "zattera:alice:nft:0xBC4C...:1234:1704672000";
   string signature = sign_ethereum_message(alice_eth_key, message);

   // Submit proof
   witness_nft_proof_operation proof_op;
   proof_op.witness_account = "alice";
   proof_op.eth_address = "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb";
   proof_op.contract_address = "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D";
   proof_op.token_id = "1234";
   proof_op.chain_id = 1;
   proof_op.eth_signature = signature;
   proof_op.timestamp = fc::time_point_sec(1704672000);

   push_transaction(proof_op, alice_key);

   // Verify proof created with pending status
   const auto& proof_idx = db->get_index<witness_nft_proof_index>()
                               .indices().get<by_witness>();
   auto proof = proof_idx.find("alice");
   BOOST_REQUIRE(proof != proof_idx.end());
   BOOST_REQUIRE_EQUAL(proof->verification_status,
                      witness_nft_proof_object::pending);
}
```

### Test Case 4: Signature Verification

```cpp
BOOST_AUTO_TEST_CASE(ethereum_signature_recovery)
{
   string message = "zattera:alice:nft:0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D:1234:1704672000";

   // Sign with known Ethereum private key
   string eth_privkey = "0x1234...";
   string expected_addr = "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb";

   string signature = sign_with_ethereum_key(eth_privkey, message);

   // Recover address from signature
   string recovered = recover_eth_address(message, signature);

   BOOST_REQUIRE_EQUAL(to_lowercase(recovered),
                      to_lowercase(expected_addr));
}
```

### Test Case 5: Witness Schedule Filtering

```cpp
BOOST_AUTO_TEST_CASE(witness_schedule_nft_filtering)
{
   ACTORS((alice)(bob)(charlie))

   // Create witnesses
   witness_create("alice", ...);
   witness_create("bob", ...);
   witness_create("charlie", ...);

   // Alice has verified NFT proof
   create_verified_nft_proof("alice", ...);

   // Bob has expired proof
   create_expired_nft_proof("bob", ...);

   // Charlie has no proof

   // Update witness schedule
   db->update_witness_schedule();

   const auto& schedule = db->get_witness_schedule_object();
   const auto& active = schedule.current_shuffled_witnesses;

   // Only Alice should be in schedule
   BOOST_REQUIRE(std::find(active.begin(), active.end(), "alice") != active.end());
   BOOST_REQUIRE(std::find(active.begin(), active.end(), "bob") == active.end());
   BOOST_REQUIRE(std::find(active.begin(), active.end(), "charlie") == active.end());
}
```

### Test Case 6: Proof Expiration

```cpp
BOOST_AUTO_TEST_CASE(nft_proof_expiration)
{
   ACTORS((alice))
   witness_create("alice", ...);

   // Create verified proof
   create_verified_nft_proof("alice", ...);

   // Initially valid
   BOOST_REQUIRE(db->validate_witness_nft_ownership("alice"));

   // Fast-forward 8 days
   generate_blocks(db->head_block_time() + fc::days(8));

   // Now expired (7 day limit)
   BOOST_REQUIRE(!db->validate_witness_nft_ownership("alice"));
}
```

## Reference Implementation

Available in feature branch:

- Branch: `feature/nft-witness-verification`
- Key Files:
  - `libraries/protocol/include/zattera/protocol/zattera_operations.hpp`
  - `libraries/chain/include/zattera/chain/nft_verification_objects.hpp`
  - `libraries/chain/nft_evaluator.cpp`
  - `libraries/chain/database.cpp` (witness schedule)
  - `libraries/plugins/nft_oracle/` (See ZEP-3)
  - `tests/tests/nft_witness_tests.cpp`

## Security Considerations

### Cryptographic Security

**Signature Verification**
- Risk: Signature forgery
- Mitigation: Use battle-tested secp256k1 recovery
- Libraries: fc::ecc, libsecp256k1

**Replay Attacks**
- Risk: Reuse old signatures
- Mitigation: Include timestamp in signed message
- Validation: Reject proofs >24 hours old

### Oracle Security

**Oracle Compromise**
- Risk: Malicious oracle approves fake proofs
- Mitigation:
  - Oracle controlled by witnesses
  - Open-source oracle code
  - Multiple oracle verification (future)
  - Monitor for anomalous approvals

**RPC Endpoint Manipulation**
- Risk: Fake RPC returns false ownership data
- Mitigation:
  - Use trusted RPC providers
  - Multiple RPC endpoints
  - Cross-verify with block explorers

**Chain Reorganization**
- Risk: NFT ownership changes in reorg
- Mitigation: Wait for confirmation depth (e.g., 12 blocks)

### Economic Security

**NFT Transfer Attacks**
- Risk: Briefly own NFT, submit proof, transfer
- Mitigation:
  - 7-day proof validity requires continuous ownership
  - Frequent re-verification detects transfers
  - NFT staking (future enhancement)

**Witness Collusion**
- Risk: Malicious witnesses approve fake collections
- Mitigation:
  - Requires 11/21 majority
  - Public justification and voting
  - Community oversight
  - Governance mechanism for removal

**DoS Attacks**
- Risk: Spam proof submissions
- Mitigation:
  - Rate limiting per witness
  - Operation fees
  - Only active witnesses can submit

### Operational Security

**Key Management**
- Risk: Ethereum private key compromise
- Mitigation:
  - Separate keys for NFT ownership and block signing
  - Hardware wallet support
  - Key rotation procedures

**RPC Availability**
- Risk: RPC endpoints go offline
- Mitigation:
  - Fallback endpoints
  - Local archive nodes
  - Graceful degradation

## Future Enhancements

### 1. Multi-NFT Requirements

Require witnesses to hold NFTs from multiple collections:

```cpp
struct multi_nft_requirement
{
   vector<nft_collection_id_type> required_collections;
   uint32_t min_collections;  // "X out of Y" logic
};
```

### 2. NFT Staking

Lock NFTs in smart contract for enhanced security:

```cpp
struct nft_staking_contract
{
   string contract_address;
   uint32_t min_stake_duration;  // Minimum lock period
};
```

### 3. Cross-Chain Bridges

Use native bridges instead of oracle:

```cpp
struct bridge_verification
{
   string bridge_contract;
   uint32_t confirmation_blocks;
};
```

### 4. Dynamic NFT Values

Weight NFTs differently based on rarity, price, etc.:

```cpp
struct nft_weight_criteria
{
   asset min_floor_price;
   map<string, uint32_t> trait_weights;
};
```

### 5. Delegated Ownership

Allow NFT owners to delegate witness rights:

```cpp
struct nft_delegation_operation
{
   string nft_owner_eth_address;
   account_name_type delegated_witness;
   time_point_sec expiration;
   string eth_signature;
};
```

### 6. Multi-Oracle Consensus

Require multiple oracles to agree:

```cpp
struct oracle_consensus_config
{
   uint32_t min_oracle_agreements;  // e.g., 3 out of 5
   vector<account_name_type> trusted_oracles;
};
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
