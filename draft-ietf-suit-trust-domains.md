---
title: SUIT Manifest Extensions for Multiple Trust Domains
abbrev: SUIT Trust Domains
docname: draft-ietf-suit-trust-domains-05
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
      email: brendan.moran.ietf@gmail.com

 -
      ins: K. Takayama
      name: Ken Takayama
      organization: SECOM CO., LTD.
      email: ken.takayama.ietf@gmail.com

normative:
  RFC3986:
  RFC6024:
  RFC7228:
  RFC8392:
  RFC8747:
  RFC9019:
  RFC9124:
  I-D.ietf-suit-manifest:

informative:
  I-D.ietf-suit-update-management:
  I-D.ietf-suit-firmware-encryption:
  I-D.ietf-teep-architecture:

--- abstract

This specification describes extensions to the SUIT Manifest format (as
defined in {{I-D.ietf-suit-manifest}}) for use in deployments with
multiple trust domains. A device has more than one trust domain when it
enables delegation of different rights to mutually distrusting entities
for use for different purposes or Components in the context of firmware
or software update.

--- middle

#  Introduction {#Introduction}

Devices that go beyond single-signer update require more complex rules for deploying software updates. For example, devices may require:

* long-term Trust Anchors with a mechanism to delegate trust to short term keys.
* software Components from multiple software signing authorities.
* a mechanism to remove an unneeded Component
* single-object Dependencies
* a partly encrypted Manifest so that distribution does not reveal private information

Dependency Manifests enable several additional use cases. In particular, they enable two or more entities who are trusted for different privileges to coordinate. This can be used in many scenarios. For example:

* A device may contain a processor in its radio in addition to the primary processor. These two processors may have separate Software with separate signing authorities. Dependencies allow the Software for the primary processor to reference a Manifest signed by a different authority.
* A network operator may wish to provide local caching of Update Payloads. The network operator overrides the URI of a Payload by providing a dependent Manifest that references the original Manifest, but replaces its URI.
* A device operator provides a device with some additional configuration. The device operator wants to test their configuration with each new Software version before releasing it. The configuration is delivered as a binary in the same way as a Software Image. The device operator references the Software Manifest from the Software author in their own Manifest which also defines the configuration.
* An Author wants to entrust a Distributor to provide devices with firmware decryption keys, but not permit the Distributor to sign code. Dependencies allow the Distributor to deliver a device's decryption information without also granting code signing authority.
* A Trusted Application Manager (TAM) wants to distribute personalisation information to a Trusted Execution Environment in addition to a Trusted Application (TA), but does not have code signing authority. Dependencies enable the TAM to construct an update containing the personalisation information and a dependency on the TA, but leaves the TA signed by the TA's Author.

By using Dependencies, Components such as Software, configuration, and other Resource data authenticated by different Trust Anchors can be delivered to devices.

These mechanisms are not part of the core Manifest specification, but they are needed for more advanced use cases, such as the architecture described in {{I-D.ietf-teep-architecture}}.

This specification extends the SUIT Manifest specification ({{I-D.ietf-suit-manifest}}).

#  Conventions and Terminology

{::boilerplate bcp14}

Additionally, the following terminology is used throughout this document:

* SUIT: Software Update for the Internet of Things, also the IETF working group for this standard.
* Payload: A piece of information to be delivered. Typically Firmware/Software, configuration, or Resource data such as text or images.
* Resource: A piece of information that is used to construct a Payload.
* Manifest: A Manifest is a bundle of metadata about one or more Components for a device, where to
find them, and the devices to which they apply.
* Envelope: A container with the Manifest, an authentication wrapper with cryptographic information protecting the Manifest, authorization information, and severable elements (see Section 5.1 of {{I-D.ietf-suit-manifest}}).
* Update: One or more Manifests that describe one or more Payloads.
* Update Authority: The owner of a cryptographic key used to sign Updates, trusted by Recipients.
* Recipient: The system that receives and processes a Manifest.
* Manifest Processor: A component of the Recipient that consumes Manifests and executes the Commands in the Manifest.
* Component: An updatable logical block of the Firmware, Software, configuration, or data of the Recipient.
* Component Set: A group of interdependent Components that must be updated simultaneously.
* Command: A Condition or a Directive.
* Condition: A test for a property of the Recipient or its Components.
* Directive: An action for the Recipient to perform.
* Trusted Invocation: A process by which a system ensures that only trusted code is executed, for example secure boot or launching a Trusted Application.
* A/B Images: Dividing a Recipient's storage into two or more bootable Images, at different offsets, such that the active Image can write to the inactive Image(s).
* Record: The result of a Command and any metadata about it.
* Report: A list of Records.
* Procedure: The process of invoking one or more sequences of Commands.
* Update Procedure: A Procedure that updates a Recipient by fetching Dependencies and Images, and installing them.
* Invocation Procedure: A Procedure in which a Recipient verifies Dependencies and Images, loading Images, and invokes one or more Image.
* Software: Instructions and data that allow a Recipient to perform a useful function.
* Firmware: Software that is typically changed infrequently, stored in nonvolatile memory, and small enough to apply to {{RFC7228}} Class 0-2 devices.
* Image: Information that a Recipient uses to perform its function, typically Firmware/Software, configuration, or Resource data such as text or images. Also, a Payload, once installed is an Image.
* Slot: One of several possible storage locations for a given Component, typically used in A/B Image systems
* Abort: An event in which the Manifest Processor immediately halts execution of the current Procedure. It creates a Record of an error Condition.
* Trust Anchor: A Trust Anchor, as defined in {{RFC6024}}, represents an
      authoritative entity via a public key and associated data.  The
      public key is used to verify digital signatures, and the
      associated data is used to constrain the types of information for
      which the Trust Anchor is authoritative.

