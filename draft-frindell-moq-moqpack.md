---
title: "QPACK Compression for MoQ Transport"
abbrev: "moq-moqpack"
docname: draft-frindell-moq-moqpack-latest
category: std
consensus: true
v: 3

ipr: trust200902
area: "Web and Internet Transport"
submissiontype: IETF
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - qpack
 - compression
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "afrind/draft-frindell-moq-moqpack"
  latest: "https://afrind.github.io/draft-frindell-moq-moqpack/draft-frindell-moq-moqpack.html"

author:
  -
    ins: A. Frindell
    name: Alan Frindell
    organization: Meta
    email: afrind@meta.com

normative:
  QUIC: RFC9000
  QPACK: RFC9204
  MOQT: I-D.ietf-moq-transport

informative:
  HPACK: RFC7541

--- abstract

This document defines an extension to Media over QUIC Transport (MOQT) that
enables QPACK compression for contol messages. By leveraging QPACK's dynamic
table, this extension significantly reduces the overhead of repeated
values such as track names and authorization tokens, improving efficiency for
sessions with many subscriptions or frequent redundant values.

--- middle


# Introduction

Media over QUIC Transport (MOQT) {{MOQT}} control message message fields and
parameters can contain large values that are repeated across many messages
within a session. The base MOQT specification transmits this information in full
each time it appears, which can result in significant overhead.

This document defines an extension that uses QPACK {{QPACK}} to compress MOQT
message parameters. QPACK provides:

* Dynamic table for referencing previously transmitted values
* Static table with pre-defined common values
* Stream blocking semantics suitable for QUIC

By treating MOQT parameters as QPACK field lines, this extension enables
efficient compression of repeated values while maintaining compatibility with
QPACK's existing infrastructure.

## Motivation

Consider a session where a client sends 100 SUBSCRIBE messages, each carrying
the same 500-byte authorization token. Without compression, this results in
50,000 bytes of token data. With QPACK compression, the token is transmitted
once and subsequent references require only a few bytes, reducing total
overhead to approximately 600 bytes.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

The terms "endpoint", "session", "publisher", and "subscriber" are defined in
{{MOQT}}.


# Extension Negotiation

This extension is negotiated during MOQT session establishment using Setup
Options. CLIENT_SETUP and SERVER_SETUP messages always use standard MOQT
encoding and are never MOQPACK compressed.

## MOQT_QPACK_MAX_TABLE_CAPACITY

The MOQT_QPACK_MAX_TABLE_CAPACITY setup option (Option Type 0x10)
specifies the maximum size in bytes of the QPACK dynamic table the endpoint
is willing to maintain for decoding. This corresponds to
SETTINGS_QPACK_MAX_TABLE_CAPACITY in HTTP/3.

The value is encoded as a variable-length integer. The default value is 0.

QPACK compression is enabled when both endpoints send this parameter with a
value greater than 0. If either endpoint omits this parameter or sends a
value of 0, QPACK compression MUST NOT be used and control messages use the
standard MOQT format.

## MOQT_QPACK_BLOCKED_STREAMS

The MOQT_QPACK_BLOCKED_STREAMS setup option (Option Type 0x11) specifies
the maximum number of streams that can be blocked waiting for dynamic table
updates. This corresponds to SETTINGS_QPACK_BLOCKED_STREAMS in HTTP/3.

The value is encoded as a variable-length integer. The default value is 0,
which prevents any stream from being blocked. When set to 0, encoders MUST NOT
reference dynamic table entries that have not been acknowledged.

This option is only meaningful when QPACK compression is enabled.

## MOQT_QPACK_INDEX_SETUP_AUTH {#implicit-dynamic-table-seeding}

The MOQT_QPACK_INDEX_SETUP_AUTH setup option (Option Type 0x12) controls
whether AUTHORIZATION TOKEN options from the Setup messages are implicitly
inserted into the dynamic table.

The value is encoded as a variable-length integer:

- 0: Do not implicitly insert setup auth tokens (default)
- 1: Implicitly insert setup auth tokens

When either endpoint omits this option or sends 0, implicit insertion does
not occur and endpoints that wish to reference auth tokens explicitly
insert them via the encoder stream.

This allows endpoints to authenticate the connection via setup auth tokens
while still using Never-Indexed Literals for subsequent auth token references
if desired.

When both endpoints send MOQT_QPACK_INDEX_SETUP_AUTH with value 1, any
AUTHORIZATION TOKEN options from the Setup messages are implicitly inserted
into the dynamic table without requiring encoder stream instructions.

Tokens from Setup message, in the order they appeared, are inserted into
the receiver's encoder dynamic table (indices 0, 1, 2, ...)

