---
v: 3

title: Software Update for the Internet of Things (SUIT) Manifest Extensions for Multiple Trust Domain
abbrev: SUIT Trust Domains
docname: draft-ietf-suit-trust-domains-11
category: std
stream: IETF

area: Security
workgroup: SUIT
keyword: Internet-Draft

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
 -    name: Brendan Moran
      organization: Arm Limited
      email: brendan.moran.ietf@gmail.com

 -    name: Ken Takayama
      organization: SECOM CO., LTD.
      email: ken.takayama.ietf@gmail.com

normative:
  I-D.ietf-suit-manifest:
  I-D.ietf-suit-firmware-encryption:
  RFC8610:
  RFC8949:

informative:
  I-D.ietf-suit-update-management:
  I-D.ietf-iotops-7228bis:
  RFC6024:
  RFC9019:
  RFC9124:
  RFC9397:

--- abstract

A device has
more than one trust domain when it enables delegation of different
rights to mutually distrusting entities for use for different
purposes or Components in the context of firmware or software update. This
specification describes extensions to the Software Update for the Internet
of Things (SUIT) Manifest format for use in deployments with multiple trust
domains.

--- middle

#  Introduction {#Introduction}

Devices that require more advanced configurations than a Manifest signed by a
single authority also require more complex rules for deploying software updates. For example, devices may require:

* Components from multiple software signing authorities
* a mechanism to remove an unneeded Component
* Dependencies delivered in the same envelope as the Manifest
* a partly encrypted Manifest so that distribution does not reveal private information
* installation performed by a different execution mode than payload fetch

Devices implementing this specification typically partition their software, dividing it, according to physical or logical features, into multiple "domains" with different requirements for authorities: multiple trust domains. Because of the more complex use cases that are typically targetted by devices implementing this specification, the applicable device class is typically Class 2 or higher and often isolation level Is8, for example Arm TrustZone for Cortex-M, as described in {{I-D.ietf-iotops-7228bis}}.

Dependencies enable several additional use cases. In particular, they enable two or more entities who are trusted for different privileges to coordinate. This can be used in many scenarios. For example:

* Devices with network interface controllers (NICs), including radios, may contain secondary processors in the NICs in addition to the device primary processor. These two processors may have separate Software with separate signing authorities. Dependencies allow the Manifest for the primary processor to reference a Manifest signed by a different authority.
* A network operator may wish to provide local caching of Update Payloads. The network creates a Dependent Manifest that provides a different URI for any Payloads they wish to cache the parameter override mechanism described in {{suit-directive-set-parameters}}.
* A Device Administrator provides a device with some additional configuration. The Device Administrator wants to test their configuration with each new Software version before releasing it. The configuration is delivered as a binary in the same way as a Software Image. The Device Administrator references the Software Manifest from the Software author in their own Manifest which also defines the configuration.
* An Author wants to entrust a Distributor to provide devices with firmware decryption keys, but not permit the Distributor to sign code. Dependencies allow the Distributor to deliver a device's decryption information without also granting code signing authority.
* A Trusted Application Manager (TAM) wants to distribute personalisation information to a Trusted Execution Environment in addition to a Trusted Application (TA), but does not have code signing authority (see {{RFC9397}}, Section 2). Dependencies enable the TAM to construct an update containing the personalisation information and a dependency on the TA, but leaves the TA signed by the TA's Author.

When a system has multiple trust domains, each domain might require independent verification of authenticity or security policies. Trust domains might be divided by separation technology such as Arm TrustZone, Intel SGX, or another Trusted Execution Environment (TEE) technology. Trust domains might also be divided into separate processors and memory spaces, with a communication interface between them.

For example, an application processor may have an attached communications module that contains a processor. The communications module might require metadata signed by a specific Trust Authority for regulatory approval. This may be a different Trust Authority than the application processor.

Dependencies enable Components such as Software, configuration, and other Resource data authenticated by different Trust Anchors to be delivered to devices.

These mechanisms are not part of the core Manifest specification ({{I-D.ietf-suit-manifest}}), but they are needed for more advanced use cases, such as the architecture described in {{RFC9397}}.

This specification extends the SUIT Manifest specification ({{I-D.ietf-suit-manifest}}) with:

* Integrated Components
* Dependencies
* Manifest Component Identifier
* Candidate Verification
* Parameter Override support
* Uninstall support


#  Conventions and Terminology

{::boilerplate bcp14}

The terminology from {{I-D.ietf-suit-manifest}}, Section 2 and {{RFC9397}}, Section 2 is used in this specification. Additionally, the following terminology is used:

* Dependency: A Manifest that is required by a second Manifest in order for operations described by the second Manifest to complete successfully.
* Dependent: A Manifest that depends on another Mani
* Root Manifest: A manifest that has no dependents and, combined with all Dependencies (recursively) specifies a complete Component Set.
* Staging Procedure: A procedure that fetches dependencies and images referenced by an Update and stores them to a Staging Area.
* Installation Procedure: A procedure that installs dependencies and images stored in a Staging Area; copying (and optionally, transforming them) into an active Image storage location.
* Staging Area: A Component or group of Components that are used for transient storage of Images between fetch and installation. Images in this area are opaque, except for use by the Installation Procedure.
* Reference Count: An implementation-defined mechanism to track the number of manifests that refer to another manifest.