#  Changes to SUIT Workflow Model

The use of the features presented for use with multiple trust domains requires some augmentation of the workflow presented in the SUIT Manifest specification ({{I-D.ietf-suit-manifest}}):

One additional assumption is added for the Update Procedure: 

* All Dependency Manifests must be present before any Payload is fetched.

One additional assumption is added to the Invocation Procedure:

* All Dependencies must be validated prior to loading.

Steps 1 and 4 are added to the expected installation workflow of a Recipient:

1. Verify Delegation Chains.
2. Verify the signature of the Manifest.
3. Verify the applicability of the Manifest.
4. Resolve Dependencies.
5. Fetch Payload(s).
6. Install Payload(s).

In addition, when multiple Manifests are used for an Update, each Manifest's steps occur in a lockstep fashion; all Manifests have Dependency resolution performed before any Manifest performs a Payload fetch, etc.

#  Changes to Manifest Metadata Structure {#metadata-structure-overview}

To accommodate the additional metadata needed to enable these features, the Envelope and Manifest have several new elements added.

The Envelope gains two more elements: Delegation Chains and Integrated Dependencies. The Common metadata section in the Manifest also gains a list of Dependencies.

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
| CoSWID                  |          | Structure Version            |
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

Delegation Chains allow a Recipient to establish a chain of trust from a Trust Anchor to the signer of a Manifest by validating delegation claims. Each delegation claim is a {{RFC8392}} CBOR Web Token (CWT). The first claim in each list is signed by a Trust Anchor. Each subsequent claim in a list is signed by the public key claimed in the preceding list element. The last element in each list claims a public key that can be used to verify a signature in the Authentication Block (See Section 5.2 of {{I-D.ietf-suit-manifest}}).

See {{delegation-info}} for more detail.

##  Delegation Chains {#delegation-info}

The suit-delegation element MAY carry one or more CBOR Web Tokens (CWTs) {{RFC8392}}, with {{RFC8747}} cnf claims. They can be used to perform enhanced authorization decisions. The CWTs are arranged into a list of lists. Each list starts with a CWT authorized by a Trust Anchor, and finishes with a key used to authenticate the Manifest (see Section 8.3 of {{I-D.ietf-suit-manifest}}). This allows an Update Authority to delegate from a long term Trust Anchor, down through intermediaries, to a delegate without any out-of-band provisioning of Trust Anchors or intermediary keys.

A Recipient MAY choose to cache intermediaries and/or delegates. If an intermediary knows that a targeted Recipient has cached some intermediaries or delegates, it MAY choose to strip any cached intermediaries or delegates from the Delegation Chains in order to reduce bandwidth and energy.


#  Dependencies 

A Dependency is another SUIT_Envelope that describes additional Components. 

As described in {{Introduction}}, Dependencies enable several common use cases.

##  Changes to Required Checks {#required-checks}

This section augments the definitions in Required Checks (Section 6.2) of {{I-D.ietf-suit-manifest}}.

