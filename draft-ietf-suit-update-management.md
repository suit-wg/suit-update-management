---
title: Update Management Extensions for Software Updates for Internet of Things (SUIT) Manifests
abbrev: SUIT Update Management Extensions
docname: draft-ietf-suit-update-management-13
category: std
stream: IETF

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
      email: Brendan.Moran.ietf@gmail.com
 -
      ins: K. Takayama
      name: Ken Takayama
      organization: SECOM CO., LTD.
      email: ken.takayama.ietf@gmail.com

normative:
  RFC9393:
  I-D.ietf-suit-manifest:
  RFC8949:

informative:
  RFC9124:
  RFC9019:
  semver:
    title: "Semantic Versioning 2.0.0"
    author:
    date: 2013-06-18
    target: https://semver.org

--- abstract
This specification describes extensions to the SUIT manifest format. These extensions allow an update
author, update distributor or device operator to more precisely control
the distribution and installation of updates to devices. These
extensions also provide a mechanism to inform a management system of
Software Identifier and Software Bill Of Materials information about an
updated device.

--- middle

#  Introduction

Full management of software updates for unattended, connected devices requires a cooperation between the update author(s) and management, distribution, policy enforcement, and auditing systems. This specification provides the extensions to the SUIT manifest {{I-D.ietf-suit-manifest}} that enable an author to coordinate with these other systems. These extensions enable authors to instruct devices to examine update priority, local update authorisation, update lifetime, and system properties. They also enable devices to report and distributors to collect Software Bill of Materials (SBOM) information.

Extensions in this specification are OPTIONAL to implement and OPTIONAL to include in manifests. A Recipient that encounters a command or parameter it does not implement MUST reject the manifest as defined in {{I-D.ietf-suit-manifest}} Section 8.4.2, ensuring that update behaviour is never ambiguous. Conversely, when a deployment relies on update-management behaviour defined here, the manifest author MUST ensure that targeted recipients advertise support for the required extensions (for example via enablement policy or capability negotiation) before shipping such manifests so that required commands will be honoured rather than rejected.

#  Conventions and Terminology {#conventions-and-terminology}

{::boilerplate bcp14}

This draft makes use of terminology defined in {{RFC9019}} and {{I-D.ietf-suit-manifest}}.

This document uses semantic versioning terminology from {{semver}}, including major, minor, patch, pre-release, and build metadata. The machine-readable version encoding defined in {{suit-parameter-version}} is a constrained integer encoding based on that terminology: it encodes release versions as one to three non-negative integers, supports only the pre-release classes defined in {{suit-parameter-version}}, and excludes build metadata from machine-readable comparisons.

# Extension Metadata

Some additional metadata makes management of SUIT updates easier:

* A semantic version number for the update represented by the manifest
* Concise Software Identifiers (CoSWID) {{RFC9393}}
* Text descriptions of requirements
* Text description of the current versions of components

## suit-set-version {#suit-set-version}

This metadata encodes a semantic version for the component set that the manifest updates, including any dependencies. This enables version comparisons to be performed on manifests. Non-manifest images encode their versions independently of the manifest.

Manifest Authors SHOULD encode suit-set-version whenever the release can be represented by the constrained version encoding defined in {{suit-parameter-version}} so that Recipients can compare manifests deterministically. Deployments that cannot supply such a version without loss of fidelity MUST omit suit-set-version and convey any human-facing numbering via suit-text-current-version ({{text-current-version}}). Because suit-set-version is a machine-readable parameter for determining compatibility and build metadata is ignored for semantic-version precedence, build metadata MUST NOT be included.

The composition of suit-set-version is the same as suit-parameter-version ({{suit-parameter-version}}).

If build metadata is desired, the manifest author MAY include it via suit-text-current-version ({{text-current-version}}).

## suit-coswid {#manifest-digest-coswid}

A CoSWID can enable Software Bill of Materials (SBOM) use-cases. Tightly coupling update and attestation ensures that verification infrastructure always knows what software to expect on each device.

suit-coswid is a member of the suit-manifest. It contains a Concise Software Identifier (CoSWID) as defined in {{RFC9393}}. This element SHOULD be made severable so that it can be discarded by the Recipient or an intermediary if it is not used by the Recipient while preserving the manifest signature. Implementations that cannot support severable elements MAY include suit-coswid non-severably, but MUST ensure that Recipients can still process the manifest.

