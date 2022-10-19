---
title: SUIT Manifest Extensions for Multiple Trust Domains
abbrev: SUIT Trust Domains
docname: draft-ietf-suit-trust-domains-00
category: std

ipr: trust200902
area: Security
workgroup: SUIT
keyword: Internet-Draft

stand_alone: yes
pi:
  rfcedstyle: yes
  toc: yes
  tocindent: yes
  sortrefs: yes
  symrefs: yes
  strict: yes
  comments: yes
  inline: yes
  text-list-symbols: -o*+
  docmapping: yes
  toc_levels: 4

author:
 -
      ins: B. Moran
      name: Brendan Moran
      organization: Arm Limited
      email: Brendan.Moran@arm.com

 -
      ins: K. Takayama
      name: Ken Takayama
      organization: SECOM CO., LTD.
      email: ken.takayama.ietf@gmail.com

normative:
  RFC3986:
  RFC7228:
  RFC8392:
  RFC8747:
  RFC9019:
  I-D.ietf-suit-manifest:

informative:
  I-D.ietf-suit-information-model:
  I-D.ietf-suit-firmware-encryption:
  I-D.ietf-teep-architecture:

--- abstract

This specification describes extensions to the SUIT manifest format (as
defined in {{I-D.ietf-suit-manifest}}) for use in deployments with
multiple trust domains. A device has more than one trust domain when it
uses different trust anchors for different purposes or components in the
context of firmware update.

--- middle

#  Introduction

Devices that go beyond single-signer update require more complex rules for deploying firmware updates. For example, devices may require:

* long-term trust anchors with a mechanism to delegate trust to short term keys. 
* software components from multiple software signing authorities.
* a mechanism to remove an uneeded component
* single-object dependencies
* a partly encrypted manifest so that distribution does not reveal private information

These mechanisms are not part of the core manifest specification, but they are needed for more advanced use cases, such as the architecture described in {{I-D.ietf-teep-architecture}}.

This specification extends the SUIT Manifest specification ({{I-D.ietf-suit-manifest}}).

#  Conventions and Terminology

{::boilerplate bcp14}

Additionally, the following terminology is used throughout this document:

* SUIT: Software Update for the Internet of Things, also the IETF working group for this standard.
* Payload: A piece of information to be delivered. Typically Firmware for the purposes of SUIT.
* Resource: A piece of information that is used to construct a payload.
* Manifest: A manifest is a bundle of metadata about the firmware for an IoT device, where to
find the firmware, and the devices to which it applies.
* Envelope: A container with the manifest, an authentication wrapper with cryptographic information protecting the manifest, authorization information, and severable elements (see: TBD).
* Update: One or more manifests that describe one or more payloads.
* Update Authority: The owner of a cryptographic key used to sign updates, trusted by Recipients.
* Recipient: The system, typically an IoT device, that receives and processes a manifest.
* Manifest Processor: A component of the Recipient that consumes Manifests and executes the commands in the Manifest.
* Component: An updatable logical block of the Firmware, Software, configuration, or data of the Recipient.
* Component Set: A group of interdependent Components that must be updated simultaneously.
* Command: A Condition or a Directive.
* Condition: A test for a property of the Recipient or its Components.
* Directive: An action for the Recipient to perform.
* Trusted Invocation: A process by which a system ensures that only trusted code is executed, for example secure boot or launching a Trusted Application.
* A/B images: Dividing a Recipient's storage into two or more bootable images, at different offsets, such that the active image can write to the inactive image(s).
* Record: The result of a Command and any metadata about it.
* Report: A list of Records.
* Procedure: The process of invoking one or more sequences of commands.
* Update Procedure: A procedure that updates a Recipient by fetching dependencies and images, and installing them.
* Invocation Procedure: A procedure in which a Recipient verifies dependencies and images, loading images, and invokes one or more image.
* Software: Instructions and data that allow a Recipient to perform a useful function.
* Firmware: Software that is typically changed infrequently, stored in nonvolatile memory, and small enough to apply to {{RFC7228}} Class 0-2 devices.
* Image: Information that a Recipient uses to perform its function, typically firmware/software, configuration, or resource data such as text or images. Also, a Payload, once installed is an Image.
* Slot: One of several possible storage locations for a given Component, typically used in A/B image systems
* Abort: An event in which the Manifest Processor immediately halts execution of the current Procedure. It creates a Record of an error condition.

