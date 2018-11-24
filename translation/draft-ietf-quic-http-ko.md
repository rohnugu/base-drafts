---
title: Hypertext Transfer Protocol Version 3 (HTTP/3)
abbrev: HTTP/3
docname: draft-ietf-quic-http-latest
date: {DATE}
category: std
ipr: trust200902
area: Transport
workgroup: QUIC

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping]

author:
-
    ins: M. Bishop
    name: Mike Bishop
    org: Akamai
    email: mbishop@evequefou.be
    role: editor

normative:

  QUIC-TRANSPORT:
    title: "QUIC: A UDP-Based Multiplexed and Secure Transport"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-transport-latest
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Fastly
        role: editor
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla
        role: editor

  QPACK:
    title: "QPACK: Header Compression for HTTP over QUIC"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-qpack-latest
    author:
      -
          ins: C. Krasic
          name: Charles 'Buck' Krasic
          org: Google, Inc
      -
          ins: M. Bishop
          name: Mike Bishop
          org: Akamai Technologies
      -
          ins: A. Frindell
          name: Alan Frindell
          org: Facebook
          role: editor


informative:


--- abstract

QUIC 전송 프로토콜은 스트림 멀티플렉싱, 스트림당 플로우 제어, 저지연 연결 설립
등, HTTP를 위한 전송이 가질 바람직한 특징들을 가지고 있다. 이 문서는 QUIC에
HTTP 의미를 매핑하는 방법을 설명한다. 이 문서는 또한 QUIC이 포함한 HTTP/2
특징을 특정하고, HTTP/2 확장을 어떻게 HTTP/3으로 이전가능한지 설명한다.

--- note_Note_to_Readers

Discussion of this draft takes place on the QUIC working group mailing list
(quic@ietf.org), which is archived at
<https://mailarchive.ietf.org/arch/search/?email_list=quic>.

Working Group information can be found at <https://github.com/quicwg>; source
code and issues list for this draft can be found at
<https://github.com/quicwg/base-drafts/labels/-http>.

본 한국어 문서는 노희준 (hjroh@korea.ac.kr)이 네트워크 프로토콜 연구를
위해 초벌 번역한 것이다. 본 Internet-Draft는 Simplified BSD License를 따르며,
번역자는 본 번역물에 대해 해당 라이센스의 허용 범위 내에서 2차 저작물로서의
모든 권리를 가진다. 단, 번역자는 번역 내용을 참고함으로써 발생할 수 있는 어떠한
문제에 대해서도 책임을 지지 않는다.

--- middle


# Introduction

HTTP semantics are used for a broad range of services on the Internet. These
semantics have commonly been used with two different TCP mappings, HTTP/1.1 and
HTTP/2.  HTTP/2 introduced a framing and multiplexing layer to improve latency
without modifying the transport layer.  However, TCP's lack of visibility into
parallel requests in both mappings limited the possible performance gains.

The QUIC transport protocol incorporates stream multiplexing and per-stream flow
control, similar to that provided by the HTTP/2 framing layer. By providing
reliability at the stream level and congestion control across the entire
connection, it has the capability to improve the performance of HTTP compared to
a TCP mapping.  QUIC also incorporates TLS 1.3 at the transport layer, offering
comparable security to running TLS over TCP, but with improved connection setup
latency.

This document describes a mapping of HTTP semantics over the QUIC transport
protocol, drawing heavily on design of HTTP/2. This document identifies HTTP/2
features that are subsumed by QUIC, and describes how the other features can be
implemented atop QUIC.

QUIC is described in {{QUIC-TRANSPORT}}.  For a full description of HTTP/2, see
{{!RFC7540}}.


## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

Field definitions are given in Augmented Backus-Naur Form (ABNF), as defined in
{{!RFC5234}}.

This document uses the variable-length integer encoding from
{{QUIC-TRANSPORT}}.

Protocol elements called "frames" exist in both this document and
{{QUIC-TRANSPORT}}. Where frames from {{QUIC-TRANSPORT}} are referenced, the
frame name will be prefaced with "QUIC."  For example, "QUIC CONNECTION_CLOSE
frames."  References without this preface refer to frames defined in {{frames}}.


# Connection Setup and Management

## Draft Version Identification

> **RFC Editor's Note:**  Please remove this section prior to publication of a
> final version of this document.

HTTP/3 uses the token "h3" to identify itself in ALPN and Alt-Svc.  Only
implementations of the final, published RFC can identify themselves as "h3".
Until such an RFC exists, implementations MUST NOT identify themselves using
this string.

Implementations of draft versions of the protocol MUST add the string "-" and
the corresponding draft number to the identifier. For example,
draft-ietf-quic-http-01 is identified using the string "h3-01".

Non-compatible experiments that are based on these draft versions MUST append
the string "-" and an experiment name to the identifier. For example, an
experimental implementation based on draft-ietf-quic-http-09 which reserves an
extra stream for unsolicited transmission of 1980s pop music might identify
itself as "h3-09-rickroll". Note that any label MUST conform to the "token"
syntax defined in Section 3.2.6 of [RFC7230]. Experimenters are encouraged to
coordinate their experiments on the quic@ietf.org mailing list.

## Discovering an HTTP/3 Endpoint

An HTTP origin advertises the availability of an equivalent HTTP/3 endpoint via
the Alt-Svc HTTP response header field or the HTTP/2 ALTSVC frame
({{!ALTSVC=RFC7838}}), using the ALPN token defined in
{{connection-establishment}}.

For example, an origin could indicate in an HTTP/1.1 or HTTP/2 response that
HTTP/3 was available on UDP port 50781 at the same hostname by including the
following header field in any response:

~~~ example
Alt-Svc: h3=":50781"
~~~

On receipt of an Alt-Svc record indicating HTTP/3 support, a client MAY attempt
to establish a QUIC connection to the indicated host and port and, if
successful, send HTTP requests using the mapping described in this document.

Connectivity problems (e.g. firewall blocking UDP) can result in QUIC connection
establishment failure, in which case the client SHOULD continue using the
existing connection or try another alternative endpoint offered by the origin.

Servers MAY serve HTTP/3 on any UDP port, since an alternative always includes
an explicit port.

### QUIC Version Hints {#alt-svc-version-hint}

This document defines the "quic" parameter for Alt-Svc, which MAY be used to
provide version-negotiation hints to HTTP/3 clients. QUIC versions are four-byte
sequences with no additional constraints on format. Leading zeros SHOULD be
omitted for brevity.

Syntax:

~~~ abnf
quic = DQUOTE version-number [ "," version-number ] * DQUOTE
version-number = 1*8HEXDIG; hex-encoded QUIC version
~~~

Where multiple versions are listed, the order of the values reflects the
server's preference (with the first value being the most preferred version).
Reserved versions MAY be listed, but unreserved versions which are not supported
by the alternative SHOULD NOT be present in the list. Origins MAY omit supported
versions for any reason.

Clients MUST ignore any included versions which they do not support.  The "quic"
parameter MUST NOT occur more than once; clients SHOULD process only the first
occurrence.

For example, suppose a server supported both version 0x00000001 and the version
rendered in ASCII as "Q034".  If it also opted to include the reserved version
(from Section 15 of {{QUIC-TRANSPORT}}) 0x1abadaba, it could specify the
following header field:

~~~ example
Alt-Svc: h3=":49288";quic="1,1abadaba,51303334"
~~~

A client acting on this header field would drop the reserved version (not
supported), then attempt to connect to the alternative using the first version
in the list which it does support, if any.

## Connection Establishment {#connection-establishment}

HTTP/3 relies on QUIC as the underlying transport.  The QUIC version being used
MUST use TLS version 1.3 or greater as its handshake protocol.  HTTP/3 clients
MUST indicate the target domain name during the TLS handshake. This may be done
using the Server Name Indication (SNI) {{!RFC6066}} extension to TLS or using
some other mechanism.

QUIC connections are established as described in {{QUIC-TRANSPORT}}. During
connection establishment, HTTP/3 support is indicated by selecting the ALPN
token "hq" in the TLS handshake.  Support for other application-layer protocols
MAY be offered in the same handshake.

While connection-level options pertaining to the core QUIC protocol are set in
the initial crypto handshake, HTTP/3-specific settings are conveyed in the
SETTINGS frame. After the QUIC connection is established, a SETTINGS frame
({{frame-settings}}) MUST be sent by each endpoint as the initial frame of their
respective HTTP control stream (see {{control-streams}}). The server MUST NOT
process any request streams or send responses until the client's SETTINGS frame
has been received.

## Connection Reuse

Once a connection exists to a server endpoint, this connection MAY be reused for
requests with multiple different URI authority components.  The client MAY send
any requests for which the client considers the server authoritative.

An authoritative HTTP/3 endpoint is typically discovered because the client has
received an Alt-Svc record from the request's origin which nominates the
endpoint as a valid HTTP Alternative Service for that origin.  As required by
{{RFC7838}}, clients MUST check that the nominated server can present a valid
certificate for the origin before considering it authoritative. Clients MUST NOT
assume that an HTTP/3 endpoint is authoritative for other origins without an
explicit signal.

A server that does not wish clients to reuse connections for a particular origin
can indicate that it is not authoritative for a request by sending a 421
(Misdirected Request) status code in response to the request (see Section 9.1.2
of {{!RFC7540}}).

The considerations discussed in Section 9.1 of {{?RFC7540}} also apply to the
management of HTTP/3 connections.

# Stream Mapping and Usage {#stream-mapping}

A QUIC stream provides reliable in-order delivery of bytes, but makes no
guarantees about order of delivery with regard to bytes on other streams. On the
wire, data is framed into QUIC STREAM frames, but this framing is invisible to
the HTTP framing layer. The transport layer buffers and orders received QUIC
STREAM frames, exposing the data contained within as a reliable byte stream to
the application.

QUIC streams can be either unidirectional, carrying data only from initiator to
receiver, or bidirectional.  Streams can be initiated by either the client or
the server.  For more detail on QUIC streams, see Section 2 of
{{QUIC-TRANSPORT}}.

When HTTP headers and data are sent over QUIC, the QUIC layer handles most of
the stream management.  HTTP does not need to do any separate multiplexing when
using QUIC - data sent over a QUIC stream always maps to a particular HTTP
transaction or connection context.

## Bidirectional Streams

All client-initiated bidirectional streams are used for HTTP requests and
responses.  A bidirectional stream ensures that the response can be readily
correlated with the request. This means that the client's first request occurs
on QUIC stream 0, with subsequent requests on stream 4, 8, and so on. In order
to permit these streams to open, an HTTP/3 client SHOULD send non-zero values
for the QUIC transport parameters `initial_max_stream_data_bidi_local`. An
HTTP/3 server SHOULD send non-zero values for the QUIC transport parameters
`initial_max_stream_data_bidi_remote` and `initial_max_bidi_streams`. It is
recommended that `initial_max_bidi_streams` be no smaller than 100, so as to not
unnecessarily limit parallelism.

These streams carry frames related to the request/response (see
{{request-response}}). When a stream terminates cleanly, if the last frame on
the stream was truncated, this MUST be treated as a connection error (see
HTTP_MALFORMED_FRAME in {{http-error-codes}}).  Streams which terminate abruptly
may be reset at any point in the frame.

HTTP/3 does not use server-initiated bidirectional streams; clients MUST omit or
specify a value of zero for the QUIC transport parameter
`initial_max_bidi_streams`.


## Unidirectional Streams