#  Changes to SUIT Workflow Model

The use of the features presented for use with multiple trust domains requires some augmentation of the workflow presented in the SUIT Manifest specification ({{I-D.ietf-suit-manifest}}):

One additional assumption is added to the list of assumptions for the Update Procedure in {{I-D.ietf-suit-manifest}}, Section 4.2: 

* All Dependencies must be fetched and integrity checked before any Payload is fetched.

One additional assumption is added to the list of assumptions for the Invocation Procedure in {{I-D.ietf-suit-manifest}}, Section 4.2:

* All Dependencies must be validated prior to loading.

Steps 3 and 5 are added to the expected installation workflow of a Recipient:

1. Verify the signature of the Manifest.
2. Verify the applicability of the Manifest.
3. Resolve Dependencies.
4. Fetch Payload(s).
5. Verify Candidate Component Set.
6. Install Payload(s).
7. Verify image(s).

In addition, when multiple Manifests are used for an Update, each Manifest's steps occur in a lockstep fashion: all Manifests have Dependency resolution performed before any Manifest performs a Payload fetch, etc.
The lockstep process is described in {{processing-dependencies}}.

#  Changes to Manifest Metadata Structure {#metadata-structure-overview}

To accommodate the additional metadata needed to enable these features, the Envelope and Manifest are augmented with several new elements:

* Envelope

    * Integrated Dependency

* Manifest

    * Common

        * Dependency Metadata

    * Component Identifier
    * Dependency Resolution SUIT\_Command\_Sequence
    * Candidate Verification SUIT\_Command\_Sequence

In addition several new SUIT\_Commands are added:

* SUIT Conditions

    * Dependency Integrity Check
    * Component Is Dependency Check

* SUIT Directives

    * Process Dependency
    * Set Parameters
    * Unlink


The Envelope gains two more elements: Integrated Dependencies and Integrated Payloads.
The Common metadata section in the Manifest also gains a list of Dependencies.

The new metadata structure is shown below.

