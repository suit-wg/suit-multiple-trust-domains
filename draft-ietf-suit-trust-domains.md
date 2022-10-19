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
| Dependencies            |      +-> +-----------------------+
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

By using dependencies, components such as software, configuration, models, and other resoruces authenticated by different trust anchors can be delivered to devices.

##  Changes to Required Checks {#required-checks}

This section augments the definitions in Required Checks (Section 6.2) of {{I-D.ietf-suit-manifest}}.

More checks are required when handling dependencies. By default, any signature of a dependency MUST be verified. However, there are some exceptions to this rule: where a device supports only one level of access (no ACLs defining which authorities have access to different componetns), it MAY choose to skip signature verification of dependencies, since they are referenced by digest. Where a device differentiates between trust levels, such as with an ACL, it MAY choose to defer the verification of signatures of dependencies until the list of affected components is known so that it can skip redundant signature verifications. For example, a dependency signed by the same author as the dependent does not require a signature verification. Similarly, if the signer of the dependent has full rights to the device, according to the ACL, then no signature verification is necessary on the dependency.

If the manifest contains more than one component and/or dependency, each command sequence MUST begin with a Set Component Index or Set Dependency Index command.

If a dependency is specified, then the manifest processor MUST perform the following checks:

1. At the beginning of each section in the dependent: all previous sections of each dependency have been executed.
2. At the end of each section in the dependent: The corresponding section in each dependency has been executed.

If the interpreter does not support dependencies and a manifest specifies a dependency, then the interpreter MUST reject the manifest.

If a Recipient supports groups of interdependent components (a Component Set), then it SHOULD verify that all Components in the Component Set are specified by one update, that is: a single manifest and all its dependencies that together:

1. have sufficient permissions imparted by their signatures
2. specify a digest and a payload for every Component in the Component Set.

The single dependent manifest is sometimes called a Root Manifest.

##  Changes to Abstract Machine Description

This section augments the Abstract Machine Description (Section 6.4) in {{I-D.ietf-suit-manifest}}
With the addition of dependencies, some changes are necessary to the abstract machine, outside the typical scope of added commands. These changes alter the behaviour of an existing command and way that the parser processes manifests:

* All commands may target dependency manifests as well as components. To support this behaviour, there is a new command instroduced: Set Dependency Index. This change works together with Set Component Index to choose the object on which the manifest is operating.
* Dependencies are processed in lock-step with the Root Manifest. This means that every dependency's current command sequence must be executed before a dependent's later command sequence may be executed. For example, every dependency's Dependency Resolution step MUST be executed before any dependent's payload fetch step.

The logic of Set Componment Index is modified as below:

As in {{I-D.ietf-suit-manifest}}, To simplify the logic describing the command semantics, the object "current" is used. It represents the component identified by the Component Index or the dependency identified by the Dependency Index:

~~~
current := components\[component-index\]
    if component-index is not false
    else dependencies\[dependency-index\]
~~~

As a result, Set Component Index is described as current := components\[arg\]. The actual operation performed for Set Component Index is described by the following pseudocode, however, because of the definition of current (above), these are semantically equivalent.

~~~
component-index := arg
dependency-index := false
~~~

Similarly, Set Dependency Index is semantically equivalent to current := dependencies\[arg\], but the actual operation performed is:

~~~
dependency-index := arg
component-index := false
~~~

Dependencies are identified by digest, but referenced in commands by Dependency Index, the index into the array of Dependencies.

##  Changes to Special Cases of Component Index and Dependency Index {#index-true}

The considerations that apply in Special Cases of Component Index and Dependency Index (Section 6.5) of {{I-D.ietf-suit-manifest}} are augmented to include Dependency Index as well as Component Index.

The target(s) assigned for each command are defined by the following pseudocode.

~~~
if component-index is true:
    current-list = components
else if component-index is array:
    current-list = [ components[idx] for idx in component-index ]
else if component-index is integer:
    current-list = [ components[component-index] ]
else if dependency-index is true:
    current-list = dependencies
else if dependency-index is array:
    current-list = [ dependencies[idx] for idx in dependency-index ]
else:
    current-list = [ dependencies[dependency-index] ]
for current in current-list:
    cmd(current)
~~~

##  Processing Dependencies {#processing-dependencies}

As described in {{required-checks}}, each manifest must invoke each of its dependencies' sections from the corresponding section of the dependent. Any changes made to parameters by the dependency persist in the dependent.

