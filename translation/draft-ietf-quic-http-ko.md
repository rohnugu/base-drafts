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

QUIC 전송 프로토콜은 스트림 멀티플렉싱, 스트림당 흐름 제어, 저지연 연결 설립
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


## 표기 관례 (Notational Conventions)

이 문서에서 다음 주요 단어들 "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY" 및
"OPTIONAL"은 여기에 있듯이 모두 대문자로 나타난 때에, 그리고 그 때에만 (when,
and only when) BCP 14 {{!RFC2119}} {{!RFC8174}}에서 설명하듯이 해석되어야 한다.

(역주) 한국어엔 이를 적절히 번역할 방법이 없으므로, 해당 문장 또는 절을 \[\]로
묶어 \["MUST" 문장\]과 같이 표기하도록 한다.

필드 정의는 {{!RFC5234}}에 정의되었듯 ABNF (Augmented Backus-Naur Form)로
주어진다.

본 문서는 {{QUIC-TRANSPORT}}에서 가져온 가변 길이 인코딩 (variable-length
integer encoding)을 사용한다.

"프레임"이라고 불리는 프로토콜 요소 본 문서와 {{QUIC-TRANSPORT}} 모두에 있다.
(본 문서에서) {{QUIC-TRANSPORT}}에서의 프레임이 참조될 때에는 프레임 이름의
앞에 "QUIC"이 붙을 (prefaced with) 것이다. 예를 들어, "QUIC CONNECTION_CLOSE
프레임"이라고 쓴다. 이 접두어가 붙지 않은 참조는 {{frames}}에 정의된 프레임을
가리킨다.


# 연결 설정 및 관리 (Connection Setup and Management)

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

## 연결 설립 (Connection Establishment) {#connection-establishment}

HTTP/3는 QUIC을 기반 전송으로써 의존한다. \["MUST" 사용되는 QUIC 버전은 반드시
핸드셰이크 프로토콜로 TLS 버전 1.3 이상을 사용해야만 한다.\] \["MUST" HTTP/3
클라이언트는 TLS 핸드셰이크 동안 타겟 도메인 명을 나타내야 한다.\] 이는 TLS 또는
다른 어떤 메커니즘에 SNI (Server Name Indication) {{!RFC6066}} 확장을 적용해서
이루어질 수 있다.

QUIC 연결은 {{QUIC-TRANSPORT}}에 설명된 대로 설립된다. 연결 설립 과정에서,
HTTP/3 지원 여부를 TLS 핸드셰이크에 ALPN 토큰 "hq"를 선택해 알린다. \["MAY" 다른
응용 계층 프로토콜의 지원 여부는 동일 핸드셰이크에서 제공될 수도 있다.\]

핵심 QUIC 프로토콜에 관한 연결 수준 옵션은 초기 암호화 핸드셰이크에 설정되지만,
HTTP/3에 특화된 설정은 SETTINGS 프레임으로 전달된다. QUIC 연결이 설립된 후에,
\["MUST" 각 엔드포인트는 반드시 각 HTTP 제어 스트림 ({{control-streams}}를
보라)의 첫 프레임으로 SETTING 프레임 ({{frame-settings}})을 보내야만 한다.\]

## 연결 재사용 (Connection Reuse)

연결이 서버 엔드포인트에 존재하면, \["MAY" 해당 연결은 서로 다른 URI 담당
컴포넌트 (authority components)를 갖는 요청에도 재사용할 수도 있다.\] (역주:
URI autority component는 URI를 구성하는 컴포넌트 중 사용자명, 호스트, 포트
컴포넌트를 합친 컴포넌트를 말한다. TCP/IP 스타일로 설명하면, 5 튜플로 정해진
TCP 연결로 서로 다른 가상 서버에 접속할 수 있도록 허용한다는 의미이다.)
\["MAY" 클라이언트는 해당 서버가 담당 (authoritative)할 것이라 여겨지는 임의의
요청을 해당 서버에 보낼 수도 있다.\]

담당 (authoritative) HTTP/3 엔드포인트는 보통 클라이언트가 원 서버 (origin)에
요청을 보냈을 때, 클라이언트가 원 서버로부터 '해당 엔드포인트를 유효한 (valid)
HTTP 대체 서비스 (HTTP Alternative Service)로 지명함'을 알리는 Alt-Svc 레코드를
받아서 알려진다. {{RFC7838}}에서 요구한 것처럼, 클라이언트는 지명된 서버를
담당이라고 간주하기 전에, \["MUST" 지명된 서버가 원 서버에 대한 유효한 인증서를
보일 수 있는지 반드시 체크해야만 한다.\] \["MUST NOT" 클라이언트는 HTTP/3
엔드포인트를 명시적인 신호가 없는데도 다른 원 서버들에 대한 담당이라고
가정해서는 절대로 안 된다.\]

클라이언트가 특정 원 서버에 대한 연결을 재사용하는 상황을 원하지 않는 서버는
해당 요청에 대한 응답으로 421 (Misdirected Request; 잘못 보내진 요청) 상태
코드를 보내서 담당이 아님을 알릴 수 있다. ({{!RFC7540}}의 9.1.2절을 보라).