Unidirectional streams, in either direction, are used for a range of purposes.
The purpose is indicated by a stream type, which is sent as a single byte header
at the start of the stream. The format and structure of data that follows this
header is determined by the stream type.

~~~~~~~~~~ drawing
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|Stream Type (8)|
+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-stream-header title="Unidirectional Stream Header"}

Some stream types are reserved ({{stream-grease}}).  Two stream types are
defined in this document: control streams ({{control-streams}}) and push streams
({{push-streams}}).  Other stream types can be defined by extensions to HTTP/3.

Both clients and servers SHOULD send a value of three or greater for the QUIC
transport parameter `initial_max_uni_streams`.

If the stream header indicates a stream type which is not supported by the
recipient, the remainder of the stream cannot be consumed as the semantics are
unknown. Recipients of unknown stream types MAY trigger a QUIC STOP_SENDING
frame with an error code of HTTP_UNKNOWN_STREAM_TYPE, but MUST NOT consider such
streams to be an error of any kind.

Implementations MAY send stream types before knowing whether the peer supports
them.  However, stream types which could modify the state or semantics of
existing protocol components, including QPACK or other extensions, MUST NOT be
sent until the peer is known to support them.

###  Control Streams

A control stream is indicated by a stream type of `0x43` (ASCII 'C').  Data on
this stream consists of HTTP/3 frames, as defined in {{frames}}.

Each side MUST initiate a single control stream at the beginning of the
connection and send its SETTINGS frame as the first frame on this stream.  If
the first frame of the control stream is any other frame type, this MUST be
treated as a connection error of type HTTP_MISSING_SETTINGS. Only one control
stream per peer is permitted; receipt of a second stream which claims to be a
control stream MUST be treated as a connection error of type
HTTP_WRONG_STREAM_COUNT.  If the control stream is closed at any point, this
MUST be treated as a connection error of type HTTP_CLOSED_CRITICAL_STREAM.

A pair of unidirectional streams is used rather than a single bidirectional
stream.  This allows either peer to send data as soon they are able.  Depending
on whether 0-RTT is enabled on the connection, either client or server might be
able to send stream data first after the cryptographic handshake completes.

### Push Streams

A push stream is indicated by a stream type of `0x50` (ASCII 'P'), followed by
the Push ID of the promise that it fulfills, encoded as a variable-length
integer. The remaining data on this stream consists of HTTP/3 frames, as defined
in {{frames}}, and fulfills a promised server push.  Server push and Push IDs
are described in {{server-push}}.

Only servers can push; if a server receives a client-initiated push stream, this
MUST be treated as a stream error of type HTTP_WRONG_STREAM_DIRECTION.

~~~~~~~~~~ drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Stream Type (8)|                  Push ID (i)                ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-push-stream-header title="Push Stream Header"}

Each Push ID MUST only be used once in a push stream header. If a push stream
header includes a Push ID that was used in another push stream header, the
client MUST treat this as a connection error of type HTTP_DUPLICATE_PUSH.

### Reserved Stream Types {#stream-grease}

Stream types of the format `0x1f * N` are reserved to exercise the requirement
that unknown types be ignored. These streams have no semantic meaning, and can
be sent when application-layer padding is desired.  They MAY also be sent on
connections where no request data is currently being transferred. Endpoints MUST
NOT consider these streams to have any meaning upon receipt.

The payload and length of the stream are selected in any manner the
implementation chooses.


# HTTP Framing Layer {#http-framing-layer}

Frames are used on control streams, request streams, and push streams.  This
section describes HTTP framing in QUIC.  For a comparison with HTTP/2 frames,
see {{h2-frames}}.

## Frame Layout

All frames have the following format:

~~~~~~~~~~ drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Length (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Type (8)   |               Frame Payload (*)             ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-frame title="HTTP/3 frame format"}

A frame includes the following fields:

  Length:
  : A variable-length integer that describes the length of the Frame Payload.
    This length does not include the Type field.

  Type:
  : An 8-bit type for the frame.

  Frame Payload:
  : A payload, the semantics of which are determined by the Type field.

Each frame's payload MUST contain exactly the identified fields.  A frame that
contains additional bytes after the identified fields or a frame that terminates
before the end of the identified fields MUST be treated as a connection error of
type HTTP_MALFORMED_FRAME.

## Frame Definitions {#frames}

### DATA {#frame-data}

DATA frames (type=0x0) convey arbitrary, variable-length sequences of bytes
associated with an HTTP request or response payload.

DATA frames MUST be associated with an HTTP request or response.  If a DATA
frame is received on either control stream, the recipient MUST respond with a
connection error ({{errors}}) of type HTTP_WRONG_STREAM.

~~~~~~~~~~ drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Payload (*)                         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-data title="DATA frame payload"}

DATA frames MUST contain a non-zero-length payload.  If a DATA frame is received
with a payload length of zero, the recipient MUST respond with a stream error
({{errors}}) of type HTTP_MALFORMED_FRAME.

### HEADERS {#frame-headers}

The HEADERS frame (type=0x1) is used to carry a header block, compressed using
QPACK. See [QPACK] for more details.

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Header Block (*)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-headers title="HEADERS frame payload"}

HEADERS frames can only be sent on request / push streams.

### PRIORITY {#frame-priority}

The PRIORITY (type=0x02) frame specifies the sender-advised priority of a
stream.  In order to ensure that prioritization is processed in a consistent
order, PRIORITY frames MUST be sent on the control stream.  A PRIORITY frame
sent on any other stream MUST be treated as a connection error of type
HTTP_WRONG_STREAM.

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|PT |DT |Empty|E|          Prioritized Element ID (i)         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Element Dependency ID (i)                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Weight (8)  |
+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-priority title="PRIORITY frame payload"}

The PRIORITY frame payload has the following fields:

  Prioritized Type:
  : A two-bit field indicating the type of element being prioritized.

  Dependency Type:
  : A two-bit field indicating the type of element being depended on.

  Empty:
  : A three-bit field which MUST be zero when sent and MUST be ignored
    on receipt.

  Exclusive:
  : A flag which indicates that the stream dependency is exclusive (see
    {{!RFC7540}}, Section 5.3).

  Prioritized Element ID:
  : A variable-length integer that identifies the element being prioritized.
    Depending on the value of Prioritized Type, this contains the Stream ID of a
    request stream, the Push ID of a promised resource, or a Placeholder ID of a
    placeholder.

  Element Dependency ID:
  : A variable-length integer that identifies the element on which a dependency
    is being expressed. Depending on the value of Dependency Type, this contains
    the Stream ID of a request stream, the Push ID of a promised resource, the
    Placeholder ID of a placeholder, or is ignored.  For details of
    dependencies, see {{priority}} and {{!RFC7540}}, Section 5.3.

  Weight:
  : An unsigned 8-bit integer representing a priority weight for the stream (see
    {{!RFC7540}}, Section 5.3). Add one to the value to obtain a weight between
    1 and 256.

A PRIORITY frame identifies an element to prioritize, and an element upon which
it depends.  A Prioritized ID or Dependency ID identifies a client-initiated
request using the corresponding stream ID, a server push using a Push ID (see
{{frame-push-promise}}), or a placeholder using a Placeholder ID (see
{{placeholders}}).

The values for the Prioritized Element Type and Element Dependency Type imply
the interpretation of the associated Element ID fields.

| Type Bits | Type Description | Element ID Contents |
| --------- | ---------------- | ------------------- |
| 00        | Request stream   | Stream ID           |
| 01        | Push stream      | Push ID             |
| 10        | Placeholder      | Placeholder ID      |
| 11        | Root of the tree | Ignored             |

Note that the root of the tree cannot be referenced using a Stream ID of 0, as
in {{!RFC7540}}; QUIC stream 0 carries a valid HTTP request.  The root of the
tree cannot be reprioritized. A PRIORITY frame that prioritizes the root of the
tree MUST be treated as a connection error of type HTTP_MALFORMED_FRAME.

When a PRIORITY frame claims to reference a request, the associated ID MUST
identify a client-initiated bidirectional stream.  A server MUST treat receipt
of PRIORITY frame with a Stream ID of any other type as a connection error of
type HTTP_MALFORMED_FRAME.

A PRIORITY frame that references a non-existent Push ID or a Placeholder ID
greater than the server's limit MUST be treated as an HTTP_MALFORMED_FRAME
error.


### CANCEL_PUSH {#frame-cancel-push}

The CANCEL_PUSH frame (type=0x3) is used to request cancellation of a server
push prior to the push stream being created.  The CANCEL_PUSH frame identifies a
server push by Push ID (see {{frame-push-promise}}), encoded as a
variable-length integer.

When a server receives this frame, it aborts sending the response for the
identified server push.  If the server has not yet started to send the server
push, it can use the receipt of a CANCEL_PUSH frame to avoid opening a push
stream.  If the push stream has been opened by the server, the server SHOULD
send a QUIC RESET_STREAM frame on that stream and cease transmission of the
response.

A server can send this frame to indicate that it will not be fulfilling a
promise prior to creation of a push stream.  Once the push stream has been
created, sending CANCEL_PUSH has no effect on the state of the push stream.  A
QUIC RESET_STREAM frame SHOULD be used instead to abort transmission of the
server push response.

A CANCEL_PUSH frame is sent on the control stream.  Sending a CANCEL_PUSH frame
on a stream other than the control stream MUST be treated as a stream error of
type HTTP_WRONG_STREAM.

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Push ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-cancel-push title="CANCEL_PUSH frame payload"}

The CANCEL_PUSH frame carries a Push ID encoded as a variable-length integer.
The Push ID identifies the server push that is being cancelled (see
{{frame-push-promise}}).

If the client receives a CANCEL_PUSH frame, that frame might identify a Push ID
that has not yet been mentioned by a PUSH_PROMISE frame.

An endpoint MUST treat a CANCEL_PUSH frame which does not contain exactly one
properly-formatted variable-length integer as a connection error of type
HTTP_MALFORMED_FRAME.


### SETTINGS {#frame-settings}

The SETTINGS frame (type=0x4) conveys configuration parameters that affect how
endpoints communicate, such as preferences and constraints on peer behavior.
Individually, a SETTINGS parameter can also be referred to as a "setting"; the
identifier and value of each setting parameter can be referred to as a "setting
identifier" and a "setting value".

SETTINGS parameters are not negotiated; they describe characteristics of the
sending peer, which can be used by the receiving peer. However, a negotiation
can be implied by the use of SETTINGS -- a peer uses SETTINGS to advertise a set
of supported values. The recipient can then choose which entries from this list
are also acceptable and proceed with the value it has chosen. (This choice could
be announced in a field of an extension frame, or in its own value in SETTINGS.)

Different values for the same parameter can be advertised by each peer. For
example, a client might be willing to consume a very large response header,
while servers are more cautious about request size.

Parameters MUST NOT occur more than once.  A receiver MAY treat the presence of
the same parameter more than once as a connection error of type
HTTP_MALFORMED_FRAME.

The payload of a SETTINGS frame consists of zero or more parameters, each
consisting of an unsigned 16-bit setting identifier and a value which uses the
QUIC variable-length integer encoding.