More checks are required when handling Dependencies. By default, any signature of a Dependency MUST be verified. However, there are some exceptions to this rule: where a device supports only one level of access (no ACLs defining which authorities have access to different Components/Commands/Parameters), it MAY choose to skip signature verification of Dependencies, since they are verified by digest. Where a device differentiates between trust levels, such as with an ACL, it MAY choose to defer the verification of signatures of Dependencies until the list of affected Components is known so that it can skip redundant signature verifications. For example, if a dependent's signer has access rights to all Components specified in a Dependency, then that Dependency does not require a signature verification. Similarly, if the signer of the dependent has full rights to the device, according to the ACL, then no signature verification is necessary on the Dependency.

Components that should be treated as Dependency Manifests are identified in the suit-common metadata. See {{structure-change}} for details.

If the Manifest contains more than one Component and/or Dependency, each Command sequence MUST begin with a Set Component Index Command.

If a Dependency is specified, then the Manifest processor MUST perform the following checks:

1. The dependent MUST populate all Command sequences for the current Procedure (Update or Invoke).
2. At the end of each section in the dependent: The corresponding section in each Dependency has been executed.

If the interpreter does not support Dependencies and a Manifest specifies a Dependency, then the interpreter MUST Abort.

If a Recipient supports groups of interdependent Components (a Component Set), then it SHOULD verify that all Components in the Component Set are specified by a single Manifest and all its Dependencies that together:

1. have sufficient permissions imparted by their signatures
2. specify a digest and a Payload for every Component in the Component Set.

The single dependent Manifest is sometimes called a Root Manifest.

##  Changes to Manifest Structure {#structure-change}

This section augments the Manifest Structure (Section 8.4) in {{I-D.ietf-suit-manifest}}. 

### Manifest Component ID {#manifest-id}

In complex systems, it may not always be clear where the Root Manifest should be stored; this is particularly complex when a system has multiple, independent Root Manifests. The Manifest Component ID resolves this contention. The manifest-component-id is intended to be used by the Root Manifest. When a Dependency Manifest also declares a Component ID, the Dependency Manifest's Component ID is overridden by the Component ID declared by the dependent.

The following CDDL describes the Manifest Component ID:

~~~CDDL
$$SUIT_Manifest_Extensions //= 
    (suit-manifest-component-id => SUIT_Component_Identifier)
~~~

### SUIT_Dependencies Manifest Element {#SUIT_Dependencies}

The suit-common section, as described in {{I-D.ietf-suit-manifest}}, Section 8.4.5 is extended with a map of Component indices that indicate a Dependency Manifest. The keys of the map are the Component indices and the values of the map are any extra metadata needed to describe those Dependency Manifests.

Because some operations treat Dependency Manifests differently from other Components, it is necessary to identify them. SUIT_Dependencies identifies which Components from suit-components (see Section 8.4.5 of {{I-D.ietf-suit-manifest}}) are to be treated as Dependency Manifest Envelopes. SUIT_Dependencies is a map of Components, referenced by Component Index. Optionally, a Component prefix or other metadata may be delivered with the Component index. The CDDL for suit-dependencies is shown below:

~~~CDDL
$$SUIT_Common-extensions //= (
    suit-dependencies => SUIT_Dependencies
)
SUIT_Dependencies = {
    + uint => SUIT_Dependency_Metadata
}
SUIT_Dependency_Metadata = {
    ? suit-dependency-prefix => SUIT_Component_Identifier
    * $$SUIT_Dependency_Extensions
}
~~~

If no extended metadata is needed for an extension, SUIT_Dependency_Metadata is an empty map (this is the same encoding size as a null). SUIT_Dependencies MUST be sorted according to CBOR canonical encoding.

The Components specified by SUIT_Dependency will contain a Manifest Envelope that describes a Dependency of the current Manifest. The Manifest is identified, but the Recipient should expect an Envelope when it acquires the Dependency. This is because the Manifest is the one invariant element of the Envelope, where other elements may change by countersigning, adding authentication blocks, or severing elements.

When executing suit-condition-image-match over a Component that is designated in SUIT_Dependency, the digest MUST be computed over just the bstr-wrapped SUIT_Manifest contained in the Manifest Envelope designated by the Component Index. This enables a Dependency reference to uniquely identify a particular Manifest structure. This is identical to the digest that is present as the first element of the suit-authentication-block in the Dependency's Envelope. The digest is calculated over the Manifest structure to ensure that removing a signature from a Manifest does not break Dependencies due to missing signature elements. This is also necessary to support the trusted intermediary use case, where an intermediary re-signs the Manifest, removing the original signature, potentially with a different algorithm, or trading COSE_Sign for COSE_Mac.

