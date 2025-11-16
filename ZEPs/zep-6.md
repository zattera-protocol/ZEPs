---
zep: 6
title: Comment Permission Control
description: Allow post authors to specify whitelists of accounts permitted to comment on their posts
author: Zattera Core Team
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2025-11-11
---

## Abstract

This ZEP introduces a permission control mechanism that allows post authors to restrict who can comment on their content. Authors can specify one of three modes: open comments (default), restricted to specific whitelisted accounts, or no comments allowed. This feature enables better content moderation and spam prevention at the protocol level.

## Motivation

Current blockchain social platforms provide no native mechanism for content creators to control who can reply to their posts. This limitation creates several problems:

1. **Spam and harassment**: Authors cannot prevent unwanted comments from spam accounts or harassers
2. **Quality control**: No way to limit discussions to verified or trusted participants
3. **Private discussions**: Cannot create posts for selective group discussions
4. **Content curation**: Authors lack tools to maintain comment quality standards
5. **DApp business model constraints**: No mechanism to build premium content or membership-based communities

Off-chain solutions (UI filtering, muting) are inadequate because:
- Comments still exist on-chain and consume resources
- Different UIs show different comment sets (inconsistent experience)
- Spam still affects reputation and discovery algorithms
- No economic disincentive for spam comments

A protocol-level solution provides:
- Consistent enforcement across all interfaces
- True comment prevention (not just hiding)
- Reduced blockchain bloat from spam
- Author sovereignty over their content space
- DApp business model implementation

### DApp Business Model Use Cases

This feature enables various business models to be implemented at the blockchain level:

1. **Premium Communities**
   - Discussion spaces accessible only to subscribers or token holders
   - Membership-based content platforms
   - Example: Educational platforms where only enrolled students can ask questions

2. **Verified User Spaces**
   - Allow comments only from KYC-verified users
   - Professional networks or industry-specific forums
   - Example: Medical paper discussions accessible only to verified healthcare professionals

3. **Customer Support and Services**
   - Product buyers can write reviews and questions
   - Customer-only support threads
   - Example: NFT holders can ask project-related questions

4. **Tiered Communities**
   - Differential access based on staking level or reputation
   - DAO membership-based governance discussions
   - Example: Proposal comments limited to specific token holders

5. **Private Group Collaboration**
   - Internal team project discussions
   - Private communication with clients
   - Example: Enterprise customer-exclusive update feeds

This functionality allows DApp developers to implement Web2-style membership models on blockchain while maintaining decentralization and transparency.

## Specification

### Comment Permission Modes

Three modes are supported:

1. **Open (default)**: Anyone can comment
   - Field: `allowed_comment_accounts` not set (`nullopt`)
   - Behavior: Existing behavior, no restrictions

2. **Whitelist**: Only specified accounts can comment
   - Field: `allowed_comment_accounts` set with account list
   - Behavior: Only listed accounts can create replies

3. **Disabled**: No comments allowed
   - Field: `allowed_comment_accounts` set to empty set
   - Behavior: All comment attempts fail

### Protocol Changes

#### 1. Comment Operation Extension

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

   // NEW: Optional comment permission control
   optional<flat_set<account_name_type>> allowed_comment_accounts;

   extensions_type   extensions;

   void validate() const;
};
```

**Validation Logic**:

```cpp
void comment_operation::validate() const
{
   // ... existing validation ...

   if (allowed_comment_accounts.valid())
   {
      // Validate each account name
      for (const auto& account : *allowed_comment_accounts)
      {
         FC_ASSERT(is_valid_account_name(account),
                   "Invalid account name: ${a}", ("a", account));
      }

      // Prevent abuse with excessively large whitelists
      FC_ASSERT(allowed_comment_accounts->size() <= ZATTERA_MAX_COMMENT_WHITELIST_SIZE,
                "Whitelist cannot exceed ${max} accounts",
                ("max", ZATTERA_MAX_COMMENT_WHITELIST_SIZE));
   }
}
```

**Protocol Constant**:
```cpp
#define ZATTERA_MAX_COMMENT_WHITELIST_SIZE 1000
```

#### 2. Comment Object Extension

```cpp
class comment_object : public object<comment_object_type, comment_object>
{
   comment_id_type   id;