suit-coswid is RECOMMENDED to implement and RECOMMENDED to include in manifests because management systems commonly need a durable software identity after update installation. CoSWID and related Software Bill of Materials metadata can support inventory, vulnerability management, compliance checks, and reconciliation between the installed update state and management-system records. This recommendation is scoped to the operational and security value of identifying installed software; it does not imply that the presence of SBOM metadata proves that the software is free of vulnerabilities or policy issues. Other extension metadata is not generally RECOMMENDED unless required by deployment policy or by a SUIT profile.

Recipients that do not use suit-coswid are not required to interpret the CoSWID content. When suit-coswid is severable, such Recipients or intermediaries can discard it without invalidating the manifest signature. When suit-coswid is not severable, a Recipient MUST NOT fail solely because a well-formed, policy-permitted suit-coswid field is present.

Recipients that use or validate suit-coswid MAY still fail or reject the manifest when the suit-coswid field or its digest is malformed, when local policy rejects the metadata, when processing would exhaust available resources, when validation of processed CoSWID metadata fails, or when a manifest relies on unsupported critical behaviour. These requirements do not imply that every Recipient implements CoSWID processing.

## suit-text-version-required {#text-version-required}

suit-text-version-required is used to represent a version-based dependency on suit-parameter-version as described in {{suit-parameter-version}} and {{suit-condition-version}}. When a Manifest Author needs to communicate such a dependency to operators, the author SHOULD populate the suit-text map with a SUIT_Component_Identifier key for the dependency component, and place in the corresponding map a suit-text-version-required key with a free text expression that is representative of the version constraints placed on the dependency so that field personnel can validate compliance. Deployments that provide operator guidance exclusively through other channels MAY omit this field. This text SHOULD be expressive enough that a device operator can be expected to understand the dependency; predefined tokens MAY be used when supporting documentation ensures equivalent clarity. Expressions in this field MUST be encoded as UTF-8 text limited to printable characters (Unicode general categories L, N, P, or Zs) and SHOULD use simple relational operators (for example `>`, `>=`, `<`, `<=`, `=`) so that automated tooling can perform lint checks. Implementations that render this text SHOULD escape or filter it to prevent markup or control-code injection. This is a free text field and there are no additional specific formatting rules beyond the requirements above.

By way of example only, to express a dependency on a component "\['x', 'y'\]", where the intended version is any v1.x later than v1.2.5, but not v2.0 or above, the author would add the following structure to the suit-text element. Note that this text is in cbor-diag notation.

~~~CDDL
['x','y'] : {
    7 : ">=1.2.5,<2"
}
~~~

## suit-text-current-version {#text-current-version}

suit-text-current-version is used to provide human-readable version information equivalent to suit-set-version ({{suit-set-version}}). This metadata MAY have a version listed for each or any component. The Manifest Processor MUST NOT consume this version; it is for human readability only.

To describe a version, a Manifest Author SHOULD populate the suit-text map with a SUIT_Component_Identifier key for the dependency component, and place in the corresponding map a suit-text-current-version key with a free text version that is representative of the version of the component so that operators can reconcile machine and human-readable records. Deployments that provide human-facing version information through other configuration channels MAY omit this text. This text SHOULD be expressive enough that a device operator can be expected to understand the version; environments that rely on catalog identifiers MAY use those identifiers when supporting documentation provides the necessary context. Values in this field MUST be encoded as UTF-8 text limited to printable characters, and implementations MUST treat suit-set-version and suit-parameter-version as authoritative when a discrepancy exists. Recipients MUST NOT interpret this text as executable code or markup and MUST treat it as display-only information. Implementations that render this text SHOULD sanitize, escape, or otherwise filter it before presentation. This is a free text field and there are no additional specific formatting rules beyond the requirements above.

It is RECOMMENDED that the Manifest Author use a Semantic Version ({{semver}}) in the free-text field to keep human-readable and machine-readable versions aligned. Unlike suit-set-version ({{suit-set-version}}), the full semantic version specification can be used.

# Extension Parameters {#extension-parameters}