{{?RFC7540}}의 9.1절에서 논의한 고려 사항은 HTTP/3 연결의 관리에도 적용될 수
있다.

# 스트림 매핑 및 용례 (Stream Mapping and Usage) {#stream-mapping}

QUIC 스트림은 신뢰적이고, 정렬된 바이트 전송 (reliable in-order delivery of
bytes)을 제공하지만, 다른 스트림에 속한 바이트의 전송 순서는 전혀 보장하지
않는다. 선로 (wire)에는 데이터가 QUIC의 STREAM 프레임으로 나뉘지만 (framed),
이 프레이밍 (framing)은 HTTP의 프레이밍 계층에는 보이지 않는다. 전송 계층은
QUIC의 STREAM 프레임을 받아서 버퍼링하고 정렬하며, 이를 통해 응용에는 신뢰적인
바이트 스트림 안에 포함된 데이터를 노출한다.

QUIC 스트림은 시작자에서 수신자로만 데이터를 보내는 단방향 스트림, 또는 양방향
스트림 중 하나일 수 있다. 스트림은 서버나 클라이언트 중 하나에 의해 시작될 수
있다. QUIC 스트림에 대한 자세한 상세는 {{QUIC-TRANSPORT}}의 2절을 보라.

HTTP 헤더와 데이터가 QUIC을 통해 보내질 때, QUIC 계층은 스트림 관리의 대부분을
다루게 된다. HTTP는 QUIC을 사용할 때 따로 멀티플렉싱을 할 필요가 없다. QUIC
스트림을 통해 보내진 데이터는 언제나 특정 HTTP 트랜젝션 또는 연결 컨텍스트
(context)에 매핑되어 있다.

## 양방향 스트림 (Bidirectional Streams)

클라이언트가 시작한 양방향 스트림은 모두 HTTP 요청과 응답에 쓰인다. 양방향
스트림은 응답을 해당 요청과 자연스럽고 (readily) 명확하게 (ensure) 연관시킨다.
이는 클라이언트의 첫 요청이 QUIC의 스트림 0에서 일어나고, 후속 요청은 스트림 4,
스트림 8, 등등으로 이어짐을 의미한다. (역주: QUIC은 스트림 0에서 암호학적
핸드셰이크를 하도록 되어 있다. 하지만 핸드셰이크 이후에 데이터를 보낼 수 있는
걸로 보인다 - 확인이 더 필요하다. 스트림 식별자의 마지막 두 비트는 시작자와
방향성을 결정하며, 클라이언트가 시작한 양방향 스트림 식별자의 비트는 00이다.)
해당 스트림이 열리기 위해서, \["SHOULD" HTTP/3 클라이언트는 QUIC의 전송
파라미터 `initial_max_stream_data_bidi_local`를 0이 아닌 값으로 보내야 한다.\]
\["SHOULD" HTTP/3 서버는 QUIC의 전송 파라미터
`initial_max_stream_data_bidi_remote`와 `initial_max_bidi_streams`를 0이 아닌
값으로 보내야 한다.\] 병렬성을 불필요하게 제약하지 않도록,
`initial_max_bidi_streams`는 100보다 작지 않도록 하는 것을 추천한다.

해당 스트림은 요청과 응답에 대한 프레임을 싣는다. ({{request-response}}를
보라.) 스트림이 깔끔하게 중단될 때 해당 스트림의 마지막 프레임이 절단되었다면
(truncated), \["MUST" 이 상황은 반드시 연결 오류로 처리되어야만 한다.
{{http-error-codes}}의 HTTP_MALFORMED_FRAME를 보라.\] 불시에 (abruptly) 중단한
스트림은 프레임의 아무 위치에서 리셋될 수도 있다.

HTTP/3은 서버가 시작한 양방향 스트림을 사용하지 않는다. \["MUST" 클라이언트는
반드시 QUIC 전송 파라미터 `initial_max_bidi_streams`의 값을 생략하거나 또는
0으로 설정해야만 한다.\]


## 단방향 스트림 (Unidirectional Streams)

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
({{push-streams}}).  Other stream types can be defined by extensions to HTTP/3;
see {{extensions}} for more details.

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

### 제어 스트림 (Control Streams)

제어 스트림은 스트림 타입이 `0x43` (ASCII 'C')인지 확인하면 알 수 있다. 이
스트림의 데이터는 {{frames}}에 정의되었듯이 HTTP/3 프레임으로 구성되어 있다.

\["MUST" 각 엔드포인트 (side)는 반드시 연결 시작 시점에 단일 제어 스트림을
시작해야만 한다.\] 또한 \["MUST" 각 엔드포인트는 반드시 해당 스트림의 첫
프레임으로 SETTINGS 프레임을 보내야만 한다.\] 만약 제어 스트림의 첫 프레임이
다른 프레임 타입이라면, \["MUST" 반드시 HTTP_MISSING_SETTINGS 타입의 연결
에러로 처리되어야만 한다.\] 두 번째로 전송 스트림이라 주장하는 스트림이 있다면,
\["MUST" 이 스트림은 반드시 HTTP_WRONG_STREAM_COUNT 타입의 연결 오류로
처리되어야만 한다.\] 제어 스트림이 어느 위치에서 (at any point) 닫히면,
\["MUST" 이 상황은 반드시 HTTP_CLOSED_CRITICAL_STREAM 타입의 연결 오류로
처리되어야만 한다.\]