The suit-dependency-prefix element contains a SUIT_Component_Identifier (see Section 8.4.5.1 of {{I-D.ietf-suit-manifest}}). This specifies the scope at which the Dependency operates. This allows the Dependency to be forwarded on to a Component that is capable of parsing its own Manifests. It also allows one Manifest to be deployed to multiple dependent Recipients without those Recipients needing consistent Component hierarchy. This element is OPTIONAL for Recipients to implement.

A Dependency prefix can be used with a Component identifier. This allows complex systems to understand where Dependencies need to be applied. The Dependency prefix can be used in one of two ways. The first simply prepends the prefix to all Component Identifiers in the Dependency.

A Dependency prefix can also be used to indicate when a Dependency Manifest needs to be processed by a secondary Manifest processor, as described in {{hierarchical-interpreters}}.

##  Changes to Abstract Machine Description

This section augments the Abstract Machine Description (Section 6.4) in {{I-D.ietf-suit-manifest}}.
With the addition of Dependencies, some changes are necessary to the abstract machine, outside the typical scope of added Commands. These changes alter the behaviour of an existing Command and way that the parser processes Manifests:

* Five new Commands are introduced:

    * Set Parameters
    * Process Dependency
    * Is Dependency
    * Dependency Integrity
    * Unlink

* Dependency Manifests are also Components. All Commands may target Dependency Manifests as well as Components, with one exception: process Dependency. Commands defined outside of this draft and {{I-D.ietf-suit-manifest}} MAY have additional restrictions.
* Dependencies are processed in lockstep with the Root Manifest. This means that every Dependency's current Command sequence must be executed before a dependent's later Command sequence may be executed. For example, every Dependency's Dependency Resolution step MUST be executed before any dependent's Payload fetch step.
* When a Manifest Processor supports multiple independent Components, they MAY have shared Dependencies.
* When a Manifest Processor supports shared Dependencies, it MUST support reference counting of those Dependencies.
* When reference counting is used, Components MUST NOT be overwritten. The Manifest Uninstall section must be called, then the component MUST be Unlinked.

##  Processing Dependencies {#processing-dependencies}

As described in {{required-checks}}, each Manifest must invoke each of its Dependencies' sections from the corresponding section of the dependent. Any changes made to Parameters by the Dependency persist in the dependent.

When a Process Dependency Command is encountered, the Manifest processor:

1. Checks whether the map of Dependencies contains an entry for the current Component Index. If not present, it causes an immediate Abort.
2. Checks whether the Dependency has been the target of a Dependency integrity check. If not, it causes an immediate Abort.
2. Loads the specified Component as a Dependency Manifest Envelope.
3. Authenticates the Dependency Manifest.
4. Executes the common-sequence section of the Dependency Manifest.
5. Executes the section of the Dependency Manifest that corresponds to the currently executing section of the dependent.

If the specified Dependency does not contain the current section, Process Dependency succeeds immediately.

The interpreter also performs the checks described in {{required-checks}} to ensure that the dependent is processing the Dependency correctly.

###  Multiple Manifest Processors {#hierarchical-interpreters}

When a system has multiple trust domains, each domain might require independent verification of authenticity or security policies. Trust domains might be divided by separation technology such as Arm TrustZone, Intel SGX, or another Trusted Execution Environment (TEE) technology. Trust domains might also be divided into separate processors and memory spaces, with a communication interface between them.

For example, an application processor may have an attached communications module that contains a processor. The communications module might require metadata signed by a specific Trust Authority for regulatory approval. This may be a different Trust Authority than the application processor.

When there are two or more trust domains, a Manifest processor might be required in each. The first Manifest processor is the normal Manifest processor as described for the Recipient in Section 6 of {{I-D.ietf-suit-manifest}}. The second Manifest processor only executes sections when the first Manifest processor requests it. An API interface is provided from the second Manifest processor to the first. This allows the first Manifest processor to request a limited set of operations from the second. These operations are limited to: setting Parameters, inserting an Envelope, and invoking a Manifest Command Sequence. The second Manifest processor declares a prefix to the first, which tells the first Manifest processor when it should delegate to the second. These rules are enforced by underlying separation of privilege infrastructure, such as TEEs, or physical separation.

When the first Manifest processor encounters a Dependency prefix, that informs the first Manifest processor that it should provide the second Manifest processor with the corresponding Dependency Envelope. This is done when the Dependency is fetched. The second Manifest processor immediately verifies any authentication information in the Dependency Envelope. When a Parameter is set for any Component that matches the prefix, this Parameter setting is passed to the second Manifest processor via an API. As the first Manifest processor works through the Procedure (set of Command sequences) it is executing, each time it sees a Process Dependency Command that is associated with the prefix declared by the second Manifest processor, it uses the API to ask the second Manifest processor to invoke that Dependency section instead.