~~~
+-------------------------+
| Envelope                |
+-------------------------+
| Authentication Block    |
| Manifest           --------------> +------------------------------+
| Severable Elements      |          | Manifest                     |
| Integrated Dependencies |          +------------------------------+
| Integrated Payloads     |          | Structure Version            |
+-------------------------+          | Sequence Number              |
                                     | Reference to Full Manifest   |
                               +------ Common Structure             |
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
{: #fig-metadata-structure artwork-align="left" title="SUIT Metadata Structure"}


This is an update of the figure in Section 4.2 of {{I-D.ietf-suit-manifest}}

#  Dependencies 

A Dependency is another SUIT_Envelope ({{I-D.ietf-suit-manifest}}, section 8.2) that describes additional Components. 


##  Changes to Required Checks {#required-checks}

This section augments the definitions in Required Checks ({{I-D.ietf-suit-manifest}}, Section 6.2).

More checks are required when handling Dependencies. By default, any signature of a Dependency MUST be verified. However, there are some exceptions to this rule: where a device supports only one level of access (no ACLs, {{I-D.ietf-suit-manifest}}, Section 9, declaring which authorities have access to different Components/Commands/Parameters), it MAY choose to skip signature verification of Dependencies, since they are verified by digest. Where a device differentiates between trust levels, such as with an ACL, it MAY choose to defer the verification of signatures of Dependencies until the list of affected Components is known so that it can skip redundant signature verifications. For example, if a dependent's signer has access rights to all Components specified in a Dependency, then that Dependency does not require a signature verification. Similarly, if the signer of the dependent has full rights to the device, according to the ACL, then no signature verification is necessary on the Dependency.

Components that should be treated as Dependencies are identified in the suit-common metadata ({{structure-change}}).

Any required check that fails MUST result in an Abort.

Prior to executing any Command Sequence:
1. If the interpreter does not support Dependencies and a Manifest specifies a Dependency, then the interpreter MUST Abort.
1. If the Manifest contains more than one Component and/or Dependency, each Command sequence MUST begin with a Set Component Index Command.

If a Dependency is specified, then the Manifest Processor MUST perform the following additional checks:
1. Prior to executing any Command Sequence: the dependent MUST populate all Command Sequences for each Procedure specified by the Dependency; either the Staging Procedure, the Update Procedure, the Installation Procedure, or the Invocation Procedure.
2. At the end of each section in the dependent: The corresponding section in each Dependency has been executed, if present.

If a Recipient supports groups of interdependent Components (a Component Set), then prior to fetching any payload, it SHOULD verify that all Components in the Component Set are specified by a single Manifest and all its Dependencies that together:

1. Have sufficient permissions imparted by their signatures.
2. Specify a digest and a Payload for every Component in the Component Set.

Failing to verify the availablility of all components may lead to
API mismatches and other version mismatch problems.

The single dependent Manifest is called a Root Manifest.

##  Changes to Manifest Structure {#structure-change}

This section augments the Manifest Structure (Section 8.4) in {{I-D.ietf-suit-manifest}}. 

### Manifest Component ID {#manifest-id}

In complex systems, it may not always be clear where the Root Manifest is stored; this is particularly complex when a system has multiple, independent Root Manifests. The Manifest Component ID resolves this contention. The manifest-component-id is intended to be used by the Root Manifest.
The manifest-component-id is only used when storing a Root Manifest.
The manifest-component-id is ignored when processing Dependencies.

The following CDDL (see {{RFC8610}}) describes the Manifest Component ID:

~~~ cddl
$$SUIT_Manifest_Extensions //= 
    (suit-manifest-component-id => SUIT_Component_Identifier)
~~~

### SUIT_Dependencies Manifest Element {#SUIT_Dependencies}

The suit-common section, as described in {{I-D.ietf-suit-manifest}}, Section 8.4.5 is extended with a map of Component indices that indicate a Dependency. The keys of the map are the Component indices and the values of the map are any extra metadata needed to describe those Dependencies.

Because some operations treat Dependencies differently from other Components, it is necessary to identify them. SUIT_Dependencies identifies which Components from suit-components ({{I-D.ietf-suit-manifest}}, Section 8.4.5) are to be treated as the SUIT\_Envelope of a Dependency. SUIT_Dependencies is a map of Components, referenced by Component Index. Optionally, a Component prefix or other metadata may be delivered with the Component index. The CDDL for suit-dependencies is shown below:

~~~ cddl
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

If no extended metadata is needed for an extension, SUIT_Dependency_Metadata is an empty map (this is the same encoding size as a null). SUIT_Dependencies MUST be sorted according to Core Deterministic Encoding Requirements ({{RFC8949}}, Section 4.2.1).

The Components specified by SUIT_Dependency_Metadata will contain a Manifest Envelope that describes a Dependency of the current Manifest. The Manifest is identified, but the Recipient should expect an Envelope when it acquires the Dependency. This is because the Manifest is the one invariant element of the Envelope, where other elements may change by countersigning, adding authentication blocks, or severing elements.

When executing suit-condition-image-match over a Component that is designated in SUIT_Dependency_Metadata, the digest MUST be computed over just the bstr-wrapped SUIT_Manifest contained in the Manifest Envelope designated by the Component Index. This enables a Dependency reference to uniquely identify a particular Manifest structure. This is identical to the digest that is present as the first element of the suit-authentication-block in the Dependency's Envelope. The digest is calculated over the Manifest structure to ensure that removing a signature from a Manifest does not break Dependencies due to missing signature elements. This is also necessary to support the trusted intermediary use case, where an intermediary re-signs the Manifest, removing the original signature, potentially with a different algorithm, or trading COSE_Sign for COSE_Mac.

The suit-dependency-prefix element contains a SUIT_Component_Identifier ({{I-D.ietf-suit-manifest}}, Section 8.4.5.1). This specifies the scope at which the Dependency operates. This allows the Dependency to be forwarded on to a Component that is capable of parsing its own Manifests. It also allows one Manifest to be deployed to multiple dependent Recipients without those Recipients needing consistent Component hierarchy. suit-dependency-prefix is OPTIONAL for Recipients to implement.

A Dependency prefix can be used with a Component identifier. This allows complex systems to understand where Dependencies need to be applied. The Dependency prefix can be used in one of two ways. The first simply prepends the prefix to all Component Identifiers in the Dependency.

A Dependency prefix can also be used to indicate when a Dependency needs to be processed by a secondary Manifest Processor, as described in {{hierarchical-interpreters}}.

##  Changes to Abstract Machine Description

This section augments the Abstract Machine Description in {{I-D.ietf-suit-manifest}}, Section 6.4.
With the addition of Dependencies, some changes are necessary to the abstract machine, outside the typical scope of added Commands. These changes alter the behaviour of an existing Command and way that the parser processes Manifests:

* Five new Commands are introduced in {{new-commands}}:

    * Set Parameters
    * Process Dependency
    * Is Dependency
    * Dependency Integrity
    * Unlink

* Dependencies have Component Identifiers. All Commands may target Dependencies as well as Components, with one exception: suit-directive-process-dependency. Future commands MAY define their own restrictions on applicability to Dependencies and non-Dependency Components.
* Dependencies are processed in lockstep with the Root Manifest. This means that every Dependency's current Command sequence must be executed before a dependent's later Command sequence may be executed. For example, every Dependency's Dependency Resolution step must be executed before any dependent's Payload fetch step.
* When a Manifest Processor supports multiple independent Components, they may have shared Dependencies.
* When a Manifest Processor supports shared Dependencies, it MUST support reference counting of those Dependencies.
* When reference counting is used, Components MUST NOT be overwritten. The Manifest Uninstall section must be called, then the component MUST be Unlinked.

##  Processing Dependencies {#processing-dependencies}

As described in {{required-checks}}, the Manifest Processor must ensure that a 
Manifest with Dependencies invokes suit-directive-process-dependency for each 
of its Dependencies' sections from the corresponding section of the dependent.
Any changes made to Parameters by the Dependency persist in the dependent.

When a Process Dependency Command is encountered, the Manifest Processor:

1. Checks whether the map of Dependencies contains an entry for the current Component Index. If not present, it causes an immediate Abort.
2. Checks whether the Dependency has been the target of a Dependency integrity check. If not, it causes an immediate Abort.
2. Performs any application-specific setup that is required to parse the specified Component as a SUIT\_Envelope of a Dependency.
3. Authenticates the Dependency.
4. Executes the common-sequence section of the Dependency.
5. Executes the section of the Dependency that corresponds to the currently executing section of the dependent.

If the specified Dependency does not contain the current section, Process Dependency succeeds immediately.

The interpreter also performs the checks described in {{required-checks}} to ensure that the dependent is processing the Dependency correctly.

###  Multiple Manifest Processors {#hierarchical-interpreters}

When there are two or more trust domains, a Manifest Processor might be required in each. The first Manifest Processor is the normal Manifest Processor as described for the Recipient in Section 6 of {{I-D.ietf-suit-manifest}}. The second Manifest Processor only executes sections when the first Manifest Processor requests it. An API interface is provided from the second Manifest Processor to the first. This allows the first Manifest Processor to request a limited set of operations from the second. These operations are limited to: setting Parameters, inserting an Envelope, and invoking a Manifest Command Sequence. The second Manifest Processor declares a prefix to the first, which tells the first Manifest Processor when it should delegate to the second. These rules are enforced by underlying separation of privilege infrastructure, such as TEEs, or physical separation.

When the first Manifest Processor encounters a Dependency prefix, that informs the first Manifest Processor that it should provide the second Manifest Processor with the corresponding Dependency Envelope. This is done when the Dependency is fetched. The second Manifest Processor immediately verifies any authentication information in the Dependency Envelope. When a Parameter is set for any Component that matches the prefix, this Parameter setting is passed to the second Manifest Processor via an API. As the first Manifest Processor works through the Procedure (set of Command sequences) it is executing, each time it sees a Process Dependency Command that is associated with the prefix declared by the second Manifest Processor, it uses the API to ask the second Manifest Processor to invoke that Dependency section instead.

This mechanism ensures that the two or more Manifest Processors do not need to trust each other, except in a very limited case. When Parameter setting across trust domains is used, it must be very carefully considered. Only Parameters that do not have an effect on security properties should be allowed. The Dependency MAY control which Parameters are allowed to be set by using the Override Parameters Directive. The second Manifest Processor MAY also control which Parameters may be set by the first Manifest Processor by means of an ACL that lists the allowed Parameters. For example, a URI may be set by a dependent without a substantial impact on the security properties of the Manifest.

##  Dependency Resolution {#suit-dependency-resolution}

The Dependency Resolution Command Sequence is a container for the Commands needed to acquire and process the Dependencies of the current Manifest. All Dependencies MUST be fetched before any Payload is fetched to ensure that all Manifests are available and authenticated before any of the (larger) Payloads are acquired.

##  Added and Modified Commands {#new-commands}

All Commands are modified in that they can also target Dependencies. However, Set Component Index has a larger modification.

| Command Name | Semantic of the Operation
|------|----
| Set Parameters | current.params\[k\] := v if not k in current.params for-each k,v in arg
| Process Dependency | exec(current\[common\]); exec(current\[current-segment\])
| Dependency Integrity | verify(current, current.params\[image-digest\])
| Is Dependency | assert(current exists in Dependencies)
| Unlink | unlink(current)
{: #tab-modified-abstract title="Added/Modified Abstract Machine Commands"}

### suit-directive-set-parameters {#suit-directive-set-parameters}

Similar to suit-directive-override-parameters ({{I-D.ietf-suit-manifest}}, section 8.4.10.3), suit-directive-set-parameters allows the Manifest to configure behavior of future Directives by changing Parameters that are read by those Directives. Set Parameters is for use when Dependencies are used because it allows a Manifest to modify the behavior of its Dependencies. Because of this modification behavior, suit-directive-set-parameters MUST only be used for parameters that are intended to be overridden.

Available Parameters are defined in {{I-D.ietf-suit-manifest}}, section 8.4.8.

If a Parameter is already set, suit-directive-set-parameters will skip setting the Parameter to its
argument. This enables parameter replacement in Manifest trees. A Dependency can specify a
default Parameter using suit-directive-set-parameters. Then, a dependent of that Dependency can use
suit-directive-set-parameters prior to invoking suit-directive-process-dependency. Since
suit-directive-set-parameters has set-if-unset behaviour, this means that the dependent has effectively
overriden the Dependency's Parameter. Manifests that wish to enforce a specific value of a Parameter
MUST use suit-directive-override-parameters instead. This satisfies USER_STORY.OVERRIDE and
REQ.USE.MFST.COMPONENT of {{RFC9124}}.

While suit-directive-set-parameters can be used outside of a Dependency use case, it has limited
applicability: in linear manifests (without try-each, {{I-D.ietf-suit-manifest}}, section 8.4.10.2)
it either behaves as suit-directive-override-parameters or has no effect, depending on whether its
targets are already set. When used as a set-if-unset construction following a try-each,
suit-directive-override-parameters has the same effect as if a suit-directive-override-parameters
were placed in the final element of the try-each with no preceding condition. This limits the
applicability of suit-directive-set-parameters outside dependency use cases.

suit-directive-set-parameters does not specify a reporting policy.


### suit-directive-process-dependency {#suit-directive-process-dependency}

Execute the Commands in the common section of the current Dependency, followed by the Commands in the equivalent section of the current Dependency. For example, if the current section is "Payload Fetch," this will execute "Common metadata" in the current Dependency, then "Payload Fetch" in the current Dependency. Once this is complete, the Command following suit-directive-process-dependency will be processed.

If the current Component Index matches any of the following conditions, the Manifest Processor MUST Abort:

* The current Component index does not have an entry in the suit-dependencies map
* The current Component index has not been the target of a suit-condition-dependency-integrity
* The current section is "Common metadata"

If the current Component is True, then this Directive applies to all Dependencies.

When suit-directive-process-dependency completes, it forwards the last status code that occurred in the Dependency;
an Abort in a Dependency causes an Abort in the suit-directive-process-dependency of the Dependent.

### suit-condition-is-dependency {#suit-condition-is-dependency}

Check whether the current Component index is present in the Dependency list. If the current Component is in the Dependency list, suit-condition-is-dependency succeeds. Otherwise, it fails. This can be used along with component-id = True to act on all Dependencies or on all non-Dependency Components ({{creating-manifests}}).

### suit-condition-dependency-integrity {#suit-condition-dependency-integrity}

Verify the integrity of a Dependency. When a Manifest Processor executes suit-condition-dependency-integrity, it performs the following operations:

1. Verify the signature of the Dependency's suit-authentication-wrapper.
2. Compare the Dependency's suit-authentication-wrapper digest to the dependent's suit-parameter-image-digest
3. Verify the Dependency against the Depedency's suit-authentication-wrapper digest

If any of these steps fails, the Manifest Processor MUST immediately Abort.

The Manifest Processor MAY cache the results of these operations for later use from the context of the current Manifest. The Manifest Processor MUST NOT use cached results from any other Manifest context.
The Manifest Processor MUST prevent tampering with the cached results, e.g. through tamper-evident memory.
If the Manifest Processor caches the results of these checks, it MUST eliminate this cache if:

* Any Fetch, or Copy operation targets the Dependency's Component ID
* An Abort is encountered
* A Procedure completes

### suit-directive-unlink {#suit-directive-unlink}

A Manifest Processor that supports multiple independent root manifests
MUST support suit-directive-unlink. When a Component is no longer
needed, the Manifest Processor unlinks the Component to inform the 
Manifest Processor that it is no longer needed.

If a Manifest is no longer needed, the Manifest Processor unlinks it.
This causes the Manifest Processor to execute the suit-uninstall section
of the unlinked Manifest, after which it decrements the reference count
of the unlinked Manifest. The suit-uninstall section of a manifest
typically contains an unlink of all its dependencies and components.

All components, including Manifests must be unlinked before deletion 
or overwrite. If the
reference count of a component is non-zero, any command that alters
that component MUST cause the Manifest Processor to Abort. Affected
commands are:

* suit-directive-copy
* suit-directive-fetch
* suit-directive-write

The unlink Command decrements an implementation-defined reference counter. This reference counter MUST persist across restarts. The reference counter MUST NOT be decremented by a given Manifest more than once, and the Manifest Processor must enforce this. The Manifest Processor MAY choose to ignore an Unlink Directive depending on device policy.

When the reference counter of a Manifest reaches zero, the suit-uninstall Command sequence is invoked ({{suit-uninstall}}).

suit-directive-unlink is OPTIONAL to implement in Manifest Processors,
but Manifest Processors that support multiple independent Root Manifests
MUST support suit-directive-unlink.

# Uninstall {#suit-uninstall}

In some systems, particularly with multiple, independent, optional Components, it may be that there is a need to uninstall the Components that have been installed by a Manifest. Where this is expected, the uninstall Command sequence can provide the sequence needed to cleanly remove the Components defined by the Manifest and its Dependencies. In general, the suit-uninstall Command Sequence will contain primarily unlink Directives.

WARNING: This can cause faults where there are loose Dependencies (e.g., version range matching, {{I-D.ietf-suit-update-management}}, Section 5.5), since a Component can be removed while it is depended upon by another Component. To avoid Dependency faults, a Manifest author MUST use explicit Dependencies where possible.
To enable applications where explicit Dependency matching is not possible, a Manifest Processor can track references to loose Dependencies via reference counting in the same way as explicit Dependencies, as described in {{suit-directive-unlink}}.

The suit-uninstall Command Sequence is not severable, since it must always be available to enable uninstalling.

# Staging and Installation

In order to coordinate between download and installation in different trust domains, the Update Procedure defined in {{I-D.ietf-suit-manifest}}, Section 8.4.6 is divided into two sub-procedures:

* The Staging Procedure: This procedure is responsible for dependency resolution and acquiring all payloads required for the Update to proceed. It is composed of two command sequences

    * suit-dependency-resolution
    * suit-payload-fetch

* The Installation Procedure: This procedure is responsible for verifying staged components and installing them. It is composed of:

    * suit-candidate-verification
    * suit-install

This extension is backwards compatible when used with a Manifest Processor that supports the Update Procedure but does not support the Staging Procedure and Installation Procedure: the payload-fetch command sequence already contains suit-condition-image tests for each payload ({{I-D.ietf-suit-manifest}}, section 7.3) which means that images are already validated when suit-install is invoked. This makes suit-candidate-verification OPTIONAL to implement.

The Staging and Installation Procedures are only required when Staging occurs in a different trust domain to Installation.

## suit-candidate-verification {#suit-candidate-verification}

This command sequence is responsible for verifying that all elements of an update are present and correct prior to installation. This is only required when Installation occurs in a trust domain different from Staging, such as an installer invoked by the bootloader.

# Creating Manifests {#creating-manifests}

This section details a set of templates for creating Manifests. These templates explain which Parameters, Commands, and orders of Commands are necessary to achieve a stated goal.

## Dependency Template {#template-dependency}

The goal of the Dependency template is to obtain, verify, and process a Dependency as appropriate.

The following Commands are added to the shared sequence:

- Set Component Index Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1)
- Set Parameters Directive ({{suit-directive-set-parameters}}) for digest ({{I-D.ietf-suit-manifest}}, Section 8.4.8.6). Note that the digest MUST match the SUIT_Digest in the Dependency's suit-authentication-block ({{I-D.ietf-suit-manifest}}, Section 8.3).

The following Commands are placed into the Dependency resolution sequence:

- Set Component Index Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1)
- Set Parameters Directive ({{suit-directive-set-parameters}}) for a URI ({{I-D.ietf-suit-manifest}}, Section 8.4.8.10)
- Fetch Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.4)
- Dependency Integrity Condition ({{suit-condition-dependency-integrity}})
- Process Dependency Directive ({{suit-directive-process-dependency}})