Several parameters are needed to define the behaviour of the commands specified in Extension Commands ({{extension-commands}}). These parameters follow the same considerations as defined in Section 8.4.8 of {{I-D.ietf-suit-manifest}}.

Name | CDDL Structure | Reference
---|---|---
Use Before | suit-parameter-use-before | {{suit-parameter-use-before}}
Minimum Battery | suit-parameter-minimum-battery | {{suit-parameter-minimum-battery}}
Update Priority | suit-parameter-update-priority | {{suit-parameter-update-priority}}
Version | suit-parameter-version | {{suit-parameter-version}}
Wait Info | suit-parameter-wait-info | {{suit-parameter-wait-info}}
Component Metadata | suit-parameter-component-metadata | {{suit-parameter-component-metadata}}

## suit-parameter-use-before {#suit-parameter-use-before}

An expiry date for the use of the manifest encoded as the non-negative integer number of seconds since 1970-01-01. Implementations that use this parameter MUST use a 64-bit internal representation of the integer. Used with {{suit-condition-use-before}}.

## suit-parameter-minimum-battery

This parameter sets the minimum battery level in mWh. This parameter is encoded as a non-negative integer. Used with suit-condition-minimum-battery ({{suit-condition-minimum-battery}}).

## suit-parameter-update-priority

This parameter sets the priority of the update. This parameter is encoded as an integer. It is used along with suit-condition-update-authorized ({{suit-condition-update-authorized}}) to ask an application for permission to initiate an update. This does not constitute a privilege inversion because an explicit request for authorization has been provided by the Update Authority in the form of the suit-condition-update-authorized command.

Numerically smaller values indicate higher update priority. Recipients and applications that compare suit-parameter-update-priority values MUST use this ordering. Local policy MAY assign deployment-specific meanings to particular values or ranges. For example, critical reliability and vulnerability fixes might be given negative numbers, while bug fixes might be given small positive numbers, and feature additions might be given larger positive numbers, which allows an application to make an informed decision about whether and when to allow an update to proceed.

## suit-parameter-version {#suit-parameter-version}

Indicates allowable versions for the specified component. One version comparison can be made with each suit-parameter-version. This parameter is compared with the version asserted by the current component when suit-condition-version ({{suit-condition-version}}) is invoked. The current component can assert the current version in many ways, including storage in a parameter storage database, in a metadata object, or in a known location within the component itself.

Each suit-parameter-version contains a comparison operator and a version, according to the following CDDL:

~~~CDDL
SUIT_Parameter_Version_Match = [
    suit-condition-version-comparison-type:
        SUIT_Condition_Version_Comparison_Types,
    suit-condition-version-comparison-value:
        SUIT_Condition_Version_Comparison_Value
]
~~~

The comparison type can be:

* Greater.
* Greater or Equal.
* Equal.
* Lesser or Equal.
* Lesser.

The version comparison value is encoded as a CBOR list of integers. Comparisons are done on each integer in sequence. Comparison stops after all integers in the list defined by the manifest have been consumed OR after an non-equal comparison has occurred. For example, if the manifest defines a comparison, "Equal \[1\]", then this will match all version sequences starting with 1. If a manifest defines both "Greater or Equal \[1,0\]" and "Lesser \[1,10\]", then it will match versions 1.0.x up to, but not including 1.10.

### suit-parameter-version Semantic Versioning encoding guidelines

The encoded versions use the constrained semantic-versioning encoding summarized in {{conventions-and-terminology}}. Manifest Authors SHOULD keep their encoding aligned with semantic versioning so that Recipients can compare versions deterministically; if another numbering scheme is required, the sequence of integers encoded here MUST still preserve release ordering (for example, `[2025,12,6]` for a calendar-based release).

Versions are composed of:

1. A release version encoded as a sequence of 1 to 3 non-negative integers (allowing zero values)
2. An optional pre-release indicator encoded as a negative integer, followed by zero or more non-negative integers

Semantic Versioning permits arbitrary pre-release identifiers and build metadata. This specification only defines encodings for alpha, beta, and release-candidate pre-release classes. Because suit-parameter-version exists solely to enable the Manifest Processor to make a decision about version compatibility, and because build metadata is ignored for semantic-version precedence, build metadata MUST NOT be included.