This mechanism ensures that the two or more Manifest processors do not need to trust each other, except in a very limited case. When Parameter setting across trust domains is used, it must be very carefully considered. Only Parameters that do not have an effect on security properties should be allowed. The Dependency Manifest MAY control which Parameters are allowed to be set by using the Override Parameters Directive. The second Manifest processor MAY also control which Parameters may be set by the first Manifest processor by means of an ACL that lists the allowed Parameters. For example, a URI may be set by a dependent without a substantial impact on the security properties of the Manifest.

##  Dependency Resolution {#suit-dependency-resolution}

The Dependency Resolution Command Sequence is a container for the Commands needed to acquire and process the Dependencies of the current Manifest. All Dependency Manifests SHOULD be fetched before any Payload is fetched to ensure that all Manifests are available and authenticated before any of the (larger) Payloads are acquired.

##  Added and Modified Commands

All Commands are modified in that they can also target Dependencies. However, Set Component Index has a larger modification.

| Command Name | Semantic of the Operation
|------|----
| Set Parameters | current.params\[k\] := v if not k in current.params for-each k,v in arg
| Process Dependency | exec(current\[common\]); exec(current\[current-segment\])
| Dependency Integrity | verify(current, current.params\[image-digest\])
| Is Dependency | assert(current exists in Dependencies)
| Unlink | unlink(current)

### suit-directive-set-parameters {#suit-directive-set-parameters}

Similar to suit-directive-override-parameters, suit-directive-set-parameters allows the Manifest to configure behavior of future Directives by changing Parameters that are read by those Directives. Set Parameters is for use when Dependencies are used because it allows a Manifest to modify the behavior of its Dependencies.

Available Parameters are defined in {{I-D.ietf-suit-manifest}}, section 8.4.8.

If a Parameter is already set, suit-directive-set-parameters will skip setting the Parameter to its argument. This allows dependent Manifests to change the behavior of a Manifest, a Dependency that wishes to enforce a specific value of a Parameter MAY use suit-directive-override-parameters instead.

suit-directive-set-parameters does not specify a reporting policy.


### suit-directive-process-dependency {#suit-directive-process-dependency}

Execute the Commands in the common section of the current Dependency, followed by the Commands in the equivalent section of the current Dependency. For example, if the current section is "Payload Fetch," this will execute "Common metadata" in the current Dependency, then "Payload Fetch" in the current Dependency. Once this is complete, the Command following suit-directive-process-dependency will be processed.

If the current Component index does not have an entry in the suit-dependencies map, then this Command MUST Abort.

If the current Component index has not been the target of a suit-condition-dependency-integrity, then this Command MUST Abort.

If the current Component is True, then this Directive applies to all Dependencies. If the current section is "Common metadata," then the Command sequence MUST Abort.

When SUIT_Process_Dependency completes, it forwards the last status code that occurred in the Dependency.

### suit-condition-is-dependency {#suit-condition-is-dependency}

Check whether the current Component index is present in the Dependency list. If the current Component is in the Dependency list, suit-condition-is-dependency succeeds. Otherwise, it fails. This can be used along with component-id = True to act on all Dependencies or on all non-Dependency Components. See {{creating-manifests}} for more details.

### suit-condition-dependency-integrity {#suit-condition-dependency-integrity}

Verify the integrity of a Dependency Manifest. When a Manifest Processor executes suit-condition-dependency-integrity, it performs the following operations:

1. Evaluate any Delegation Chains
2. Verify the signature of the Manifest hash
3. Compare the Manifest hash to the provided hash
4. Verify the Manifest against the Manifest hash

If any of these steps fails, the Manifest Process MUST immediately Abort.

The Manifest Processor MAY cache the results of these operations for later use from the context of the current Manifest. The Manifest Processor MUST NOT use cached results from any other Manifest context. If the Manifest Processor caches the results of these checks, it MUST eliminate this cache if any Fetch, or Copy operation targets the Dependency Manifest's Component ID. 

### suit-directive-unlink {#suit-directive-unlink}

A manifest processor that supports multiple independent root manifests
MUST support suit-directive-unlink. When a Component is no longer
needed, the Manifest processor unlinks the Component to inform the 
Manifest processor that it is no longer needed.

If a Manifest is no longer needed, the Manifest Processor unlinks it.
This causes the Manifest Processor to execute the suit-uninstall section
of the unlinked Manifest, after which it decrements the reference count
of the unlinked Manifest. The suit-uninstall section of a manifest
typically contains an unlink of all its dependencies and components.