Then, the validate sequence contains the following operations:

- Set Component Index Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1)
- Dependency Integrity Condition ({{suit-condition-dependency-integrity}})
- Process Dependency Directive ({{suit-directive-process-dependency}})

If any Dependency is declared, the dependent MUST populate all Command sequences for the current Procedure (Update or Invoke).

NOTE: Any changes made to Parameters in a Dependency persist in the dependent.

### Integrated Dependencies {#integrated-dependencies}

An implementer MAY choose to place a Dependency's Envelope in the Envelope of its dependent. The dependent Envelope key for the Dependency Envelope MUST be a text string. The URI for the Dependency MUST match the text string key of the dependent's Envelope key. It is RECOMMENDED to make the text string key a resolvable URI so that a Dependency that is removed from the Envelope can still be fetched.

## Encrypted Manifest Template {#template-encrypted-manifest}

The goal of the Encrypted Manifest template is to fetch and decrypt a Manifest so that it can be used as a Dependency. To use an encrypted Manifest, create a plaintext dependent, and add the encrypted Manifest as a Dependency. The dependent can include very little information.

NOTE: This template also requires the extensions defined in {{I-D.ietf-suit-firmware-encryption}}.

The following Commands are added to the shared sequence:

- Set Component Index Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1)
- Set Parameters Directive ({{suit-directive-set-parameters}}) for digest ({{I-D.ietf-suit-manifest}}, Section 8.4.8.6). Note that the digest MUST match the SUIT_Digest in the Dependency's suit-authentication-block ({{I-D.ietf-suit-manifest}}, Section 8.3).