~~~~~~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identifier (16)       |           Value (i)         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~
{: #fig-ext-settings title="SETTINGS parameter format"}

Each value MUST be compared against the remaining length of the SETTINGS frame.
A variable-length integer value which cannot fit within the remaining length of
the SETTINGS frame MUST cause the SETTINGS frame to be considered malformed and
trigger a connection error of type HTTP_MALFORMED_FRAME.

An implementation MUST ignore the contents for any SETTINGS identifier it does
not understand.

SETTINGS frames always apply to a connection, never a single stream.  A SETTINGS
frame MUST be sent as the first frame of each control stream (see
{{control-streams}}) by each peer, and MUST NOT be sent subsequently or on any
other stream. If an endpoint receives a SETTINGS frame on a different stream,
the endpoint MUST respond with a connection error of type HTTP_WRONG_STREAM. If
an endpoint receives a second SETTINGS frame, the endpoint MUST respond with a
connection error of type HTTP_UNEXPECTED_FRAME.

The SETTINGS frame affects connection state. A badly formed or incomplete
SETTINGS frame MUST be treated as a connection error ({{errors}}) of type
HTTP_MALFORMED_FRAME.


#### Defined SETTINGS Parameters {#settings-parameters}

The following settings are defined in HTTP/3:

  SETTINGS_NUM_PLACEHOLDERS (0x3):
  : This value SHOULD be non-zero.  The default value is 16.

  SETTINGS_MAX_HEADER_LIST_SIZE (0x6):
  : The default value is unlimited.

Setting identifiers of the format `0x?a?a` are reserved to exercise the
requirement that unknown identifiers be ignored.  Such settings have no defined
meaning. Endpoints SHOULD include at least one such setting in their SETTINGS
frame. Endpoints MUST NOT consider such settings to have any meaning upon
receipt.

Because the setting has no defined meaning, the value of the setting can be any
value the implementation selects.

Additional settings MAY be defined by extensions to HTTP/3.

#### Initialization

When a 0-RTT QUIC connection is being used, the client's initial requests will
be sent before the arrival of the server's SETTINGS frame.  Clients MUST store
the settings the server provided in the session being resumed and MUST comply
with stored settings until the server's current settings are received.
Remembered settings apply to the new connection until the server's SETTINGS
frame is received.

A server can remember the settings that it advertised, or store an
integrity-protected copy of the values in the ticket and recover the information
when accepting 0-RTT data. A server uses the HTTP/3 settings values in
determining whether to accept 0-RTT data.

A server MAY accept 0-RTT and subsequently provide different settings in its
SETTINGS frame. If 0-RTT data is accepted by the server, its SETTINGS frame MUST
NOT reduce any limits or alter any values that might be violated by the client
with its 0-RTT data.

When a 1-RTT QUIC connection is being used, the client MUST NOT send requests
prior to receiving and processing the server's SETTINGS frame.

### PUSH_PROMISE {#frame-push-promise}

The PUSH_PROMISE frame (type=0x05) is used to carry a promised request header
set from server to client, as in HTTP/2.

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Push ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Header Block (*)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-push-promise title="PUSH_PROMISE frame payload"}

The payload consists of:

Push ID:
: A variable-length integer that identifies the server push operation.  A Push
  ID is used in push stream headers ({{server-push}}), CANCEL_PUSH frames
  ({{frame-cancel-push}}), and PRIORITY frames ({{frame-priority}}).

Header Block:
: QPACK-compressed request header fields for the promised response.  See [QPACK]
  for more details.

A server MUST NOT use a Push ID that is larger than the client has provided in a
MAX_PUSH_ID frame ({{frame-max-push-id}}).  A client MUST treat receipt of a
PUSH_PROMISE that contains a larger Push ID than the client has advertised as a
connection error of type HTTP_MALFORMED_FRAME.

A server MAY use the same Push ID in multiple PUSH_PROMISE frames.  This allows
the server to use the same server push in response to multiple concurrent
requests.  Referencing the same server push ensures that a PUSH_PROMISE can be
made in relation to every response in which server push might be needed without
duplicating pushes.

A server that uses the same Push ID in multiple PUSH_PROMISE frames MUST include
the same header fields each time.  The bytes of the header block MAY be
different due to differing encoding, but the header fields and their values MUST
be identical.  Note that ordering of header fields is significant.  A client
MUST treat receipt of a PUSH_PROMISE with conflicting header field values for
the same Push ID as a connection error of type HTTP_MALFORMED_FRAME.

Allowing duplicate references to the same Push ID is primarily to reduce
duplication caused by concurrent requests.  A server SHOULD avoid reusing a Push
ID over a long period.  Clients are likely to consume server push responses and
not retain them for reuse over time.  Clients that see a PUSH_PROMISE that uses
a Push ID that they have since consumed and discarded are forced to ignore the
PUSH_PROMISE.


### GOAWAY {#frame-goaway}

The GOAWAY frame (type=0x7) is used to initiate graceful shutdown of a
connection by a server.  GOAWAY allows a server to stop accepting new requests
while still finishing processing of previously received requests.  This enables
administrative actions, like server maintenance.  GOAWAY by itself does not
close a connection.

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Stream ID (i)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-goaway title="GOAWAY frame payload"}

The GOAWAY frame carries a QUIC Stream ID for a client-initiated bidirectional
stream encoded as a variable-length integer.  A client MUST treat receipt of a
GOAWAY frame containing a Stream ID of any other type as a connection error of
type HTTP_MALFORMED_FRAME.

Clients do not need to send GOAWAY to initiate a graceful shutdown; they simply
stop making new requests.  A server MUST treat receipt of a GOAWAY frame on any
stream as a connection error ({{errors}}) of type HTTP_UNEXPECTED_FRAME.

The GOAWAY frame applies to the connection, not a specific stream.  A client
MUST treat a GOAWAY frame on a stream other than the control stream as a
connection error ({{errors}}) of type HTTP_UNEXPECTED_FRAME.

See {{connection-shutdown}} for more information on the use of the GOAWAY frame.

### MAX_PUSH_ID {#frame-max-push-id}

The MAX_PUSH_ID frame (type=0xD) is used by clients to control the number of
server pushes that the server can initiate.  This sets the maximum value for a
Push ID that the server can use in a PUSH_PROMISE frame.  Consequently, this
also limits the number of push streams that the server can initiate in addition
to the limit set by the QUIC MAX_STREAM_ID frame.

The MAX_PUSH_ID frame is always sent on a control stream.  Receipt of a
MAX_PUSH_ID frame on any other stream MUST be treated as a connection error of
type HTTP_WRONG_STREAM.

A server MUST NOT send a MAX_PUSH_ID frame.  A client MUST treat the receipt of
a MAX_PUSH_ID frame as a connection error of type HTTP_MALFORMED_FRAME.

The maximum Push ID is unset when a connection is created, meaning that a server
cannot push until it receives a MAX_PUSH_ID frame.  A client that wishes to
manage the number of promised server pushes can increase the maximum Push ID by
sending MAX_PUSH_ID frames as the server fulfills or cancels server pushes.

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Push ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-max-push title="MAX_PUSH_ID frame payload"}

The MAX_PUSH_ID frame carries a single variable-length integer that identifies
the maximum value for a Push ID that the server can use (see
{{frame-push-promise}}).  A MAX_PUSH_ID frame cannot reduce the maximum Push ID;
receipt of a MAX_PUSH_ID that contains a smaller value than previously received
MUST be treated as a connection error of type HTTP_MALFORMED_FRAME.

A server MUST treat a MAX_PUSH_ID frame payload that does not contain a single
variable-length integer as a connection error of type HTTP_MALFORMED_FRAME.

### Reserved Frame Types {#frame-grease}

Frame types of the format `0xb + (0x1f * N)` are reserved to exercise the
requirement that unknown types be ignored ({{extensions}}). These frames have no
semantic value, and can be sent when application-layer padding is desired. They
MAY also be sent on connections where no request data is currently being
transferred. Endpoints MUST NOT consider these frames to have any meaning upon
receipt.

The payload and length of the frames are selected in any manner the
implementation chooses.


# HTTP Request Lifecycle

## HTTP Message Exchanges {#request-response}

A client sends an HTTP request on a client-initiated bidirectional QUIC
stream. A server sends an HTTP response on the same stream as the request.

An HTTP message (request or response) consists of:

1. the message header (see {{!RFC7230}}, Section 3.2), sent as a single HEADERS
   frame (see {{frame-headers}}),

2. the payload body (see {{!RFC7230}}, Section 3.3), sent as a series of DATA
   frames (see {{frame-data}}),

3. optionally, one HEADERS frame containing the trailer-part, if present (see
   {{!RFC7230}}, Section 4.1.2).

A server MAY interleave one or more PUSH_PROMISE frames (see
{{frame-push-promise}}) with the frames of a response message. These
PUSH_PROMISE frames are not part of the response; see {{server-push}} for more
details.

The "chunked" transfer encoding defined in Section 4.1 of {{!RFC7230}} MUST NOT
be used.

Trailing header fields are carried in an additional header block following the
body. Senders MUST send only one header block in the trailers section;
receivers MUST discard any subsequent header blocks.

A response MAY consist of multiple messages when and only when one or more
informational responses (1xx, see {{!RFC7231}}, Section 6.2) precede a final
response to the same request.  Non-final responses do not contain a payload body
or trailers.

An HTTP request/response exchange fully consumes a bidirectional QUIC stream.
After sending a request, a client closes the stream for sending; after sending a
final response, the server closes the stream for sending and the QUIC stream is
fully closed.  Requests and responses are considered complete when the
corresponding QUIC stream is closed in the appropriate direction.

A server can send a complete response prior to the client sending an entire
request if the response does not depend on any portion of the request that has
not been sent and received. When this is true, a server MAY request that the
client abort transmission of a request without error by triggering a QUIC
STOP_SENDING frame with error code HTTP_EARLY_RESPONSE, sending a complete
response, and cleanly closing its stream. Clients MUST NOT discard complete
responses as a result of having their request terminated abruptly, though
clients can always discard responses at their discretion for other reasons.

Changes to the state of a request stream, including receiving a QUIC
RESET_STREAM with any error code, do not affect the state of the server's
response. Servers do not abort a response in progress solely due to a state
change on the request stream.  However, if the request stream terminates without
containing a usable HTTP request, the server SHOULD abort its response with the
error code HTTP_INCOMPLETE_REQUEST.


### Header Formatting and Compression

HTTP message headers carry information as a series of key-value pairs, called
header fields. For a listing of registered HTTP header fields, see the "Message
Header Field" registry maintained at
<https://www.iana.org/assignments/message-headers>.

Just as in previous versions of HTTP, header field names are strings of ASCII
characters that are compared in a case-insensitive fashion.  Properties of HTTP
header field names and values are discussed in more detail in Section 3.2 of
{{!RFC7230}}, though the wire rendering in HTTP/3 differs.  As in HTTP/2, header
field names MUST be converted to lowercase prior to their encoding.  A request
or response containing uppercase header field names MUST be treated as
malformed.

As in HTTP/2, HTTP/3 uses special pseudo-header fields beginning with the ':'
character (ASCII 0x3a) to convey the target URI, the method of the request, and
the status code for the response.  These pseudo-header fields are defined in
Section 8.1.2.3 and 8.1.2.4 of {{!RFC7540}}. Pseudo-header fields are not HTTP
header fields.  Endpoints MUST NOT generate pseudo-header fields other than
those defined in {{!RFC7540}}.  The restrictions on the use of pseudo-header
fields in Section 8.1.2.1 of {{!RFC7540}} also apply to HTTP/3.

HTTP/3 uses QPACK header compression as described in [QPACK], a variation of
HPACK which allows the flexibility to avoid header-compression-induced
head-of-line blocking.  See that document for additional details.

An HTTP/3 implementation MAY impose a limit on the maximum size of the header it
will accept on an individual HTTP message.  This limit is conveyed as a number
of bytes in the `SETTINGS_MAX_HEADER_LIST_SIZE` parameter. The size of a header
list is calculated based on the uncompressed size of header fields, including
the length of the name and value in bytes plus an overhead of 32 bytes for each
header field.  Encountering a message header larger than this value SHOULD be
treated as a stream error of type `HTTP_EXCESSIVE_LOAD`.

