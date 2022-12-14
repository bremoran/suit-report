---
title: Secure Reporting of Update Status
abbrev: Secure Reporting of Update Status
docname: draft-ietf-suit-report-04
category: info

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
author:
 -
       ins: B. Moran
       name: Brendan Moran
       organization: Arm Limited
       email: brendan.moran.ietf@gmail.com
 - 
       ins: H. Birkholz
       name: Henk Birkholz
       organization: Fraunhofer SIT
       email: henk.birkholz@sit.fraunhofer.de

informative:
  I-D.ietf-rats-eat: EAT
  I-D.birkholz-rats-corim: CoRIM
  I-D.ietf-suit-trust-domains:

normative:
  I-D.ietf-suit-manifest:

--- abstract

The Software Update for the Internet of Things (SUIT) manifest provides
a way for many different update and boot
workflows to be described by a common format. However, this does not
provide a feedback mechanism for developers in the event that an update
or boot fails.

This specification describes a lightweight feedback mechanism that
allows a developer in possession of a manifest to reconstruct the
decisions made and actions performed by a manifest processor.

--- middle

# Introduction

A SUIT manifest processor can fail to install or boot an update for many
reasons. Frequently, the error codes generated by such systems fail to
provide developers with enough information to find root causes and
produce corrective actions, resulting in extra effort to reproduce
failures. Logging the results of each SUIT command can simplify this
process.

While it is possible to report the results of SUIT commands through
existing logging or attestation mechanisms, this comes with several
drawbacks:

* data inflation, particularly when designed for text-based logging
* missing information elements
* missing support for multiple components

The CBOR objects defined in this document allow devices to:

* report a trace of how an update was performed
* report expected vs. actual values for critical checks
* describe the installation of complex multi-component architectures
* describe the measured properties of a system
* report the exact reason for a parsing failure

This document provides a definition of a SUIT-specific logging container
that may be used in a variety of scenarios.

#  Conventions and Terminology {#terminology}

{::boilerplate bcp14-tagged}

Terms used in this specification include:

* Boot: initialization of an executable image. Although this
  specification refers to boot, any boot-specific operations described
  are equally applicable to starting an executable in an OS context.

# The SUIT Record

If the developer can be assumed to have a copy of the
manifest, then they need little information to reconstruct what the
manifest processor has done. They simply need any data that influences
the control flow of the manifest. The manifest only supports the
following control flow primitives:

- Set Component
- Set/Override Parameters
- Try-Each
- Run Sequence
- Conditions

Of these, only conditions change the behavior of the processor from the
default, and then only when the condition fails.

Then, to reconstruct the flow of a manifest, all a developer needs is
a list of metadata about failed conditions:

- the current manifest
- the current section
- the offset into the current section
- the current component index
- the "reason" for failure

Most conditions compare a parameter to an actual value, so the "reason"
is typically simply the actual value.

Since it is possible that a non-condition command (directive) may fail in an
exceptional circumstance, this must be included as well. However, 
a failed directive will terminate processing of the manifest. To accommodate
for a failed command and for explicit "completion," an additional "result"
element is added as well. In the case of a command failure,
the failure reason is typically a numeric error code. However, these error
codes need to be standardised in order to be useful.

Reconstructing what a device has done in this way is compact,
however it requires some reconstruction effort. This is an issue that
can be solved by tooling.

~~~
SUIT_Record = [
    suit-record-manifest-id        : [* uint ],
    suit-record-manifest-section   : int,
    suit-record-section-offset     : uint,
    suit-record-component-index    : uint,
    suit-record-properties         : SUIT_Parameters,
    $$SUIT_Record_Extensions
]
~~~

suit-record-manifest-id is used to identify which manifest contains the
command that caused the record to be generated. The manifest id is a
list of integers that form a walk of the manifest tree, starting at the
root. An empty list indicates that the command was contained in the
root manifest. If the list is not empty, the command was contained in
one of the root manifest's dependencies, or nested even further below
that.

For example, suppose that the root manifest has 3 dependencies
and each of those dependencies has 2 dependencies of its own:

* Root

    * Dependency A

        * Dependency A0
        * Dependency A1

    * Dependency B

        * Dependency B0
        * Dependency B1

    * Dependency C

        * Dependency C0
        * Dependency C1