(제어 스트림으로는) 단일 양방향 스트림보다는 단방향 스트림 페어가 사용된다.
이를 통해 상대방이 데이터를 보낼 수 있자마자 보내는 것이 가능해진다. 해당
연결에 0-RTT가 사용가능한지에 따라, 클라이언트 또는 서버 중 하나는 암호학적
핸드셰이크가 먼저 완료된 뒤에 스트림 데이터를 보낼 수 있을 것이다.

### 푸시 스트림 (Push Streams)

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

### 예약된 스트림 타입 (Reserved Stream Types) {#stream-grease}

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
can be implied by the use of SETTINGS -- each peer uses SETTINGS to advertise a
set of supported values. The definition of the setting would describe how each
peer combines the two sets to conclude which choice will be used.  SETTINGS does
not provide a mechanism to identify when the choice takes effect.

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

  SETTINGS_MAX_HEADER_LIST_SIZE (0x6):
  : The default value is unlimited.  See {{header-formatting}} for usage.

  SETTINGS_NUM_PLACEHOLDERS (0x8):
  : The default value is 0.  However, this value SHOULD be set to a non-zero
    value by servers.  See {{placeholders}} for usage.

Setting identifiers of the format `0x?a?a` are reserved to exercise the
requirement that unknown identifiers be ignored.  Such settings have no defined
meaning. Endpoints SHOULD include at least one such setting in their SETTINGS
frame. Endpoints MUST NOT consider such settings to have any meaning upon
receipt.

Because the setting has no defined meaning, the value of the setting can be any
value the implementation selects.

Additional settings MAY be defined by extensions to HTTP/3; see {{extensions}}
for more details.

#### 초기화 (Initialization)

\["MUST NOT" HTTP 구현은 상대방의 설정에 대해 현재 알고 있는 바에 따르면
유효하지 않은 것으로 보이는 프레임이나 요청을 절대로 보내선 안 된다.\] 모든
설정은 초기값으로 시작해야 하며 SETTINGS 프레임의 수신에 따라 갱신된다. 서버의
경우, 각 클라이언트 설정에 대한 초기값은 기본값 (default value)이다.

1-RTT QUIC 연결을 사용하는 클라이언트의 경우, 각 서버 설정의 초기값은
기본값이다. 0-RTT QUIC 연결이 사용되는 때에는, 각 서버 설정의 초기값은 직전
세션에 사용되었던 값이다. \["MUST" 클라이언트는 재개하려는 세션에서 (과거에)
서버가 제공했던 설정을 반드시 저장해야만 한다.\] 또한, \["MUST" 클라이언트는
현재 서버 설정을 수신할 때까지 반드시 저장해둔 설정을 준수해야만 한다.\]

서버는 스스로 알린 설정을 기억하고 있을 수도 있으며, 또는 티켓 안에 해당 값들이
무결성이 보호된 채 복사되도록 저장한 뒤,0-RTT 데이터를 수락할 때 복구할 수도
있다. 서버는 0-RTT 데이터의 수락 여부를 결정하고자 해당 HTTP/3 설정값을
사용한다. (역주: QUIC의 전송 파라미터에서도 유사한 문구가 있다.)

\["MAY" 서버는 0-RTT (데이터)를 수용할 수 있다.\] \["MAY" 그 후에 서버의
SETTINGS 프레임으로 다른 설정을 제공할 수도 있다.\] 서버가 0-RTT 데이터를
수락하면, \["MUST NOT" 서버의 SETTING 프레임은 (0-RTT를 통해 알 수 있는)
제한사항을 줄이거나 해당 0-RTT 데이터를 사용하는 클라이언트에 의해 위배될 수
있는 값을 대체하는 행위를 절대로 해서는 안 된다.\]

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

### 예약된 프레임 타입 (Reserved Frame Types) {#frame-grease}

Frame types of the format `0xb + (0x1f * N)` are reserved to exercise the
requirement that unknown types be ignored ({{extensions}}). These frames have no
semantic value, and can be sent when application-layer padding is desired. They
MAY also be sent on connections where no request data is currently being
transferred. Endpoints MUST NOT consider these frames to have any meaning upon
receipt.

The payload and length of the frames are selected in any manner the
implementation chooses.


# HTTP 요청의 생애주기 (HTTP Request Lifecycle)

## HTTP 메시지 교환 (HTTP Message Exchanges) {#request-response}

클라이언트는 HTTP 요청을 클라이언트가 시작한 특정 양방향 QUIC 스트림에 보낸다.
서버는 해당 요청에 대한 HTTP 응답을 같은 스트림에 보낸다.

HTTP (요청 또는 응답) 메시지는 다음으로 구성된다:

1. 단일 HEADERS 프레임 ({{frame-headers}}를 보라)으로 보내지는 메시지 헤더
   (message header) ({{!RFC7230}}의 3.2절을 보라),