When a Process Dependency command is encountered, the interpreter loads the dependency identified by the Current Dependency Index. The interpreter first executes the common-sequence section of the identified dependency, then it executes the section of the dependency that corresponds to the currently executing section of the dependent.

If the specified dependency does not contain the current section, Process Dependency succeeds immediately.

The Manifest Processor MUST also support a Dependency Index of True, which applies to every dependency, as described in {{index-true}}

The interpreter also performs the checks described in {{required-checks}} to ensure that the dependent is processing the dependency correctly.

###  Multiple Manifest Processors {#hierarchical-interpreters}

When a system has multiple security domains, each domain might require independent verification of authenticity or security policies. Security domains might be divided by separation technology such as Arm TrustZone, Intel SGX, or another TEE technology. Security domains might also be divided into separate processors and memory spaces, with a communication interface between them.

For example, an application processor may have an attached communications module that contains a processor. The communications module might require metadata signed by a specific Trust Authority for regulatory approval. This may be a different Trust Authority than the application processor.

When there are two or more security domains (see {{I-D.ietf-teep-architecture}}), a manifest processor might be required in each. The first manifest processor is the normal manifest processor as described for the Recipient in Section 6 of {{I-D.ietf-suit-manifest}}. The second manifest processor only executes sections when the first manifest processor requests it. An API interface is provided from the second manifest processor to the first. This allows the first manifest processor to request a limited set of operations from the second. These operations are limited to: setting parameters, inserting an Envelope, invoking a Manifest Command Sequence. The second manifest processor declares a prefix to the first, which tells the first manifest processor when it should delegate to the second. These rules are enforced by underlying separation of privilege infrastructure, such as TEEs, or physical separation.

When the first manifest processor encounters a dependency prefix, that informs the first manifest processor that it should provide the second manifest processor with the corresponding dependency Envelope. This is done when the dependency is fetched. The second manifest processor immediately verifies any authentication information in the dependency Envelope. When a parameter is set for any component that matches the prefix, this parameter setting is passed to the second manifest processor via an API. As the first manifest processor works through the Procedure (set of command sequences) it is executing, each time it sees a Process Dependency command that is associated with the prefix declared by the second manifest processor, it uses the API to ask the second manifest processor to invoke that dependency section instead.

This mechanism ensures that the two or more manifest processors do not need to trust each other, except in a very limited case. When parameter setting across security domains is used, it must be very carefully considered. Only parameters that do not have an effect on security properties should be allowed. The dependency manifest MAY control which parameters are allowed to be set by using the Override Parameters directive. The second manifest processor MAY also control which parameters may be set by the first manifest processor by means of an ACL that lists the allowed parameters. For example, a URI may be set by a dependent without a substantial impact on the security properties of the manifest.

##  Dependency Resolution {#suit-dependency-resolution}

The Dependency Resolution Command Sequence is a container for the commands needed to acquire and process the dependencies of the current manifest. Ideally, all dependency manifests should fetched before any payload is fetched to ensure that all manifests are available and authenticated before any of the (larger) payloads are acquired.

##  Added and Modified Commands

All commands are modified in that they can also target dependencies. However, Set Component Index has a larger modification.

| Command Name | Semantic of the Operation
|------|----
| Set Component Index | current := components\[arg\]
| Set Dependency Index | current := dependencies\[arg\]
| Set Parameters | current.params\[k\] := v if not k in params for-each k,v in arg
| Process Dependency | exec(current\[common\]); exec(current\[current-segment\])
| Unlink | unlink(current)

####  suit-directive-set-component-index {#suit-directive-set-component-index}

Set Component Index defines the component to which successive directives and conditions will apply. The supplied argument MUST be one of three types:

1. An unsigned integer (REQUIRED to implement in parser)
2. A boolean (REQUIRED to implement in parser ONLY IF 2 or more components supported)
3. An array of unsigned integers (REQUIRED to implement in parser ONLY IF 3 or more components supported)

If the following commands apply to ONE component, an unsigned integer index into the component list is used. If the following commands apply to ALL components, then the boolean value "True" is used instead of an index. If the following commands apply to more than one, but not all components, then an array of unsigned integer indices into the component list is used.
See {{index-true}} for more details.

If the following commands apply to NO components, then the boolean value "False" is used. When suit-directive-set-dependency-index is used, suit-directive-set-component-index = False is implied. When suit-directive-set-component-index is used, suit-directive-set-dependency-index = False is implied.

If component index is set to True when a command is invoked, then the command applies to all components, in the order they appear in suit-common-components. When the Manifest Processor invokes a command while the component index is set to True, it must execute the command once for each possible component index, ensuring that the command receives the parameters corresponding to that component index.