#  Changes to SUIT Workflow Model

The use of the features presented for use with multiple trust domains requires some augmentation of the workflow presented in the SUIT Manifest specification ({{I-D.ietf-suit-manifest}}):

One additional assumption is added for the Update Procedure: 

* All dependency manifests should be present before any payload is fetched.

One additional assumption is added to the Invocation Procedure:

* All dependencies must be validated prior to loading.

Two steps are added to the expected installation workflow of a Recipient:

1. **Verify delegation chains**
2. Verify the signature of the manifest.
3. Verify the applicability of the manifest.
4. **Resolve dependencies.**
5. Fetch payload(s).
6. Install payload(s).

In addition, when multiple manifests are used for an update, each manifest's steps occur in a lockstep fashion; all manifests have dependency resolution performed before any manifest performs a payload fetch, etc.

#  Changes to Manifest Metadata Structure {#metadata-structure-overview}

To accomodate the additional metadata needed to enable these features, the envelope and manifest have several new elements added.

The Envelope gains two more elements: Delegation chains and Integrated Dependencies
The Common metadata section in the Manifest also gains a list of dependencies.

The new metadata structure is shown below.

~~~
+-------------------------+
| Envelope                |
+-------------------------+
| Delegation Chains       |
| Authentication Block    |
| Manifest           --------------> +------------------------------+
| Severable Elements      |          | Manifest                     |
| Human-Readable Text     |          +------------------------------+
| COSWID                  |          | Structure Version            |
| Integrated Dependencies |          | Sequence Number              |
| Integrated Payloads     |          | Reference to Full Manifest   |
+-------------------------+    +------ Common Structure             |
                               | +---- Command Sequences            |
+-------------------------+    | |   | Digests of Envelope Elements |
| Common Structure        | <--+ |   +------------------------------+
+-------------------------+      |
| Dependency Indices      |      +-> +-----------------------+
| Component IDs           |          | Command Sequence      |
| Common Command Sequence ---------> +-----------------------+
+-------------------------+          | List of ( pairs of (  |
                                     |   * command code      |
                                     |   * argument /        |
                                     |      reporting policy |
                                     | ))                    |
                                     +-----------------------+
~~~

#  Delegation Chains {#ovr-delegation}

Delegation Chains allow a Recipient to establish a chain of trust from a Trust Anchor to the signer of a manifest by validating delegation claims. Each delegation claim is a {{RFC8392}} CBOR Web Tokens (CWTs). The first claim in each list is signed by a Trust Anchor. Each subsequent claim in a list is signed by the public key claimed in the preceding list element. The last element in each list claims a public key that can be used to verify a signature in the Authentication Block (See Sectino 5.2 of {{I-D.ietf-suit-manifest}}).

See {{delegation-info}} for more detail.

##  Delegation Chains {#delegation-info}

The suit-delegation element MAY carry one or more CBOR Web Tokens (CWTs) {{RFC8392}}, with {{RFC8747}} cnf claims. They can be used to perform enhanced authorization decisions. The CWTs are arranged into a list of lists. Each list starts with a CWT authorized by a Trust Anchor, and finishes with a key used to authenticate the Manifest (see Section 8.3 of {{I-D.ietf-suit-manifest}}). This allows an Update Authority to delegate from a long term Trust Anchor, down through intermediaries, to a delegate without any out-of-band provisioning of Trust Anchors or intermediary keys.

A Recipient MAY choose to cache intermediaries and/or delegates. If an Update Distributor knows that a targeted Recipient has cached some intermediaries or delegates, it MAY choose to strip any cached intermediaries or delegates from the Delegation Chains in order to reduce bandwidth and energy.


#  Dependencies 

A dependency is another SUIT_Envelope that describes additional components. 

Dependency manifests enable several additional use cases. In particular, they enable two or more entities who are trusted for different privileges to coordinate. This can be used in many scenarios, for example:

* An IoT device may contain a processor in its radio in addition to the primary processor. These two processors may have separate firmware with separate signing authorities. Dependencies allow the firmware for the primary processor to reference a manifest signed by a different authority.
* A network operator may wish to provide local caching of update payloads. The network operator overrides the URI of payload by providing a dependent manifest that references the original manifest, but replaces its URI.
* A device operator provides a device with some additional configuration. The device operator wants to test their configuration with each new firmware version before releasing it. The configuration is delivered as a binary in the same way as a firmware image. The device operator references the firmware manifest from the firmware author in their own manifest which also defines the configuration.

