---
zep: 1
title: ZEP Purpose and Guidelines
description: Defines the Zattera Enhancement Proposal (ZEP) and Zattera Request for Comment (ZRC) process
author: Zattera Core Team
discussions-to: TBD
status: Living
type: Meta
created: 2025-11-08
---

## What is a ZEP?

ZEP stands for Zattera Enhancement Proposal. A ZEP is a design document providing information to the Zattera community, or describing a new feature for Zattera or its processes or environment. The ZEP should provide a concise technical specification of the feature and a rationale for the feature. The ZEP author is responsible for building consensus within the community and documenting dissenting opinions.

## What is a ZRC?

ZRC stands for Zattera Request for Comment. A ZRC is a subset of ZEPs focused on application-level standards and conventions. This includes token standards, plugin interfaces, wallet formats, API conventions, and other standards that don't require consensus changes.

The ZEP/ZRC system is inspired by Ethereum's EIP/ERC process, Bitcoin's BIP process, and Python's PEP process.

## ZEP Rationale

We intend ZEPs to be the primary mechanisms for proposing new features, for collecting community technical input on an issue, and for documenting the design decisions that have gone into Zattera. Because the ZEPs are maintained as text files in a versioned repository, their revision history is the historical record of the feature proposal.

For Zattera implementers, ZEPs are a convenient way to track the progress of their implementation. Ideally each implementation maintainer would list the ZEPs that they have implemented. This will give end users a convenient way to know the current status of a given implementation or library.

## ZEP Types

There are three types of ZEP:

- **Standards Track ZEP**: Describes any change that affects most or all Zattera implementations, such as:
  - A change to the network protocol
  - A change in block or transaction validity rules
  - Proposed application standards/conventions
  - Any change or addition that affects the interoperability of applications using Zattera

- **Meta ZEP**: Describes Zattera-related processes or proposes a change to (or an event in) a process. Meta ZEPs are like Standards Track ZEPs but apply to areas other than the Zattera protocol itself. Examples include procedures, guidelines, changes to the decision-making process, and changes to the tools or environment used in Zattera development.

- **Informational ZEP**: Describes a Zattera design issue, or provides general guidelines or information to the Zattera community, but does not propose a new feature. Informational ZEPs do not necessarily represent Zattera community consensus or a recommendation.

## Standards Track ZEP Categories

Standards Track ZEPs consist of three categories:

- **Core**: Improvements requiring a consensus fork (e.g., changes to operations, evaluators, consensus rules, reward algorithms)
- **Networking**: Improvements around network protocol specifications
- **Interface**: Improvements around client API/RPC specifications and standards, including database_api and plugin APIs

## ZRC vs ZEP

ZRCs are a subset of ZEPs. Specifically, **ZRCs are Standards Track ZEPs** that focus on application-level standards. The distinction is:

- **ZEPs**: Protocol-level changes requiring consensus (Core), network changes (Networking), or interface changes (Interface)
- **ZRCs**: Application-level standards that don't require protocol changes (token standards, plugin interfaces, data formats)

Examples of ZRCs:
- Plugin standard interfaces
- Wallet key derivation standards
- JSON metadata schemas
- URI schemes for Zattera resources

## ZEP Workflow

### 1. Idea Phase

Before submitting a ZEP, the idea should be thoroughly vetted. Ask the Zattera community first if an idea is original to avoid wasting time on something that will be rejected.

**Channels for discussion**:
- Zattera community forums
- Developer Discord/Telegram
- GitHub Discussions

### 2. Draft Phase

Once the idea has been vetted, the author should:

1. **Create ZEP document** using the template
2. **Submit pull request** to the ZEPs repository
3. **Get ZEP number** assigned by editors
4. Status: **Draft**

**Draft requirements**:
- Must follow the ZEP template format
- Must include all required sections
- Must be technically sound
- Must be properly formatted markdown

### 3. Review Phase

Once submitted, the ZEP enters community review:

1. **Community feedback** via GitHub comments and discussions
2. **Author revisions** based on feedback
3. **Technical review** by core developers
4. Status changes to **Review** when ready for wider consideration

### 4. Last Call Phase

When the author believes the ZEP is ready:

1. Editor moves status to **Last Call**
2. **Last Call deadline** set (typically 14 days)
3. Final opportunity for community feedback
4. Final revisions made

### 5. Final Phase

For a ZEP to move to **Final**:

- **Standards Track**: Must have reference implementation and pass tests
- **Meta/Informational**: Must have rough consensus
- No unresolved objections

Status: **Final**

### Other Statuses

- **Stagnant**: Inactive for 6+ months in Draft or Review
- **Withdrawn**: Author withdraws the proposal
- **Living**: Continually updated (e.g., ZEP-1)

## What belongs in a successful ZEP?

Each ZEP should have the following parts:

### Preamble

RFC 822 style headers containing metadata about the ZEP, including the ZEP number, a short descriptive title (limited to a maximum of 44 characters), a description (limited to a maximum of 140 characters), and the author details.