### Request Cancellation

Either client or server can cancel requests by aborting the stream (QUIC
RESET_STREAM and/or STOP_SENDING frames, as appropriate) with an error code of
HTTP_REQUEST_CANCELLED ({{http-error-codes}}).  When the client cancels a
response, it indicates that this response is no longer of interest.
Implementations SHOULD cancel requests by aborting both directions of a stream.

When the server aborts its response stream using HTTP_REQUEST_CANCELLED, it
indicates that no application processing was performed.  The client can treat
requests cancelled by the server as though they had never been sent at all,
thereby allowing them to be retried later on a new connection.  Servers MUST NOT
use the HTTP_REQUEST_CANCELLED status for requests which were partially or fully
processed.

  Note:
  : In this context, "processed" means that some data from the stream was
    passed to some higher layer of software that might have taken some action as
    a result.

If a stream is cancelled after receiving a complete response, the client MAY
ignore the cancellation and use the response.  However, if a stream is cancelled
after receiving a partial response, the response SHOULD NOT be used.
Automatically retrying such requests is not possible, unless this is otherwise
permitted (e.g., idempotent actions like GET, PUT, or DELETE).


## The CONNECT Method

The pseudo-method CONNECT ({{!RFC7231}}, Section 4.3.6) is primarily used with
HTTP proxies to establish a TLS session with an origin server for the purposes
of interacting with "https" resources. In HTTP/1.x, CONNECT is used to convert
an entire HTTP connection into a tunnel to a remote host. In HTTP/2, the CONNECT
method is used to establish a tunnel over a single HTTP/2 stream to a remote
host for similar purposes.

A CONNECT request in HTTP/3 functions in the same manner as in HTTP/2. The
request MUST be formatted as described in {{!RFC7540}}, Section 8.3. A CONNECT
request that does not conform to these restrictions is malformed. The request
stream MUST NOT be closed at the end of the request.

A proxy that supports CONNECT establishes a TCP connection ({{!RFC0793}}) to the
server identified in the ":authority" pseudo-header field. Once this connection
is successfully established, the proxy sends a HEADERS frame containing a 2xx
series status code to the client, as defined in {{!RFC7231}}, Section 4.3.6.

All DATA frames on the stream correspond to data sent or received on the TCP
connection. Any DATA frame sent by the client is transmitted by the proxy to the
TCP server; data received from the TCP server is packaged into DATA frames by
the proxy. Note that the size and number of TCP segments is not guaranteed to
map predictably to the size and number of HTTP DATA or QUIC STREAM frames.

The TCP connection can be closed by either peer. When the client ends the
request stream (that is, the receive stream at the proxy enters the "Data Recvd"
state), the proxy will set the FIN bit on its connection to the TCP server. When
the proxy receives a packet with the FIN bit set, it will terminate the send
stream that it sends to the client. TCP connections which remain half-closed in
a single direction are not invalid, but are often handled poorly by servers, so
clients SHOULD NOT close a stream for sending while they still expect to receive
data from the target of the CONNECT.

A TCP connection error is signaled with QUIC RESET_STREAM frame. A proxy treats
any error in the TCP connection, which includes receiving a TCP segment with the
RST bit set, as a stream error of type HTTP_CONNECT_ERROR
({{http-error-codes}}).  Correspondingly, a proxy MUST send a TCP segment with
the RST bit set if it detects an error with the stream or the QUIC connection.

## Request Prioritization {#priority}

HTTP/3 uses a priority scheme similar to that described in {{!RFC7540}}, Section
5.3. In this priority scheme, a given stream can be designated as dependent upon
another request, which expresses the preference that the latter stream (the
"parent" request) be allocated resources before the former stream (the
"dependent" request). Taken together, the dependencies across all requests in a
connection form a dependency tree. The structure of the dependency tree changes
as PRIORITY frames add, remove, or change the dependency links between requests.

The PRIORITY frame {{frame-priority}} identifies a prioritized element. The
elements which can be prioritized are:

- Requests, identified by the ID of the request stream
- Pushes, identified by the Push ID of the promised resource
  ({{frame-push-promise}})
- Placeholders, identified by a Placeholder ID

An element can depend on another element or on the root of the tree.  A
reference to an element which is no longer in the tree is treated as a reference
to the root of the tree.

### Placeholders

In HTTP/2, certain implementations used closed or unused streams as placeholders
in describing the relative priority of requests.  However, this created
confusion as servers could not reliably identify which elements of the priority
tree could safely be discarded. Clients could potentially reference closed
streams long after the server had discarded state, leading to disparate views of
the prioritization the client had attempted to express.

In HTTP/3, a number of placeholders are explicitly permitted by the server using
the `SETTINGS_NUM_PLACEHOLDERS` setting. Because the server commits to maintain
these IDs in the tree, clients can use them with confidence that the server will
not have discarded the state.

Placeholders are identified by an ID between zero and one less than the number
of placeholders the server has permitted.

### Priority Tree Maintenance

Servers can aggressively prune inactive regions from the priority tree, because
placeholders will be used to "root" any persistent structure of the tree which
the client cares about retaining.  For prioritization purposes, a node in the
tree is considered "inactive" when the corresponding stream has been closed for
at least two round-trip times (using any reasonable estimate available on the
server).  This delay helps mitigate race conditions where the server has pruned
a node the client believed was still active and used as a Stream Dependency.

Specifically, the server MAY at any time:

- Identify and discard branches of the tree containing only inactive nodes
  (i.e. a node with only other inactive nodes as descendants, along with those
  descendants)
- Identify and condense interior regions of the tree containing only inactive
  nodes, allocating weight appropriately

~~~~~~~~~~  drawing
    x                x                 x
    |                |                 |
    P                P                 P
   / \               |                 |
  I   I     ==>      I      ==>        A
     / \             |                 |
    A   I            A                 A
    |                |
    A                A