This allows the client to immediately reference its setup auth token in
the first SUBSCRIBE message using a dynamic table reference, without
resending it on the encoder stream.

The implicit entries count against the MOQT_QPACK_MAX_TABLE_CAPACITY limit.
If the implicit entries would exceed the peer's advertised capacity, the
excess entries (in reverse order) are not inserted and cannot be referenced.

Encoder stream insertions after setup use indices starting after the
implicit entries. For example, if CLIENT_SETUP contained 2 auth tokens,
the first explicit insertion would be at index 2.

# QPACK Streams

When QPACK compression is negotiated, each endpoint opens two unidirectional
streams for QPACK signaling.

## Stream Types

This extension defines two new MOQT unidirectional stream types:

QPACK_ENCODER_STREAM (0x1f107a60):
: Carries QPACK encoder instructions from the endpoint that opens the stream.
  The format and semantics are defined in Section 4.3.1 of {{QPACK}}.

QPACK_DECODER_STREAM (0x1f107a61):
: Carries QPACK decoder instructions from the endpoint that opens the stream.
  The format and semantics are defined in Section 4.4.1 of {{QPACK}}.

## Stream Initialization

Each endpoint MUST open exactly one QPACK encoder stream and one QPACK decoder
stream for QPACK use. These streams MUST be opened before sending any message
with a Compressed Block.

An endpoint MAY open these streams immediately after sending its Setup message
if it included MOQT_QPACK_MAX_TABLE_CAPACITY with a non-zero value. If a
receiving endpoint does not enable QPACK (omits the parameter or sends value 0),
it MAY send STOP_SENDING on these streams; this is not an error and the streams
SHOULD be reset.

Note that implicitly-inserted dynamic table entries from Setup auth tokens
(see {{implicit-dynamic-table-seeding}}) do not require encoder stream
instructions. A Compressed Block MAY reference these implicit entries even
if no encoder stream instructions have been sent.

If an endpoint receives a Compressed Block that references a dynamic table
entry beyond the implicit entries before receiving any encoder stream data,
it MUST buffer the message until the required encoder instructions arrive
or close the session with PROTOCOL_VIOLATION if buffering limits are exceeded.

## Stream Lifetime

QPACK encoder and decoder streams MUST remain open for the duration of the MOQT
session. If either stream is closed after QPACK is negotiated, the endpoint MUST
close the MOQT session with PROTOCOL_VIOLATION.


# Compressed Message Formats

## MOQPACK Flag Bit

This extension uses a flag bit in the message type to indicate if QPACK is
used. Bit 6 (0x40) of the message type indicates MOQPACK format:

- Type & 0x40 == 0: Standard MOQT format
- Type & 0x40 == 0x40: MOQPACK format

For example:
- SUBSCRIBE standard = 0x03
- SUBSCRIBE MOQPACK = 0x43

When MOQPACK is negotiated, endpoints MUST accept both standard and MOQPACK
formats for all applicable messages. An endpoint MAY send either format, but
SHOULD prefer MOQPACK format to benefit from compression.

When MOQPACK is NOT negotiated, endpoints MUST NOT send MOQPACK format messages
and MUST close the session with PROTOCOL_VIOLATION if they receive one.

In MOQPACK format, Track Namespace, Track Name, and Parameters are moved into a
QPACK Compressed Block. Other message-specific fields including Properties
remain unchanged.

## Pseudo-Parameter Types

The following pseudo-parameter types are reserved for encoding namespace and
track name fields in the Compressed Block:

| Type | Name | Description |
|------|------|-------------|
| 0x00 | TRACK_NAMESPACE_ELEMENT | Single element of a namespace tuple |
| 0x01 | TRACK_NAMESPACE_SET | Full serialized namespace tuple |
| 0x02 | TRACK_NAME | Track Name |

These pseudo-types use the same encoding as regular parameters: Literal with
Static Name Reference for new values, or Indexed with Dynamic Table for
previously-inserted values.

## Namespace Reconstruction

A Track Namespace, Track Namespace Prefix or Track Namespace Suffix is
reconstructed by assembling consecutive TRACK_NAMESPACE_ELEMENT and
TRACK_NAMESPACE_SET field lines, which MUST appear first in the Compressed Block
before TRACK_NAME or any parameters.

TRACK_NAMESPACE_ELEMENT adds a single element to the namespace tuple.
TRACK_NAMESPACE_SET appends all elements from a serialized tuple. These
can be intermixed and appear in any combination. For example:

~~~
Namespace ("conference", "room1", "audio"):

Option A - All elements:
  TRACK_NAMESPACE_ELEMENT: "conference"
  TRACK_NAMESPACE_ELEMENT: "room1"
  TRACK_NAMESPACE_ELEMENT: "audio"