2. 일련의 DATA 프레임들 ({{frame-data}}를 보라)로 보내지는 페이로드 본체
   (payload body) ({{!RFC7230}}의 3.3절을 보라),

3. 선택사항으로, 만약 있다면 트레일러 부분 ({{!RFC7230}}의 4.1.2절을 보라)을
   담고 있는 단일 HEADERS 프레임.

\["MAY" 서버는 응답 메세지의 프레임들에 하나 이상의 PUSH_PROMISE 프레임
({{frame-push-promise}})을 끼워넣을 수도 있다.\] 이 PUSH_PROMISE 프레임은
응답의 일부가 아니다. 자세한 사항은 {{server-push}}를 보라.

\["MUST NOT" {{!RFC7230}}의 4.1 절에 정의된 "청크 단위의" 전송 인코딩
("chunked" transfer encoding)은 절대로 사용되어서는 안 된다. \]

트레일러 헤더 (trailing header)의 필드는 본체 뒤에 추가적인 헤더 블록에 실린다.
(역주: <https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Trailer>
참고) \["MUST" 송신자 (sender)는 트레일러 섹션에 반드시 단 하나의 헤더 블록만을
보내야만 한다.\] \["MUST" 수신자 (receiver)는 그 뒤의 헤더 블록은 반드시
폐기해야만 한다.\]

\["MAY" 응답은 하나 이상의 정보성 응답 (informational response) (1xx,
{{!RFC7231}}의 6.2절을 보라)이 동일 요청에 대한 최종 응답에 선행될 때, 그리고
그 때에만 여러 개의 메시지로 구성될 수도 있다.\] 최종 응답이 아닌 응답은
페이로드 본체 또는 트레일러를 가지지 않는다.

HTTP 요청/응답 교환은 양방향 QUIC 스트림을 완전히 소비한다. 요청이 보내진 뒤에,
클라이언트는 보내기용 스트림을 닫는다. 서버는 최종 응답을 송신한 뒤에 보내기용
스트림을 닫고, 그러면 QUIC 스트림은 완전히 닫힌다. 요청과 응답은 대응하는 QUIC
스트림이 적절한 방향에서 닫힐 때 완료되었다고 간주된다.

만약 (클라이언트가) 아직 보내지 않아 받지 못한 요청의 부분이 응답과 무관하면,
서버는 클라이언트가 전체 요청을 다 보내기 전에 완전한 응답을 보낼 수 있다.
이런 경우에, \["MAY" 서버는 클라이언트에게 요청의 전송을 중단토록 요청할 수도
있다.\] 서버의 이 요청은 에러 코드 HTTP_EARLY_RESPONSE가 담긴 QUIC의
STOP_SENDING 프레임을 야기해서, 완전한 응답을 송신한 뒤에, 스트림을 깔끔하게
닫음으로써 행한다. 클라이언트는 (곧 말할 이유를 제외하고) 다른 이유로는 재량껏
응답을 폐기할 수 있지만, \["MUST NOT" 클라이언트가 보낸 요청을 중단시키는
결과를 낳은 완전한 응답을 절대로 폐기해서는 안 된다. \]

에러 코드를 포함하고 있는 QUIC의 RESET_STREAM을 (서버가) 받는 상태 변화를
포함해, 요청 스트림의 상태의 변화는 서버의 응답 상태에 영향을 주지 않는다.
서버는 요청 스트림의 상태 변화만을 이유로 진행 중은 응답을 중단하지 않는다.
하지만, 요청 스트림이 사용가능한 HTTP 요청을 포함하지 않고 요청 스트림을
중단하면, \["SHOULD" 서버는 에러 코드 HTTP_INCOMPLETE_REQUEST와 함께 응답을
중단하여야 한다.\]


### 헤더 포맷 및 압축 (Header Formatting and Compression) {#header-formatting}

HTTP 메시지 헤더는 헤더 필드 (header field)라고 불리는 일련의 키-값 쌍
(key-value pair) 형태로 정보를 운반한다. 등록된 HTTP 헤더 필드의 나열은
<https://www.iana.org/assignments/message-headers>에서 관리하는 "Message
Header Field" 레지스트리를 보라.

기존 HTTP 버전과 마찬가지로, 헤더 필드명은 ASCII 문자로 구성된 문자열이며,
비교시에는 대소문자를 가리지 않는다. HTTP 헤더 필드명과 해당 필드값의 성질은
{{!RFC7230}}의 3.2절에서 더욱 자세히 논의된다. 다만 HTTP/3의 데이터 전달 방식
(wire rendering)은 해당 문서와는 다르다. (역주: wire rendering은 Wire Protocol
의 렌더링을 가리키는 것임) HTTP/2에서처럼, \["MUST" 헤더 필드명은 반드시 인코딩
전에 소문자로 변환되어야만 한다.\] \["MUST" 대문자가 들어간 헤더 필드명을
포함한 요청 또는 응답은 잘못된 형식 (malformed)으로 처리되어야만 한다.\]