All components, including Manifests must be unlinked before deletion 
or overwrite. If the
reference count of a component is non-zero, any command that alters
that component MUST cause an immediate ABORT. Affected commands are:

* suit-directive-copy
* suit-directive-fetch
* suit-directive-write

The unlink Command decrements an implementation-defined reference counter. This reference counter MUST persist across restarts. The reference counter MUST NOT be decremented by a given Manifest more than once, and the Manifest processor must enforce this. The Manifest processor MAY choose to ignore an Unlink Directive depending on device policy.

When the reference counter of a Manifest reaches zero, the suit-uninstall Command sequence is invoked (see {{suit-uninstall}}).

suit-directive-unlink is OPTIONAL to implement in Manifest processors,
but Manifest processors that support multiple independent Root Manifests
MUST support suit-directive-unlink.

# Uninstall {#suit-uninstall}

In some systems, particularly with multiple, independent, optional Components, it may be that there is a need to uninstall the Components that have been installed by a Manifest. Where this is expected, the uninstall Command sequence can provide the sequence needed to cleanly remove the Components defined by the Manifest and its Dependencies. In general, the suit-uninstall Command Sequence will contain primarily unlink Directives.

WARNING: This can cause faults where there are loose Dependencies (e.g., version range matching, see {{I-D.ietf-suit-update-management}}), since a Component can be removed while it is depended upon by another Component. To avoid Dependency faults, a Manifest author MAY use explicit Dependencies where possible, or a Manifest processor MAY track references to loose Dependencies via reference counting in the same way as explicit Dependencies, as described in {{suit-directive-unlink}}.

The suit-uninstall Command Sequence is not severable, since it must always be available to enable uninstalling.

# Creating Manifests {#creating-manifests}

This section details a set of templates for creating Manifests. These templates explain which Parameters, Commands, and orders of Commands are necessary to achieve a stated goal.

## Dependency Template {#template-dependency}

The goal of the Dependency template is to obtain, verify, and process a Dependency Manifest as appropriate.

The following Commands are added to the shared sequence:

- Set Component Index Directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Set Parameters Directive (see {{suit-directive-set-parameters}}) for digest (see Section 8.4.8.6 of {{I-D.ietf-suit-manifest}}). Note that the digest MUST match the SUIT_Digest in the Dependency's suit-authentication-block (see Section 8.3 of {{I-D.ietf-suit-manifest}}).

The following Commands are placed into the Dependency resolution sequence:

- Set Component Index Directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Set Parameters Directive (see {{suit-directive-set-parameters}}) for a URI (see Section 8.4.8.10 of {{I-D.ietf-suit-manifest}})
- Fetch Directive (see Section 8.4.10.4 of {{I-D.ietf-suit-manifest}})
- Dependency Integrity Condition (see {{suit-condition-dependency-integrity}})
- Process Dependency Directive (see {{suit-directive-process-dependency}})

Then, the validate sequence contains the following operations:

- Set Component Index Directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Dependency Integrity Condition (see {{suit-condition-dependency-integrity}})
- Process Dependency Directive (see {{suit-directive-process-dependency}})

If any Dependency is declared, the dependent MUST populate all Command sequences for the current Procedure (Update or Invoke).

NOTE: Any changes made to Parameters in a Dependency persist in the dependent.

### Integrated Dependencies {#integrated-dependencies}

An implementer MAY choose to place a Dependency's Envelope in the Envelope of its dependent. The dependent Envelope key for the Dependency Envelope MUST be a text string. The URI for the Dependency MUST match the text string key of the dependent's Envelope key. It is RECOMMENDED to make the text string key a resolvable URI so that a Dependency Manifest that is removed from the Envelope can still be fetched.

## Encrypted Manifest Template {#template-encrypted-manifest}

The goal of the Encrypted Manifest template is to fetch and decrypt a Manifest so that it can be used as a Dependency. To use an encrypted Manifest, create a plaintext dependent, and add the encrypted Manifest as a Dependency. The dependent can include very little information.

NOTE: This template also requires the extensions defined in {{I-D.ietf-suit-firmware-encryption}}.

The following Commands are added to the shared sequence:

- Set Component Index Directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Set Parameters Directive (see {{suit-directive-set-parameters}}) for digest (see Section 8.4.8.6 of {{I-D.ietf-suit-manifest}}). Note that the digest MUST match the SUIT_Digest in the Dependency's suit-authentication-block (see Section 8.3 of {{I-D.ietf-suit-manifest}}).