Option B - Full set:
  TRACK_NAMESPACE_SET: ("conference", "room1", "audio")

Option C - Mixed:
  TRACK_NAMESPACE_ELEMENT: "conference"
  TRACK_NAMESPACE_SET: ("room1", "audio")
~~~

This allows encoders to maximize compression by inserting commonly-reused
elements or partial tuples into the dynamic table.

The message type determines the semantics of the assembled namespace:

* SUBSCRIBE, PUBLISH, FETCH, TRACK_STATUS, PUBLISH_NAMESPACE: Full namespace
* SUBSCRIBE_NAMESPACE: Namespace prefix
* NAMESPACE, NAMESPACE_DONE: Namespace suffix

## Field Ordering

Namespace elements (TRACK_NAMESPACE_ELEMENT and TRACK_NAMESPACE_SET) MUST
appear first in the Compressed Block, followed by TRACK_NAME (if present),
followed by parameters in increasing order of parameter type.

Within the namespace elements section, entries appear in the order they
contribute to the namespace tuple and are not required to be in increasing
order of type. TRACK_NAMESPACE_ELEMENT (0x00) and TRACK_NAMESPACE_SET (0x01)
MAY be intermixed.

Parameters (types 0x03 and above) MUST appear in increasing order of their
parameter type. If parameters appear out of order, the receiver MUST close
the session with PROTOCOL_VIOLATION.

## Required Fields

When decoding a Compressed Block, the receiver MUST verify that all required
fields are present:

* SUBSCRIBE, PUBLISH, TRACK_STATUS, Standalone FETCH: At least one namespace
  element (TRACK_NAMESPACE_ELEMENT or TRACK_NAMESPACE_SET) and TRACK_NAME
* SUBSCRIBE_NAMESPACE, PUBLISH_NAMESPACE, NAMESPACE, NAMESPACE_DONE: At least
  one namespace element
* Joining FETCH, parameter-only messages: No required pseudo-parameters

An empty namespace (zero elements) is valid only if explicitly allowed by the
message semantics.

If a required field is missing, the receiver MUST close the session with
PROTOCOL_VIOLATION.

## MOQPACK Message Formats

### SUBSCRIBE

~~~
SUBSCRIBE Message (MOQPACK) {
  Type (vi64) = 0x43,
  Length (16),
  Request ID (vi64),
  Track Alias (vi64),
  Compressed Block (..)
}
~~~

The Compressed Block contains namespace elements (TRACK_NAMESPACE_ELEMENT and/or
TRACK_NAMESPACE_SET), TRACK_NAME (0x02), and any parameters
(AUTHORIZATION_TOKEN, SUBSCRIBER_PRIORITY, etc.).

### PUBLISH

~~~
PUBLISH Message (MOQPACK) {
  Type (vi64) = 0x5D,
  Length (16),
  Request ID (vi64),
  Track Alias (vi64),
  Compressed Block Length (vi64),
  Compressed Block (..),
  Track Extensions (..)
}
~~~

The Compressed Block contains namespace elements, TRACK_NAME, and any
parameters.  Track Extensions remain outside the compressed block and use
standard MOQT encoding, including IMMUTABLE_EXTENSIONS which are not QPACK
compressed.

### FETCH

~~~
Standalone Fetch (MOQPACK) {
  Start Location (Location),
  End Location (Location),
  Compressed Block (..)
}

Joining Fetch (MOQPACK) {
  Joining Request ID (vi64),
  Join Type (vi64),
  Joining Start (vi64),
  Compressed Block (..)
}

FETCH Message (MOQPACK) {
  Type (vi64) = 0x56,
  Length (16),
  Request ID (vi64),
  Fetch Type (vi64),
  [Standalone (Standalone Fetch QPACK),]
  [Joining (Joining Fetch QPACK),]
}
~~~

For Standalone Fetch, the Compressed Block contains namespace elements,
TRACK_NAME, and parameters. For Joining Fetch, the Compressed Block contains
only parameters (the track is inherited from the joined subscription).

### SUBSCRIBE_NAMESPACE

~~~
SUBSCRIBE_NAMESPACE Message (MOQPACK) {
  Type (vi64) = 0x51,
  Length (16),
  Request ID (vi64),
  Subscribe Options (vi64),
  Compressed Block (..)
}
~~~

The Compressed Block contains namespace prefix elements and any parameters.

### PUBLISH_NAMESPACE

~~~
PUBLISH_NAMESPACE Message (MOQPACK) {
  Type (vi64) = 0x46,
  Length (16),
  Request ID (vi64),
  Compressed Block (..)
}
~~~