HTTP/2와 마찬가지로, HTTP/3는 타겟 URI, 요청 방법, 응답의 상태 코드를 담기 위해
':' 문자 (ASCII 0x3a)로 시작하는 특수 가상헤더 (pseudo-header) 필드를
사용한다. 이 가상헤더 필드는 {{!RFC7540}}의 8.1.2.3절과 8.1.2.4절에 정의되어
있다. \["MUST NOT" 엔드포인트는 {{!RFC7540}}에 정의된 것과 다른 가상헤더 필드를
절대 생성해서는 안 된다.\] {{!RFC7540}}의 8.1.2.1 절에서 가상헤더 필드의 사용
시 제약사항은 HTTP/3에도 적용된다.

HTTP/3은 [QPACK]에 설명된 QPACK 헤더 압축을 사용한다. QPACK 헤더 압축은 HPACK의
변형으로, 헤더 압축으로 야기되는 head-of-line 차단 (head-of-line blocking)을
피하기 위한 유연성이 있다. 자세한 내용은 해당 문서를 보라.

\["MAY" HTTP/3 구현은 헤더의 최대 크기를 개별 HTTP 메시지의 수용 여부를 결정할
제한사항으로 둘 수도 있다.\] \["SHOULD" 이 값보다 큰 메시지 헤더를 직면하면
`HTTP_EXCESSIVE_LOAD` 타입의 스트림 오류로 처리된다.\] 구현이 상대방에게 이
제한사항을 알리고 싶으면, 그 제한사항은 `SETTINGS_MAX_HEADER_LIST_SIZE`
파라미터에 바이트 수로 담길 수 있다. 헤더 리스트의 크기는 압축되지 않은 헤더
필드의 크기에 기반해 계산되며, 이 크기에는 바이트 단위의 필드명의 길이, 바이트
단위의 필드값의 길이, 그리고 각 헤더 필드 당 32 바이트의 오버헤드가 더해진다.

### 요청 취소 (Request Cancellation)

클라이언트 혹은 서버는 오류 코드 HTTP_REQUEST_CANCELLED
({{http-error-codes}})와 함께 스트림을 중단함으로써 (적절하게는, QUIC의
RESET_STREAM 그리고/또는 STOP_SENDING 프레임으로) 요청을 취소시킬 수 있다.
클라이언트가 응답을 취소시킨다는 것은 해당 응답이 더 이상 관심없음을 나타낸다.
\["SHOULD" 구현은 스트림 양방향을 중단시켜서 요청을 취소시킬 수 있어야 한다.\]

서버가 HTTP_REQUEST_CANCELLED를 사용해 응답 스트림을 중단할 때는 응용이 전혀
처리하지 않았음을 나타낸다. 클라이언트는 서버에 의해 취소된 요청을 요청 자체가
보내지지 않았던 것처럼, 그래서 새 연결에서 추후 재전송될 수 있도록 처리할 수
있다. \["MUST NOT" 서버는 일부 또는 전부가 처리된 요청에 대해 절대로
HTTP_REQUEST_CANCELLED 상태를 사용해서는 안 된다.\]

  Note:
  : 여기서 "처리"는 해당 스트림의 어떤 데이터가 소프트웨어의 특정 상위 계층에
    전달되어 최종적으로 어떤 행동을 야기했을 수 있는 상황을 가리킨다.

스트림이 완전한 응답을 받은 뒤 취소되었다면, \["MAY" 클라이언트는 그런 취소는
무시하고, 해당 응답을 사용할 수도 있다.\] 하지만, 스트림이 부분적인 응답만
받은 후에 취소되었다면, \["SHOULD NOT" 그런 응답은 사용되어선 안 된다.\]
(GET, PUT, DELETE와 같이 멱등적인 (idempotent) 행동이라서) 재시도가 허용된
경우가 아니라면, 그런 요청을 자동적으로 재시도할 수 없다. (역주: 멱등적인
행동은 동일 행동이 반복해서 실행되어도 한 번만 실행한 것과 완전히 동일한
효과를 갖는 행동을 말한다.)


## CONNECT 메소드 (The CONNECT Method)

가상-메소드 (pseudo-method)인 CONNECT ({{!RFC7231}}의 4.3.6절)는 HTTP 프록시가
원 서버 (origin server)와 "https" 리소스를 교환할 (interacting) 목적으로 TLS
세션을 설립할 때 주로 쓰인다. HTTP/1.x에서는 CONNECT가 전체 HTTP 연결을 원격
호스트로의 터널로 변환하고자 사용되었다. HTTP/2에서는 비슷한 목적으로 원격
호스트에 단일 HTTP/2 스트림 위의 터널을 설립할 때 CONNECT 메소드가 사용된다.

HTTP/3의 CONNECT 요청은 HTTP/2에서 동일한 방식으로 기능한다. \["MUST" CONNECT
요청은 {{!RFC7540}}의 8.3절에 설명된 포맷을 반드시 사용해야만 한다.\] 이
제한사항을 준수하지 않는 CONNECT 요청은 잘못된 형식 (malformed)을 가진 것이다.
\["MUST NOT" 그러한 요청 스트림은 요청 끝에 절대로 닫아져서는 안 된다.\]
(역주: 해당 스트림을 통해서 HEADERS, DATA 프레임을 교환해야 하므로.)