In semantic versioning terminology:

1. The first integer represents the major number. This indicates breaking changes to the component.
2. The second integer represents the minor number. This is typically reserved for new features or large, non-breaking changes.
3. The third integer is the patch version. This is typically reserved for bug fixes.

The pre-release indicator MUST NOT appear as element 0. The pre-release indicator is encoded as:

* -1: Release Candidate (RC)
* -2: Beta
* -3: Alpha

This allows these releases to compare correctly with final releases. For example, Version 2.0, RC1 is lower than Version 2.0.0 and higher than any Version 1.x. By encoding RC as -1, this works correctly: \[2,0,-1,1\] compares as lower than \[2,0,0\]. Similarly, beta (-2) is lower than RC and alpha (-3) is lower than RC.

Pre-release identifiers other than alpha, beta, and release candidate cannot be represented directly in this encoding. Deployments that need other identifiers MUST either map them to one of the defined classes while preserving the intended ordering or omit the machine-readable version field and convey the identifier as suit-text-current-version ({{text-current-version}}).

For example:

* 1.2.3 = \[1,2,3\].
* 1.2-rc.3 = \[1,2,-1,3\].
* 1.2-beta = \[1,2,-2\].
* 1.2-alpha = \[1,2,-3\].
* 1.2.3-alpha.4 = \[1,2,3,-3,4\].

## suit-parameter-wait-info

suit-directive-wait ({{suit-directive-wait}}) directs the manifest processor to pause until a specified event occurs. The suit-parameter-wait-info encodes the parameters needed for the directive.

The exact implementation of the pause is implementation-defined. For example, this could be done by blocking on a semaphore, registering an event handler and suspending the manifest processor, polling for a notification, or aborting the update entirely, then restarting when a notification is received.

suit-parameter-wait-info is encoded as a map of wait events. All wait events MUST be satisfied before the Manifest Processor continues. The wait events currently defined are described in the following table.

Name | Encoding | Description
---|---|---
suit-wait-event-authorization | int | Same as suit-parameter-update-priority
suit-wait-event-power | int | Wait until power state
suit-wait-event-network | int | Wait until network state
suit-wait-event-other-device-version | See below | Wait for other device to match version
suit-wait-event-time | uint | Wait until time (seconds since 1970-01-01)
suit-wait-event-time-of-day | uint | Wait until seconds since 00:00:00 Local Time
suit-wait-event-time-of-day-utc | uint | Wait until seconds since 00:00:00 UTC
suit-wait-event-day-of-week | uint | Wait until days since Sunday Local Time
suit-wait-event-day-of-week-utc | uint | Wait until days since Sunday UTC

Local Time means the Recipient's configured local civil time zone at the time the wait event is evaluated, including any daylight-saving-time rules available to the Recipient. If the local time zone changes while a Recipient is waiting, the Recipient reevaluates the wait event using the updated time-zone configuration. During daylight-saving-time transitions, a skipped local time is treated as satisfied at the first representable local time after the skipped interval, and a repeated local time is satisfied at its first occurrence. Recipients that do not have configured local-time and daylight-saving-time information MUST treat local-time wait events as unsupported. Manifest Authors SHOULD use the UTC wait events when a deployment does not have a common local-time policy.

suit-wait-event-other-device-version reuses the encoding of SUIT_Parameter_Version_Match. It is encoded as a sequence that contains an implementation-defined bstr identifier for the other device, and a list of one or more SUIT_Parameter_Version_Match. For interoperable use, the deployment profile or management system MUST define the namespace and byte-string encoding of other-device identifiers, the mechanism by which the Recipient obtains the referenced device's version, and the version encoding used by that referenced device. A Recipient MUST treat suit-wait-event-other-device-version as unsupported unless those deployment-specific definitions are available.

## suit-parameter-component-metadata {#suit-parameter-component-metadata}

In some instances, a system needs to know the file metadata for a component. This metadata can include:

* creator
* creation time
* modification time
* default permissions (rwx)
* a map of user/permission pairs
* a map of role/permission pairs
* a map of group/permission pairs
* file type