The Compressed Block contains namespace elements and any parameters.

### NAMESPACE

~~~
NAMESPACE Message (MOQPACK) {
  Type (vi64) = 0x48,
  Length (16),
  Compressed Block (..)
}
~~~

The Compressed Block contains namespace suffix elements. This message has
no parameters.

### NAMESPACE_DONE

~~~
NAMESPACE_DONE Message (MOQPACK) {
  Type (vi64) = 0x4E,
  Length (16),
  Compressed Block (..)
}
~~~

The Compressed Block contains namespace suffix elements. This message has
no parameters.

### TRACK_STATUS

TRACK_STATUS uses the same format as SUBSCRIBE but with type 0x4D.
The Compressed Block contains namespace elements, TRACK_NAME, and any applicable
parameters.

### Parameter-Only Messages

The following messages have parameters but no namespace or track name fields.
When MOQPACK is negotiated, these messages MAY use MOQPACK format with the
0x40 flag bit set. The MOQPACK format replaces the standard Parameters field
with a Compressed Block:

| Standard Type | MOQPACK Type | Message |
|---------------|--------------|---------|
| 0x02 | 0x42 | REQUEST_UPDATE |
| 0x04 | 0x44 | SUBSCRIBE_OK |
| 0x05 | 0x45 | REQUEST_ERROR |
| 0x07 | 0x47 | REQUEST_OK |
| 0x18 | 0x58 | FETCH_OK |
| 0x1E | 0x5E | PUBLISH_OK |

~~~
Parameter-Only Message (MOQPACK) {
  Type (vi64) = <standard type> | 0x40,
  Length (16),
  [Message-specific fields...],
  Compressed Block (..)
}
~~~

The Compressed Block contains only parameters (no pseudo-parameter types).
Message-specific fields (Request ID, error codes, etc.) remain unchanged.


# QPACK Encoding

This extension uses QPACK's wire encoding formats exactly as specified in
{{QPACK}}. The only difference is the interpretation of static table
references: instead of indexing into the HTTP static table of string
name-value pairs, the static table index IS the MOQT parameter type.

This allows existing QPACK encoder/decoder implementations to be reused
with minimal modification.

## Compressed Block Format

Each Compressed Block begins with the standard QPACK encoded field section
prefix as defined in Section 4.5.1 of {{QPACK}}:

~~~
Compressed Block {
  Required Insert Count (8+),
  Sign and Delta Base (8+),
  Encoded Field Lines (..)
}
~~~

Required Insert Count:
: Encoded as specified in Section 4.5.1.1 of {{QPACK}}. Indicates the
  minimum dynamic table state needed to decode this block. A value of 0
  means the block has no dynamic table references.

Base:
: Encoded as a sign bit and Delta Base as specified in Section 4.5.1.2
  of {{QPACK}}. Used to resolve relative indices in field line
  representations.

## Indexing

Dynamic table references in field lines use relative indexing as specified
in {{QPACK}} Section 3.2.5. A relative index of 0 refers to the entry with
absolute index equal to Base - 1. Encoders and decoders MUST use relative
indices, not absolute indices, in Compressed Blocks.

Post-Base indexing (Section 3.2.6 of {{QPACK}}) MAY be used for entries
inserted after the Base. This enables single-pass encoding where the encoder
inserts entries while encoding a field section and references them using
Post-Base indices.

## MOQT Static Table Semantics

In standard QPACK, a static table index retrieves a predefined (name, value)
pair. In this extension, the static table conceptually contains entries
where the "name" is the parameter type integer and there is no predefined
value.

The static table index equals the MOQT parameter type:

For example:
* Static index 0x02 represents DELIVERY_TIMEOUT
* Static index 0x03 represents AUTHORIZATION_TOKEN
* Static index 0x20 represents SUBSCRIBER_PRIORITY

This means any valid MOQT parameter type can be referenced by static index
without requiring pre-registration in a table.

### Field Line Interpretation

In QPACK field line encodings, the T bit selects between static table (T=1)
and dynamic table (T=0). Since MOQT parameter types are integers that map
directly to static table indices, the following encodings are used:

Literal Field Line With Static Name Reference (T=1):
: The Name Index is the MOQT parameter type. The Value field contains the
  parameter value. Use this to send a parameter value.

Indexed Field Line with Dynamic Table (T=0):
: References the dynamic table using a relative index. The retrieved entry
  contains a complete MOQT parameter (type and value).

Indexed Field Line with Post-Base Index:
: References a dynamic table entry inserted after the Base. Used for
  single-pass encoding. See {{QPACK}} Section 4.5.3.

The following QPACK field line encodings are prohibited:

Indexed Field Line with Static Table (T=1):
: PROHIBITED. The MOQT static table has no predefined values.

Literal Field Line With Dynamic Name Reference (T=0):
: PROHIBITED. Parameter types are always known integers; there is no need
  to reference the dynamic table for a parameter type.

Literal Field Line with Post-Base Name Reference:
: PROHIBITED. Parameter types are always known integers; there is no need
  to reference the dynamic table for a parameter type.

Literal Field Line With Literal Name:
: PROHIBITED. Parameter types are integers, never string literals.

Huffman-encoded string literals (H=1):
: PROHIBITED. The CPU cost outweighs the minimal space savings for typical
  MOQT values. Encoders MUST set the H bit to 0 for all string literals.

Receivers MUST treat prohibited encodings as a PROTOCOL_VIOLATION.

## Dynamic Table

Dynamic table entries store complete MOQT parameters (type and value).
The entry size calculation follows {{QPACK}} Section 3.2.1, using a fixed
name size of 4 bytes for the parameter type regardless of its encoded length.

### Encoder Stream Instructions

Set Dynamic Table Capacity:
: Sets the dynamic table capacity up to the peer's
  MOQT_QPACK_MAX_TABLE_CAPACITY. Encoders MAY reduce capacity dynamically.
  See {{QPACK}} Section 4.3.1.

Insert With Static Name Reference (T=1):
: The Name Index is the MOQT parameter type. Inserts a new dynamic table
  entry with that parameter type and the provided value.

Duplicate:
: Duplicates an existing dynamic table entry at a new index. Useful when
  an entry is near eviction but still frequently referenced. See {{QPACK}}
  Section 4.3.4.

Insert With Dynamic Name Reference (T=0):
: PROHIBITED. Parameter types are always known integers.

Insert With Literal Name:
: PROHIBITED. Parameter types are integers, never strings.

Receivers MUST close the session with PROTOCOL_VIOLATION if a prohibited
encoder instruction is received.

### Parameter Value Encoding

QPACK values contain the raw parameter value bytes without any MOQT length
prefix. The QPACK value length field serves as the length for binary values.

For binary parameters (odd parameter types in MOQT):
: The QPACK value contains the raw binary bytes. The QPACK value length
  replaces the MOQT Length field. No length prefix is included in the value.

For integer parameters (even parameter types in MOQT):
: The QPACK value contains the varint-encoded integer. Note that this
  encoding carries redundant length information: the QPACK value length
  specifies the byte count, while the varint encoding is self-delimiting.
  If the varint-encoded length does not match the QPACK value length, the
  receiver MUST close the session with PROTOCOL_VIOLATION.

For example, DELIVERY_TIMEOUT with value 200:
~~~
QPACK Value Length: 2
QPACK Value: 0xC8 0x01  (varint encoding of 200)
~~~

The receiver decodes the varint and verifies it consumed exactly 2 bytes.

### Authorization Token Encoding

The AUTHORIZATION TOKEN parameter value contains the Token structure:

~~~
Token Value = Token Type (vi64) || Token Payload (..)
~~~

For example, a JWT token (Token Type 1) is encoded as:

~~~
Literal Field Line With Name Reference:
  Name Index: 0x03 (AUTHORIZATION_TOKEN)
  Value: 0x01 || "eyJhbGciOiJIUzI1NiIs..."
~~~

When this token is inserted into the dynamic table, subsequent references
use only a single-byte Indexed Field Line.

## Decoding

When receiving a message with a Compressed Block, the receiver:

1. Parses fixed message fields (Request ID, Track Alias, etc.)
2. If a Compressed Block is present:
   a. Waits for any referenced dynamic table entries to become available,
      subject to MOQT_QPACK_BLOCKED_STREAMS limits
   b. Decodes the QPACK Compressed Block to recover namespace elements,
      track name, and parameters
3. Processes the fully decoded message

If QPACK decoding fails, the receiver MUST close the session with
MOQPACK_DECOMPRESSION_FAILED.

The total size of the decompressed message fields (the sum of all parameter
values, namespace elements, and track name, excluding interior length fields)
MUST NOT exceed 65535 bytes. If a decompressed message exceeds this limit,
the receiver MUST close the session with MOQPACK_DECOMPRESSION_FAILED.

# Dynamic Table Management

## Encoder Behavior

Encoders SHOULD insert frequently-used parameter values into the dynamic table.
Authorization tokens, Track Namespace Elements and Track names that will be
reused across multiple messages are prime candidates for insertion.

Short values like brief track names ("audio", "video") MAY be sent as literals
rather than inserted, since the overhead of insertion and indexed reference
is similar to sending the literal value. Insertion is more beneficial for:
- Long values (large auth tokens, long namespace elements)
- Values that will be reused across multiple messages (namespace elements
  shared by many tracks, auth tokens used for many subscriptions)