By using dependencies, components such as software, configuration, models, and other resources authenticated by different trust anchors can be delivered to devices.

##  Changes to Required Checks {#required-checks}

This section augments the definitions in Required Checks (Section 6.2) of {{I-D.ietf-suit-manifest}}.

More checks are required when handling dependencies. By default, any signature of a dependency MUST be verified. However, there are some exceptions to this rule: where a device supports only one level of access (no ACLs defining which authorities have access to different components/commands/parameters), it MAY choose to skip signature verification of dependencies, since they are verified by digest. Where a device differentiates between trust levels, such as with an ACL, it MAY choose to defer the verification of signatures of dependencies until the list of affected components is known so that it can skip redundant signature verifications. For example, if a dependent's signer has access rights to all components specified in a dependency, then that dependency does not require a signature verification. Similarly, if the signer of the dependent has full rights to the device, according to the ACL, then no signature verification is necessary on the dependency.

Components that should be treated as dependency manifests are identified in the suit-common metadata. See section {{structure-change}} for details.

If the manifest contains more than one component and/or dependency, each command sequence MUST begin with a Set Component Index command.

If a dependency is specified, then the manifest processor MUST perform the following checks:

1. The dependent MUST populate all command sequences for the current procedure (Update or Invoke).
2. At the end of each section in the dependent: The corresponding section in each dependency has been executed.

If the interpreter does not support dependencies and a manifest specifies a dependency, then the interpreter MUST Abort.

If a Recipient supports groups of interdependent components (a Component Set), then it SHOULD verify that all Components in the Component Set are specified by one update, that is: a single manifest and all its dependencies that together:

1. have sufficient permissions imparted by their signatures
2. specify a digest and a payload for every Component in the Component Set.

The single dependent manifest is sometimes called a Root Manifest.

##  Changes to Manifest Structure {#structure-change}

This section augments the Manifest Structure (Section 8.4) in {{I-D.ietf-suit-manifest}}. 

##  Changes to Abstract Machine Description

This section augments the Abstract Machine Description (Section 6.4) in {{I-D.ietf-suit-manifest}}
With the addition of dependencies, some changes are necessary to the abstract machine, outside the typical scope of added commands. These changes alter the behaviour of an existing command and way that the parser processes manifests:

* Two new commands are introduced.

    * Process dependency.
    * Is Dependency.

* Dependency manifests are also components. All commands may target dependency manifests as well as components, with one exception: process dependency. Commands defined outside of this draft and {{I-D.ietf-suit-manifest}} MAY have additional restrictions.
* Dependencies are processed in lock-step with the Root Manifest. This means that every dependency's current command sequence must be executed before a dependent's later command sequence may be executed. For example, every dependency's Dependency Resolution step MUST be executed before any dependent's payload fetch step.
* When performing a suit-condition-image-match operation on a component, the manifest processor MUST first determine whether or not the component is a dependency manifest. If identified as a dependency manifest envelope, the manifest processor MUST compute the digest over only the SUIT_Manifest bstr, not the complete SUIT_Manifest_Envelope. This is so that severable elements, added or removed signatures, and delegations do not affect the integrity measurements of the manifest.

##  Processing Dependencies {#processing-dependencies}

As described in {{required-checks}}, each manifest must invoke each of its dependencies' sections from the corresponding section of the dependent. Any changes made to parameters by the dependency persist in the dependent.

When a Process Dependency command is encountered, the manifest processor:

1. Checks whether the map of dependencies contains an entry for the current Component Index. If not present, it causes an immediate Abort.
2. Loads the specified component as a dependency manifest envelope.
3. Authenticates the dependency manifest
4. Executes the common-sequence section of the dependency manifest
5. Executes the section of the dependency manifest that corresponds to the currently executing section of the dependent.

If the specified dependency does not contain the current section, Process Dependency succeeds immediately.

The interpreter also performs the checks described in {{required-checks}} to ensure that the dependent is processing the dependency correctly.

###  Multiple Manifest Processors {#hierarchical-interpreters}

When a system has multiple security domains, each domain might require independent verification of authenticity or security policies. Security domains might be divided by separation technology such as Arm TrustZone, Intel SGX, or another TEE technology. Security domains might also be divided into separate processors and memory spaces, with a communication interface between them.