~~~~~~~~~~
{: #fig-pruning title="Example of Priority Tree Pruning"}

In the example in {{fig-pruning}}, `P` represents a Placeholder, `A` represents
an active node, and `I` represents an inactive node.  In the first step, the
server discards two inactive branches (each a single node).  In the second step,
the server condenses an interior inactive node.  Note that these transformations
will result in no change in the resources allocated to a particular active
stream.

Clients SHOULD assume the server is actively performing such pruning and SHOULD
NOT declare a dependency on a stream it knows to have been closed.

## Server Push

HTTP/3 server push is similar to what is described in HTTP/2 {{!RFC7540}}, but
uses different mechanisms.

Each server push is identified by a unique Push ID. The same Push ID can be used
in one or more PUSH_PROMISE frames (see {{frame-push-promise}}), then included
with the push stream which ultimately fulfills those promises.

Server push is only enabled on a connection when a client sends a MAX_PUSH_ID
frame (see {{frame-max-push-id}}). A server cannot use server push until it
receives a MAX_PUSH_ID frame. A client sends additional MAX_PUSH_ID frames to
control the number of pushes that a server can promise. A server SHOULD use Push
IDs sequentially, starting at 0. A client MUST treat receipt of a push stream
with a Push ID that is greater than the maximum Push ID as a connection error of
type HTTP_PUSH_LIMIT_EXCEEDED.

The header of the request message is carried by a PUSH_PROMISE frame (see
{{frame-push-promise}}) on the request stream which generated the push. This
allows the server push to be associated with a client request. Ordering of a
PUSH_PROMISE in relation to certain parts of the response is important (see
Section 8.2.1 of {{!RFC7540}}).  Promised requests MUST conform to the
requirements in Section 8.2 of {{!RFC7540}}.

When a server later fulfills a promise, the server push response is conveyed on
a push stream (see {{push-streams}}). The push stream identifies the Push ID of
the promise that it fulfills, then contains a response to the promised request
using the same format described for responses in {{request-response}}.

If a promised server push is not needed by the client, the client SHOULD send a
CANCEL_PUSH frame. If the push stream is already open or opens after sending the
CANCEL_PUSH frame, a QUIC STOP_SENDING frame with an appropriate error code can
also be used (e.g., HTTP_PUSH_REFUSED, HTTP_PUSH_ALREADY_IN_CACHE; see
{{errors}}). This asks the server not to transfer additional data and indicates
that it will be discarded upon receipt.

# 연결 폐쇄 (Connection Closure)

연결이 한 번 설립되면, HTTP/3 연결은 연결이 닫힐 때까지 여러 요청과 응답
과정에 사용될 수 있다. 연결 폐쇄는 여러 가지 경우에 발생할 수 있다.

## 휴지 연결 (Idle Connections)

각 QUIC 엔드포인트는 핸드셰이크 과정에서 휴지 타임아웃을 선언한다. 해당
연결이 (상대방이 선언한) 타임아웃보다 더 긴 시간 휴지 상태 (어떤 패킷도
도착하지 않음)이면 상대방은 해당 연결이 닫힌 (closed) 것으로 간주한다. 만약
기존 연결이 서버가 고지한 (advertised) 휴지 타임아웃보다 긴 시간 동안 휴지
상태이면 HTTP/3 구현은 새 요청에 대해 새 연결을 열 필요가 있으며, \["SHOULD"
특히 휴지 타임아웃에 가까워지면 그런 동작을 해야 한다.\]

HTTP 클라이언트는 요청이나 서버 푸시에 대한 응답이 있는 동안에는 연결을
유지하기 위해 QUIC PING 프레임을 사용할 것이 기대되어진다. 만약 클라이언트가
서버로부터 응답을 기대하지 않는다면, 휴지 연결을 타임아웃하는 것이 필요없을
수 있는 연결을 유지하려는 노력을 더 하는 것에 비해서는 낫다. \["MAY"
게이트웨이라면 필요할 것이라 에상되는 연결에 대해, 서버로의 연결 설립에 드는
지연 비용을 야기할 바에는, 연결을 유지하고자 PING을 사용할 수도 있다.\]
\["SHOULD" 서버는 연결을 열린 상태로 유지하고자 PING 프레임을 사용하면 안
된다.\]

## 연결 종료 (Connection Shutdown)

연결이 휴지 상태가 아닐지라도, 각 엔드포인트는 연결 사용을 멈추어 해당 연결이
정상적으로 (gracefully) 종료 것을 결정할 수 있다. (클라이언트의 경우)
클라이언트가 요청 생성을 주도하므로, 클라이언트가 해당 연결에 추가 요청을 더
보내지 않는 식으로 연결 종료를 수행한다.기존 요청에 대한 응답 및 푸시된 응답은
완료될 때까지 지속될 것이다. (반면에) 서버는 클라이언트와 통신하여 연결 종료
기능을 수행한다.

서버는 GOAWAY ({{frame-goaway}}) 프레임을 보내서 연결 종료를 시작한다. GOAWAY
프레임은, (해당 프레임보다) 낮은 스트림 ID를 가진 '클라이언트가 시작한 요청'
(client-initiated requests)이 이미 처리되었거나 처리될 수 있음을 알려준다. 반면
이를 알려준 스트림 ID와 그 이후의 스트림 ID는 수용되지 않는다 (not accepted).
이는 클라이언트와 서버가 연결 종료 전에 어떤 요청을 수용할지에 대해 확인할 수
있게 한다. \["MAY" 이 스트림 ID는 QUIC의 MAX_STREAM_ID 프레임에서 확인된 스트림
한계 (stream limit)보다는 작을 수 있다.\] 또한 \["MAY" 이 스트림 ID는 요청이
처리된 적 없었다면 0일 수 있다. \] \["SHOULD NOT" 서버는 GOAWAY 프레임을
보낸 뒤에 QUIC의 MAX_STREAM_ID 한계를 증가시켜서는 안 된다.\]

GOAWAY 프레임이 보내지면, \["MUST" 서버는 확인된 최신 스트림 ID보다 높은 ID를
가진 스트림으로 보내진 요청을 취소해야만 한다.\] \["MUST NOT" 클라이언트는
GOAWAY 프레임을 받은 뒤 해당 연결에 새 요청을 절대 보내서는 안 된다.\] 다만
이미 전달 과정에 있는 요청이 있을 수는 있다. 새 연결은 새 요청으로 설립될 수
있다.

클라이언트가 GOAWAY 프레임에서 확인된 스트림 ID보다 더 높은 ID를 가진
스트림으로 요청을 보냈다면, 해당 요청은 취소({{request-cancellation}})된 것으로
간주된다. \["SHOULD" 클라이언트는 에러 코드 HTTP_REQUEST_CANCELLED와 함께 해당
스트림 ID로 보낸 모든 스트림을 리셋해야 한다.\] \["MAY" 서버는 확인된 스트림
ID보다 낮은 스트림 ID를 갖는 스트림도, 해당 요청이 처리되지 않았다면 취소할
수도 있다 .\]

GOAWAY 프레임의 스트림 ID보다 낮은 스트림 ID를 통한 요청은 처리될 수도 있다.
해당 요청이 성공적으로 완료되었거나, 개별적으로 리셋되었거나, 또는 연결이
중단되기 전까지는, 처리 여부에 관한 상태를 (클라이언트가) 알 수 없다.

연결이 닫힐 것을 미리 알고 있다면, 설령 곧바로 연결이 닫히더라도 \["SHOULD"
서버는 GOAWAY 프레임을 보내야 한다.\] 이를 통해 원격 상대방이 특정 스트림이
부분적으로 처리가 된 것인지 아닌지를 알 수 있게 된다. 예를 들어, HTTP
클라이언트가 POST 메시지를 보내고 동시에 서버가 서버가 QUIC 연결을 닫는다면,
그런데도 서버가 어떤 스트림이 처리 중인지를 알려주는 GOAWAY 프레임을 보내주지
않는다면, 클라이언트는 서버가 POST 요청을 처리하기 시작했는지를 알 수 없다.

서버가 연결을 닫을 때, 요청을 재시도할 수 없는 클라이언트는  전달 중인 (in
flight) 모든 요청을 잃어버리게 된다. \["MAY" 서버는 다른 스트림 ID를 갖는
여러 개의 GOAWAY 프레임을 전달할 수 있다.\] 하지만 \["MUST NOT" 서버는
마지막 스트림 ID로 보낸 값을 증가시켜서는 절대 안 된다.\] 왜냐면 클라이언트는
다른 연결에서 처리되지 않은 요청을 이미 재전송하고 있을 수 있기 때문이다.
\["SHOULD" 연결을 정상적으로 (gracefully) 종료하고자 하는 서버는,
QUIC의 MAX_STREAM_ID의 현재 값으로 설정된 '마지막 스트림 ID'를 가진 첫 GOAWAY
프레임을 송신해야 한다.\] 또한 \["SHOULD NOT" 서버는 MAX_STREAM_ID를 그 이후에
증가시켜선 안 된다.\] 이는 클라이언트에게 연결 종료가 임박했음을 알리고, 추가
요청을 시작하는 것은 금지되었음을 알린다. 혹시 있을 전달 중인 (in-flight)
요청을 위한 시간 (적어도 1 RTT)을 준 뒤에, \["MAY" 서버는 앞에서 갱신되었던
마지막 스트림 ID를 가진 GOAWAY 프레임을 새로 보낼 수도 있다.\] 이는 연결이
요청을 잃는 일 없이 깔끔하게 종료하도록 보장한다.

모든 승인된 요청이 처리되면, 서버는 해당 연결을 휴지 연결이 되도록 허용할
수 있으며, 또는 \["MAY" 해당 연결의 즉시 폐쇄를 시작할 수도 있다.\] \["SHOULD"
정상적인 종료를 완료한 엔드포인트는 연결을 닫을 때 HTTP_NO_ERROR 코드를
사용해야 한다.\]

## 응용에 의한 즉시 폐쇄 (Immediate Application Closure)

HTTP/3 구현은 아무 때나 QUIC 연결을 즉시 닫을 수 있다. 이는 상대에게 QUIC의
CONNECTION_CLOSE 프레임을 보내게 된다. 상대는 이 프레임의 에러 코드에서
연결이 왜 닫히는지를 알게 된다.연결이 닫힐 때 어떤 에러 코드가 쓰일 수 있는지
{{errors}}를 보라.

\["MAY" 연결을 닫기 전에, 클라이언트가 요청을 재시도할 수 있도록 GOAWAY
프레임이 보내질 수있다.\] QUIC의 CONNECTION_CLOSE 프레임과 GOAWaY 프레임을
같은 패킷에 포함시키면, 해당 프레임이 클라이언트에게 받아질 기회가 늘어난다.

## 전송에 의한 폐쇄 (Transport Closure)

다양한 이유로, QUIC 전송은 응용 계층에게 연결이 중단되었음을 (terminated) 알릴
수 있다. 상대가 명시적으로 닫거나, 전송 계층에서의 오류가 발생했거나, 연결성을
방해하는 네트워크 토폴로지의 변화 등이 이유일 수 있다.

연결이 GOAWAY 프레임 없이 중단되면, \["MUST" 클라이언트는 이미 보내진 요청은,
그게 전체가 보내졌든 부분만 보내졌든, 처리되었을 거라고 반드시 가정해야만
한다.\]

# HTTP/3의 확장 (Extensions to HTTP/3) {#extensions}

HTTP/3은 프로토콜 확장을 허용한다. 이 절에서 설명된 한계점 내에서, 추가
서비스를 제공하거나 프로토콜의 특정 측면을 대체하기 위해 프로토콜 확장을 사용할
수 있다. 단일 HTTP/3 연결의 범위 내에서만 확장이 유효하다.

이 확장은 본 문서에서 정의된 프로토콜의 구성 요소에 적용된다. HTTP 메소드
(method), 상태 코드, 헤더 필드 등을 새로이 정의하는 HTTP의 확장은 기존 옵션에
영향을 끼치지 않는다.

확장은 새 프레임 타입 ({{frames}}), 새 설정 ({{settings-parameters}}), 새 에러
코드 ({{errors}}), 또는 새 단방향 스트림 타입 ({{unidirectional-streams}})을
사용하는 것이 허용된다. 레지스트리 (registries)는 프레임 타입
({{iana-frames}}), 설정 ({{iana-settings}}), 에러 코드 ({{iana-error-codes}}),
스트림 타입 ({{iana-stream-types}})이라는 확장 포인트 (extension point)를
설립한다.

\["MUST" 구현은 모든 확장가능한 프로토콜 요소에서 알 수 없거나 지원되지 않는
값을 반드시 무시해야만 한다.\] \["MUST" 구현은 알 수 없거나 지원되지 않는
타입의 프레임과 단방향 스트림을 반드시 폐기해야만 (discard) 한다.\] 이는 그런
(값이나 타입을 가진) 확장 포인트를 확장이 사전 처리 (arrangement)나 협상
(negotiation) 없이도 안전하게 사용할 수 있음을 의미한다.\]

\["MUST" 기존 프로토콜 요소 (component)의 의미 (semantics)를 바꿀 가능성이 있는
확장은 반드시 사용하기 전에 협상을 해야 한다.\] 예를 들어, HEADERS 프레임의
레이아웃을 바꾸는 확장은 상대방이 이를 받아들인다는 긍정적인 신호를 줄 때까지
사용되어서는 안 된다.\] 또한, 변경된 레이아웃이 적용될 때 조정 (coordinate)이
필요할 수도 있다.

이 문서는 확장의 사용에 관한 협상을 하는 특정한 방법을 명시하진 않는다. 하지만
그럴 목적으로 설정 ({{settings-parameters}})을 사용할 수도 있음을 참고하라.
쌍방 (both peers)이 해당 확장을 쓰겠다는 의지를 나타내는 값을 설정하면, 해당
확장은 사용할 수 있다. 확장 협상을 위해 설정을 사용한다면, \["MUST" 해당 설정이
생략되었을 때 반드시 해당 확장을 비활성화하는 것이 기본값이 되도록 해야만
한다.\]


# 에러 처리 (Error Handling) {#errors}