### suit-directive-set-dependency-index {#suit-directive-set-dependency-index}

Set Dependency Index defines the manifest to which successive directives and conditions will apply. The supplied argument MUST be either a boolean or an unsigned integer index into the dependencies, or an array of unsigned integer indices into the list of dependencies. If the following directives apply to ALL dependencies, then the boolean value "True" is used instead of an index. If the following directives apply to NO dependencies, then the boolean value "False" is used. When suit-directive-set-component-index is used, suit-directive-set-dependency-index = False is implied. When suit-directive-set-dependency-index is used, suit-directive-set-component-index = False is implied.

If dependency index is set to True when a command is invoked, then the command applies to all dependencies, in the order they appear in suit-common-components. When the Manifest Processor invokes a command while the dependency index is set to True, the Manifest Processor MUST execute the command once for each possible dependency index, ensuring that the command receives the parameters corresponding to that dependency index. If the dependency index is set to an array of unsigned integers, then the Manifest Processor MUST execute the command once for each listed dependency index, ensuring that the command receives the parameters corresponding to that dependency index.

See {{index-true}} for more details.

Typical operations that require suit-directive-set-dependency-index include setting a source URI or Encryption Information, invoking "Fetch," or invoking "Process Dependency" for an individual dependency.

### suit-directive-process-dependency {#suit-directive-process-dependency}

Execute the commands in the common section of the current dependency, followed by the commands in the equivalent section of the current dependency. For example, if the current section is "fetch payload," this will execute "common" in the current dependency, then "fetch payload" in the current dependency. Once this is complete, the command following suit-directive-process-dependency will be processed.

If the current dependency is False, this directive has no effect. If the current dependency is True, then this directive applies to all dependencies. If the current section is "common," then the command sequence MUST be terminated with an error.

When SUIT_Process_Dependency completes, it forwards the last status code that occurred in the dependency.

#### suit-directive-set-parameters {#suit-directive-set-parameters}

suit-directive-set-parameters allows the manifest to configure behavior of future directives by changing parameters that are read by those directives. When dependencies are used, suit-directive-set-parameters also allows a manifest to modify the behavior of its dependencies.

If a parameter is already set, suit-directive-set-parameters will skip setting the parameter to its argument. This provides the core of the override mechanism, allowing dependent manifests to change the behavior of a manifest.

suit-directive-set-parameters does not specify a reporting policy.

### suit-directive-unlink {#suit-directive-unlink}

suit-directive-unlink marks the current component as unused in the current manifest. This can be used to remove temporary storage or remove components that are no longer needed. Example use cases:

* Temporary storage for encrypted download
* Temporary storage for verifying decompressed file before writing to flash
* Removing Trusted Service no longer needed by Trusted Application

Once the current Command Sequence is complete, the manifest processors checks each marked component to see whether any other manifests have referenced it. Those marked components with no other references are deleted. The manifest processor MAY choose to ignore a Unlink directive depending on device policy.

suit-directive-unlink is OPTIONAL to implement in manifest processors.

## SUIT_Dependency Manifest Element {#SUIT_Dependency}

SUIT_Dependency specifies a manifest that describes a dependency of the current manifest. The Manifest is identified, but the Recipient should expect an Envelope when it acquires the dependency. This is because the Manifest is the one invariant element of the Envelope, where other elements may change by countersigning, adding authentication blocks, or severing elements.

The suit-dependency-digest specifies the dependency manifest uniquely by identifying a particular Manifest structure. This is identical to the digest that would be present as the payload of any suit-authentication-block in the dependency's Envelope. The digest is calculated over the Manifest structure instead of the COSE Sig_structure or Mac_structure. This is necessary to ensure that removing a signature from a manifest does not break dependencies due to missing signature elements. This is also necessary to support the trusted intermediary use case, where an intermediary re-signs the Manifest, removing the original signature, potentially with a different algorithm, or trading COSE_Sign for COSE_Mac.

The suit-dependency-prefix element contains a SUIT_Component_Identifier (see Section 8.4.5.1 of {{I-D.ietf-suit-manifest}}). This specifies the scope at which the dependency operates. This allows the dependency to be forwarded on to a component that is capable of parsing its own manifests. It also allows one manifest to be deployed to multiple dependent Recipients without those Recipients needing consistent component hierarchy. This element is OPTIONAL for Recipients to implement.