A manifest-id of \[1,0\] would indicate that the current command was
contained within Dependency B0. Similarly, a manifest-id of \[2,1\]
would indicate Dependency C1

suit-record-manifest-section indicates which section of the manifest was
active. This is used in addition to an offset so that the developer can
index into severable sections in a predictable way. The value of this
element is the value of the key that identified the section in the
manifest.

suit-record-section-offset is the number of bytes into the current
section at which the current command is located.

suit-record-component-index is the index of the component that was
specified at the time that the report was generated. This field is
necessary due to the availability of set-current-component values of
True and a list of components. Both of these values cause the manifest
processor to loop over commands using a series of component-ids, so the
developer needs to know which was selected when the command executed.

suit-record-properties contains any measured properties that led to the
command failure.
For example, this could be the actual value of a SUIT_Digest or
class identifier. This is encoded in a SUIT_Parameters block as defined
in {{I-D.ietf-suit-manifest}}.

# The SUIT Report

Some metadata is common to all records, such as the root manifest:
the manifest that is the entry-point for the manifest processor. This
metadata is aggregated with a list of SUIT_Records. The SUIT_Report
may also contain a list of any system properties that were measured
and reported, and a reason for a failure if one occured.

~~~
SUIT_Report = {
  suit-reference              => SUIT_Reference,
  ? suit-report-nonce         => bstr,
  suit-report-records         => [ * SUIT_Record / system-property-claims ],
  suit-report-result          => true / {
    suit-report-result-code   => int, ; could condense to enum later
    suit-report-result-record => SUIT_Record,
  }
  $$SUIT_Report_Extensions
}
system-property-claims = {
  system-component-id => SUIT_Component_Identifier,
  + SUIT_Parameters,
}
~~~

The suit-reference provides a reference URI and digest for a suit
manifest. The uri SHOULD be the canonical URI that is provided in the
manifest. The digest is the digest of the manifest.

NOTE: The digest is used
in preference to other identifiers in the manifest because it allows
a manifest to be uniquely identified (collision resistance) whereas
other identifiers, such as the sequence number, can collide,
particularly in scenarios with multiple trusted signers.

The following CDDL describes a SUIT_Reference.

~~~CDDL
SUIT_Reference = {
    suit-report-manifest-uri  => tstr,
    suit-report-manifest-digest => SUIT_Digest,
}
~~~

suit-report-manifest-digest provides a SUIT_Digest (as defined in
{{I-D.ietf-suit-manifest}}) that is the characteristic digest of the
Root manifest.

suit-report-manifest-uri provides the reference URI that was provided in
the root manifest.

suit-report-nonce provides a container for freshness or replay
protection information. This field MAY be omitted where the suit-report
is authenticated within a container that provides freshness already.
For example, attestation evidence typically contains a proof of
freshness.

suit-report-records is a list of 0 or more SUIT Records or 
system-property-claims. Because SUIT Records are only generated on failure,
in simple cases this can be an empty list. SUIT_Records and 
suit-system-property-claims are merged into a single list because this
reduces the overhead for a constrained node that generates this report.
The use of a single append-only log allows report generators to use simple
memory management. Because the system-property-claims are encoded as maps
and SUIT_Records are encoded as lists, a recipient need only filter the
CBOR Type-5 entries from suit-report-records to obtain all 
system-property-claims.

System properties can be extracted from suit-report-records by filtering
suit-report-records for maps. System Properties are a list of measured 
or asserted properties
of the system that creates the suit report. These properties are scoped by
component identifier. Because this list is expected to be constructed on
the fly by a constrained node, component identifiers may appear more than
once. A recipient may convert the result to a more conventional structure:

~~~~CDDL
SUIT_Record_System_Properties = {
  * component-id => {
    + SUIT_Parameters,
  }
}
~~~~

suit-report-result provides a mechanism to show that the SUIT procedure
completed successfully (value is true) or why it failed (value is a map
of an error code and a SUIT_Record).

The suit-report-result-code indicates the reason for the failure. Values
are expected to be CBOR parsing failures, Schema validation failures,
COSE validation failures or SUIT processing failures.

The suit-report-result-record indicates the exact point in the manifest
or manifest dependency tree where the error occured.

#  Attestation