Unless otherwise stated, all string values in this structure MUST be encoded as UTF-8 without control characters (Unicode general categories Cc or Cf) and SHOULD be limited to human-readable identifiers such as names or POSIX-style paths. Binary values conveyed via `bstr` MUST be well-formed for the consuming platform (for example, a UUID or permissions bitmap) and MUST NOT exceed the minimum length required to represent the value canonically.

Component metadata is applied at time of fetch, copy, or write; see {{I-D.ietf-suit-manifest}}, Sections 8.4.10.4, 8.4.10.5, and 8.4.10.6. Therefore, the component metadata parameter MUST be set in advance of the component being fetched, copied into, or written.

 
### Creator {#suit-meta-creator}

Sometimes, management of file systems requires that the creator of each file is correctly recorded. Because the default creator of files will be the update agent, this can obscure the actual creator of each file. The Creator metadata element allows overriding the default behaviour and setting the correct creator.

The creator is defined as follows:

~~~ CDDL
SUIT_meta_actor_id = UUID_Tagged / bstr / tstr / int
UUID_Tagged = #6.37(bstr)
~~~

The actor ID can be whatever is most appropriate for any given system. For example, the actor ID might be a string (e.g., username), integer (e.g., POSIX userid), or UUID (e.g., TEEP TA UUID).


### Creation & Modification Time

The creation and modification times are defined by CBOR time types. These are defined in {{RFC8949}}, Section 3.4.2. The CBOR tag is REQUIRED when either creation or modification time are provided.

~~~ CDDL
suit-meta-modification-time => #6.1(uint)
suit-meta-creation-time => #6.1(uint)
~~~

### Component Default Permissions

Typical permissions management systems require read, write, and execute permissions that are applied to all users who do not have their own explicit permissions. These are the default permissions for the current component. Default permissions are described by the following CDDL:

~~~ CDDL
SUIT_meta_permissions = uint .bits SUIT_meta_permission_bits
SUIT_meta_permission_bits = &(
    write_attr_ex: 13,
    read_attr_ex: 12,
    sync: 11,
    delete: 10,
    recurse_delete: 9,
    write_attr: 8,
    change_owner: 7,
    change_perm: 6,
    read_perm: 5,
    read_attr: 4,
    createdir_append: 3,
    list_read: 2,
    create_write: 1,
    traverse_exec: 0,
    * $$SUIT_meta_permission_bits_extensions
)
~~~

### User, Role, Group permissions

Many filesystems have users and groups. Additionally some have roles. Actors that have these associations can have specific permissions associated with them for each component. Each of these sets of permissions is defined the same way: with a map of actor identifiers to permissions.

~~~CDDL
SUIT_meta_permission_map = {
    + SUIT_meta_actor_id => SUIT_meta_permissions
}
~~~

The SUIT_meta_actor_id is the same as defined for Creator, {{suit-meta-creator}}.


### File Type

File Type typically identifies whether a file is a directory, regular file, or symbolic link. If not specified, File Type defaults to regular file.

This enables specific management operations for SUIT command sequences:

* To create a directory

    * Set the Component Index to the Component Identifier of the directory to be created
    * Set the Component metadata, including the file type for directory
    * Set suit-parameter-content to an empty bstr
    * Invoke suit-directive-write

* To create a symbolic link

    * Set the Component Index to the Component Identifier of the link to be created
    * Set the Component metadata, including the file type for symbolic link
    * Set suit-parameter-content to the link target 
    * Invoke suit-directive-write

For example, the following Payload Fetch & Install sequences will create a new /usr/local/bin directory, download https://cdn.example/example3.bin into a new file: /usr/local/bin/example3, then create a symlink at /usr/bin/example that points to /usr/local/bin/example3.

* Common has components for:

    * /usr/bin/example
    * /usr/local/bin
    * /usr/local/bin/example3

* Payload fetch:

    * set component index = 1
    * set parameters:

        * content = h''
        * metadata = {file-type: directory}

    * write
    * set component index = 2
    * set URI = "https://cdn.example/example3.bin"
    * fetch
    * condition image digest

* Install:

    * set component index = 0
    * set parameters:

        * content = "/usr/local/bin/example3"
        * metadata = {file-type: symlink}

    * write

# Extension Commands