For example, an application processor may have an attached communications module that contains a processor. The communications module might require metadata signed by a specific Trust Authority for regulatory approval. This may be a different Trust Authority than the application processor.

When there are two or more security domains (see {{I-D.ietf-teep-architecture}}), a manifest processor might be required in each. The first manifest processor is the normal manifest processor as described for the Recipient in Section 6 of {{I-D.ietf-suit-manifest}}. The second manifest processor only executes sections when the first manifest processor requests it. An API interface is provided from the second manifest processor to the first. This allows the first manifest processor to request a limited set of operations from the second. These operations are limited to: setting parameters, inserting an Envelope, invoking a Manifest Command Sequence. The second manifest processor declares a prefix to the first, which tells the first manifest processor when it should delegate to the second. These rules are enforced by underlying separation of privilege infrastructure, such as TEEs, or physical separation.

When the first manifest processor encounters a dependency prefix, that informs the first manifest processor that it should provide the second manifest processor with the corresponding dependency Envelope. This is done when the dependency is fetched. The second manifest processor immediately verifies any authentication information in the dependency Envelope. When a parameter is set for any component that matches the prefix, this parameter setting is passed to the second manifest processor via an API. As the first manifest processor works through the Procedure (set of command sequences) it is executing, each time it sees a Process Dependency command that is associated with the prefix declared by the second manifest processor, it uses the API to ask the second manifest processor to invoke that dependency section instead.

This mechanism ensures that the two or more manifest processors do not need to trust each other, except in a very limited case. When parameter setting across security domains is used, it must be very carefully considered. Only parameters that do not have an effect on security properties should be allowed. The dependency manifest MAY control which parameters are allowed to be set by using the Override Parameters directive. The second manifest processor MAY also control which parameters may be set by the first manifest processor by means of an ACL that lists the allowed parameters. For example, a URI may be set by a dependent without a substantial impact on the security properties of the manifest.

##  Added and Modified Commands

All commands are modified in that they can also target dependencies. However, Set Component Index has a larger modification.

| Command Name | Semantic of the Operation
|------|----
| Process Dependency | exec(current\[common\]); exec(current\[current-segment\])
| Is Dependency | assert(current exists in dependencies)
| Unlink | unlink(current)


### suit-directive-process-dependency {#suit-directive-process-dependency}

Execute the commands in the common section of the current dependency, followed by the commands in the equivalent section of the current dependency. For example, if the current section is "fetch payload," this will execute "common" in the current dependency, then "fetch payload" in the current dependency. Once this is complete, the command following suit-directive-process-dependency will be processed.

If the current component index does not have an entry in the suit-dependencies map, then this command MUST Abort.

If the current component is True, then this directive applies to all dependencies. If the current section is "common," then the command sequence MUST Abort.

When SUIT_Process_Dependency completes, it forwards the last status code that occurred in the dependency.

### suit-condition-is-dependency {#suit-directive-is-dependency}

Check whether or not the current component index is present in the dependency list. If the current component is in the dependency list, suit-condition-is-dependency succeeds. Otherwise, it fails. This can be used along with component-id = True to act on all dependencies or on all non-dependency components. See {{creating-manifests}} for more details.

### suit-directive-unlink {#suit-directive-unlink}

suit-directive-unlink marks the current component as unused in the current manifest. This can be used to remove temporary storage or remove components that are no longer needed. Example use cases:

* Temporary storage for encrypted download
* Temporary storage for verifying decompressed file before writing to flash
* Removing Trusted Service no longer needed by Trusted Application

Once the current Command Sequence is complete, the manifest processors checks each marked component to see whether any other manifests have referenced it. Those marked components with no other references are deleted. The manifest processor MAY choose to ignore a Unlink directive depending on device policy.

suit-directive-unlink is OPTIONAL to implement in manifest processors.

## SUIT_Dependencies Manifest Element {#SUIT_Dependencies}

Because some operations treat dependency manifests differently from other components, it is necessary to identify them. SUIT_Dependencies identifies which components from suit-components (See Section 8.4.5 of {{I-D.ietf-suit-manifest}}) are to be treated as dependency manifest envelopes. SUIT_Dependencies is a map of Components, referenced by Component Index. Optionally, a component prefix or other metadata may be delivered with the component index. The CDDL for suit-dependencies is shown below:

~~~CDDL
SUIT_Dependencies = {
    + uint => SUIT_Dependency_Metadata
}
SUIT_Dependency_Metadata = {
    ? suit-dependency-prefix => SUIT_Component_Identifier
    $$SUIT_Dependency_Extensions
}
~~~

If no extended metadata is needed for an extension, SUIT_Dependency_Metadata is an empty map (this is the same encoding size as a null). SUIT_Dependencies MUST be sorted according to CBOR canonical encoding.

The components specified by SUIT_Dependency will contain a Manifest Envelope that describes a dependency of the current manifest. The Manifest is identified, but the Recipient should expect an Envelope when it acquires the dependency. This is because the Manifest is the one invariant element of the Envelope, where other elements may change by countersigning, adding authentication blocks, or severing elements.

When executing suit-condition-image-match over a component that is designated in SUIT_Dependency, the digest MUST be computed over just the bstr-wrapped SUIT_Manifest contained in the Manifest Envelope designated by the Component Index. This enables a dependency reference to uniquely identify a particular Manifest structure. This is identical to the digest that is present as the first element of the suit-authentication-block in the dependency's Envelope. The digest is calculated over the Manifest structure to ensure that removing a signature from a manifest does not break dependencies due to missing signature elements. This is also necessary to support the trusted intermediary use case, where an intermediary re-signs the Manifest, removing the original signature, potentially with a different algorithm, or trading COSE_Sign for COSE_Mac.

The suit-dependency-prefix element contains a SUIT_Component_Identifier (see Section 8.4.5.1 of {{I-D.ietf-suit-manifest}}). This specifies the scope at which the dependency operates. This allows the dependency to be forwarded on to a component that is capable of parsing its own manifests. It also allows one manifest to be deployed to multiple dependent Recipients without those Recipients needing consistent component hierarchy. This element is OPTIONAL for Recipients to implement.

A dependency prefix can be used with a component identifier. This allows complex systems to understand where dependencies need to be applied. The dependency prefix can be used in one of two ways. The first simply prepends the prefix to all Component Identifiers in the dependency.

A dependency prefix can also be used to indicate when a dependency manifest needs to be processed by a secondary manifest processor, as described in {{hierarchical-interpreters}}.

# Creating Manifests {#creating-manifests}

This section details a set of templates for creating manifests. These templates explain which parameters, commands, and orders of commands are necessary to achieve a stated goal.

## Dependency Template {#template-dependency}

The goal of the Dependency template is to obtain, verify, and process a dependency manifest as appropriate.

The following commands are added to the shared sequence:

- Set Component Index directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Set Parameters directive (see {{suit-directive-set-parameters}}) for digest (see Section 8.4.8.6 of {{I-D.ietf-suit-manifest}}). Note that the digest MUST match the SUIT_Digest in the dependency's suit-authentication-block (See Section 8.3 of {{I-D.ietf-suit-manifest}}).

The following commands are placed into the dependency resolution sequence:

- Set Component Index directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Set Parameters directive (see {{suit-directive-set-parameters}}) for URI (see Section 8.4.8.10 of {{I-D.ietf-suit-manifest}})
- Fetch directive (see Section 8.4.10.4 of {{I-D.ietf-suit-manifest}})
- Check Image Match condition (see Section 8.4.9.2 of {{I-D.ietf-suit-manifest}} of {{I-D.ietf-suit-manifest}})
- Process Dependency directive (see {{suit-directive-process-dependency}})

Then, the validate sequence contains the following operations:

- Set Component Index directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Check Image Match condition (see Section 8.4.9.2 of {{I-D.ietf-suit-manifest}})
- Process Dependency directive (see {{suit-directive-process-dependency}})

If any dependency is declared, the dependent MUST populate all command sequences for the current procedure (Update or Invoke).

NOTE: Any changes made to parameters in a dependency persist in the dependent.

### Composite Manifests {#composite-manifests}

An implementer MAY choose to place a dependency's envelope in the envelope of its dependent. The dependent envelope key for the dependency envelope MUST be a text string. The URI for the dependency MUST match the text string key of the dependent's envelope key. It is RECOMMENDED to make the text string key a resolvable URI so that a dependency manifest that is removed from the envelope can still be fetched.

## Encrypted Manifest Template {#template-encrypted-manifest}

The goal of the Encrypted Manifest template is to fetch and decrypt a manifest so that it can be used as a dependency. To use an encrypted manifest, create a plaintext dependent, and add the encrypted manifest as a dependency. The dependent can include very little information.