The following operations are placed into the Dependency resolution block:

- Set Component Index Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1)
- Set Parameters Directive ({{suit-directive-set-parameters}}) for
    - URI ({{I-D.ietf-suit-manifest}}, Section 8.4.8.9)
    - Encryption Info ({{I-D.ietf-suit-firmware-encryption}})
- Fetch Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.4)
- Dependency Integrity Condition ({{suit-condition-dependency-integrity}})
- Process Dependency Directive ({{suit-directive-process-dependency}})

Then, the validate block contains the following operations:

- Set Component Index Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1)
- Check Image Match Condition ({{I-D.ietf-suit-manifest}}, Section 8.4.9.2)
- Process Dependency Directive ({{suit-directive-process-dependency}})

A plaintext Manifest and its encrypted Dependency may also form a composite Manifest ({{integrated-dependencies}}).

## Overriding Encryption Info Template {#template-override-encryption-info}

The goal of overriding the Encryption Info template is to separate the role of generating encrypted Payload and Encryption Info with Key-Encryption Key addressing Section 3 of {{I-D.ietf-suit-firmware-encryption}}.

As an example, this template describes two manifests:
- The dependent Manifest created by the Distribution System contains Encryption Info, allowing the Device to generate the Content-Encryption Key.
- The Dependency created by the Author contains Commands to decrypt the encrypted Payload using Encryption Info above and to validate the plaintext Payload with SUIT_Digest.