The following table defines the semantics of the commands defined in this specification in the same way as in the Abstract Machine Description, Section 6.4, of {{I-D.ietf-suit-manifest}}.

All commands defined in this specification are OPTIONAL to implement. A Recipient that encounters a command it does not implement MUST reject the manifest as defined in {{I-D.ietf-suit-manifest}} Section 8.4.2, ensuring that update behaviour is never ambiguous.

| Command Name | CDDL Identifier | Semantic of the Operation
|------|---|----
| Use Before  | suit-condition-use-before | assert(now() < current.params\[use-before\])
| Check Image Not Match | suit-condition-image-not-match | assert(not binary-match(digest(current), current.params\[digest\]))
| Check Minimum Battery | suit-condition-minimum-battery | assert(battery >= current.params\[minimum-battery\])
| Check Update Authorized | suit-condition-update-authorized | assert( isAuthorized( current.params\[priority\]))
| Check Version | suit-condition-version | assert(version_check(current, current.params\[version\]))
| Wait For Event | suit-directive-wait | until event(arg), wait
| Override Multiple | suit-directive-override-multiple | components\[i\].params\[k\] := v for-each k,v in d for-each i,d in arg
| Copy Params | suit-directive-copy-params | current.params\[k\] = components\[i\].params\[k\] for k in l for i,l in arg


## suit-condition-use-before

Verify that the current time is BEFORE the specified time. suit-condition-use-before is used to specify the last time at which an update is to be installed. The recipient evaluates the current time against the suit-parameter-use-before parameter ({{suit-parameter-use-before}}), which MUST have already been set as a parameter, encoded as seconds after 1970-01-01 00:00:00 UTC. Timestamp conditions MUST be evaluated in 64 bits, regardless of encoded CBOR size. suit-condition-use-before is OPTIONAL to implement.

## suit-condition-image-not-match

Verify that the current component does not match the suit-parameter-image-digest (Section 8.4.8.6 of {{I-D.ietf-suit-manifest}}). If no digest is specified, the condition fails. suit-condition-image-not-match is OPTIONAL to implement.

## suit-condition-minimum-battery

suit-condition-minimum-battery provides a mechanism to test a Recipient's battery level before installing an update. This condition is primarily for use in primary-cell applications, where a primary cell is a single-use, non-rechargeable battery and energy budgeting is therefore a one-way operation. For batteries that are charged, suit-directive-wait is more appropriate, since it defines a "wait" until the battery level is sufficient to install the update. suit-condition-minimum-battery is specified in mWh. suit-condition-minimum-battery is OPTIONAL to implement. suit-condition-minimum-battery consumes suit-parameter-minimum-battery ({{suit-parameter-minimum-battery}}).

## suit-condition-update-authorized

Request authorization from the application and fail if not authorized. This can allow a user to decline an update. suit-parameter-update-priority ({{suit-parameter-update-priority}}) provides an integer priority level that the application can use to determine whether or not to authorize the update. Smaller integer values indicate higher priority; deployment policy defines the action taken for a given priority. suit-condition-update-authorized is OPTIONAL to implement.

## suit-condition-version

suit-condition-version allows comparing versions of firmware. Verifying image digests is preferred to version checks because digests are more precise. suit-condition-version examines a component's version against the version info specified in suit-parameter-version ({{suit-parameter-version}}).

## suit-directive-wait {#suit-directive-wait}

suit-directive-wait directs the manifest processor to pause until a specified event occurs. Some possible events include:

1. Authorization
2. External power
3. Network availability
4. Other device firmware version
5. Time
6. Time of day
7. Day of week

## suit-directive-override-multiple

This directive enables setting parameters for multiple components at the same time. This allows a small reduction in encoding overhead:

* without override-multiple, the encoding for each component consists of:

  * set-component-index (2 bytes)
  * override-parameters (1 byte + parameter map)

* with override-multiple, the encoding for each component consists of:

  * the component index key (1 byte)
  * the parameter map

Override-multiple requires the command (1-2 bytes) and one additional map to hold the parameter sets (1 byte). For one component, there is no savings. For multiple components, there is an encoding savings of 2 bytes per component.

Implementations can structure code so that override-multiple follows a code-path nearly identical to set-component-index + override-parameters.