### Abstract

A short (~200 word) description of the technical issue being addressed.

### Motivation *(optional)*

The motivation section should clearly explain why the existing protocol specification is inadequate to address the problem that the ZEP solves. This section may be omitted if the motivation is evident.

### Specification

The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations.

### Rationale

The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work.

### Backwards Compatibility

All ZEPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The ZEP must explain how the author proposes to deal with these incompatibilities.

### Test Cases

Test cases for an implementation are mandatory for ZEPs that are affecting consensus changes. Tests should either be inlined in the ZEP as data or included in `../assets/zep-###/<filename>`.

### Reference Implementation *(optional)*

An optional section that contains a reference/example implementation that people can use to assist in understanding or implementing this specification.

### Security Considerations

All ZEPs must contain a section that discusses the security implications/considerations relevant to the proposed change. This section is mandatory and must discuss potential security risks.

### Copyright

All ZEPs must be in the public domain. The copyright waiver must use the following wording:

```
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
```

## ZEP Formats and Templates

ZEPs should be written in markdown format. There is a [template](../zep-template.md) to follow.

### Auxiliary Files

Images, diagrams and auxiliary files should be included in a subdirectory of the `assets` folder for that ZEP as follows: `assets/zep-N` (where **N** is to be replaced with the ZEP number). When linking to an image in the ZEP, use relative links such as `../assets/zep-1/image.png`.

## ZEP Header Preamble

Each ZEP must begin with an RFC 822 style header preamble in YAML format. The headers must appear in the following order:

- `zep`: ZEP number
- `title`: The ZEP title (maximum 44 characters)
- `description`: Description (maximum 140 characters)
- `author`: List of authors' names and contact info
- `discussions-to`: URL pointing to the official discussion thread
- `status`: Draft | Review | Last Call | Final | Stagnant | Withdrawn | Living
- `type`: Standards Track | Meta | Informational
- `category`: Core | Networking | Interface (required for Standards Track only)
- `created`: Date created (ISO 8601 format: yyyy-mm-dd)
- `requires`: ZEP number(s) (optional)
- `withdrawal-reason`: Explanation if withdrawn (optional)

### Author Header

The `author` header lists the names and contact information for the authors/owners of the ZEP. Format:

```
author: FirstName LastName (@GitHubUsername), FirstName LastName <email@example.com>
```

At least one author must use a GitHub username in parentheses or an email address in angle brackets. The email address or GitHub username may be omitted.

### discussions-to Header

While a ZEP is a draft, a `discussions-to` header will indicate the URL where the ZEP is being discussed. The preferred discussion URL is a GitHub Discussion in the ZEPs repository.

### type Header

The `type` header specifies the type of ZEP: Standards Track, Meta, or Informational.

### category Header

The `category` header specifies the ZEP's category. This is required for Standards Track ZEPs only.

### created Header

The `created` header records the date that the ZEP was assigned a number. It should be in ISO 8601 format (yyyy-mm-dd).

### requires Header

ZEPs may have a `requires` header, indicating the ZEP numbers that this ZEP depends on.

## Linking to External Resources

Links to external resources **SHOULD NOT** be included. External resources may disappear, move, or change unexpectedly.

## Linking to other ZEPs

References to other ZEPs should follow the format `ZEP-N` where `N` is the ZEP number you are referring to.

## Transferring ZEP Ownership

It occasionally becomes necessary to transfer ownership of ZEPs to a new champion. In general, we'd like to retain the original author as a co-author of the transferred ZEP, but that's really up to the original author. A good reason to transfer ownership is because the original author no longer has the time or interest in updating it or following through with the ZEP process, or has fallen off the face of the 'net.

## ZEP Editors

The current ZEP editors are:

- Zattera Core Team

### ZEP Editor Responsibilities

For each new ZEP submitted, an editor does the following:

- Read the ZEP to check if it is ready, sound, and complete
- Check that title accurately describes the content
- Check that the ZEP follows the formatting rules
- Check the ZEP for language (spelling, grammar, sentence structure, etc.)
- If the ZEP is not ready, send it back to the author for revision with specific instructions

Once the ZEP is ready for the repository, the ZEP editor will:

- Assign a ZEP number (generally the PR number)
- Merge the pull request
- Send a message to the author with the next steps

The editors don't pass judgment on ZEPs. They merely perform administrative & editorial functions.

## Style Guide

### ZEP Numbers

When referring to a ZEP, it should be written in the form `ZEP-X` where `X` is the ZEP's assigned number.

### RFC 2119 and RFC 8174

ZEPs are encouraged to follow [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.html) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174.html) for terminology regarding requirement levels.

### Capitalizations

- Zattera (proper noun, always capitalized)
- ZEP/ZRC (always uppercase)
- blockchain (lowercase, unless starting a sentence)

## History

This document was derived heavily from Ethereum's EIP-1, which was derived from Bitcoin's BIP-0001 written by Amir Taaki, which in turn was derived from Python's PEP-0001. In many places text was simply copied and modified.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