NOTE: This template also requires the extensions defined in {{I-D.ietf-suit-firmware-encryption}}.

The following operations are placed into the Dependency resolution block of dependent Manifest:

- Set Component Index Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1) pointing at Dependency
- Set Parameters Directive ({{suit-directive-set-parameters}}) for
    - Image Digest ({{I-D.ietf-suit-manifest}}, Section 8.4.8.6)
    - URI ({{I-D.ietf-suit-manifest}}, Section 8.4.8.9) of Dependency
- Fetch Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.4)
- Dependency Integrity Condition ({{suit-condition-dependency-integrity}})

The following Commands are placed into the Fetch/Install block of dependent Manifest

- Set Component Index Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1) pointing at encrypted Payload
- Set Parameters Directive ({{suit-directive-set-parameters}}) for
    - URI ({{I-D.ietf-suit-manifest}}, Section 8.4.8.9)
- Set Component Index Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1) pointing at Dependency
- Set Parameters Directive ({{suit-directive-set-parameters}}) for
    - Encryption Info ({{I-D.ietf-suit-firmware-encryption}})
- Process Dependency Directive ({{suit-directive-process-dependency}})

The following Commands are placed into the same block of Dependency:

- Set Component Index Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1) pointing at encrypted Payload
- Fetch Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.4)
- Set Component Index Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1) pointing at to be decrypted Payload
- Override Parameters Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.3) for
    - Source Component ({{I-D.ietf-suit-manifest}}, Section 8.4.8.11) pointing at encrypted Payload