Encoders MUST respect the peer's MOQT_QPACK_MAX_TABLE_CAPACITY and
MOQT_QPACK_BLOCKED_STREAMS limits when making insertion and reference
decisions.

Encoders SHOULD use the QPACK duplicate instruction when a dynamic table
entry is at risk of eviction but is still frequently referenced.

### Known Received Count

Encoders track the Known Received Count as specified in {{QPACK}} Section 2.1.4
to determine which entries can be referenced without blocking. Encoders MUST
only reference dynamic table entries with absolute index less than the Known
Received Count when the number of streams that would be blocked by the
reference equals MOQT_QPACK_BLOCKED_STREAMS.

### Never-Indexed Literals

The 'N' bit in Literal Field Line representations signals that a value
MUST NOT be indexed by intermediaries. Encoders MAY set the 'N' bit
for sensitive values when:
- The value should not be cached by relays
- The value has low entropy and is vulnerable to compression attacks

Note that when MOQT_QPACK_INDEX_SETUP_AUTH is enabled, setup auth tokens are
implicitly inserted into the dynamic table (see
{{implicit-dynamic-table-seeding}}).  Endpoints that require 'N' bit semantics
for auth tokens MUST NOT enable MOQT_QPACK_INDEX_SETUP_AUTH and SHOULD instead
send tokens as Never-Indexed Literals in each message.

Intermediaries that re-encode MOQT messages MUST preserve the 'N' bit
semantics: values encoded with N=1 MUST NOT be inserted into the dynamic
table.

### Avoiding Flow Control Deadlocks

Writing large encoder instructions can cause deadlocks if the decoder
withholds flow control credit until the instruction is complete. To avoid
this, encoders SHOULD NOT write an encoder instruction unless sufficient
stream and connection flow-control credit is available for the entire
instruction. See {{QPACK}} Section 2.1.3.

## Decoder Behavior

Decoders MUST process encoder instructions from the QPACK encoder stream
before processing any message that might reference those insertions.

Decoders MUST send Section Acknowledgment instructions on the QPACK decoder
stream after successfully decoding a Compressed Block that references the
dynamic table (Required Insert Count > 0). Compressed Blocks that contain
only Literal Field Lines with Static Name References do not require
acknowledgment.

### Decoder Instructions with Request IDs

QPACK decoder instructions that reference streams (Section Acknowledgment,
Stream Cancellation) use Request IDs instead of QUIC Stream IDs. This
allows MOQT to operate over WebTransport where QUIC Stream IDs are not
exposed to the application.

Section Acknowledgment:
: Carries a Request ID. Acknowledges successful decoding of a Compressed
  Block that referenced the dynamic table. For streams with multiple such
  Compressed Blocks (e.g., SUBSCRIBE_NAMESPACE response stream with multiple
  NAMESPACE messages referencing the dynamic table), successive Section
  Acknowledgments for the same Request ID acknowledge successive Compressed
  Blocks in the order they were sent. Compressed Blocks with no dynamic
  table references are not counted.

Stream Cancellation:
: Carries a Request ID. Indicates the stream was cancelled before the
  Compressed Block(s) could be decoded. The encoder MUST NOT count
  references from that stream when determining eviction eligibility.

Insert Count Increment:
: Unchanged from {{QPACK}}. Carries an increment value, no Request ID.

The encoder tracks how many Compressed Blocks it has sent for each
Request ID. When it receives a Section Acknowledgment for a Request ID,
it knows which Compressed Block was acknowledged based on the count.


# Error Handling

## MOQPACK_DECOMPRESSION_FAILED {#moqpack_decompression_failed}

This document defines a new MOQT session error code:

MOQPACK_DECOMPRESSION_FAILED (0xTBD):
: A QPACK Compressed Block could not be decoded, or the decompressed message
  exceeded implementation limits. This is always a session error; the
  endpoint MUST close the MOQT session.

## QPACK Errors

QPACK decoding errors (as defined in Section 6 of {{QPACK}}) result in
session termination. The specific mapping is:

| QPACK Error | MOQT Session Error |
|-------------|-------------------|
| QPACK_DECOMPRESSION_FAILED | MOQPACK_DECOMPRESSION_FAILED |
| QPACK_ENCODER_STREAM_ERROR | PROTOCOL_VIOLATION |
| QPACK_DECODER_STREAM_ERROR | PROTOCOL_VIOLATION |


# Security Considerations

## Dynamic Table State

The QPACK dynamic table maintains state across messages. An attacker with
knowledge of dynamic table contents could potentially:

* Determine which authorization tokens have been used
* Infer subscription patterns from parameter compression ratios

Implementations SHOULD consider these privacy implications when deciding
which values to insert into the dynamic table.

## Compression Oracle Attacks

As with HTTP compression, implementers need to take care to avoid compression
oracle attacks where an attacker can infer secret values by observing compressed
message sizes. Applications SHOULD NOT mix attacker-controlled data with
secret authorization tokens in the same field section.

## Resource Exhaustion

Endpoints MUST enforce the negotiated table capacity limits to prevent
resource exhaustion attacks. An endpoint that attempts to exceed these
limits causes a session error.


# IANA Considerations

## Setup Parameter Types

This document registers the following Setup Parameter Types in the
"MOQT Setup Parameters" registry:

| Parameter Type | Parameter Name | Specification |
|----------------|----------------|---------------|
| 0x10 | MOQT_QPACK_MAX_TABLE_CAPACITY | {{extension-negotiation}} |
| 0x11 | MOQT_QPACK_BLOCKED_STREAMS | {{extension-negotiation}} |
| 0x12 | MOQT_QPACK_INDEX_SETUP_AUTH | {{extension-negotiation}} |

## Unidirectional Stream Types

This document registers the following unidirectional stream types in the
"MOQT Stream Types" registry:

| Stream Type | Name | Specification |
|-------------|------|---------------|
| 0x1f107a60 | QPACK_ENCODER_STREAM | {{stream-types}} |
| 0x1f107a61 | QPACK_DECODER_STREAM | {{stream-types}} |

## Pseudo-Parameter Types

This document registers the following pseudo-parameter types in the
"MOQT Message Parameters" registry. These types are reserved for use in
QPACK Compressed Blocks and MUST NOT appear in standard Parameters fields.

| Parameter Type | Parameter Name | Specification |
|----------------|----------------|---------------|
| 0x00 | TRACK_NAMESPACE_ELEMENT | {{pseudo-parameter-types}} |
| 0x01 | TRACK_NAMESPACE_SET | {{pseudo-parameter-types}} |
| 0x02 | TRACK_NAME | {{pseudo-parameter-types}} |

## Session Error Codes

This document registers the following session error code in the
"MOQT Session Error Codes" registry:

| Error Code | Name | Specification |
|------------|------|---------------|
| 0xTBD | MOQPACK_DECOMPRESSION_FAILED | {{moqpack_decompression_failed}} |


--- back

# Acknowledgments
{:numbered="false"}

The QPACK specification {{QPACK}} provides the foundation for this work.
The design of HTTP/3 header compression informed many decisions in this
document.

Claude, by Anthropic, assisted in drafting this document.


# Example Encoding
{:numbered="false"}

This appendix provides an example of QPACK-compressed MOQT messages,
demonstrating the Required Insert Count, Base, and relative indexing.

## Scenario
{:numbered="false"}

A client sends three SUBSCRIBE messages to the same track with the same
authorization token:

* Track Namespace: ("conference", "room42")
* Track Name: "audio"
* Token Type: 1 (e.g., JWT)
* Token Value: "eyJhbGciOiJIUzI1NiIs..." (500 bytes)

The auth token was sent in CLIENT_SETUP and is implicitly inserted at
dynamic table absolute index 0 after QPACK negotiation succeeds.

## First SUBSCRIBE
{:numbered="false"}

The encoder inserts namespace elements on the encoder stream. The short track
name "audio" is sent as a literal rather than inserted:

~~~
Encoder Stream:
  Insert With Static Name Reference
    Name Index: 0x00 (TRACK_NAMESPACE_ELEMENT)
    Value: "conference"
  Insert With Static Name Reference
    Name Index: 0x00 (TRACK_NAMESPACE_ELEMENT)
    Value: "room42"
~~~

Dynamic table state after insertions:

| Absolute Index | Parameter Type | Value |
|----------------|----------------|-------|
| 0 | AUTH_TOKEN (implicit) | Token Type 1, "eyJ..." |
| 1 | TRACK_NAMESPACE_ELEMENT | "conference" |
| 2 | TRACK_NAMESPACE_ELEMENT | "room42" |

The encoder sends the SUBSCRIBE with Required Insert Count = 3 and Base = 3:

~~~
SUBSCRIBE Message:
  Type: 0x3
  Request ID: 1
  Track Alias: 100
  Compressed Block Length: 12
  Compressed Block:
    Required Insert Count: 3 (encoded per RFC 9204 Section 4.5.1.1)
    Base: Sign=0, Delta=0 (Base = Required Insert Count = 3)
    Indexed Field Line (Dynamic, relative index 1)  // abs 1 = "conference"
    Indexed Field Line (Dynamic, relative index 0)  // abs 2 = "room42"
    Literal Field Line (Static Name 0x02, Value "audio")  // TRACK_NAME
    Indexed Field Line (Dynamic, relative index 2)  // abs 0 = AUTH_TOKEN