This command is purely an encoding alias for set-component-index and override-parameters. The component index is set to the last component listed in the override-multiple argument when override-multiple completes.

The following CDDL defines the argument for suit-directive-override-multiple:

```CDDL
SUIT_Override_Mult_Arg = {
    + uint => {+ $$SUIT_Parameters}
}
```

## suit-directive-copy-params

suit-directive-copy-params enables a manifest author to specify one or more components to copy parameters from, and a list of parameters to copy from each specified source component.

The behaviour is exactly the same as override parameters, but with parameter values defined in existing components. Parameters are only copied between identical keys (no copying from URI to digest, for example).

For each entry in the map, the manifest processor sets the source component to be the component identified by the index contained in the map key. For each parameter identified in the copy list, the manifest processor copies the parameter from the source component to the current component.

The following CDDL defines the argument for suit-directive-copy-params:

```CDDL
SUIT_Directive_Copy_Params = {
    + uint => [+ int]
}
```

#  IANA Considerations {#iana}

IANA is requested to allocate the commands, parameters, and metadata values shown in the following tables.

## SUIT Envelope Elements

Label | Name | Reference
---|---|---
14 | CoSWID | {{manifest-digest-coswid}}

## SUIT Manifest Elements

Label | Name | Reference
---|---|---
6 | Set Version | {{suit-set-version}}
14 | CoSWID | {{manifest-digest-coswid}}

## SUIT Commands

Label | Name | Reference
---|---|---
4 | Use Before | {{suit-condition-use-before}}
25 | Image Not Match | {{suit-condition-image-not-match}}
26 | Minimum Battery | {{suit-condition-minimum-battery}}
27 | Update Authorized | {{suit-condition-update-authorized}}
28 | Version | {{suit-condition-version}}
29 | Wait For Event | {{suit-directive-wait}}
34 | Override Multiple | {{suit-directive-override-multiple}}
35 | Copy Params | {{suit-directive-copy-params}}

## SUIT Parameters

Label | Name | Reference
---|---|---
4 | Use Before | {{suit-parameter-use-before}}
26 | Minimum Battery | {{suit-parameter-minimum-battery}}
27 | Update Priority | {{suit-parameter-update-priority}}
28 | Version | {{suit-parameter-version}}
29 | Wait Info | {{suit-parameter-wait-info}}
30 | Component Metadata | {{suit-parameter-component-metadata}}

## SUIT Component Text Values

Label | Name | Reference
---|---|---
7 | Component Version Required | {{text-version-required}}
8 | Current Version | {{text-current-version}}

#  Security Considerations

This document extends the SUIT manifest specification. The extensions defined here are optional and do not make support for update-management extensions mandatory for implementations of the base SUIT manifest specification. A detailed security treatment can be found in the architecture {{RFC9019}} and in the information model {{RFC9124}} documents.

The free-text fields introduced in Sections {{text-version-required}} and {{text-current-version}} are intended solely for human consumption. Recipients MUST treat those values as untrusted input: they MUST NOT evaluate the text, execute embedded markup, or override machine-readable decisions derived from suit-set-version or suit-parameter-version. Implementations SHOULD bound the length of displayed text to mitigate interface flooding and log injection.

The suit-coswid element can expose detailed software identity and SBOM information. Such information can help authorized operators assess inventory, vulnerability exposure, and compliance, but the same information can also help an attacker or unauthorized observer quickly identify software components and versions on a device. Deployments SHOULD treat manifests containing suit-coswid as sensitive metadata, limit access to authorized parties, and consider using severable suit-coswid content so that intermediaries and Recipients that do not need this metadata can discard it without invalidating the manifest signature.

Component metadata ({{suit-parameter-component-metadata}}) can expose operator identifiers, file paths, or other locally meaningful strings. Deployments SHOULD validate these values against local policy before applying them, and MUST handle missing or malformed metadata defensively so that the update agent does not escalate privileges or disclose sensitive information inadvertently.


--- back

#  Full CDDL {#full-cddl}

To be valid, the following CDDL MUST be appended to the SUIT Manifest CDDL. The SUIT CDDL is defined in Appendix A of {{I-D.ietf-suit-manifest}}.

~~~ CDDL
{::include draft-ietf-suit-update-management.cddl}
~~~
