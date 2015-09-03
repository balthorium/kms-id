---
title: Group Key Management Architecture
abbrev: key-management-service
docname: draft-abiggs-saag-group-key-management-00
date: 2015-07-03
category: std
ipr: trust200902
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: A. Biggs
    name: Andrew Biggs
    organization: Cisco Systems
    email: adb@cisco.com
 -
    ins: S. Cooley 
    name: Shaun Cooley
    organization: Cisco Systems
    email: shcooley@cisco.com

normative:

  RFC2119:

  RFC5280:

  RFC5869:

  RFC7159:

  I-D.ietf-jose-json-web-encryption:

  I-D.ietf-jose-json-web-key:

  I-D.ietf-jose-json-web-signature:

  I-D.ietf-jose-json-web-algorithms:

informative:

  I-D.barnes-pervasive-problem:

  I-D.newton-json-content-rules:

--- abstract

Insert pithy abstract here.

--- middle

# Introduction

This specification proposes a strategy for the establishment of E2E confidentiality of communications and content sharing among multiple parties.  Specifically, it aims to provide a basis for addressing three basic elements of secure group communications:

  (1) the manner in which content encryption keys are created,
  (2) the manner in which content encryption keys are securely shared, and
  (3) the manner in which entities are authorized to receive these keys.

Additional objectives of this specification include support for decentralized deployment over untrusted networks and a strong preference for leveraging existing and modern standards in cryptographic technology and infrastructure.  Groups should also be readily formed on an ad-hoc basis and their membership mutable over the lifetime of the group.

It is expressly not a goal of this specification to define how communications and shared content are actually encrypted or transported among members.  It is presumed that given a means for securely creating, sharing, and authorizing access to cryptographic keys, any communications application can readily determine a content encryption scheme and transport mechanism suitable to their particular needs.

## Terminology {#terminology}

This document uses the terminology from {{I-D.ietf-jose-json-web-signature}}, {{I-D.ietf-jose-json-web-encryption}}, {{I-D.ietf-jose-json-web-key}}, and {{I-D.ietf-jose-json-web-algorithms}} when discussing JOSE technologies.  Most security-related terms in this document are to be understood in the sense defined in {{RFC4949}}.

## Notational Conventions

In this document, the key words "MUST", "MUST NOT", "REQUIRED",
"SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY",
and "OPTIONAL" are to be interpreted as described in BCP 14, RFC 2119
{{RFC2119}}.

# Confidentiality Groups

Central to this specification is the notion of a "confidentiality group".  For the purposes of this document we refer to this as simply a "group", and define it as a logical collection of two or more entities that wish to secure some communications resource in a manner that permits data emitted to that communications resource by each member to be consumable by each other member, but not by non-members.  

We also define an "entity" as any user, client, or automated process which is in posession of an asymmetric key pair for which there exists a discoverable PKIX certificate to authenticate that entity's public key component.  The means by which this certificate may be discovered is out of scope for this specification.

## Group Membership

Membership within a group is defined though a membership block chain.  A membership block chain is an ordered list of blocks representing a public and tamper-proof ledger of group membership updates.  Each block is composed of a JWS object with a JSON object as payload signed with the private key of the entity that appended the block to the chain.  The JSON object within each signed block includes at least the following information:

  * the public key of the entity that appended the block (as a JWK),
  * an array of group membership operation objects, and
  * a hash of the preceeding block in the membership chain.

As indicated above, a block contains an array of operation objects, each of which represents an update to the group membership.  An operation object has two fields:

  * a field indicating the operation type ("add" or "remove"),
  * the public key of the entity being added or removed (as a JWK).

Membership in the group is implicit and may be determined by processing the block chain in chronological order.  At any given point in time the membership of the group is defined as that set of entities for each of which there exists a block containing an "add" operation and for which there does not exist a subsequent block containing a "remove" operation.

To protect against unauthorized tampering, the membership chain may be validated by traversing the chain from beginning to end and confirming that each block contains a valid hash of the preceding block and is signed by an entity that is among the group's membership as deterimined by the segment of chain preceding that block.

The means by which a membership block chain is shared with other entities is out of scope for this document.  Given that the content of the genesis node is not necessarily confidential and can be authenticated by virtue of the signatures applied to the chain blocks, it would be not be unreasonable for the an application of this framework to publish the membership block chain publicly or transmit it openly over an insecure channel.  On the other hand, a deployment may choose to store membership block chain information centrally within a trusted environment, perhaps to provide an additional level of confidentiality around group membership itself.  This specification does not presume to advocate an approach that is suitable for all applications.

## Group Creation

An entity, referred to in this document as the "initiator", creates a new group by creating a new group block chain containing a single genesis block.  The genesis block is a JWS signed JSON object containing the fields described in the previous section plus one additional field that identifies the URI of some communications resource.  Note that the genesis block must also contain one or more "add" operations to represent the original group membership (though this need not necessarily include the initiator itself). 

## Group Membership Updates

Any current member of a group may update the group membership by appending new blocks to the group membership chain.  To do so the member contructs a new membership block including a hash of the previous tail block of the chain, and adds operations to the new block to represent the "add" and/or "remove" operation(s) they wish to enact.  The member then signs the block with their entity private key and publishes the new tail to other members.  The means by which this is performed is, as mentioned earlier, out of scope for this document.

Since the means by which membership chain updates are exchanged between members is not specified in this document, we must consider the case where there exists no externally applied synchornization around this operation and therefore the possibility that conflicts may arise.  This can happen, for instance, if two members attempt to each append a new block to the same tail block of the chain.  In this case, conflict is resolved by convention where the new block with the numerically smallest hash value prevails and all other competing blocks are rejected.

## Group Membership Validation

A critical property of the membership block chain is that it can be readily validated against tampering to its content or structure by non-member entities.  Furthermore, this validation can be performed by any other entity, regardless of whether the validating entity is itself a member of the group.

The means by which a membership block chain may be validated is as follows.

  1. The validating entity confirms that the public key indicated in the genesis block represents the entity it expects to be the originating entity of the group.

  2. The validating entity confirms that the resource URI indicated in the genesis block is the URI of the communications resource it expects to be secured for access by this group.

  3. The validating entity uses the public key indicated in the genesis block to validate the signature on the genesis block.

  4. The validating entity records all "add" operations included in the genesis block and regards these as representing the initial group membership state.

  5. The remaining steps are performed in sequence for each subsequent block in the membership chain:

  5a. The validating entity confirms that the hash attribute indicated in the block is the hash of the preceding block.

  5b. The validating entity confirms that the public key indicated in the block is among those in the current group membership state.

  5c. The validating entity uses the public key indicated in the block to validate the signature of the block.

  5d. The validating entity records all "add" and "remove" operations included in the block and updates the current group membership state accordingly.
  


# Mandatory-to-Implement


# Security Considerations


# Appendix A. Acknowledgments


# Appendix B. Document History

\-00

  * Initial draft.