NOTE: This template also requires the extensions defined in {{I-D.ietf-suit-firmware-encryption}}

The following commands are added to the shared sequence:

- Set Component Index directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Set Parameters directive (see {{suit-directive-set-parameters}}) for digest (see Section 8.4.8.6 of {{I-D.ietf-suit-manifest}}). Note that the digest MUST match the SUIT_Digest in the dependency's suit-authentication-block (See Section 8.3 of {{I-D.ietf-suit-manifest}}).

The following operations are placed into the dependency resolution block:

- Set Component Index directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Set Parameters directive (see {{suit-directive-set-parameters}}) for
    - URI (see Section 8.4.8.9 of {{I-D.ietf-suit-manifest}})
    - Encryption Info (See {{I-D.ietf-suit-firmware-encryption}})
- Fetch directive (see Section 8.4.10.4 of {{I-D.ietf-suit-manifest}})
- Check Image Match condition (see Section 8.4.9.2 of {{I-D.ietf-suit-manifest}})
- Process Dependency directive (see {{suit-directive-process-dependency}})

Then, the validate block contains the following operations:

- Set Component Index directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Check Image Match condition (see Section 8.4.9.2 of {{I-D.ietf-suit-manifest}})
- Process Dependency directive (see {{suit-directive-process-dependency}})

A plaintext manifest and its encrypted dependency may also form a composite manifest ({{composite-manifests}}).

## Operating on Multiple Components

In order to produce compact encoding, it is efficient to perform operations on multiple components simultaneously. Because Dependency Manifests and Component Images are processed at different times, there is a mechanism to distinguish between these elements: suit-condition-is-manifest. This can be used with suit-directive-try-each to perform operations just on Dependency Manifests or just on Component Images.

For example, to fetch all dependency manifests, the following commands are added to the dependency resolution block:

- Set Component Index directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Set Parameters directive (see {{suit-directive-set-parameters}}) for
    - URI (see Section 8.4.8.9 of {{I-D.ietf-suit-manifest}})
- Set Component Index directive, with argument "True" (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Try Each Directive
    - Sequence 0
        - Condition Is Manifest
        - Fetch
        - Condition Image Match
        - Process Dependency
    - Sequence 1 (Empty; no commands, succeeds immediately)

Another example is to fetch and validate all Component Images. The image fetch sequence contains the following commands:

- Set Component Index directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Set Parameters directive (see {{suit-directive-set-parameters}}) for
    - URI (see Section 8.4.8.9 of {{I-D.ietf-suit-manifest}})
- Set Component Index directive, with argument "True" (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Try Each Directive
    - Sequence 0
        - Condition Is Manifest
        - Process Dependency
    - Sequence 1 (Empty; no commands, succeeds immediately)
        - Fetch
        - Condition Image Match

When some components are "installed" or "loaded" it is more productive to use lists of component indices rather than Component Index = True. For example, to install several components, the following commands should be placed in the image install sequence:

- Set Component Index directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Set Parameters directive (see {{suit-directive-set-parameters}}) for
    - Source Component (see Section 8.4.8.11 of {{I-D.ietf-suit-manifest}})
- Set Component Index directive, with argument containing list of destination component indices (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Copy
- Set Component Index directive, with argument containing list dependency component indices (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Process Dependency

#  IANA Considerations {#iana}

IANA is requested to allocate the following numbers in the listed registries:

## SUIT Commands

Label | Name | Reference
---|---|---
7 | Is Dependency | suit-directive-is-dependency | {{suit-directive-is-dependency}}
18 | Process Dependency | suit-directive-process-dependency | {{suit-directive-process-dependency}}
19 | Set Parameters | {{suit-directive-set-parameters}}
33 | Unlink | {{suit-directive-unlink}}

#  Security Considerations

This document is about a manifest format protecting and describing how to retrieve, install, and invoke firmware images and as such it is part of a larger solution for delivering firmware updates to IoT devices. A detailed security treatment can be found in the architecture {{RFC9019}} and in the information model {{I-D.ietf-suit-information-model}} documents.


--- back

# A. Full CDDL {#full-cddl}

To be valid, the following CDDL MUST be appended to the SUIT Manifest CDDL. The SUIT CDDL is defined in Appendix A of {{I-D.ietf-suit-manifest}}

~~~ CDDL
{::include draft-ietf-suit-trust-domains.cddl}
~~~