   string            author;
   string            permlink;
   // ... existing fields ...

   // NEW: Comment permission control
   // - nullopt = open to all
   // - empty vector = no comments allowed
   // - non-empty vector = whitelist of allowed accounts
   optional<shared_vector<account_name_type>> allowed_comment_accounts;

   // ... rest of object ...
};
```

**Note**: Use `shared_vector` instead of `flat_set` because chainbase requires allocator-aware containers for memory-mapped storage.

#### 3. Comment Evaluator Changes

**File**: `libraries/chain/zattera_evaluator.cpp`

```cpp
void comment_evaluator::do_apply(const comment_operation& o)
{
   // If this is a reply (not a root post), check permissions
   if (o.parent_author != ZATTERA_ROOT_POST_PARENT)
   {
      const auto& parent = _db.get_comment(o.parent_author, o.parent_permlink);

      // Check if parent has comment restrictions
      if (parent.allowed_comment_accounts.valid())
      {
         const auto& allowed = *parent.allowed_comment_accounts;

         if (allowed.empty())
         {
            // Empty whitelist = no comments allowed
            FC_ASSERT(false,
                     "Comments are disabled for this post");
         }
         else
         {
            // Check if commenter is in whitelist
            auto it = std::find(allowed.begin(), allowed.end(), o.author);
            FC_ASSERT(it != allowed.end(),
                     "Account ${a} is not allowed to comment on this post",
                     ("a", o.author));
         }
      }
   }
   else  // Root post
   {
      // Set comment permissions if specified
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

   // ... rest of existing comment creation logic ...
}
```

### Behavior Specifications

#### Permission Inheritance

**Rule**: Comment permission settings do NOT inherit to child comments.

**Rationale**:
- Each comment is independently controlled by its author
- Prevents confusing nested permission scenarios
- Keeps implementation simple

**Example**:
```
Post (Alice, whitelist: [Bob, Charlie])
  ├─ Comment 1 (Bob, open to all)
  │    └─ Comment 1.1 (Dan, allowed because Bob's comment is open)
  └─ Comment 2 (Charlie, no comments allowed)
       └─ (No replies possible)
```

#### Permission Immutability

**Rule**: Comment permissions CANNOT be changed after post creation.

**Rationale**:
- Prevents bait-and-switch (allow comments, then lock)
- Simpler implementation (no update logic needed)
- Clear expectations for commenters

**Future Enhancement**: Separate update operation could be added in future ZEP.

#### Edit Behavior

**Rule**: Editing post body/title does NOT affect comment permissions.

**Implementation**: Permission field only set during initial creation, ignored on updates.

```cpp
// In evaluator
if (comment == nullptr)  // Creating new post
{
   // Set allowed_comment_accounts
}
else  // Editing existing post
{
   // Do NOT modify allowed_comment_accounts
}
```

### API Extensions

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
 * Returns comment permission settings for a post
 *
 * @param author Post author
 * @param permlink Post permlink
 * @returns Permission information
 */
get_comment_permissions_return get_comment_permissions(
   get_comment_permissions_args args);
```

**Implementation**:

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
         result.comments_enabled = true;  // Open to all
      }

      return result;
   });
}
```

## Rationale

### Design Decisions

#### 1. Whitelist vs Blacklist

**Decision**: Use whitelist (allow list) instead of blacklist (deny list)

**Reasons**:
- **Scalability**: Whitelists typically smaller than blacklists
- **Privacy**: Blacklists publicly expose conflicts/harassers
- **Sybil resistance**: Blacklists ineffective against new spam accounts
- **Gas efficiency**: Smaller data structure to store/check

#### 2. Size Limit (1000 accounts)

**Decision**: Maximum 1000 accounts per whitelist

**Reasons**:
- **DoS prevention**: Prevents excessively large whitelists
- **Performance**: Linear search O(n) acceptable for n ≤ 1000
- **Practicality**: Covers legitimate use cases (1000+ person discussions rare)
- **Gas cost**: Limits transaction size and validation time

**Alternative considered**: No limit - rejected due to DoS concerns

#### 3. Immutable After Creation

**Decision**: Permissions cannot be changed after post creation

**Reasons**:
- **Simplicity**: No need for separate update operation
- **Trust**: Commenters know rules won't change mid-discussion
- **Compatibility**: Avoids complex migration logic

**Trade-off**: Less flexibility for authors who want to adjust later

**Future**: Could add update operation in later ZEP if demand exists

#### 4. No Nested Restrictions

**Decision**: Child comments controlled independently

**Reasons**:
- **Autonomy**: Comment authors control their sub-threads
- **Simplicity**: Avoids complex permission resolution
- **Flexibility**: Allows open discussion under restricted posts

**Example**: Bob (whitelisted) can allow anyone to reply to his comment on Alice's restricted post

#### 5. Protocol-Level vs. Application-Level

**Decision**: Enforce at protocol level, not just UI

**Reasons**:
- **Consistency**: All clients enforce same rules
- **Resource efficiency**: Prevents spam from reaching blockchain
- **Economic incentive**: Failed transactions waste spammer resources
- **True enforcement**: Cannot be bypassed by alternative UIs

#### 6. Optional Field

**Decision**: Use `optional<>` instead of always-present field

**Reasons**:
- **Backwards compatibility**: Existing posts unchanged
- **Storage efficiency**: Only pay for feature when used
- **Default behavior**: Null = open (matches existing behavior)

### Alternative Approaches Considered

#### A. Role-Based Permissions

**Idea**: Allow comments from accounts with certain criteria (reputation, stake, etc.)

**Rejected because**:
- More complex implementation
- Criteria can be gameable
- Still need whitelist for fine-grained control

**Future**: Could be separate ZEP building on this one

#### B. Reputation Threshold

**Idea**: Minimum reputation required to comment

**Rejected because**:
- Disadvantages new legitimate users
- Reputation systems vary across implementations
- Not flexible enough for private discussions

**Future**: Could combine with whitelist (whitelist OR reputation > N)

#### C. Token-Gated Comments

**Idea**: Require holding specific tokens to comment

**Rejected because**:
- Requires token system integration
- Too complex for initial implementation
- Overlaps with SMT functionality

**Future**: Possible extension for token-gated communities

## Backwards Compatibility

This ZEP introduces a consensus-breaking change requiring a hard fork.

### Compatibility Analysis

**Existing Operations**:
- ✅ Old `comment_operation` without `allowed_comment_accounts` remains valid
- ✅ Null/absent field treated as "open to all"
- ✅ No changes to existing comment display

**Database Schema**:
- ⚠️ Requires database migration to add optional field
- ⚠️ Shared memory file format changes (requires replay)

**APIs**:
- ✅ Existing API calls continue working
- ✅ New fields added to responses (backwards compatible)
- ✅ Clients ignoring new fields see default behavior

### Migration Path

**Pre-Hardfork**:
1. All posts are open to comments
2. `allowed_comment_accounts` field does not exist

**Hardfork Activation**:
1. Database schema updated
2. Existing posts: `allowed_comment_accounts = nullopt` (open)
3. New operations can include permission field

**Post-Hardfork**:
1. Authors can create restricted posts
2. Old posts remain open unless recreated
3. APIs return permission information

**No Breaking Changes For**:
- Users who don't use the feature
- Existing posts and comments
- Read-only API consumers
- Wallets that ignore the field

## Test Cases

### Test Case 1: Open Comments (Default)

```cpp
BOOST_AUTO_TEST_CASE(comment_permission_open_default)
{
   ACTORS((alice)(bob))

   // Alice creates post without permission field
   comment_operation post;
   post.parent_author = "";
   post.author = "alice";
   post.permlink = "test-post";
   post.title = "Test";
   post.body = "Content";
   // allowed_comment_accounts not set

   push_transaction(post, alice_private_key);

   // Bob can comment
   comment_operation reply;
   reply.parent_author = "alice";
   reply.parent_permlink = "test-post";
   reply.author = "bob";
   reply.permlink = "bobs-reply";
   reply.body = "Nice post!";

   // Should succeed
   push_transaction(reply, bob_private_key);

   const auto& comment = db->get_comment("bob", "bobs-reply");
   BOOST_REQUIRE(comment.parent_author == "alice");
}
```

### Test Case 2: Whitelist - Allowed Account

```cpp
BOOST_AUTO_TEST_CASE(comment_permission_whitelist_allowed)
{
   ACTORS((alice)(bob)(charlie))

   // Alice creates post with whitelist
   comment_operation post;
   post.parent_author = "";
   post.author = "alice";
   post.permlink = "restricted-post";
   post.title = "Restricted";
   post.body = "Only bob and charlie can comment";
   post.allowed_comment_accounts = flat_set<account_name_type>{"bob", "charlie"};

   push_transaction(post, alice_private_key);

   // Bob (in whitelist) can comment
   comment_operation bob_reply;
   bob_reply.parent_author = "alice";
   bob_reply.parent_permlink = "restricted-post";
   bob_reply.author = "bob";
   bob_reply.permlink = "bobs-reply";
   bob_reply.body = "Thanks for including me!";

   // Should succeed
   push_transaction(bob_reply, bob_private_key);

   const auto& comment = db->get_comment("bob", "bobs-reply");
   BOOST_REQUIRE(comment.id._id > 0);
}
```

### Test Case 3: Whitelist - Rejected Account

```cpp
BOOST_AUTO_TEST_CASE(comment_permission_whitelist_rejected)
{
   ACTORS((alice)(bob)(dan))

   // Alice creates post with whitelist [bob]
   comment_operation post;
   post.parent_author = "";
   post.author = "alice";
   post.permlink = "restricted-post";
   post.allowed_comment_accounts = flat_set<account_name_type>{"bob"};

   push_transaction(post, alice_private_key);

   // Dan (not in whitelist) tries to comment
   comment_operation dan_reply;
   dan_reply.parent_author = "alice";
   dan_reply.parent_permlink = "restricted-post";
   dan_reply.author = "dan";
   dan_reply.permlink = "dans-reply";
   dan_reply.body = "Can I comment?";

   // Should fail
   ZATTERA_REQUIRE_THROW(
      push_transaction(dan_reply, dan_private_key),
      fc::exception
   );
}
```

### Test Case 4: No Comments Allowed

```cpp
BOOST_AUTO_TEST_CASE(comment_permission_disabled)
{
   ACTORS((alice)(bob))

   // Alice creates post with empty whitelist
   comment_operation post;
   post.parent_author = "";
   post.author = "alice";
   post.permlink = "no-comments";
   post.allowed_comment_accounts = flat_set<account_name_type>();  // Empty

   push_transaction(post, alice_private_key);

   // Bob tries to comment
   comment_operation bob_reply;
   bob_reply.parent_author = "alice";
   bob_reply.parent_permlink = "no-comments";
   bob_reply.author = "bob";
   bob_reply.permlink = "bobs-reply";
   bob_reply.body = "Testing...";

   // Should fail
   ZATTERA_REQUIRE_THROW(
      push_transaction(bob_reply, bob_private_key),
      fc::exception
   );
}
```

### Test Case 5: Whitelist Size Limit

```cpp
BOOST_AUTO_TEST_CASE(comment_permission_size_limit)
{
   ACTORS((alice))

   // Create whitelist with 1001 accounts (exceeds limit)
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

   // Should fail validation
   BOOST_REQUIRE_THROW(
      post.validate(),
      fc::exception
   );
}
```

### Test Case 6: Invalid Account Name

```cpp
BOOST_AUTO_TEST_CASE(comment_permission_invalid_account)
{
   comment_operation post;
   post.parent_author = "";
   post.author = "alice";
   post.permlink = "test";
   post.allowed_comment_accounts = flat_set<account_name_type>{"invalid.name!"};

   // Should fail validation
   BOOST_REQUIRE_THROW(
      post.validate(),
      fc::exception
   );
}
```

### Test Case 7: Child Comment Independence

```cpp
BOOST_AUTO_TEST_CASE(comment_permission_child_independence)
{
   ACTORS((alice)(bob)(charlie))

   // Alice creates restricted post [bob only]
   comment_operation post;
   post.author = "alice";
   post.allowed_comment_accounts = flat_set<account_name_type>{"bob"};
   push_transaction(post, alice_private_key);

   // Bob comments (allowed) with open permissions
   comment_operation bob_reply;
   bob_reply.parent_author = "alice";
   bob_reply.author = "bob";
   // No allowed_comment_accounts = open to all
   push_transaction(bob_reply, bob_private_key);

   // Charlie replies to Bob's comment (should succeed)
   comment_operation charlie_reply;
   charlie_reply.parent_author = "bob";
   charlie_reply.author = "charlie";
   push_transaction(charlie_reply, charlie_private_key);

   // Should succeed because Bob's comment is open
   const auto& comment = db->get_comment("charlie", charlie_reply.permlink);
   BOOST_REQUIRE(comment.parent_author == "bob");
}
```

## Reference Implementation

Available in feature branch:

- Branch: `feature/comment-permission-control`
- Key Files:
  - `libraries/protocol/include/zattera/protocol/zattera_operations.hpp`
  - `libraries/chain/include/zattera/chain/comment_object.hpp`
  - `libraries/chain/zattera_evaluator.cpp`
  - `libraries/plugins/apis/database_api/database_api.cpp`
  - `tests/tests/comment_permission_tests.cpp`

## Security Considerations

### Potential Vulnerabilities

#### 1. DoS via Large Whitelists

**Risk**: Attackers create posts with maximum-size whitelists

**Impact**:
- Increased transaction size
- Slower validation (O(n) lookup)
- Higher storage costs

**Mitigation**:
- Hard limit: 1000 accounts max
- Linear search acceptable for n ≤ 1000
- Consider binary search optimization if needed

**Monitoring**: Track average whitelist size distribution

#### 2. Spam Account Creation

**Risk**: Spammers create new accounts to bypass blacklists

**Impact**: Whitelist approach already addresses this

**Mitigation**: Whitelist model inherently resistant to Sybil attacks

#### 3. Permission Verification Bypass

**Risk**: Bug in evaluator allows unauthorized comments

**Impact**: Feature completely broken

**Mitigation**:
- Comprehensive test coverage
- Audit by security experts
- Bug bounty program

**Testing**: Fuzz testing with various account combinations

#### 4. API Information Leakage

**Risk**: Privacy concerns with publicly visible whitelists

**Impact**: Reveals author's social graph

**Mitigation**: This is intentional and unavoidable
- Whitelists are public by nature of blockchain
- Users should be aware of privacy implications
- Future: encrypted whitelists (advanced feature)

### Economic Considerations

#### Storage Costs

- Small impact: Most posts won't use feature
- Whitelist users pay proportional RC costs
- Optional field = no cost when unused

#### Network Effects

- Positive: Reduces spam, improves quality
- Negative: May reduce engagement metrics
- Neutral: User choice (opt-in feature)

## Future Enhancements

Potential extensions for future ZEPs:

### 1. Permission Updates

Allow authors to modify permissions after creation:

```cpp
struct update_comment_permissions_operation
{
   account_name_type author;
   string            permlink;
   optional<flat_set<account_name_type>> new_allowed_accounts;
};
```

### 2. Role-Based Access

Combine whitelist with criteria:

```cpp
struct comment_permission_criteria
{
   optional<flat_set<account_name_type>> whitelist;
   optional<uint32_t> min_reputation;
   optional<asset> min_stake;
};
```

### 3. Delegation

Allow moderators to manage permissions:

```cpp
struct delegate_comment_moderation_operation
{
   account_name_type author;
   string            permlink;
   account_name_type moderator;
   time_point_sec    expiration;
};
```

### 4. Blacklist Mode

Complement whitelist with deny list:

```cpp
struct comment_permission_v2
{
   optional<flat_set<account_name_type>> allow_list;
   optional<flat_set<account_name_type>> deny_list;
   permission_mode mode;  // whitelist, blacklist, open, closed
};
```

### 5. Time-Based Restrictions

Lock comments after period:

```cpp
struct comment_operation
{
   // ... existing fields ...
   optional<time_point_sec> comment_lock_time;
};
```

### 6. NFT/Token Gating

Require token ownership:

```cpp
struct token_gated_comments
{
   string token_symbol;
   asset  minimum_balance;
};
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
