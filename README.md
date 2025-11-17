<div align="center">

![Zattera Enhancement Proposals (ZEPs)](assets/images/main-banner.png)

[한국어](README.ko.md)

</div>

# Zattera Enhancement Proposals (ZEPs)

This repository contains all Zattera Enhancement Proposals (ZEPs) for the Zattera blockchain protocol.

## What are ZEPs?

**ZEPs (Zattera Enhancement Proposals)** are design documents providing information to the Zattera community, or describing a new feature for Zattera or its processes or environment. ZEPs provide a concise technical specification of the feature and the rationale for the feature.

ZEPs cover both protocol-level changes and application-level standards. Application-level standards (similar to Ethereum's ERCs) are Standards Track ZEPs with category "Application".

This structure is inspired by Ethereum's EIP/ERC system.

## ZEP Types

ZEPs are separated into several types:

- **Standards Track ZEP**: Changes affecting most or all Zattera implementations
  - **Core**: Improvements requiring a consensus fork (e.g., ZEP-2, ZEP-4)
  - **Networking**: Improvements to network protocol specifications (e.g., ZEP-3)
  - **Interface**: API/RPC specifications and standards
  - **Application**: Application-level standards and conventions, including:
    - Plugin interfaces and behaviors
    - Wallet formats and key management
    - API conventions
    - JSON metadata formats and URI schemes

- **Meta ZEP**: Process changes or decisions about processes (e.g., ZEP-1 - procedures, guidelines, decision-making)

- **Informational ZEP**: Design issues, general guidelines, or information to the community (does not propose new features)

## ZEP Workflow

### 1. Idea Stage
- Discuss idea in community forums
- Vet the concept with core developers
- Ensure idea is original and technically sound

### 2. Draft Stage
- Author creates ZEP document using the template
- Submit pull request to this repository
- ZEP number assigned by editors
- Status: **Draft**

### 3. Review Stage
- Community reviews and provides feedback
- Author makes revisions based on feedback
- Technical review by core developers
- Status: **Review** or **Last Call**

### 4. Final Stage
- Rough consensus achieved
- Implementation complete (for Standards Track)
- Status: **Final**

### Other Statuses
- **Stagnant**: Inactive for 6+ months
- **Withdrawn**: Author withdraws proposal
- **Living**: Continually updated (e.g., ZEP-1)

## ZEP Format

All ZEPs must follow this structure:

```markdown
---
zep: <ZEP number>
title: <ZEP title>
description: <Short description>
author: <Name> (@github), <Name> <email>
discussions-to: <URL>
status: Draft | Review | Last Call | Final | Stagnant | Withdrawn | Living
type: Standards Track | Meta | Informational
category: Core | Networking | Interface | (Standards Track only)
created: YYYY-MM-DD
requires: <ZEP numbers> (optional)
---

## Abstract
Brief technical summary

## Motivation
Why is this needed?

## Specification
Technical specification

## Rationale
Design decisions and alternatives

## Backwards Compatibility
Breaking changes and migration

## Test Cases
Test cases for implementation

## Reference Implementation
Link to implementation

## Security Considerations
Security implications

## Copyright
Copyright waiver
```

## Contributing

To submit a ZEP:

1. **Fork** this repository
2. **Copy** `zep-template.md` to `ZEPs/zep-XXXX.md` (check existing ZEPs for next available number)
3. **Fill out** the template with your proposal
4. **Submit** a pull request
5. **Editor reviews** and may reassign the ZEP number if needed before merging

### Before Submitting

- **Read [ZEP-1: ZEP Purpose and Guidelines](ZEPs/zep-1.md)** for detailed process and formatting requirements
- Ensure your idea hasn't been proposed before
- Discuss in community channels first
- Follow the ZEP format strictly
- Include all required sections
- Use proper markdown formatting
- Provide clear technical specifications

### ZEP Editors

ZEP editors are responsible for:
- Assigning ZEP numbers
- Reviewing ZEP formatting
- Checking technical soundness
- Merging approved ZEPs
- Managing ZEP status changes

Current editors:
- Zattera Core Team

## License

All ZEPs and ZRCs are released under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).