The following operations are placed into the Dependency resolution block:

- Set Component Index Directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Set Parameters Directive (see {{suit-directive-set-parameters}}) for
    - URI (see Section 8.4.8.9 of {{I-D.ietf-suit-manifest}})
    - Encryption Info (See {{I-D.ietf-suit-firmware-encryption}})
- Fetch Directive (see Section 8.4.10.4 of {{I-D.ietf-suit-manifest}})
- Dependency Integrity Condition (see {{suit-condition-dependency-integrity}})
- Process Dependency Directive (see {{suit-directive-process-dependency}})

Then, the validate block contains the following operations:

- Set Component Index Directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Check Image Match Condition (see Section 8.4.9.2 of {{I-D.ietf-suit-manifest}})
- Process Dependency Directive (see {{suit-directive-process-dependency}})

A plaintext Manifest and its encrypted Dependency may also form a composite Manifest ({{integrated-dependencies}}).

## Overriding Encryption Info Template {#template-override-encryption-info}

The goal of overriding the Encryption Info template is to separate the role of generating encrypted Payload and Encryption Info with Key-Encryption Key addressing Section 3 of {{I-D.ietf-suit-firmware-encryption}}.

As an example, this template describes two manifests:
- The dependent Manifest created by the Distribution System contains Encryption Info, allowing the Device to generate the Content-Encryption Key.
- The dependency Manifest created by the Author contains Commands to decrypt the encrypted Payload using Encryption Info above and to validate the plaintext Payload with SUIT_Digest.

NOTE: This template also requires the extensions defined in {{I-D.ietf-suit-firmware-encryption}}.

The following operations are placed into the Dependency resolution block of dependent Manifest:

- Set Component Index Directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}}) pointing at dependency Manifest
- Set Parameters Directive (see {{suit-directive-set-parameters}}) for
    - Image Digest (see Section 8.4.8.6 of {{I-D.ietf-suit-manifest}})
    - URI (see Section 8.4.8.9 of {{I-D.ietf-suit-manifest}}) of dependency Manifest
- Fetch Directive (see Section 8.4.10.4 of {{I-D.ietf-suit-manifest}})
- Dependency Integrity Condition (see {{suit-condition-dependency-integrity}})

The following Commands are placed into the Fetch/Install block of dependent Manifest

- Set Component Index Directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}}) pointing at encrypted Payload
- Set Parameters Directive (see {{suit-directive-set-parameters}}) for
    - URI (see Section 8.4.8.9 of {{I-D.ietf-suit-manifest}})
- Set Component Index Directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}}) pointing at dependency Manifest
- Set Parameters Directive (see {{suit-directive-set-parameters}}) for
    - Encryption Info (See {{I-D.ietf-suit-firmware-encryption}})
- Process Dependency Directive (see {{suit-directive-process-dependency}})

The following Commands are placed into the same block of dependency Manifest:

- Set Component Index Directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}}) pointing at encrypted Payload
- Fetch Directive (see Section 8.4.10.4 of {{I-D.ietf-suit-manifest}})
- Set Component Index Directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}}) pointing at to be decrypted Payload
- Override Parameters Directive (see Section 8.4.10.3 of {{I-D.ietf-suit-manifest}}) for
    - Source Component (see Section 8.4.8.11 of {{I-D.ietf-suit-manifest}}) pointing at encrypted Payload
- Copy Directive (see Section 8.4.10.5 of {{I-D.ietf-suit-manifest}}) consuming the Encryption Info above

The Distribution System can Set the Parameter URI in the Fetch/Install block of dependent Manifest if it wants to overwrite the URI of encrypted Payload.

Because the Author and the Distribution System have different roles and MAY be separate entities, it is highly RECOMMENDED to leverage permissions (see Section 9 of {{I-D.ietf-suit-manifest}}).
For example, The Device can protect itself from attacker who breaches the Distribution System by allowing only the Author's Manifest to modify the Component of (to be) decrypted Payload.

## Operating on Multiple Components

In order to produce compact encoding, it is efficient to perform operations on multiple Components simultaneously. Because Dependency Manifests and Component Images are processed at different times, there is a mechanism to distinguish between these elements: suit-condition-is-dependency. This can be used with suit-directive-try-each to perform operations just on Dependency Manifests or just on Component Images.

For example, to fetch all Dependency Manifests, the following Commands are added to the Dependency resolution block:

- Set Component Index Directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Set Parameters Directive (see {{suit-directive-set-parameters}}) for a URI (see Section 8.4.8.9 of {{I-D.ietf-suit-manifest}})
- Set Component Index Directive, with argument "True" (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Try Each Directive
    - Sequence 0
        - Condition Is Dependency Manifest
        - Fetch
        - Dependency Integrity Condition (see {{suit-condition-dependency-integrity}})
        - Process Dependency
    - Sequence 1 (Empty; no Commands, succeeds immediately)

Another example is to fetch and validate all Component Images. The Image fetch sequence contains the following Commands:

- Set Component Index Directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Set Parameters Directive (see {{suit-directive-set-parameters}}) for a URI (see Section 8.4.8.9 of {{I-D.ietf-suit-manifest}})
- Set Component Index Directive, with argument "True" (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Try Each Directive
    - Sequence 0
        - Condition Is Dependency Manifest
        - Process Dependency
    - Sequence 1
        - Fetch
        - Condition Image Match

When some Components are "installed" or "loaded" it is more productive to use lists of Component indices rather than Component Index = True. For example, to install several Components, the following Commands should be placed in the Image Install Sequence:

- Set Component Index Directive (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Set Parameters Directive (see {{suit-directive-set-parameters}}) for the Source Component (see Section 8.4.8.11 of {{I-D.ietf-suit-manifest}})
- Set Component Index Directive, with argument containing list of destination Component indices (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Copy
- Set Component Index Directive, with argument containing list Dependency Component indices (see Section 8.4.10.1 of {{I-D.ietf-suit-manifest}})
- Process Dependency

#  IANA Considerations {#iana}

IANA is requested to allocate the following numbers in the listed registries created by draft-ietf-suit-manifest:

## SUIT Envelope Elements

Label | Name | Reference
---|---|---
1  | Delegation | {{ovr-delegation}}
15 | Dependency Resolution | {{suit-dependency-resolution}}

## SUIT Manifest Elements

Label | Name | Reference
---|---|---
5 | Manifest Component ID | {{manifest-id}}
15 | Dependency Resolution | {{suit-dependency-resolution}}
24 | Uninstall | {{suit-uninstall}}

## SUIT Common Elements

Label | Name | Reference
---|---|---
1 | Dependencies | {{SUIT_Dependencies}}

## SUIT Commands

Label | Name | Reference
---|---|---
7 | Dependency Integrity | {{suit-condition-dependency-integrity}}
8 | Is Dependency | {{suit-condition-is-dependency}}
11 | Process Dependency | {{suit-directive-process-dependency}}
19 | Set Parameters | {{suit-directive-set-parameters}}
33 | Unlink | {{suit-directive-unlink}}

#  Security Considerations

This document is about a Manifest format protecting and describing how to retrieve, install, and invoke Images and as such it is part of a larger solution for delivering software updates to devices. A detailed security treatment can be found in the architecture {{RFC9019}} and in the information model {{RFC9124}} documents.


--- back

# A. Full CDDL {#full-cddl}

To be valid, the following CDDL MUST be appended to the SUIT Manifest CDDL. The SUIT CDDL is defined in Appendix A of {{I-D.ietf-suit-manifest}}

~~~ CDDL
{::include draft-ietf-suit-trust-domains.cddl}
~~~

# B. Examples {#examples}

The following examples demonstrate a small subset of the functionalities in this document.

The examples are signed using the following ECDSA secp256r1 key:

~~~
-----BEGIN PRIVATE KEY-----
MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQgApZYjZCUGLM50VBC
CjYStX+09jGmnyJPrpDLTz/hiXOhRANCAASEloEarguqq9JhVxie7NomvqqL8Rtv
P+bitWWchdvArTsfKktsCYExwKNtrNHXi9OB3N+wnAUtszmR23M4tKiW
-----END PRIVATE KEY-----
~~~

The corresponding public key can be used to verify these examples:

~~~
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEhJaBGq4LqqvSYVcYnuzaJr6qi/Eb
bz/m4rVlnIXbwK07HypLbAmBMcCjbazR14vTgdzfsJwFLbM5kdtzOLSolg==
-----END PUBLIC KEY-----
~~~

Each example uses SHA256 as the digest function.

## Example 0: Delegation Chain

This example uses functionalities:

* manifest component id
* delegation chain

{::include examples/example0_delegation.txt}

## Example 1: Process Dependency

This example uses functionalities:

* manifest component id
* dependency resolution
* process dependency

{::include examples/example1_process.txt}

## Example 2: Integrated Dependency

* manifest component id
* dependency resolution
* process dependency
* integrated dependency

{::include examples/example2_integrated.txt}