~~~

Relative index calculation: relative = Base - 1 - absolute

* "conference": relative = 3 - 1 - 1 = 1
* "room42": relative = 3 - 1 - 2 = 0
* AUTH_TOKEN: relative = 3 - 1 - 0 = 2

Compressed Block: ~12 bytes (including prefix and literal track name)
Uncompressed equivalent: ~526 bytes

## Subsequent SUBSCRIBEs to Same Track
{:numbered="false"}

No encoder stream instructions needed; namespace elements are in the table,
track name is sent as literal again:

~~~
SUBSCRIBE Message:
  Type: 0x3
  Request ID: 2
  Track Alias: 101
  Compressed Block:
    Required Insert Count: 3
    Base: Sign=0, Delta=0
    Indexed Field Line (Dynamic, relative index 1)  // "conference"
    Indexed Field Line (Dynamic, relative index 0)  // "room42"
    Literal Field Line (Static Name 0x02, Value "audio")  // TRACK_NAME
    Indexed Field Line (Dynamic, relative index 2)  // AUTH_TOKEN
~~~

Each subsequent SUBSCRIBE to the same track: ~12 bytes instead of ~526 bytes.

## SUBSCRIBE to Different Track, Same Namespace
{:numbered="false"}

The track name "video" is also short, so we send it as a literal. No encoder
stream instructions needed:

~~~
SUBSCRIBE Message:
  Type: 0x3
  Request ID: 3
  Track Alias: 102
  Compressed Block:
    Required Insert Count: 3
    Base: Sign=0, Delta=0
    Indexed Field Line (Dynamic, relative index 1)  // "conference"
    Indexed Field Line (Dynamic, relative index 0)  // "room42"
    Literal Field Line (Static Name 0x02, Value "video")  // TRACK_NAME
    Indexed Field Line (Dynamic, relative index 2)  // AUTH_TOKEN
~~~

The namespace elements are reused; the different track name is sent as a literal.

## Total Savings
{:numbered="false"}

| Encoding | Sub 1 | Sub 2 | Sub 3 | Total |
|----------|-------|-------|-------|-------|
| Uncompressed | 526 | 526 | 526 | 1578 |
| MOQPACK | 32* | 12 | 12 | 56 |

\* Includes encoder stream insertions for namespace elements (~20 bytes)

Savings: ~96% reduction in namespace/name/token overhead.


# Code Point Summary
{:numbered="false"}

This appendix summarizes all code points defined or used by this extension.

## Setup Parameters
{:numbered="false"}

| Type | Name |
|------|------|
| 0x10 | MOQT_QPACK_MAX_TABLE_CAPACITY |
| 0x11 | MOQT_QPACK_BLOCKED_STREAMS |
| 0x12 | MOQT_QPACK_INDEX_SETUP_AUTH |

## Stream Types
{:numbered="false"}

| Type | Name |
|------|------|
| 0x1f107a60 | QPACK_ENCODER_STREAM |
| 0x1f107a61 | QPACK_DECODER_STREAM |

## Pseudo-Parameter Types
{:numbered="false"}

| Type | Name |
|------|------|
| 0x00 | TRACK_NAMESPACE_ELEMENT |
| 0x01 | TRACK_NAMESPACE_SET |
| 0x02 | TRACK_NAME |

## MOQPACK Message Types
{:numbered="false"}

The MOQPACK flag bit (0x40) is OR'd with standard MOQT message types:

| Standard | MOQPACK | Message |
|----------|---------|---------|
| 0x02 | 0x42 | REQUEST_UPDATE |
| 0x03 | 0x43 | SUBSCRIBE |
| 0x04 | 0x44 | SUBSCRIBE_OK |
| 0x05 | 0x45 | REQUEST_ERROR |
| 0x06 | 0x46 | PUBLISH_NAMESPACE |
| 0x07 | 0x47 | REQUEST_OK |
| 0x08 | 0x48 | NAMESPACE |
| 0x0D | 0x4D | TRACK_STATUS |
| 0x0E | 0x4E | NAMESPACE_DONE |
| 0x11 | 0x51 | SUBSCRIBE_NAMESPACE |
| 0x16 | 0x56 | FETCH |
| 0x18 | 0x58 | FETCH_OK |
| 0x1D | 0x5D | PUBLISH |
| 0x1E | 0x5E | PUBLISH_OK |