This document describes how a well-informed verifier can infer the trustworthiness of a remote device. Remote attestation is done by using the SUIT_Manifest_Envelope along with the SUIT_Report to reconstruct the state of the device at boot time. By embedding data used for remote attestation in the SUIT_Report, a remote device can use an append-only log to collect both measurements and debug/failure information into the same document. This document can then be conveyed to a verifier as a part of the attestation evidence. A remote attestation format to convey attestation evidence, such as an Entity Attestation Token (EAT, see {{-EAT}}), that contains a SUIT_Report MUST also include an integrity measurement of the Manifest Processor & Report Generator. 

When a Concise Reference Integrity Manifest (CoRIM, see {{-CoRIM}} is delivered in a SUIT_Manifest_Envelope, this codifies the delivery of verification information to the verifier:

* The Firmware Distributor:
    * sends the SUIT_Manifest_Envelope to the Verifier without payload or text, but with CoRIM
    * sends the SUIT_Manifest_Envelope to the recipient without CoRIM, or text, but with payload
* The Recipient:
    * Installs the firmware as described in the SUIT_Manifest and generates a SUIT_report, which is encapsulated in an EAT by the installer and sent to the Firmware Distributor.
    * Boots the firmware as described in the SUIT_Manifest and creates a SUIT_report, which is encapsulated in an EAT by the installer and sent to the Firmware Distributor.
* The Firmware Distributor sends both reports to the verifier (separately or together)
* The Verifier:
    * Reconstructs the state of the device using the manifest
    * Compares this state to the CoRIM
    * Returns an Attestation Report to the Firmware Distributor

This approach simplifies the design of the bootloader since it is able to use an append-only log. It allows a verifier to validate this report against a signed CoRIM that is provided by the firmware author, which simplifies the delivery chain of verification information to the verifier.

This information is not intended as Attestation Evidence and while an Attestation Report MAY provide this information for conveying error codes and/or failure reports, it SHOULD be translated into general-purpose claims for use by the Relying Party.

# Capability Reporting

Because SUIT is extensible, a manifest author must know what capabilities a device has available. To enable this, a capability report is a set of lists that define which commands, parameters, algorithms, and component IDs are supported by a manifest processor.

The CDDL for a SUIT_Capability_Report follows:

~~~~CDDL
SUIT_Capability_Report = {
  suit-component-capabilities        => [+ SUIT_Component_Capability ]
  suit-command-capabilities          => [+ int],
  suit-parameters-capabilities       => [+ int],
  suit-crypt-algo-capabilities       => [+ int],
  ? suit-envelope-capabilities       => [+ int],
  ? suit-manifest-capabilities       => [+ int],
  ? suit-common-capabilities         => [+ int],
  ? suit-text-component-capabilities => [+ int],
  ? suit-text-capabilities           => [+ int],
  ? suit-dependency-capabilities     => [+ int],
  * [+int]                           => [+ int],
  $$SUIT_Capability_Report_Extensions
}

SUIT_Component_Capability = [*bstr,?true]
~~~~

A SUIT_Component_Capability is similar to a SUIT_Component_ID, with one difference: it may optionally be terminated by a CBOR 'true' which acts as a wild-card match for any component with a prefix matching the SUIT_Component_Capability leading up to the 'true.' This feature is for use with filesystem storage, key value stores, or any other arbitrary-component-id storage systems.

When reporting capabilities, it is OPTIONAL to report capabilities that are declared mandatory by the SUIT Manifest {{I-D.ietf-suit-manifest}}. Capabilities defined by extensions MUST be reported.

Additional capability reporting can be added as follows: if a manifest element does not exist in this map, it can be added by specifying the CBOR path to the manifest element in an array and using this as the key. For example SUIT_Dependencies, as described in {{I-D.ietf-suit-trust-domains}} could have an extension added, which was key 3 in the SUIT_Dependencies map. This capability would be reported as: \[3, 3, 1\] => \[3\], where the key consists of the key for SUIT_Manifest (3), the key for SUIT_Common (3), and the key for SUIT_Dependencies (1). Then the value indicates that this manifest processor supports the extension (3).

#  IANA Considerations {#iana}

IANA is requested to allocate a CBOR tag for the SUIT Report.

#  Security Considerations

The SUIT Report should either be carried over a secure transport, or
signed, or both. Ideally, attestation should be used to prove that the
report was generated by legitimate hardware.

#  Acknowledgements

The authors would like to thank Dave Thaler for his feedback.