QUIC은 응용이 에러에 직면했을 때 갑자기 개별 스트림이나 전체 연결을 중단
(리셋)하는 것을 허용한다. 이를 각각 "스트림 오류"와 "연결 오류"라고 하며,
{{QUIC-TRANSPORT}}에서 상세히 설명한다. \["MAY" 엔드포인트는 스트림 오류를 연결
오류처럼 다룰 수도 있다.

이 절은 연결 오류나 스트림 오류의 원인을 설명하기 위해 사용가능한 HTTP/3을 위한
에러 코드를 설명한다.

## HTTP/3 에러 코드 (HTTP/3 Error Codes) {#http-error-codes}

다음 에러 코드는 HTTP/3을 사용할 때 QUIC의 RESET_STREAM 프레임, STOP_SENDING
프레임, CONNECTION_CLOSE 프레임에 사용되도록 정의된 것이다.

HTTP_NO_ERROR (0x00):
: 오류 없음. 연결이나 스트림이 닫힐 필요가 있지만, 오류가 없음을 알릴 때
  사용함.

HTTP_PUSH_REFUSED (0x02):
: 클라이언트가 해당 연결에서 수용하지 않는 컨첸츠를 서버가 푸시하려고 했음.

HTTP_INTERNAL_ERROR (0x03):
: HTTP 스택에서 내부 오류가 발생함.

HTTP_PUSH_ALREADY_IN_CACHE (0x04):
: 서버가 클라이언트가 캐시해둔 컨텐츠를 푸시하려고 했음.

HTTP_REQUEST_CANCELLED (0x05):
: 클라이언트가 요청된 데이터를 더 이상 필요로 하지 않음.

HTTP_INCOMPLETE_REQUEST (0x06):
: 클라이언트의 스트림이 제대로 갖춰진 (fully-formed) 요청 없이 중단됨.

HTTP_CONNECT_ERROR (0x07):
: CONNECT 요청의 응답으로 설립된 연결이 리셋되거나 비정상적으로 닫힘.

HTTP_EXCESSIVE_LOAD (0x08):
: 엔드포인트가 상대방이 과도한 로드를 생성할 수 있는 행위를 하려고 함을 감지함.

HTTP_VERSION_FALLBACK (0x09):
: 요청된 오퍼레이션이 HTTP/3에서 행할 (served) 수 없음. 상대방은 HTTP/1.1을
  통해 재전송해야 함.

HTTP_WRONG_STREAM (0x0A):
: 프레임이 허용되지 않은 스트림을 통해 받아짐.

HTTP_PUSH_LIMIT_EXCEEDED (0x0B):
: 현재 최대 푸시 ID보다 큰 푸시 ID가 참조됨.

HTTP_DUPLICATE_PUSH (0x0C):
: 한 푸시 ID가 서로 다른 두 스트림 헤더에서 참조됨.

HTTP_UNKNOWN_STREAM_TYPE (0x0D):
: 단방향 스트림 헤더가 알 수 없는 스트림 타입을 가짐.

HTTP_WRONG_STREAM_COUNT (0x0E):
: 단방향 스트림 타입이 해당 타입에 허용된 횟수 이상으로 사용됨.

HTTP_CLOSED_CRITICAL_STREAM (0x0F):
: 해당 연결에서 필수적인 스트림이 닫혔거나 리셋됨.

HTTP_WRONG_STREAM_DIRECTION (0x0010):
: 상대방이 허용되지 않는 단방향 스트림 타입을 사용했음.

HTTP_EARLY_RESPONSE (0x0011):
: 응답을 생성하는데 클라이언트의 요청의 나머지 부분이 필요하지 않음.
  STOP_SENDING에서만 사용함.

HTTP_MISSING_SETTINGS (0x0012):
: 제어 스트림이 시작 시점에 SETTING 프레임을 전혀 받지 못함.

HTTP_UNEXPECTED_FRAME (0x0013):
: 현재 상태에서 허용되지 않는 프레임을 받음.

HTTP_GENERAL_PROTOCOL_ERROR (0x00FF):
: 상대방이 특정 에러 코드와 매치되지는 않는 프로토콜 요구사항을 위반했거나,
  (에러가 발생했지만) 엔드포인트가 더 구체적인 에러 코드를 사용하기를 거부함.

HTTP_MALFORMED_FRAME (0x01XX):
: 특정 프레임 타입에서의 오류. 에러 코드의 마지막 바이트로 프레임 타입이
  명기됨. 예를 들어, MAX_PUSH_ID 프레임에서의 에러는 이 같은 코드 (0x10D)로
  알려질 것임.


# 보안 고려 사항 (Security Considerations)

HTTP/3의 보안 고려 사항은 TLS를 같이 쓰는 HTTP/2의 보안 고려사항과 비교되어야
한다. HTTP/2가 연결이 트래픽 분석에 대항하도록 PADDING 프레임 및 다른 프레임의
Padding 필드를 활용할 때, HTTP/3은 QUIC의 PADDING 프레임에 의존하거나
{{frame-grease}}와 {{stream-grease}}에 논의된 예약된 프레임과 스트림 타입을
활용할 수 있다.

HTTP 대체 서비스 (HTTP Alternative Services)가 HTTP/3 엔드포인트를 찾기 위한
용도로 쓰일 때, {{!ALTSVC}}의 보안 고려 사항 또한 고려한다.

몇몇 프로토콜 요소는 중첩된 길이 요소를 포함하는데, 보통 가변 길이 정수로
표현된 (containing) 명시적 길이와 같이 프레임의 형태로 되어 있다. 이는 부주의한
구현가에게 보안 위협을 초래할 수 있다. \["MUST" 구현은 반드시 담고 있는
필드들의 길이와 프레임의 길이가 정확히 맞아떨어지도록  보장해야만 한다.\]


# IANA 고려 사항 (IANA Considerations)

## HTTP3 식별 문자열의 등록 (Registration of HTTP/3 Identification String)

이 문서는 HTTP/3의 식별을 위해 {{?RFC7301}}에서 설립된 "Application Layer
Protocol Negotiation (ALPN) Protocol IDs"의 레지스트리에 신규 등록을 생성한다.

"h3" 문자열은 HTTP/3을 식별한다:

  Protocol:
  : HTTP/3

  Identification Sequence:
  : 0x68 0x33 ("h3")

  Specification:
  : This document

## QUIC 버전 힌트 Alt-Svc 파라미터의 등록

이 문서는 {{!RFC7838}}에서 설립한 "Hypertext Transfer Protocol (HTTP) Alt-Svc
Parameter" 레지스트리에서 버전 협상 힌트의 새로운 등록을 생성한다.

  Parameter:
  : "quic"

  Specification:
  : This document, {{alt-svc-version-hint}}

## 프레임 타입 (Frame Types) {#iana-frames}

이 문서는 HTTP/3 프레임 타입 코드의 레지스트리를 설립한다. "HTTP/3 Frame Type"
레지스트리는 8-bit 공간을 관리한다. "HTTP/3 Frame Type" 레지스트리는 0x00에서
0xef (0xef 포함)한 값을 "IETF Review" 나 "IESG Approval" 정책 {{?RFC8126}} 중
하나에 따라 레지스트리를 운용하며, 0xf0에서 0xff 까지 (0xff 포함)의 값은 실험
사용을 위해 예약되었다.

이 레지스트리는 {{RFC7540}}에 정의된 "HTTP/2 Frame Type" 레지스트리와는
분리되어 있지만, (엔트리) 할당은 서로 유사한 것이 선호된다. 한 레지스트리에만
특정 엔트리가 있다면, \["SHOULD" 대응하는 값을 관련 없는 연산에 할당하는 것을
최대한 피하고자 노력해야만 한다.\]

이 레지스트리에 들어가는 새 엔트리는 다음 정보가 요구된다:

Frame Type:
: 프레임 타입의 이름 또는 레이블.

Code:
: 프레임 타입에 할당된 8 비트 코드

Specification:
: 프레임 레이아웃과 그 의미 (semantics)의 설명을 포함하는 명세에 대한 참조.
  명세는 조건에 따라 등장하는 프레임의 부분도 포함해야 함.

다음 테이블의 엔트리는 본 문서에 등록되어 있다.

| ---------------- | ------ | -------------------------- |
| Frame Type       | Code   | Specification              |
| ---------------- | :----: | -------------------------- |
| DATA             | 0x0    | {{frame-data}}             |
| HEADERS          | 0x1    | {{frame-headers}}          |
| PRIORITY         | 0x2    | {{frame-priority}}         |
| CANCEL_PUSH      | 0x3    | {{frame-cancel-push}}      |
| SETTINGS         | 0x4    | {{frame-settings}}         |
| PUSH_PROMISE     | 0x5    | {{frame-push-promise}}     |
| Reserved         | 0x6    | N/A                        |
| GOAWAY           | 0x7    | {{frame-goaway}}           |
| Reserved         | 0x8    | N/A                        |
| Reserved         | 0x9    | N/A                        |
| MAX_PUSH_ID      | 0xD    | {{frame-max-push-id}}      |
| ---------------- | ------ | -------------------------- |

추가적으로, N의 값이 (0..7)의 범위에 있을 때 `0xb + (0x1f * N)` 꼴의 각 코드
(즉, `0xb`, `0x2a`, `0x49`, `0x68`, `0x87`, `0xa6`, `0xc5`, `0xe4`)에 대해서는
다음 값이 등록되어 있어야 한다:

Frame Type:
: 예약됨 - GREASE

Specification:
: {{frame-grease}}

## 파리미터 설정 (Settings Parameters) {#iana-settings}

본 문서는 HTTP/3 설정에 관한 레지스트리를 설립한다. "HTTP/3 Setting"
레지스트리는 16 비트 공간을 관리한다. "HTTP/3 Setting" 레지스트리는 0x0000에서
0xefff 범위의 값에 대해서는 "Expert Review" 정책 {{?RFC8126}}에 따라 운용되며,
0xf000에서 0xffff 사이의 값에 대해서는 실험 사용으로 예약되어 있다. 지정
전문가는 {{RFC7540}}에 정의된 "HTTP/2 Setting" 레지스트리와 동일하다.

이 레지스트리는 {{RFC7540}}에 정의된 "HTTP/2 Setting" 레지스트리와는 분리되어
있지만, (엔트리) 할당은 서로 유사한 것이 선호된다. 한 레지스트리에만 특정
엔트리가 있다면, \["SHOULD" 대응하는 값을 관련 없는 연산에 할당하는 것을 최대한
피하고자 노력해야만 한다.\]

새 등록은 다음 정보를 제공하는 것이 좋다:

Name:
: 해당 설정에 대한 심볼 이름 설정 이름을 명시하는 것은 선택사항임.

Code:
: 해당 설정에 할당된 16 비트 코드.

Specification:
: 해당 설정의 사용을 설명하는 명세에의 참조 (선택사항).

다음 테이블의 엔트리는 본 문서에 등록되어 있다.

| ---------------------------- | ------ | ------------------------- |
| Setting Name                 | Code   | Specification             |
| ---------------------------- | :----: | ------------------------- |
| Reserved                     | 0x2    | N/A                       |
| NUM_PLACEHOLDERS             | 0x3    | {{settings-parameters}}   |
| Reserved                     | 0x4    | N/A                       |
| Reserved                     | 0x5    | N/A                       |
| MAX_HEADER_LIST_SIZE         | 0x6    | {{settings-parameters}}   |
| ---------------------------- | ------ | ------------------------- |

추가적으로, 각 `?`이 4비트 값이라고 할 때 `0x?a?a` 꼴의 각 코드 (즉, `0x0a0a`,
`0x0a1a`, 부터 `0xfafa`까지)에 대해서는 다음 값이 등록되어 있어야 한다:

Name:
: 예약됨 - GREASE

Specification:
: {{settings-parameters}}

## 에러 코드 (Error Codes) {#iana-error-codes}

본 문서는 HTTP/3 에러 코드에 관한 레지스트리를 설립한다. "HTTP/3 Error Code"
레지스트리는 16 비트 공간을 관리한다. "HTTP/3 Error Code" 레지스트리는 "Expert
Review" 정책 {{?RFC8126}}에 따라 운용된다.

에러 코드의 등록에는 에러 코드의 설명을 포함하도록 요구된다. 전문 리뷰어는 새
에러 코드 등록이 기존 에러 코드와 충돌할 가능성이 있는지 살펴보기 (examine)
바란다.

새 등록은 다음 정보를 제공하는 것이 좋다:

Name:
: 에러 코드 이름. 에러 코드 이름을 명시하는 것은 선택사항임.

Code:
: 16 비트 에러 코드 값.

Description:
: 에러 코드 의미의 간략한 설명, 만약 더 자세한 명세를 제공할 수 없다면.

Specification:
: 에러 코드를 정의하는 명세에의 참조 (선택사항).

다음 테이블의 엔트리는 본 문서에 등록되어 있다.

| ----------------------------------- | ---------- | ---------------------------------------- | ---------------------- |
| Name                                | Code       | Description                              | Specification          |
| ----------------------------------- | ---------- | ---------------------------------------- | ---------------------- |
| HTTP_NO_ERROR                       | 0x0000     | No error                                 | {{http-error-codes}}   |
| HTTP_PUSH_REFUSED                   | 0x0002     | Client refused pushed content            | {{http-error-codes}}   |
| HTTP_INTERNAL_ERROR                 | 0x0003     | Internal error                           | {{http-error-codes}}   |
| HTTP_PUSH_ALREADY_IN_CACHE          | 0x0004     | Pushed content already cached            | {{http-error-codes}}   |
| HTTP_REQUEST_CANCELLED              | 0x0005     | Data no longer needed                    | {{http-error-codes}}   |
| HTTP_INCOMPLETE_REQUEST             | 0x0006     | Stream terminated early                  | {{http-error-codes}}   |
| HTTP_CONNECT_ERROR                  | 0x0007     | TCP reset or error on CONNECT request    | {{http-error-codes}}   |
| HTTP_EXCESSIVE_LOAD                 | 0x0008     | Peer generating excessive load           | {{http-error-codes}}   |
| HTTP_VERSION_FALLBACK               | 0x0009     | Retry over HTTP/1.1                      | {{http-error-codes}}   |
| HTTP_WRONG_STREAM                   | 0x000A     | A frame was sent on the wrong stream     | {{http-error-codes}}   |
| HTTP_PUSH_LIMIT_EXCEEDED            | 0x000B     | Maximum Push ID exceeded                 | {{http-error-codes}}   |
| HTTP_DUPLICATE_PUSH                 | 0x000C     | Push ID was fulfilled multiple times     | {{http-error-codes}}   |
| HTTP_UNKNOWN_STREAM_TYPE            | 0x000D     | Unknown unidirectional stream type       | {{http-error-codes}}   |
| HTTP_WRONG_STREAM_COUNT             | 0x000E     | Too many unidirectional streams          | {{http-error-codes}}   |
| HTTP_CLOSED_CRITICAL_STREAM         | 0x000F     | Critical stream was closed               | {{http-error-codes}}   |
| HTTP_WRONG_STREAM_DIRECTION         | 0x0010     | Unidirectional stream in wrong direction | {{http-error-codes}}   |
| HTTP_EARLY_RESPONSE                 | 0x0011     | Remainder of request not needed          | {{http-error-codes}}   |
| HTTP_MISSING_SETTINGS               | 0x0012     | No SETTINGS frame received               | {{http-error-codes}}   |
| HTTP_UNEXPECTED_FRAME               | 0x0013     | Frame not permitted in the current state | {{http-error-codes}}   |
| HTTP_MALFORMED_FRAME                | 0x01XX     | Error in frame formatting                | {{http-error-codes}}   |
| ----------------------------------- | ---------- | ---------------------------------------- | ---------------------- |

## 스트림 타입 (Stream Types) {#iana-stream-types}

이 문서는 HTTP/3 단방향 스트림 타입 코드의 레지스트리를 설립한다. "HTTP/3
Stream Type" 레지스트리는 8-bit 공간을 관리한다. "HTTP/3 Stream Type"
레지스트리는 0x00에서 0xef (0xef 포함)한 값을 "IETF Review" 나 "IESG
Approval" 정책 {{?RFC8126}} 중 하나에 따라 레지스트리를 운용하며, 0xf0에서
0xff 까지 (0xff 포함)의 값은 실험 사용을 위해 예약되었다.

이 레지스트리에 들어가는 새 엔트리는 다음 정보가 요구된다:

Stream Type:
: 스트림 타입의 이름 또는 레이블.

Code:
: 스트림 타입에 할당된 8 비트 코드.

Specification:
: 스트림 타입에 대한 설명을 포함한 명세에의 참조. 명세는 페이로드의 레이아웃
  의미도 포힘해야 함.

Sender:
: 이 타입의 스트림을 시작할 수 있는 연결 상의 엔드포인트. 값은 "Client",
  "Server", 또는 "Both".

다음 테이블의 엔트리는 본 문서에 등록되어 있다.

| ---------------- | ------ | -------------------------- | ------ |
| Stream Type      | Code   | Specification              | Sender |
| ---------------- | :----: | -------------------------- | ------ |
| Control Stream   | 0x43   | {{control-streams}}        | Both   |
| Push Stream      | 0x50   | {{server-push}}            | Server |
| ---------------- | ------ | -------------------------- | ------ |

추가적으로, N의 값이 (0..8)의 범위에 있을 때 `0xb + (0x1f * N)` 꼴의 각 코드
(즉, `0x00`, `0x1f`, `0x3e`, `0x5d`, `0x7c`, `0x9b`, `0xba`, `0xd9`, `0xf8`)에
대해서는 다음 값이 등록되어 있어야 한다:

Stream Type:
: 예약됨 - GREASE

Specification:
: {{stream-grease}}

Sender:
: Both

--- back

# HTTP/2에서 전환 시 고려 사항 (Considerations for Transitioning from HTTP/2)

HTTP/3은 HTTP/2에서 많은 영향을 받았고, 많은 공통점을 가진다. 이 절에서는
HTTP/3의 설계에 이르게 한 접근법을 설명하고, HTTP/2와의 주요 차이점을 짚고,
HTTP/2 확장을 HTTP/3로 매핑하는 방법을 설명한다.

HTTP/3은 HTTP/2의 유사성은 선호하되, 필수 요구사항은 아니라는 전제에서
시작했다. HTTP/3은 QUIC과 TCP 사이의 동작 차이 (순서 없음, 스트림 지원)를
수용할 필요가 있을 때 주로 HTTP/2와 멀어졌다. 우리는 '두 프로토콜에 동시에
적용가능한 동일 의미 (semantics)를 갖는 확장'을 만들기 어렵거나 불가능하게 하는
불필요한 변경을 피하고자 하였다.

본 절에서는 그 차이를 기록하였다.

## 스트림 (Streams) {#h2-streams}

HTTP/3은 HTTP/2보다 훨씬 많은 수 (2^62-1)의 스트림의 사용을 허락한다. 스트림 ID
공간의 부족 (exhaustion)에 관한 생각이 적용된 것이지만, 이 공간은 QUIC의 다른
제한사항 (limit), 예를 들어 연결 플로우 제어 윈도우의 제한사항 보다 훨씬 크다.

## HTTP 프레임 타입 (HTTP Frame Types) {#h2-frames}

HTTP/2의 여러 프레임 개념들은 QUIC에서는 생략될 수 있는데, QUIC 전송이 이
개념들을 다룰 수 있기 때문이다. 프레임은 이미 한 스트림에 있기 때문에,
(HTTP/3는) 스트림 번호를 생략할 수 있다. (QUIC의 멀티플렉싱은 본 계층 아래에서
일어나기 때문에) 프레임은 멀티플렉싱을 블록할 수 없고, 따라서 가변-최대-길이
(variable-maximum-length) 패킷의 지원은 제거될 수 있다. 스트림 중단은 QUIC이
다루므로, END_STREAM 플래그는 필요하지 않다. 이는 일반적인 프레임 레이아웃에서
Flags 필드의 제거를 허용한다.

프레임 페이로드는 주로 {{!RFC7540}}에서 가져왔다. 하지만 QUIC은 HTTP/2에서도
나타나는 여러 특징 (이를테면 플로우 제어)을 가지고 있다. 이 특징들을 (HTTP/3의)
HTTP 매핑은 다시 구현하지 않는다. 때문에, 몇몇 HTTP/2 프레임 타입은
HTTP/3에서는 필요하지 않다. HTTP/2가 정의한 특정 프레임이 더 이상 사용되지
않더라도, 프레임 ID는 HTTP/2와 HTTP/3 구현 간의 이식성을 최대화하기 위해서
예약되었다. 하지만 두 매핑에서 대등한 (equivalent) 프레임이라도 완전히 같지는
않다.

HTTP/2가 모든 스트림에 걸쳐서 프레임 간 절대 순서를 제공하는 점에서 여러 차이가
발생하는데, QUIC은 각 스트림에 대해서만 프레임 간 절대 순서를 보장하기
떄문이다. 결과적으로 (HTTP2의) 어떤 프레임 타입이 다른 스트림에서 온 프레임도
보내진 순서대로 받아야 한다는 가정을 하고 있다면, HTTP/3는 이를 위반하게 된다.

이를테면, HTTP/2의 우선순위 방안 (prioritization scheme)은 우선순위 변경사항
(즉, 의존성 트리 변형)에 따라 순차전달 (in-order delivery)하는 것을 암묵적으로
생각한다. 특정 서브트리의 부모 바꾸기 (reparenting) 같은 의존성 트리에의 연산은
교환적 (cummutative)이지 않으므로, 송신자 (sender)와 수신자 (receiver)는 양측이
스트림 의존성 트리에 대해 일관된 (consistent) 뷰를 갖도록 연산을 반드시 같은
순서로 적용해만 한다. HTTP/2는 PRIORITY 프레임에 우선순위 할당을 명시하고,
(선택적으로) HEADERS 프레임에서도 우선순위 할당을 명시한다. HTTP/3에서는
우선순위 변경사항에 따라 순차전달하기 위해서 제어 스트림으로 PRIORITY 프레임을
보내고, HEADERS 프레임에서는 PRIORITY 섹션을 제거한다.

마찬가지로, HPACK는 순차전달 (in-order delivery) 가정으로 설계되었다. 인코딩된
헤더 블록 열 (sequence)는 반드시 인코딩 된 순서 그대로 엔드포인트에
도착되어야만 (그리고 디코딩되어야만) 한다. 이는 두 엔드포인트에서의 동적 상태가
동기화된 채로 유지됨을 보장한다. 결과적으로, HTTP/3은 [QPACK]에 설명된 것과
같이 HPACK의 수정된 버전을 사용한다.

HTTP/3의 프레임 타입 정의는 종종 QUIC의 가변 길이 정수 (variable-length
integer) 인코딩을 사용한다. 특히, 스트림 ID는 이 인코딩을 사용하며, 이를 통해
HTTP/2에서 사용된 인코딩보다 더 넓은 범위의 값이 가능하도록 허용한다. HTTP/3의
몇몇 프레임은 스트림 ID보다는 자체적인 식별자를 사용한다. (예를 들어 PRIORITY
프레임의 푸시 ID가 있다.) 해당 인코딩이 (HTTP/2의) 스트림 ID를 포함한다면, 확장
프레임 타입의 인코딩은 재정의될 필요가 있을 수 있다.

Flags 필드는 일반적인 HTTP/3 프레임에서는 등장하지 않으므로, 플래그의 존재에
의존적인 프레임들은 해당 프레임 페이로드의 일부에 플래그를 위한 공간을 할당할
필요가 있다.

이 이슈를 제외하면, 프레임 타입 HTTP/2 확장은 보통 HTTP/2에서의 스트림 0을
HTTP/3의 제어 스트림으로 대체하면 쉽게 QUIC으로 이전할 수 있다. HTTP/3 확장은
순서에 관한 가정을 하지 않고, 따라서 (HTTP/3 확장은) 순서로 인해 문제가 되지
않을 것이므로, 같은 방식으로 HTTP/2로 이전할 수 있을 것이다.

아래는 HTTP/2 프레임 타입이 어떻게 매핑되는지에 관해 나열한 것이다:

DATA (0x0):
: HTTP/3 프레임에서는 패딩을 정의하지 않는다. {{frame-data}}를 보라.

HEADERS (0x1):
: 위에서 설명하였듯, HEADERS의 PRIORITY 영역이 지원되지 않는다. \["MUST" 분리된
  PRIORITY 프레임이 반드시 사용되어야만 한다.\] HTTP/3 프레임에서는 패딩을
  정의하지 않는다. {{frame-headers}}를 보라.

PRIORITY (0x2):
: 위에서 설명하였듯, PRIORITY 프레임이 제어 스트림에 전송되며, 여러 식별자를
  참조할 수 있다. {{frame-priority}}를 보라.

RST_STREAM (0x3):
: QUIC이 스트림 생명주기 관리를 제공하므로 RST_STREAM 프레임은 존재하지 않는다.
  같은 코드 포인트 (역주: 프레임 타입에 부여된 숫자, 여기서는 0x3)가
  CANCEL_PUSH 프레임에 사용된다 ({{frame-cancel-push}}).

SETTINGS (0x4):
: SETTINGS 프레임은 연결 시작에만 보내진다. {{frame-settings}}와
  {{h2-settings}}를 보라.

PUSH_PROMISE (0x5):
: PUSH_PROMISE 프레임은 스트림을 참조하지 않는다. 대신, 푸시 스트림은 푸시
  ID를 사용해 PUSH_PROMISE 프레임을 참조한다. {{frame-push-promise}}를 보라.

PING (0x6):
: PING 프레임은 존재하지 않는다. QUIC이 동등한 기능을 제공하기 때문이다.

GOAWAY (0x7):
: GOAWAY는 서버에서 클라이언트로만 보내지며, 에러 코드를 가지지 않는다.
  {{frame-goaway}}를 보라.

WINDOW_UPDATE (0x8):
: WINDOW_UPDATE 프레임은 존재하지 않는다. QUIC이 플로우 제어를 제공하기
  떄문이다.

CONTINUATION (0x9):
: CONTINUATION 프레임은 존재하지 않는다. 대신, HTTP/2보다 더 큰
  HEADERS/PUSH_PROMISE는 허용된다.

HTTP/2에서 확장으로 정의된 프레임 타입은 HTTP/3에서도 적용가능하다면 분리하여
등록될 필요가 있다. {{!RFC7540}}에 정의된 프레임의 ID는 단순성을 위해 이미
예약되었다. {{iana-frames}}를 보라.

## HTTP/2 SETTINGS 파라미터 (HTTP/2 SETTINGS Parameters) {#h2-settings}

(HTTP/3가) HTTP/2와 다른 중요한 점은, 연결 시작 시점에서 설정이 한 번 보내지면
그 이후에 바꿀 수 없다는 점이다. 이는 변경으로 인한 동기화 문제와 관련된
많은 특이 사례 (corner cases)를 제거한다.

SETTINGS 프레임을 통해 HTTP/2가 명시한 몇몇 전송 수준 옵션은 HTTP/3에서의
QUIC 전송 파라미터에 의해 대체된다. HTTP/3에서 유지된 HTTP 수준 옵션은 HTTP/2와
동일한 값을 갖는다.

아래는 각 HTTP/2 SETTINGS 파라미터가 어떻게 매핑되는지를 나열한 것이다:

SETTINGS_HEADER_TABLE_SIZE:
: [QPACK]를 보라.

SETTINGS_ENABLE_PUSH:
: 서버 푸시를 좀 더 세분화하여 제어할 수 있는 MAX_PUSH_ID의 장점으로 제거됨.

SETTINGS_MAX_CONCURRENT_STREAMS:
: QUIC은 플로우 제어 로직의 일부로 열려있는 스트림 중 가장 큰 값의 스트림 ID를
  제어한다. SETTINGS 프레임에 SETTINGS_MAX_CONCURRENT_STREAMS를 명시하면 오류가
 발생한다.

SETTINGS_INITIAL_WINDOW_SIZE:
: QUIC은 초기 전송 핸드셰이크에서 스트림 플로우 제어 윈도우 크기와 연결 플로우
  제어 윈도우 크기 모두를 명시하도록 요구한다. SETTINGS 프레임에서
  SETTINGS_INITIAL_WINDOW_SIZE를 명시하면 오류가 발생한다.

SETTINGS_MAX_FRAME_SIZE:
: HTTP/3에서는 동등한 설정이 없다. SETTINGS 프레임에서 이를 명시하면 오류가
  발생한다.

SETTINGS_MAX_HEADER_LIST_SIZE:
: {{settings-parameters}}를 보라.

HTTP/2에서는 설정값으로 고정 길이의 32비트 필드가 사용되었지만, HTTP/3에서의
설정값은 가변 길이 정수 (variable-length intergers) (6, 14, 30, 또는 62 비트
길이)이다. 이는 때로 더 짧은 인코딩을 만들지만, 32 비트 공간을 모두 사용하는
설정에서는 더 긴 인코딩을 만든다. HTTP/2에서 이전된 설정은 62 비트 인코딩이
사용되는 걸 피하도록 설정 포맷을 재정의해볼 수 있을 것이다.

설정은 HTTP/2와 HTTP/3를 분리해서 정의할 필요가 있다. {{!RFC7540}}에 정의된
설정에 관한 ID들은 단순성을 위해 이미 예약되었다. {{iana-settings}}를 보라.


## HTTP/2 에러 코드 (HTTP/2 Error Codes)

QUIC는 HTTP/2가 제공하는 "스트림" 에러 및 "연결" 오류와 동일한 개념을 가진다.
하지만, HTTP/2의 에러 코드를 바로 이전할 수 있는 이전성은 없다.

{{!RFC7540}}의 7절에 정의된 HTTP/2의 에러 코드는 HTTP/3의 에러 코드에 다음과
같이 매핑된다:

NO_ERROR (0x0):
: {{http-error-codes}}에서의 HTTP_NO_ERROR.

PROTOCOL_ERROR (0x1):
: 단일 매핑 없음. {{http-error-codes}}에서 정의된 새 HTTP_MALFORMED_FRAME 에러
  코드를 보라.

INTERNAL_ERROR (0x2):
: {{http-error-codes}}에서의 HTTP_INTERNAL_ERROR.

FLOW_CONTROL_ERROR (0x3):
: QUIC이 플로우 제어를 하므로 활용할 수 없음. QUIC 계층에서
  QUIC_FLOW_CONTROL_RECEIVED_TOO_MUCH_DATA를 발생시킬 것이다.

SETTINGS_TIMEOUT (0x4):
: SETTINGS에 대한 응답 (acknowledgement)가 정의되지 않았으므로 활용할 수 없음.

STREAM_CLOSED (0x5):
: QUIC이 스트림 관리를 다루므로 활용할 수 없음. QUIC 계층에서
  QUIC_STREAM_DATA_AFTER_TERMINATION을 발생시킬 것이다.

FRAME_SIZE_ERROR (0x6):
: {{http-error-codes}}에서 정의된 HTTP_MALFORMED_FRAME 에러 코드.

REFUSED_STREAM (0x7):
: QUIC이 스트림 관리를 다루므로 활용할 수 없음. QUIC 계층에서 STREAM_ID_ERROR를
  발생시킬 것이다.

CANCEL (0x8):
: {{http-error-codes}}에서의 HTTP_REQUEST_CANCELLED.

COMPRESSION_ERROR (0x9):
: [QPACK]에 여러 에러 코드가 정의됨.

CONNECT_ERROR (0xa):
: {{http-error-codes}}에서의 HTTP_CONNECT_ERROR.

ENHANCE_YOUR_CALM (0xb):
: {{http-error-codes}}에서의 HTTP_EXCESSIVE_LOAD.

INADEQUATE_SECURITY (0xc):
: QUIC 모든 연결에 대한 충분한 보안을 제공한다고 가정하기 때문에 활용할 수 없음.

HTTP_1_1_REQUIRED (0xd):
: {{http-error-codes}}에서의 HTTP_VERSION_FALLBACK.

HTTP/2와 HTTP/3의 에러 코드는 분리되어 정의될 필요가 있다.
{{iana-error-codes}}를 보라.

# 변경 사항

> **RFC Editor's Note:**  Please remove this section prior to publication of a
> final version of this document.

## Since draft-ietf-quic-http-16

- Rename "HTTP/QUIC" to "HTTP/3" (#1973)

## Since draft-ietf-quic-http-15

Substantial editorial reorganization; no technical changes.

## Since draft-ietf-quic-http-14

- Recommend sensible values for QUIC transport parameters (#1720,#1806)
- Define error for missing SETTINGS frame (#1697,#1808)
- Setting values are variable-length integers (#1556,#1807) and do not have
  separate maximum values (#1820)
- Expanded discussion of connection closure (#1599,#1717,#1712)
- HTTP_VERSION_FALLBACK falls back to HTTP/1.1 (#1677,#1685)

## Since draft-ietf-quic-http-13

- Reserved some frame types for grease (#1333, #1446)
- Unknown unidirectional stream types are tolerated, not errors; some reserved
  for grease (#1490, #1525)
- Require settings to be remembered for 0-RTT, prohibit reductions (#1541,
  #1641)
- Specify behavior for truncated requests (#1596, #1643)

## Since draft-ietf-quic-http-12

- TLS SNI extension isn't mandatory if an alternative method is used (#1459,
  #1462, #1466)
- Removed flags from HTTP/3 frames (#1388, #1398)
- Reserved frame types and settings for use in preserving extensibility (#1333,
  #1446)
- Added general error code (#1391, #1397)
- Unidirectional streams carry a type byte and are extensible (#910,#1359)
- Priority mechanism now uses explicit placeholders to enable persistent
  structure in the tree (#441,#1421,#1422)

## Since draft-ietf-quic-http-11

- Moved QPACK table updates and acknowledgments to dedicated streams (#1121,
  #1122, #1238)

## Since draft-ietf-quic-http-10

- Settings need to be remembered when attempting and accepting 0-RTT (#1157,
  #1207)

## Since draft-ietf-quic-http-09

- Selected QCRAM for header compression (#228, #1117)
- The server_name TLS extension is now mandatory (#296, #495)
- Specified handling of unsupported versions in Alt-Svc (#1093, #1097)

## Since draft-ietf-quic-http-08

- Clarified connection coalescing rules (#940, #1024)

## Since draft-ietf-quic-http-07

- Changes for integer encodings in QUIC (#595,#905)
- Use unidirectional streams as appropriate (#515, #240, #281, #886)
- Improvement to the description of GOAWAY (#604, #898)
- Improve description of server push usage (#947, #950, #957)

## Since draft-ietf-quic-http-06

- Track changes in QUIC error code usage (#485)

## Since draft-ietf-quic-http-05

- Made push ID sequential, add MAX_PUSH_ID, remove SETTINGS_ENABLE_PUSH (#709)
- Guidance about keep-alive and QUIC PINGs (#729)
- Expanded text on GOAWAY and cancellation (#757)

## Since draft-ietf-quic-http-04

- Cite RFC 5234 (#404)
- Return to a single stream per request (#245,#557)
- Use separate frame type and settings registries from HTTP/2 (#81)
- SETTINGS_ENABLE_PUSH instead of SETTINGS_DISABLE_PUSH (#477)
- Restored GOAWAY (#696)
- Identify server push using Push ID rather than a stream ID (#702,#281)
- DATA frames cannot be empty (#700)

## Since draft-ietf-quic-http-03

None.

## Since draft-ietf-quic-http-02

- Track changes in transport draft

## Since draft-ietf-quic-http-01

- SETTINGS changes (#181):
    - SETTINGS can be sent only once at the start of a connection;
      no changes thereafter
    - SETTINGS_ACK removed
    - Settings can only occur in the SETTINGS frame a single time
    - Boolean format updated

- Alt-Svc parameter changed from "v" to "quic"; format updated (#229)
- Closing the connection control stream or any message control stream is a
  fatal error (#176)
- HPACK Sequence counter can wrap (#173)
- 0-RTT guidance added
- Guide to differences from HTTP/2 and porting HTTP/2 extensions added
  (#127,#242)

## Since draft-ietf-quic-http-00

- Changed "HTTP/2-over-QUIC" to "HTTP/QUIC" throughout (#11,#29)
- Changed from using HTTP/2 framing within Stream 3 to new framing format and
  two-stream-per-request model (#71,#72,#73)
- Adopted SETTINGS format from draft-bishop-httpbis-extended-settings-01
- Reworked SETTINGS_ACK to account for indeterminate inter-stream order (#75)
- Described CONNECT pseudo-method (#95)
- Updated ALPN token and Alt-Svc guidance (#13,#87)
- Application-layer-defined error codes (#19,#74)


## Since draft-shade-quic-http2-mapping-00

- Adopted as base for draft-ietf-quic-http
- Updated authors/editors list

# Acknowledgements
{:numbered="false"}

본 명세의 원 저자는 Robbie Shade와 Mike Warres임.

Mike Warres가 공헌한 상당 부분은 그가 고용되었던 Microsoft의 지원을 받음.