CONNECT를 지원하는 프록시는 가상 헤더 필드 ":authority"에서 특정된 서버로 TCP
연결 ({{!RFC0793}})을 설립한다. 연결이 성공적으로 설립되었다면, 프록시는
{{!RFC7231}}의 4.3.6절에 정의되었듯이 클라이언트에게 2xx 대의 상태 코드를 담은
HEADERS 프레임을 보낸다.

해당 스트림의 모든 DATA 프레임은 해당 TCP 연결에서 보내고 받은 데이터에
대응한다. 프록시는 클라이언트가 보내온 각 DATA 프레임을 TCP 서버로 송신한다.
프록시는 TCP 서버로부터 받은 데이터를 DATA 프레임으로 패키징한다. TCP
세그먼트의 크기와 수가 HTTP DATA와 QUIC STREAM 프레임의 크기와 수에 정확히
매핑되는 것이 보장되지 않음에 주의하라.

TCP 연결은 각 상대방에 의해 닫힐 수 있다. 클라이언트가 요청 스트림을 마칠 때,
(즉, 프록시의 수신 스트림이 "Data Recvd" 상태에 들어갈 때), 프록시는 TCP
서버로의 연결에 FIN 비트를 설정할 것이다. 프록시가 FIN 비트를 설정한 패킷을
받을 때, 클라이언트로 보내는 (프록시의) 송신 스트림은 중단될 것이다. 단일
방향으로 절반 닫기 (half-closed) 상태인 TCP 연결은 유효하지 않은 (invalid) 것은
아니지만, 종종 서버가 제대로 처리하지 못한다. 따라서 \["SHOULD NOT"
클라이언트는 CONNECT의 타겟 (서버)으로부터 받을 데이터가 아직 있을 것 같으면
송신 스트림을 닫지 않아야 한다.\]

TCP 연결 오류는 QUIC의 RESET_STREAM 프레임으로 알려진다. 프록시는 TCP 연결의
어떤 오류도 처리하며, 이는 RSB 비트가 설정된 TCP 세그먼트를 받는 상황도
포함하는데 이는 HTTP_CONNECT_ERROR ({{http-error-codes}}) 타입의 스트림 오류로
처리된다. 마찬가지로, 프록시는 스트림이나 QUIC 연결에 오류를 감지하면 \["MUST"
RSB 비트가 설정된 TCP 세그먼트를 (원 서버로) 반드시 보내야만 한다.\]

## 요청의 우선순위 결정법 (Request Prioritization) {#priority}

HTTP/3은 {{!RFC7540}}의 5.3절에 설명된 것과 비슷한 우선순위 결정법을 사용한다.
이 우선순위 결정법에서, 특정 스트림은 다른 요청에 의존적이다라고 지정될 수
있으며, 이는 다른 요청의 스트림 ("부모" 요청)이 특정 스트림 ("의존" 요청)보다
먼저 자원이 할당되는 것을 선호함을 나타낸다. (이러한 선호도를) 합쳐서 보면,
특정 연결의 모든 요청 간의 의존성은 의존성 트리를 형성한다. 의존성 트리의
구조는 PRIORITY 프레임이 요청 간의 의존성 링크를 추가하거나, 제거하거나, 또는
변경할 때 바뀐다.

PRIORITY 프레임 {{frame-priority}}는 우선시되는 요소를 특정한다. 우선시가능한
요소는 다음과 같다:

- 요청 스트림의 ID로 특정되는 요청
- 약속된 자원의 푸시 ID로 특정되는 푸시 ({{frame-push-promise}})
- 플레이스홀더 ID로 특정되는 플레이스홀더 (Placeholder)

특정 요소는 트리의 루트나 다른 요소에 의존할 수 있다. 더 이상 트리에 없는
요소에의 참조는 트리의 루트로의 참조로 처리된다.

### 플레이스홀더 (Placeholders)

HTTP/2의 특정 구현에서 닫힌 또는 사용하지 않는 (unused) 스트림을 '요청의
상대적인 우선순위를 나타내고자 플레이스홀더 (placeholder)로 사용했다. 하지만,
이는 우선순위 트리의 어떤 요소를 안전하게 폐기할 수 있는지를 확실히 판단할 수
없게 만들어서 혼란을 야기했다. 서버가 (어떤 요소를) 폐기 상태로 둔 뒤 오래
뒤에야 클라이언트가 잠재적으로 닫힌 스트림을 참조할 수 있었다. 이는
클라이언트가 표현하고자 한 우선순위와는 전혀 다른 (disparate) 뷰를 야기했다.
(역주: <https://chadaustin.me/2014/10/http2-request-priorities-a-summary/>와
<https://chadaustin.me/2014/11/update-on-http2-priority/>를 보면 좀 더 잘
이해할 수 있다. 여기서 플레이스홀더는 쓸모없는데 자리를 차지하는 스트림이다.
하지만 서버의 구현 측면에서 플레이스홀더 간의 우선순위의 고저를 플레이스홀더
간의 의존성으로 구분하고, 일정 시간이 지난 뒤 불필요한 플레이스홀더를 제거하는
방법을 써볼 수 있을 것이다. 한편, 특정 스트림이 플레이스홀더인지를 알 수 없는
클라이언트로서는 명시된 가중치만 가지고 의존성 트리를 만들어야 한다. 이런
상황에서 이는 플레이스홀더로 지정한 스트림을 바로 지우지 않는 것은 클라이언트와
서버의 의존성 트리가 서로 달라지는 결과를 낳는 것이다.)