A dependency prefix can be used with a component identifier. This allows complex systems to understand where dependencies need to be applied. The dependency prefix can be used in one of two ways. The first simply prepends the prefix to all Component Identifiers in the dependency.

A dependency prefix can also be used to indicate when a dependency manifest needs to be processed by a secondary manifest processor, as described in {{hierarchical-interpreters}}.

# Uninstall {#suit-uninstall}

In some systems, particularly with multiple, independent, optional components, it may be that there is a need to uninstall the components that have been installed by a manifest. Where this is expected, the uninstall command sequence can provide the sequence needed to cleanly remove the components defined by the manifest and its dependencies. In general, suit uninstall will contain primarily unlink directives.

WARNING: This can cause faults where there are loose dependencies (e.g. version-only, see {{I-D.ietf-suit-update-management}}), since a component can be removed while it is depended upon by another component.

The Uninstall command sequence is not severable, since it must always be available to enable uninstalling.

# Creating Manifests {#creating-manifests}

This section details a set of templates for creating manifests. These templates explain which parameters, commands, and orders of commands are necessary to achieve a stated goal.

## Dependency Template {#template-dependency}

The goal of the Dependency template is to obtain, verify, and process a dependency manifest as appropriate.

The following commands are placed into the dependency resolution sequence:

- Set Dependency Index directive (see {{suit-directive-set-dependency-index}})
- Set Parameters directive (see {{suit-directive-set-parameters}}) for URI (see Section 8.4.8.9 of {{I-D.ietf-suit-manifest}})
- Fetch directive (see Section 8.4.10.4 of {{I-D.ietf-suit-manifest}})
- Check Image Match condition (see Section 8.4.9.2 of {{I-D.ietf-suit-manifest}} of {{I-D.ietf-suit-manifest}})
- Process Dependency directive (see {{suit-directive-process-dependency}})

Then, the validate sequence contains the following operations:

- Set Dependency Index directive (see {{suit-directive-set-dependency-index}})
- Check Image Match condition (see Section 8.4.9.2 of {{I-D.ietf-suit-manifest}})
- Process Dependency directive (see {{suit-directive-process-dependency}})

NOTE: Any changes made to parameters in a dependency persist in the dependent.

### Composite Manifests {#composite-manifests}

An implementer MAY choose to place a dependency's envelope in the envelope of its dependent. The dependent envelope key for the dependency envelope MUST NOT be a value between 0 and 24 and it MUST NOT be used by any other envelope element in the dependent manifest.

The URI for a dependency enclosed in this way MUST be expressed as a fragment-only reference, as defined in {{RFC3986}}, Section 4.4. The fragment identifier is the stringified envelope key of the dependency. For example, an envelope that contains a dependency at key 42 would use a URI "#42", key -73 would use a URI "#-73".

## Encrypted Manifest Template {#template-encrypted-manifest}

The goal of the Encrypted Manifest template is to fetch and decrypt a manifest so that it can be used as a dependency. To use an encrypted manifest, create a plaintext dependent, and add the encrypted manifest as a dependency. The dependent can include very little information.

NOTE: This template also requires the extensions defined in {{I-D.ietf-suit-firmware-encryption}}

The following operations are placed into the dependency resolution block:

- Set Dependency Index directive (see {{suit-directive-set-dependency-index}})
- Set Parameters directive (see {{suit-directive-set-parameters}}) for
    - URI (see Section 8.4.8.9 of {{I-D.ietf-suit-manifest}})
    - Encryption Info (See {{I-D.ietf-suit-firmware-encryption}})
- Fetch directive (see Section 8.4.10.4 of {{I-D.ietf-suit-manifest}})
- Check Image Match condition (see Section 8.4.9.2 of {{I-D.ietf-suit-manifest}})
- Process Dependency directive (see {{suit-directive-process-dependency}})

Then, the validate block contains the following operations:

- Set Dependency Index directive (see {{suit-directive-set-dependency-index}})
- Check Image Match condition (see Section 8.4.9.2 of {{I-D.ietf-suit-manifest}})
- Process Dependency directive (see {{suit-directive-process-dependency}})

A plaintext manifest and its encrypted dependency may also form a composite manifest ({{composite-manifests}}).



#  IANA Considerations {#iana}

IANA is requested to allocate the following numbers in the listed registries:

## SUIT Command Sequences

Label | Name | Reference
---|---|---
7  | Dependency Resolution | 
24 | Uninstall | {{suit-uninstall}}

## SUIT Commands

Label | Name | Reference
---|---|---
13 | Set Dependency Index | {{suit-directive-set-dependency-index}}
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