- Copy Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.5) consuming the Encryption Info above

The Distribution System can Set the URI Parameter in the Fetch/Install block of dependent Manifest if it wants to overwrite the URI of the encrypted Payload.

Because the Author and the Distribution System have different roles and may be separate entities, it is highly recommended to leverage permissions ({{I-D.ietf-suit-manifest}}, Section 9).
For example, the Device can protect itself from an attacker who breaches the Distribution System by allowing only the Author's Manifest to modify the Component of (to be) decrypted Payload.

## Operating on Multiple Components

In order to produce compact encoding, it is efficient to perform operations on multiple Components simultaneously. Because Dependencies and Component Images are processed at different times, there is a mechanism to distinguish between these elements: suit-condition-is-dependency. This can be used with suit-directive-try-each to perform operations just on Dependencies or just on Component Images.

For example, to fetch all Dependencies, the following Commands are added to the Dependency resolution block:

- Set Component Index Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1)
- Set Parameters Directive ({{suit-directive-set-parameters}}) for a URI ({{I-D.ietf-suit-manifest}}, Section 8.4.8.9)
- Set Component Index Directive, with argument "True" ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1)
- Try Each Directive
    - Sequence 0
        - Condition Is Dependency
        - Fetch
        - Dependency Integrity Condition ({{suit-condition-dependency-integrity}})
        - Process Dependency
    - Sequence 1 (Empty; no Commands, succeeds immediately)

Another example is to fetch and validate all Component Images. The Image fetch sequence contains the following Commands:

- Set Component Index Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1)
- Set Parameters Directive ({{suit-directive-set-parameters}}) for a URI ({{I-D.ietf-suit-manifest}}, Section 8.4.8.9)
- Set Component Index Directive, with argument "True" ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1)
- Try Each Directive
    - Sequence 0
        - Condition Is Dependency
        - Process Dependency
    - Sequence 1
        - Fetch
        - Condition Image Match

When some Components are "installed" or "loaded" it is more productive to use lists of Component indices rather than Component Index = True. For example, to install several Components, the following Commands should be placed in the Image Install Sequence:

- Set Component Index Directive ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1)
- Set Parameters Directive ({{suit-directive-set-parameters}}) for the Source Component ({{I-D.ietf-suit-manifest}}, Section 8.4.8.11)
- Set Component Index Directive, with argument containing list of destination Component indices ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1)
- Copy
- Set Component Index Directive, with argument containing list Dependency Component indices ({{I-D.ietf-suit-manifest}}, Section 8.4.10.1)
- Process Dependency

#  IANA Considerations {#iana}

IANA is requested to allocate the following numbers in the listed registries created by draft-ietf-suit-manifest:

## SUIT Envelope Elements