HTTP/3에서는 서버가 `SETTINGS_NUM_PLACEHOLDERS` 설정을 사용하면, 여러
플레이스홀더를 명시적으로 허용하게 된다. 서버가 트리 내에 유지할 플레이스홀더 ID를
밝히므로, 클라이언트는 서버가 해당 상태를 폐기하지 않을 것이라는 확신을 가지고
플레이스홀더를 사용할 수 있다.

플레이스홀더는 0과 서버가 허용한 플레이스 홀더 수보다 1 작은 수 사이의 범위를
가지는 ID로 특정된다.

### 우선순위 트리 유지하기 (Priority Tree Maintenance)

서버는 우선순위 트리에서 비활성 영역 (inactive region)을 적극적으로 잘라낼
(prune) 수 있다. 플레이스홀더는 클라이언트가 유지하려고 (retaining) 노력하는
트리가 일정한 (persistent) 구조로 "뿌리내리도록" 하기 위해서 사용될 것이다.
우선순위 결정을 위해서, 트리의 노드는, 노드에 대응하는 스트림이 2 RTT
(round-trip times) 동안 닫혀있었을 때, "비활성 노드"라고 간주된다. 이 지연은
어떤 노드가 클라이언트(의 뷰)에서는 아직 스트림 의존성에 사용되는 활성 노드로
간주됨에도 서버에 의해서 잘릴 경우 발생할 수 있는 경쟁 조건 (race condition)을
경감하는데 도움이 된다.

구체적으로, \["MAY" 서버는 다음(의 행동)을 아무 때에나 할 수 있다:\]

- 비활성 노드만으로 구성된 트리 가지를 식별하고 폐기 (즉, 후손과 그 후손들이
  모두 비활성 노드로만 구성된 노드)
- 비활성 노드로만 구성된 트리 내부 영역을 특정하고 적절히 가중치를 할당하면서
  압축

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

{{fig-pruning}}의 예제에서, `P`는 플레이스홀더이고, `A`는 활성 노드이며 `I`는
비활성 노드이다. 1단계로, 서버는 두 비활성 브랜치 (각각 단일 노드)를 폐기한다.
2단계로, 서버는 (트리의) 내부 비활성 노드를 압축한다. 이 변형은 특정 활성
스트림에 할당된 자원에 변화를 주지 않을 것이다.

\["SHOULD" 클라이언트는 서버가 적극적으로 그런 잘라내기를 할 것임을 가정해야
한다.\] 또한 \["SHOULD NOT" 클라이언트는 닫힌 것이 확실한 스트림에 의존성을
선언 (declare)해서는 안 된다.\]

## 서버 푸시 (Server Push)

HTTP/3 서버 푸시는 HTTP/2 {{!RFC7540}}에서 설명한 것과 비슷하지만 다른
메커니즘을 사용한다.

각 서버 푸시는 유일한 푸시 ID로 식별된다. 동일한 푸시 ID가 하나 이상의
PUSH_PROMISE 프레임에서 사용될 수 있다 ({{frame-push-promise}}를 보라). 그 뒤
해당 프레임은 해당 약속을 최종적으로 충족시킬 푸시 스트림에 포함된다.