Label | Name | Reference
---|---|---
15 | Dependency Resolution | {{suit-dependency-resolution}}
18 | Candidate Verification | {{suit-candidate-verification}}
{: #tab-suit-env-elements title="New SUIT Envelope Elements"}

## SUIT Manifest Elements

Label | Name | Reference
---|---|---
5 | Manifest Component ID | {{manifest-id}}
15 | Dependency Resolution | {{suit-dependency-resolution}}
18 | Candidate Verification | {{suit-candidate-verification}}
24 | Uninstall | {{suit-uninstall}}
{: #tab-suit-mfst-elements title="New SUIT Manifest Elements"}

## SUIT Common Elements

Label | Name | Reference
---|---|---
1 | Dependencies | {{SUIT_Dependencies}}
{: #tab-suit-common-elements title="New SUIT Common Elements"}

## SUIT Commands

Label | Name | Reference
---|---|---
7 | Dependency Integrity | {{suit-condition-dependency-integrity}}
8 | Is Dependency | {{suit-condition-is-dependency}}
11 | Process Dependency | {{suit-directive-process-dependency}}
19 | Set Parameters | {{suit-directive-set-parameters}}
33 | Unlink | {{suit-directive-unlink}}
{: #tab-suit-commands title="New SUIT Commands"}

#  Security Considerations

This specification is about a Manifest format protecting and describing how to retrieve, install, and invoke Images and as such it is part of a larger solution for delivering software updates to devices. A detailed security treatment can be found in the SUIT architecture {{RFC9019}} and in the SUIT information model {{RFC9124}}.

The features added in this specification introduce several new threats. The introduction
of Dependencies enables multiple entities to operate on a device with different privileges.
While this is necessary to fulfill REQ.USE.MFST.COMPONENT ({{RFC9124}}, Section 4.5.4), it
also introduces a new requirement: REQ.SEC.ACCESS_CONTROL ({{RFC9124}}, Section 4.3.13),
which is required to address THREAT.MFST.OVERRIDE ({{RFC9124}}, Section 4.2.13) and
THREAT.UPD.UNAPPROVED ({{RFC9124}}, Section 4.2.11).

Simultaneous processing of multiple Manifests, as enabled by Dependency processing,
introduces risks of TOCTOU threats (THREAT.MFST.TOCTOU: {{RFC9124}}, Section 4.2.18). 
Holding multiple Manifest Envelopes in memory
simultaneously can exceed the capacity of the Manifest Processor's tamper-protected
memory (REQ.SEC.MFST.CONST: {{RFC9124}}, Section 4.3.21). To address this threat,
the Manifest Processor MAY use modular processing as described in REQ.USE.PAYLOAD
({{RFC9124}}, Section 4.5.12). If retaining the Manifests only, excluding envelopes,
in immutable memory does not provide enough capacity, the Manifest Processor MAY
reduce overhead by retaining the following elements for each manifest in immutable memory: 

* Manifest Digest
* Parameters
* Current Component Index
* Current Command Sequence
* Current Command Sequence Offset

This allows a Manifest Processor to resume processing a manifest as follows:

* Copy the Manifest into immutable memory
* Validate the Manifest using the stored Manifest Digest
* Parse forward to find the Current Command Sequence
* Jump within the Command Seqeunce to the stored Command Sequence Offset

When identifying a Root Manifest's correct storage location, 
the Component Identifier MUST be evaluated vs. the access priviliges of an Author.
Otherwise, the Component Identifier may permit an escalation of privilege: an
authorised Author causes a manifest to be installed in a location for which the
Author does not have access rights.

Since Dependencies are stored as Components, Dependency Integrity Checks
and Image Verification are slightly different operations. While a typical Image
is immutable, a Manifest Envelope can be modified in some ways (e.g. removing
a Severable Element) without changing the Integrity Check result. Because of
these factors, suit-directive-process-dependency requires that a dependency first
be validated with suit-check suit-condition-dependency-integrity.

--- back

# A. Full CDDL {#full-cddl}

To be valid, the following CDDL (see {{RFC8610}}) MUST be appended to the SUIT Manifest CDDL. The SUIT CDDL is defined in Appendix A of {{I-D.ietf-suit-manifest}}

~~~ cddl
{::include-fold draft-ietf-suit-trust-domains.cddl}
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

## Example 0: Process Dependency

This example uses functionalities:

* manifest component id
* dependency resolution
* process dependency

The Dependency:

~~~ cbor-diag
{::include-fold examples/example1_process.diag}
~~~

Total size of Envelope with COSE authentication object: 373

~~~ cbor-pretty
{::include-fold examples/example1_process.hex}
~~~

The dependent Manifest (fetched from "https://example.com/dependent.suit"):

~~~ cbor-diag
{::include-fold examples/example0_dependent.diag}
~~~

Total size of Envelope with COSE authentication object: 190

~~~ cbor-pretty
{::include-fold examples/example0_dependent.hex}
~~~

## Example 1: Integrated Dependency

* manifest component id
* dependency resolution
* process dependency
* integrated dependency

~~~ cbor-diag
{::include-fold examples/example2_integrated.diag}
~~~

Total size of Envelope with COSE authentication object: 519

Envelope with COSE authentication object:

~~~ cbor-pretty
{::include-fold examples/example2_integrated.hex}
~~~