서버 푸시는 클라이언트가 해당 연결에 MAX_PUSH_ID 프레임을 보낸 때에만
사용가능하다. ({{frame-max-push-id}}) 서버는 MAX_PUSH_ID 프레임을 받을 때까지
서버 푸시를 쓸 수 없다. 클라이언트는 서버가 약속할 수 있는 푸시 개수를
제어하고자 추가로 MAX_PUSH_ID 프레임을 보낸다. \["SHOULD" 서버는 푸시 ID를
0에서 시작해서 순차적으로 사용해야만 한다.\] \["MUST" 클라이언트는 최대 푸시
ID보다 큰 푸시 ID를 가진 푸시 스트림을 받았을 때 반드시
HTTP_PUSH_LIMIT_EXCEEDED 타입의 연결 오류로 처리해야만 한다.

요청 메시지의 헤더는 PUSH_PROMISE 프레임 ({{frame-push-promise}})에 의해
이송되며, 푸시를 생성한 요청 스트림을 통해 PUSH_PROMISE 프레임이 전송된다.
이는 서버 푸시가 특정 클라이언트 요청과 결합되도록 한다. 응답의 특정 부분과
PUSH_PROMISE의 순서 관계는 중요하다 ({{!RFC7540}}의 8.2.1절을 보라). \["MUST"
약속된 요청은 반드시 {{!RFC7540}}의 8.2 절의 요구사항을 지켜야만 한다.\]

서버가 추후 약속을 충족할 때, 서버 푸시 응답이 푸시 스트림으로 이송된다.
({{push-streams}}를 보라.) 해당 푸시 스트림은 서버가 충족할 약속의 푸시 ID를
특정하고, 그 뒤 {{request-response}}에서의 응답과 같은 포맷을 사용해 '약속된
요청에 대한 응답'을 포함한다.

클라이언트가 약속된 서버 푸시를 필요로 하지 않으면, \["SHOULD" 클라이언트는
CANCEL_PUSH 프레임을 보내야 한다.\] 푸시 스트림이 이미 열려있거나 CANCEL_PUSH
프레임을 보낸 후에 열렸다면, 적절한 에러 코드를 가진 QUIC의 STOP_SENDING
프레임이 사용될 수 있다. (예를 들어 HTTP_PUSH_REPUSED,
HTTP_PUSH_ALREADY_IN_CACHE 등이 있다. {{errors}}를 보라.) 이는 서버가 추가
데이터를 보내지 않도록 요청하고, 받는 즉시 폐기할 것임을 알린다.

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

HTTP/3 구현은 아무 때나 QUIC 연결을 즉시 닫을 수 있다. 이는 상대방에게 QUIC의
CONNECTION_CLOSE 프레임을 보내게 된다. 상대방은 이 프레임의 에러 코드에서
연결이 왜 닫히는지를 알게 된다.연결이 닫힐 때 어떤 에러 코드가 쓰일 수 있는지
{{errors}}를 보라.

\["MAY" 연결을 닫기 전에, 클라이언트가 요청을 재시도할 수 있도록 GOAWAY
프레임이 보내질 수있다.\] QUIC의 CONNECTION_CLOSE 프레임과 GOAWAY 프레임을
같은 패킷에 포함시키면, 해당 프레임이 클라이언트에게 받아질 기회가 늘어난다.

## 전송에 의한 폐쇄 (Transport Closure)

다양한 이유로, QUIC 전송은 응용 계층에게 연결이 중단되었음을 (terminated) 알릴
수 있다. 상대방이 명시적으로 닫거나, 전송 계층에서의 오류가 발생했거나,
연결성을 방해하는 네트워크 토폴로지의 변화 등이 이유일 수 있다.

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


# 오류 처리 (Error Handling) {#errors}

QUIC은 응용이 오류에 직면했을 때 갑자기 개별 스트림이나 전체 연결을 중단
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
  (오류가 발생했지만) 엔드포인트가 더 구체적인 에러 코드를 사용하기를 거부함.

HTTP_MALFORMED_FRAME (0x01XX):
: 특정 프레임 타입에서의 오류. 에러 코드의 마지막 바이트로 프레임 타입이
  명기됨. 예를 들어, MAX_PUSH_ID 프레임에서의 에러는 이 같은 코드 (0x10D)로
  알려질 것임.


# 보안 고려 사항 (Security Considerations)

HTTP/3의 보안 고려 사항은 TLS를 같이 쓰는 HTTP/2의 보안 고려 사항과 비교되어야
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
| Reserved                     | 0x3    | N/A                       |
| Reserved                     | 0x4    | N/A                       |
| Reserved                     | 0x5    | N/A                       |
| MAX_HEADER_LIST_SIZE         | 0x6    | {{settings-parameters}}   |
| NUM_PLACEHOLDERS             | 0x8    | {{settings-parameters}}   |
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
제한사항 (limit), 예를 들어 연결 흐름 제어 윈도우의 제한사항 보다 훨씬 크다.

## HTTP 프레임 타입 (HTTP Frame Types) {#h2-frames}

HTTP/2의 여러 프레임 개념들은 QUIC에서는 생략될 수 있는데, QUIC 전송이 이
개념들을 다룰 수 있기 때문이다. 프레임은 이미 한 스트림에 있기 때문에,
(HTTP/3는) 스트림 번호를 생략할 수 있다. (QUIC의 멀티플렉싱은 본 계층 아래에서
일어나기 때문에) 프레임은 멀티플렉싱을 블록할 수 없고, 따라서 가변-최대-길이
(variable-maximum-length) 패킷의 지원은 제거될 수 있다. 스트림 중단은 QUIC이
다루므로, END_STREAM 플래그는 필요하지 않다. 이는 일반적인 프레임 레이아웃에서
Flags 필드의 제거를 허용한다.

프레임 페이로드는 주로 {{!RFC7540}}에서 가져왔다. 하지만 QUIC은 HTTP/2에서도
나타나는 여러 특징 (이를테면 흐름 제어)을 가지고 있다. 이 특징들을 (HTTP/3의)
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
: WINDOW_UPDATE 프레임은 존재하지 않는다. QUIC이 흐름 제어를 제공하기
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
: QUIC은 흐름 제어 로직의 일부로 열려있는 스트림 중 가장 큰 값의 스트림 ID를
  제어한다. SETTINGS 프레임에 SETTINGS_MAX_CONCURRENT_STREAMS를 명시하면 오류가
 발생한다.

SETTINGS_INITIAL_WINDOW_SIZE:
: QUIC은 초기 전송 핸드셰이크에서 스트림 흐름 제어 윈도우 크기와 연결 흐름
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

QUIC는 HTTP/2가 제공하는 "스트림" 오류 및 "연결" 오류와 동일한 개념을 가진다.
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
: QUIC이 흐름 제어를 하므로 활용할 수 없음. QUIC 계층에서
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
