---
title: "QUIC: UDP 기반의 다중화된, 보안 전송"
abbrev: QUIC 전송 프로토콜
docname: draft-ietf-quic-transport-latest
date: {DATE}
category: std
ipr: trust200902
area: Transport
workgroup: QUIC

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping]

author:
  -
    ins: J. Iyengar
    name: Jana Iyengar
    org: Fastly
    email: jri.ietf@gmail.com
    role: editor
  -
    ins: M. Thomson
    name: Martin Thomson
    org: Mozilla
    email: martin.thomson@gmail.com
    role: editor

normative:

  QUIC-RECOVERY:
    title: "QUIC 손실 탐지 및 혼잡 제어"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-recovery-latest
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Fastly
        role: editor
      -
        ins: I. Swett
        name: Ian Swett
        org: Google
        role: editor

  QUIC-TLS:
    title: "QUIC의 보안을 위한 전송 계층 보안 (TLS) 사용"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-tls-latest
    author:
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla
        role: editor
      -
        ins: S. Turner
        name: Sean Turner
        org: sn3rd
        role: editor

informative:

  QUIC-INVARIANTS:
    title: "QUIC의 버전 독립적 성질들"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-invariants-latest
    author:
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla

  EARLY-DESIGN:
    title: "QUIC: UDP 위에서 동작하는 다중화 프로토콜"
    author:
      - ins: J. Roskind
    date: 2013-12-02
    target: "https://goo.gl/dMVtFi"

  SLOWLORIS:
    title: "Welcome to Slowloris..."
    author:
      - ins: R. RSnake Hansen
    date: 2009-06
    target:
     "https://web.archive.org/web/20150315054838/http://ha.ckers.org/slowloris/"


--- abstract

이 문서는 QUIC 전송 프로토콜의 핵심을 정의한다. 이 문서는 연결 설립, 패킷 포맷,
다중화 및 신뢰성을 설명한다. 동반된 문서는 암호학적 핸드셰이크와 손실 탐지를
설명한다.


--- note_Note_to_Readers

이 드래프트에 관한 토론은 QUIC 워킹 그룹 메일링 리스트 (quic@ietf.org)에서
진행되며 \<https://mailarchive.ietf.org/arch/search/?email_list=quic\>에
보관되어 있다.

워킹 그룹에 관한 정보는 \<https://github.com/quicwg\>에서 찾을 수 있다.; 이
드래프트의 소스 코드 및 이슈 리스트는
\<https://github.com/quicwg/base-drafts/labels/-transport\>에서 찾을 수 있다.

본 한국어 문서는 노희준 (hjroh@korea.ac.kr)이 네트워크 프로토콜 연구를
위해 초벌 번역한 것이다. 본 Internet-Draft는 Simplified BSD License를 따르며,
번역자는 본 번역물에 대해 해당 라이센스의 허용 범위 내에서 2차 저작물로서의
모든 권리를 가진다. 단, 번역자는 번역 내용을 참고함으로써 발생할 수 있는 어떠한
문제에 대해서도 책임을 지지 않는다.

--- middle

# 도입

QUIC은 UDP 위에서 동작하는 다중화된 보안 전송 프로토콜의 일종이다. QUIC은 여러
어플리케이션에 대한 일반-목적의 보안 전송을 가능케하는 유연한 특성들의 집합을
제공하는 걸 목적으로 한다.

* 버전 협상

* 저지연 연결 설립

* 인증되고 암호화된 헤더와 페이로드

* 스트림 다중화

* 스트림 및 연결 수준 흐름 제어

* 연결 이전 및 NAT 재바인딩에의 탄력 (resilience)

QUIC은 TCP, SCTP, 및 다른 전송 프로토콜을 경험하며 배운 기법을 구현한다. QUIC은
배포할 수 있도록 기존 클라이언트 운영 체제 및 미들박스에 수정을 요구하지 않고자
UDP를 토대(substrate)로 사용한다. QUIC은 헤더 전체를 인증하며, 시그널링을 포함,
교환하는 데이터 대부분을 암호화한다. 이는 프로토콜이 미들박스의 업그레이드(를
요구하는) 의존성 없이 발전할 수 있게 한다. 이 문서는 핵심 QUIC 프로토콜을
설명하며, 컨셉 디자인 (conceptual design), 선로 포맷 (wire format), 연결 설립을
위한 QUIC 프로토콜의 메커니즘, 스트림 다중화, 스트림과 연결 수준의 흐름 제어,
연결 이전, 데이터 신뢰성을 포함한다.

동반된 문서들은 QUIC의 손실 탐지와 혼잡 제어{{QUIC-RECOVERY}}를 설명하며, 키
협상을 위해 TLS 1.3을 사용하는 것{{QUIC-TLS}}을 설명한다.

QUIC version 1은 {{QUIC-INVARIANTS}}에 명기된 프로토콜 불변성 (protocol
invariants) 를 따른다 (conform).


# 관례와 정의

이 문서에서 다음 주요 단어들 "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY" 및
"OPTIONAL"은 여기에 있듯이 모두 대문자로 나타난 때에, 그리고 그 때에만 (when,
and only when) BCP 14 {{!RFC2119}} {{!RFC8174}}에서 설명하듯이 해석되어야 한다.

(역주) 한국어엔 이를 적절히 번역할 방법이 없으므로, 해당 문장 또는 절을 \[\]로
묶어 \["MUST" 문장\]과 같이 표기하도록 한다.


이 문서에서 사용되는 용어의 정의:

클라이언트 (Client):

: QUIC 연결을 시작하는 엔드포인트.

서버 (Server):

: 들어오는 QUIC 연결들을 수락하는 엔드포인트.

엔드포인트 (Endpoint):

: 어떤 연결에서 클라이언트 또는 서버.

스트림 (Stream):

: 어떤 QUIC 연결 내에서 정렬된 바이트로 이루어진 논리적인, 양방향 채널.

연결 (Connection):

: 여러 스트림을 다중화할 수 있는 단일 암호화 컨텍스트를 가진 두 QUIC 엔드포인트
  사이에서의 대화.

연결 ID (Connection ID):

: 엔드포인트에서 특정 QUIC 연결을 구분하기 위한 불투명한 (opaque) 식별자. 각
  엔드포인트는 패킷에 상대 (peer) 엔드포인트가 포함시킨 값으로 설정한다.

QUIC 패킷 (QUIC packet):

: QUIC 수신측에 의해 파싱가능한, 형식을 맞춘 (well-formed) UDP 페이로드.

QUIC은 이름일 뿐이며, 약어가 아니다.


## 표기 관례

패킷과 프레임 다이어그램은 다음의 추가 관례를 더하여 {{?RFC2360}}의 3.1절에
설명된 포캣을 사용한다:

\[x\]
: 는 x가 선택적(optional)임을 나타낸다.
(역주) 선택적(optional)이란, x가 있을 수도 있고, 없을 수도 있다는 의미이다.

x (A)
: 는 x의 길이가 A 비트임을 나타낸다.

x (A/B/C) ...
: 는 x의 길이가 비트 단위로 A, B, C 중 하나임을 나타낸다.

x (i) ...
: 는 x가 {{integer-encoding}}에 명시된 가변-길이 인코딩을 사용함을 나타낸다.

x (*) ...
: 는 x가 가변 길이임을 나타낸다.


# 버전 {#versions}

QUIC 버전은 32비트의 unsigned 수로 구분된다.

버전 0x00000000은 버전 협상을 나타내기 위해 점유 (reserved)되었다. 본 명세의
버전은 수 0x00000001로 구분된다.

다른 QUIC 버전은 이 버전과는 다른 성질을 가질 수 있다. 프로토콜의 모든 버전에
걸쳐 항상적으로 (consistent) 보장되어야 하는 QUIC의 성질은
{{QUIC-INVARIANTS}}에 설명되어 있다.

0x00000001 버전의 QUIC은 {{QUIC-TLS}}에 설명되었듯 TLS를 암호학적 핸드셰이크
프로토콜로 사용한다.

상위 16비트가 0으로 설정된 (cleared) 버전은 향후 IETF 의견일치 (consensus)
문서에서 활용하도록 점유되었다.

패턴 0x?a?a?a?a을 만족하는 버전들은 버전 협상을 강제적으로 행사하는데 (to be
exercised) 사용하기 위해서 점유되었다. 즉, 모든 옥텟의 하위 4비트가 (이진수로)
1010인 어떠한 버전이라도 이에 해당된다. \["MAY" 클라이언트나 서버는 점유된 버전
중 어느 하나라도 지원한다는 것을 알릴 (advertise) 수 있다.\]

점유된 버전은 아마 실제 프로토콜을 나타내서는 안 될 (will probably never)
것이다; \["MAY" 클라이언트는 서버가 버전 협상을 시작하리라고 기대하며 이 버전들
중 하나를 사용할 터이다\]; \["MAY" 서버는 이 값들 중 하나를 지원한다고 알릴 수
있는데\], 클라이언트가 그 값을 무시할 것이라 기대할 것이다.

\[\[RFC editor: 출판 전에 본 절의 이어지는 부분을 삭제하기 바람.]]

이 명세 (0x00000001)의 최종 버전의 버전 번호는 RFC로 출판될 프로토콜의 버전
번호로 점유되어 있다.

IETF 드래프트를 구분하기 위해 사용된 버전 번호는 0xff000000에 드래프트 번호를
더해서 만든다. 예를 들어, draft-ietf-quic-transfort-13은 0xff00000D로 구분될
것이다.

구현가들은 개별적인 실험을 위해 사용하고 있는 QUIC 버전 번호를 GitHub 위키
\<https://github.com/quicwg/base-drafts/wiki/QUIC-Versions\>에 등록하는 것을
권장한다.


# 패킷 타입과 포맷

먼저, 메커니즘 설명에서 참조될 QUIC의 패킷 타입과 그 포맷을 설명한다.

모든 수치 (numeric value)는 네트워크 바이트 순서 (즉, big-endian)으로
인코딩하고 모든 필드 크기는 비트 단위로 표기한다. 필드의 각 비트에 대해 설명할
때에는 최하위 비트를 비트 0으로 둔다. 16진수 표기법이 필드 값을 설명하고자
사용된다.

(역주) 모든 필드는 가독성을 위해 번역하지 않는다. 필요에 따라 구문의 가독성을
높이기 위해 ''를 사용한다.

모든 QUIC 패킷은 긴 헤더와 짧은 헤더 중 하나를 가지며, 이는 Header Form 비트를
보고 알 수 있다. '버전 협상과 1-RTT 키의 설립' 전의 연결 초기에 긴 헤더가
사용될 것이다. 짧은 헤더는 최소한의 버전 별 헤더를 갖는데, 이는 버전 협상과
1-RTT 키 설립 후에 사용된다.

## 긴 헤더 {#long-header}

~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
|1|   Type (7)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Version (32)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|DCIL(4)|SCIL(4)|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Destination Connection ID (0/32..144)         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Source Connection ID (0/32..144)            ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Length (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Packet Number (8/16/32)                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Payload (*)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~
{: #fig-long-header title="Long Header Format"}

긴 헤더는 버전 협상과 1-RTT 키의 설립이 완료되기 전에 보내지는 패킷에 사용된다.
두 조건 (역주: 협상과 설립)이 만족되면, 송신측은 짧은 헤더({{short-header}})
패킷을 송신하기로 전이 (switch)할 수 있다. 이 긴 헤더 형식은 uniform한(??)
고정 길이 패킷 포맷으로 나타내지는 특별한 패킷 - 이를테면 버전 협상 패킷 - 을
허용한다. 긴 헤더는 다음 필드를 담고 있다.

Header Form:

: 옥텟 0(첫 옥텟)의 최상위 비트 (0x80)는 긴 헤더에서 1로 설정된다.

Long Packet Type:

: 옥텟 0의 나머지 일곱 비트는 패킷 타입을 담는다. 이 필드는 128 개의 패킷 타입
  중 하나임을 알린다. 본 (명세의) 버전에 명시된 타입은 {{long-packet-types}}에
  나열되어 있다.

Version:

: QUIC Version 필드는 Type 필드 바로 뒤에 있는 32 비트 필드이다. 이 필드는
  사용 중인 QUIC의 버전을 나타내고 이어지는 프로토콜 필드를 어떻게 해석할 지
  결정한다.

DCIL and SCIL:

: 옥텟 1은 이어지는 두 연결 ID 필드의 각 길이를 담고 있다. 두 길이는 4 비트의
  unsigned 정수로 인코딩되어 있다. Destination Connection ID Length (DCIL)
  필드는 옥텟 1의 상위 4비트를 점유하고 Source Connection ID Length (SCIL)
  필드는 옥텟 1의 하위 4비트를 점유한다. 인코딩된 길이 값이 0이라는 것은 연결
  ID가 길이로 0 옥텟임을 나타낸다. 0이 아닌 인코딩된 길이에 3을 더하면 연결
  ID의 총 길이를 구할 수 있고, 이는 경계값을 포함해 4 옥텟에서 18옥텟 사이의
  길이가 된다. 예를 들어 0x50이라는 필드값은, 8 옥텟의 Destination
  Connection ID와 0 옥텟의 Source Connection ID가 이어질 것임을 나타낸다.

Destination Connection ID:

: Destination Connection ID 필드는 DCIL and SCIL 바로 뒤에 이어지며, 그 길이는
  0 옥텟이거나 4에서 18 옥텟 사이이다. {{connection-id}}에서 이 필드의 사용에
  관해 더욱 자세히 설명한다.

Source Connection ID:

: Source Connection ID 필드는 Destination Connection ID 필드 바로 뒤에
  이어지며, 그 길이는  0 옥텟이거나 4에서 18 옥텟 사이이다.
  {{connection-id}}에서 이 필드의 사용에 관해 더욱 자세히 설명한다.

Length:

: 패킷의 나머지 부분 (즉, Packet Number 필드와 Payload 필드)의 옥텟 단위 길이를
  가변 길이 정수로 인코딩한 것이다 ({{integer-encoding}}).

Packet Number:

: 패킷 번호 (packet number) 필드의 길이는 옥텟 단위로 1, 2, 4 중 하나이다. 패킷
  번호는 기밀성 보호를 받는데, 이는 {{QUIC-TLS}}의 5.6절에 설명된 것처럼 패킷
  보호와는 분리되어 있다. 패킷 번호 필드의 길이는 plaintext packet number(??)로
  인코딩되어 있다. 상세는 {{packet-numbers}}를 보라.

Payload:

: 패킷의 페이로드이다.

다음 패킷 타입이 정의되어 있다:

| Type | Name (이름)                   | Section (절)                |
|:-----|:------------------------------|:----------------------------|
| 0x7F | Initial (초기화)              | {{packet-initial}}          |
| 0x7E | Retry   (재시도)              | {{packet-retry}}            |
| 0x7D | Handshake (핸드셰이크)        | {{packet-handshake}}        |
| 0x7C | 0-RTT Protected (0-RTT 보호됨)| {{packet-protected}}        |
{: #long-packet-types title="Long Header Packet Types"}

긴 헤더 패킷의 Header Form, Type, Connection ID Lengths 옥텟, Destination ID,
Source Connection ID, Version 필드는 버전 독립적이다. Packet Number 필드와
{{long-packet-types}}에 정의된 패킷 타입 값은 버전 특화적이다. 다른 버전의
QUIC에서 패킷을 어떻게 해석할지에 대한 상세는 {{QUIC-INVARIANTS}}를 보라.

각 필드와 페이로드의 해석은 버전과 타입에 따라 개별적 (specific)이다. 본 버전의
타입 별 시맨틱(semantics)은 이어지는 절에서 설명한다.

패킷의 끝은 Length 필드에 의해 결정된다. Length 필드는 Packet Number와 Payload
필드 모두를 포함하는데, 각 필드는 기밀성이 보호되고 길이를 알 수가 없다.
Payload 필드의 크기는 패킷 번호 보호가 제거된 뒤에야 알게 된다.

송신측은 때때로 여러 패킷을 하나의 UDP 데이터그램으로 합칠 수 있다. 상세는
{{packet-coalesce}}를 보라.


## 짧은 헤더 {#short-header}

~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
|0|K|1|1|0|R R R|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Destination Connection ID (0..144)           ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Packet Number (8/16/32)                ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Protected Payload (*)                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~
{: #fig-short-header title="Short Header Format"}

짧은 헤더는 버전과 1-RTT 키가 협상된 이후에 사용할 수 있다. 이 헤더 형식은
다음 필드를 가진다:

Header Form:

: 옥텟 0의 최상위 비트 (0x80) 가 짧은 헤더에서는 0으로 설정된다.

Key Phase Bit:

: 옥텟 0의 두 번째 비트 (0x40)는 키 (교환) 단계 (key phase) (??)를 나타내며
  패킷의 수신자가 패킷 보호를 위해 사용할 패킷 보호 키를 특정할 수 있게 한다.
  자세한 사항은 {{QUIC-TLS}}를 보라.

\[\[Editor's Note: 이 절은 삭제되어야 하며 이 드래프트가 IESG로 전달되기 전에
비트 정의가 변경되었다.]]

Third Bit:

: 옥텟 0의 세 번째 비트는 (0x20) 1로 설정된다.

\[\[Editor's Note: 이 절은 삭제되어야 하며 이 드래프트가 IESG로 전달되기 전에
비트 정의가 변경되었다.]]

Fourth Bit:

: 옥텟 0의 네 번째 비트는 (0x10) 1로 설정된다.

\[\[Editor's Note: 이 절은 삭제되어야 하며 이 드래프트가 IESG로 전달되기 전에
비트 정의가 변경되었다.]]

Google QUIC Demultipexing Bit:

: 옥텟 0의 다섯 번째 비트 (0x8)는 0으로 설정한다. 이는 Google QUIC 구현이
  Google QUIC 패킷과 클라이언트가 보낸 짧은 헤더 패킷을 구분하게 해준다. 왜냐면
  Google QUIC 서버는 연결 ID가 반드시 있을 걸로 기대하기 때문이다.
  \["SHOULD" 이 비트에 대한 특별한 해석은 Google QUIC이 새로운 헤더 포맷으로
  전환을 완료하면 삭제되어야 한다.\]

Reserved:

: 옥텟 0의 6, 7, 8번째 비트 (0x7)는 실험을 위해 점유되었다.

Destination Connection ID:

: Destination Connection ID란 의도된 패킷 수신자에 의해 선택된 연결 ID이다.
  상세는 {{connection-id}}를 보라.

Packet Number:

: 패킷 번호 (packet number) 필드의 길이는 옥텟 단위로 1, 2, 4 중 하나이다. 패킷
  번호는 기밀성 보호를 받는데, 이는 {{QUIC-TLS}}의 5.6절에 설명된 것처럼 패킷
  보호와는 분리되어 있다. 패킷 번호 필드의 길이는 plaintext packet number(??)로
  인코딩되어 있다. 상세는 {{packet-numbers}}를 보라.

Protected Payload:

: 짧은 헤더 패킷은 1-RTT (키로) 보호된 페이로드를 포함한다.

짧은 헤더 패킷의 Header Form과 Connection ID 필드는 버전 독립적이다. 나머지
필드는 선택된 QUIC 버전에 특화적이다. 다른 버전의 QUIC에서 패킷을 어떻게
해석할지에 대한 상세는 {{QUIC-INVARIANTS}}를 보라.


## 버전 협상 패킷 {#packet-version}

버전 협상 패킷은 본질적으로 버전 특화적일 수 없고, 따라서 긴 헤더를 쓰지
않는다. ({{long-header}}를 보라.) 클라이언트가 받을 때엔 긴 헤더로 보이지만
Version 필드가 0 값을 가짐을 보고 버전 협상 패킷임을 구분할 수 있게 된다.

버전 협상 패킷은 서버가 지원하지 않는 버전의 클라이언트 패킷에 대한 응답이고,
따라서 서버만이 보낼 수 있다.

버전 협상 패킷의 레이아웃은 다음과 같다:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
|1|  Unused (7) |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Version (32)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|DCIL(4)|SCIL(4)|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Destination Connection ID (0/32..144)         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Source Connection ID (0/32..144)            ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Supported Version 1 (32)                 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   [Supported Version 2 (32)]                ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                               ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   [Supported Version N (32)]                ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #version-negotiation-format title="Version Negotiation Packet"}

Unused 필드의 값은 서버에 의해 랜덤하게 선택된다.

\["MUST" 버전 협상 패킷의 Version 필드는 0x00000000으로 설정되어야 한다.\]

\["MUST" 서버는 받은 패킷의 Source Connection ID 필드의 값을 Destination
Connection ID 필드에 포함시켜야 한다.] \["MUST" Source Connection ID 값은 받은
패킷의 Destination Connection ID로부터 복사되어야 하며], 이는 초기에
클라이언트에 의해 랜덤하게 선택된 값이다. 두 연결 ID를 똑같이 되보내면
클라이언트는 '서버가 패킷을 잘 받았으며 버전 협상 패킷이 경로 바깥의 공격자에
의해 생성되지 않았음'을 어느 정도 확신할 수 있다.

버전 협상 패킷의 나머지 부분은 32-비트로 표현된, 서버가 지원하는 버전의
리스트이다.

클라이언트는 (서버에게) 버전 협상 패킷에 대한 확인응답 (acknowledgement)을 ACK
프레임을 이용한 명시적인 방법으로 할 수 없다. (서버는) 다른 초기화 패킷을
수신하는 암묵적인 방법으로 버전 협상 패킷의 정상적인 송신을 확인한다.
(역주: 문장이 깔끔하게 번역되지 않음)

버전 협상 패킷은 긴 헤더 형식을 사용하는 다른 패킷에 등장하는 Packet Number
필드 및 Length 필드를 포함하지 않는다. 따라서, 한 UDP 데이터그램 전체가
버전 협상 패킷으로 사용되어져야 한다.

버전 협상 과정의 설명에 대해서는 {{version-negotiation}}를 보라.

## 암호학적 핸드셰이크 패킷 {#handshake-packets}

버전 협상이 완료되면, (사용할) 암호키에 대한 동의를 위해 암호학적 핸드셰이크가
사용된다. 암호학적 핸드셰이크는 초기화 패킷 ({{packet-initial}}), 재시도 패킷
({{packet-retry}}), 그리고 핸드셰이크 패킷 ({{packet-handshake}}) 으로
전달된다.

이 모든 패킷들은 긴 헤더 형식을 쓰며 Version 필드에 현재 QUIC 버전을 담고 있다.

버전을 고려하지 않는 미들박스로 인한 변조 (tampering)를 방지하기 위해,
{{QUIC-TLS}}에 설명되었듯이 핸드셰이크 패킷은 연결 별, 그리고 버전 별 키로
보호된다. 이 보호는 경로 상의 공격자에 대한 기밀성 (confidentiality)이나 무결성
(integrity)은 제공하지 않으나, 경로 외의 공격자에 대한 일정 수준의 보호를
제공한다.

### 초기화 패킷 {#packet-initial}

초기화 패킷은 Type 필드 값이 0x7F인 긴 헤더를 사용한다. 클라이언트가 보낸
첫 암호학적 핸드셰이크 메시지를 담고 있다.

클라이언트가 기존에 서버로부터 재시도 패킷을 받지 않았다면, 랜덤하게 선택된
값으로 Destination Connection ID 필드를 채운다. \["MUST" 이 값은 길이로 8 옥텟
이상이어야 한다.\] (실질적으로 새로운 연결 시도를 하는 것이 되도록) Source
Connection ID도 바꾸는 것이 아니라면, 서버로부터 패킷을 받을 때까지 \["MUST"
클라이언트는 반드시 같은 랜덤 값을 써야 한다.\] 랜덤하게 선택된 Destination
Connection ID는 패킷 보호 키들을 결정하기 위해 사용한다.

클라이언트가 재시도 패킷을 받았고 두 번째로 초기화 패킷을 보내고 있다면,
재시도 패킷의 Source Connection ID 값을 (두 번째 초기화 패킷의) Destination
Connection ID로 설정한다. Destination Connection ID의 변경은 초기화 패킷을
보호하기 위해 사용된 키들의 변경도 초래한다.

클라이언트는 Source Connection ID 필드를 선택한 값으로 채우면서 SCIL 필드를
그 값과 맞아떨어지게 설정한다.

클라이언트가 보낸 첫 초기화 패킷은 패킷 번호로 0을 담고 있다. 모든 후속
패킷들은 적어도 1 이상 증가한 패킷 번호를 포함하는데, 이에 관해서
({{packet-numbers}})를 보아라.

초기화 패킷의 페이로드는 암호학적 핸드셰이크 메시지를 담고 있는, 스트림 0의
STREAM 프레임 (또는 프레임들)을 나른다. 이 패킷의 스트림은 언제나 오프셋 0으로
시작하며 ({{stateless-retry}}를 보라), \["MUST" 완전한 암호학적 핸드셰이크
메시지는 한 패킷 안에 들어가야 한다 ({{handshake}}를 보라).]

\["MUST" 초기화 패킷을 담은 UDP 데이터그램의 페이로드는 1200 옥텟 이상으로
확장되어야 하는데 ({{packetization}}을 보라), PADDING 프레임들을 초기화 패킷에
더하는 방법 그리고/또는 초기 패킷에 0-RTT 패킷을 합하는 방법을 써야 한다
({{packet-coalesce}}를 보라).\]

클라이언트는 '초기 암호화된 핸드셰이크 메시지'를 담은 모든 패킷에 대해 초기화
패킷 타입을 사용한다. 이 상황은 '초기 암호화된 메시지'를 포함한 새 패킷을
만드는 상황을 포함한다. 또한 버전 협상 패킷 ({{packet-version}}) 또는 재시도
패킷 ({{packet-retry}})를 받은 뒤에 보내는 패킷들도 포함한다.


### 재시도 패킷 {#packet-retry}

재시도 패킷은 Type 필드 값이 0x7E인 긴 헤더를 사용한다. 재시도 패킷은 암호학적
핸드셰이크 메시지와 확인응답(acknowledgement)를 담고 있다. 재시도 패킷은 무상태
재시도 (stateless retry)를 수행하고 싶은 서버에 의해 사용된다
({{stateless-retry}}를 보라).

서버는 클라이언트가 초기화 패킷에 담아 보낸 Source Connection ID 필드의 연결
ID를 Destination Connection ID 필드에 포함한다. 이 값은 길이가 0인 값일 수
있다.

서버는 Source Connection ID 필드에 스스로 고른 연결 ID를 포함한다. \["MUST"
클라이언트는 앞으로 보낼 후속 패킷의 Destination Connection ID 필드에 이 연결
ID를 사용해야 한다.\]

\["MUST" 재시도 패킷의 Packet Number 필드는 0으로 설정되어야 한다.\] 이 값은
일반적인 경우처럼 보호(되어 전송)된다. \[\[Editor's Note: 이는 이상적이지 않다.
왜냐면 클라이언트가 값을 가정하는 "꼼수"를 야기하기 때문이다. 이는 문제가
되므로 일반적인 프로세싱을 하도록 2^30 미만의 임의의 값을 넣도록 제안하고 싶다.
물론 적절히 평가되어야 한다. ]]

클라이언트는 (서버에게) 재시도 패킷에 대한 확인응답 (acknowledgement)을 ACK
프레임을 이용한 명시적인 방법으로 할 수 없다. (서버는) 다른 초기화 패킷을
수신하는 암묵적인 방법으로 재시도 패킷의 정상적인 송신을 확인한다.
(역주: 문장이 깔끔하게 번역되지 않음)

클라이언트는 재시도 패킷을 받은 뒤에 다음 암호학적 핸드셰이크 메시지를 담은
새로운 초기화 패킷을 사용한다. 클라이언트는 암호학적 핸드셰이크의 상태를
유지하되, 모든 전송 상태를 폐기한다. 재시도 패킷의 응답으로 생성된 초기화
패킷은 오프셋 0부터 다시 시작하는 스트림 0에서의 STREAM 프레임들을 포함한다.

암호학적 핸드셰이크를 지속하는 것은 공격자가 어떠한 암호학적 파라미터도
강제적으로 다운그레이드할 수 없도록 보장하기 위해 필수적이다. 지속적인
암호학적 핸드셰이크를 할 뿐만 아니라, \["MUST" 클라이언트는 일어났던 모든 버전
협상 결과들을 기억해야만 한다 ({{version-negotiation}}을 보라).\] \["MAY"
클라이언트는 또한 해당 흐름에 대해 누산한 바 있는 측정 RTT나 혼잡 상태를
유지할 수 있지만\], \["MUST" 다른 전송 상태는 반드시 폐기하여야 한다.\]

재시도 패킷의 페이로드는 적어도 두 개의 프레임을 담고 있다. \["MUST" 서버의
암호학적 무상태 재시도 (stateless retry) 내용물을 담은, 오프셋 0의 스트림
0에서의 STREAM 프레임을 포함하여야 한다.\] \["MUST" 또한 클라이언트의 초기화
패킷의 확인응답을 위한 ACK 프레임을 포함하여야 한다.\] \["MAY" 추가적으로
PADDING 프레임을 포함할 수도 있다.\] 서버가 보낼 다음 STREAM 프레임 또한
스트림 오프셋 0에서 시작할 것이다.


### 핸드셰이크 패킷 {#packet-handshake}

핸드셰이크 패킷은 Type 필드 값이 0x7D인 긴 헤더를 사용한다. 서버 및
클라이언트로부터의 확인응답과 암호학적 핸드셰이크 메시지를 담기 위해 사용한다.
(역주: 핸드셰이크 패킷과 핸드셰이크 메시지가 다르다는 점을 구분하라.)

서버는, 재시도 패킷을 보낸 적이 없다면, 초기화 패킷의 응답으로 하나 이상의
핸드셰이크 패킷에 서버의 암호학적 핸드셰이크를 보낸다. 클라이언트가 서버로부터
핸드셰이크 패킷을 받으면, 서버로 후속 암호학적 핸드셰이크 메시지와 확인응답을
보내기 위해 핸드셰이크 패킷을 사용한다.

핸드셰이크 패킷의 Destination Connection ID 필드는 패킷의 수신자에 의해 선택된
연결 ID를 포함하고 있다; Source Connection ID 필드는 패킷을 사용코자
하는 송신측의 연결 ID를 포함하고 있다 ({{connection-id}}를 보라).
(역주: 즉, 악수를 청하는 사람이 Source, 악수를 받는 사람이 Destination이다.)

서버가 보낸 첫 핸드셰이크 패킷은 패킷 번호 0을 담고 있다. 다른 핸드셰이크
패킷에서는 패킷 넘버는 보통 증가한다.

\["MUST NOT" 검증된 (verified) 소스 주소로부터 패킷을 받은 게 아니라면, 서버는
3 개를 초과하는 핸드셰이크 패킷을 보내선 안된다.\] 소스 주소는 주소 검증 토큰
(address validation token), 클라이언트로부터 최종 암호학적 메시지의 수신, 또는
클라이언트로부터 유효한 PATH_RESPONSE 프레임을 받음으로써 검증될 수 있다.

서버가 초기화 패킷의 응답으로 3 개를 초과한 핸드셰이크 패킷을 생성할
예정이라면, \["SHOULD" 보낼 각 핸드셰이크 패킷에 PATH_CHALLENGE 프레임을
포함하여야 한다.\] 적어도 하나의 유효한 PATH_RESPONSE 프레임을 받은 후에,
서버는 남은 핸드셰이크 패킷을 보낼 수 있다. 대신, 서버는 재시작 패킷을 사용해
주소 검증을 수행할 수 있다; 이는 서버에 상태 정보를 거의 요구하지 않지만, 구현
선택에 따라 추가적인 계산 노력을 수반할 수 있다.

핸드셰이크 패킷의 페이로드는 STREAM 프레임을 포함하고 있고, PADDING, ACK,
PATH_CHALLENGE, 또는 PATH_RESPONSE 프레임을 포함할 수 있다. \["MAY" 핸드셰이크
패킷은 핸드셰이크가 성공하지 못할 경우에 CONNECTION_CLOSE 프레임을 포함할 수
있다. \]


## 보호된 패킷 {#packet-protected}

모든 QUIC 패킷은 패킷 보호를 사용한다. 정적 (static) 핸드셰이크 키나 0-RTT 키로
보호된 패킷은 긴 헤더와 함께 보내진다; 1-RTT 키로 보호된 모든 패킷은 짧은
헤더와 함께 보내진다. 이러한 다른 패킷 타입은 명시적으로 암호화 수준을
알려주고, 따라서 패킷 보호를 제거하기 위해 사용될 키들을 알려준다.

핸드셰이크 키로 보호된 패킷들은 패킷의 송신자가 네트워크 경로 상에 있음을
확신하기 위해서만 패킷 보호를 사용한다. 이러한 패킷 보호는 실질적인 (effective)
기밀성 보호가 아니다; 클라이언트로부터 초기화 패킷을 받은 어느 엔티티도 '패킷
보호를 제거하기 위해' 또는 '성공적으로 인증될 수 있는 패킷을 생성하기 위해'
필수적인 키를 복구할 수 있다.

0-RTT와 1-RTT 키로 보호된 패킷은 기밀성과 데이터 원천 (origin) 인증을 가지고
있다고 기대된다; 암호학적 핸드셰이크는 통신하는 엔드포인트들만이 각각에
대응하는 키를 받도록 확신시킨다.

0-RTT 키로 보호된 패킷은 Type 필드의 값로 0x7C를 사용한다. \["MUST" 0-RTT
패킷의 Connection ID는 초기화 패킷에서 사용된 값들과 대응되어야만 한다
({{packet-initial}}).\]

클라이언트는 핸드셰이크 패킷을 받은 후에 만약 해당 패킷이 핸드셰이크를 끝내지
않을 때 0-RTT 패킷을 보낼 수 있다. 클라이언트가 핸드셰이크 패킷에서 다른 연결
ID를 받더라도, \["MUST" 0-RTT 패킷에서도 동일한 Destination Connection ID를
사용해야만 한다.\] 이에 관해 {{connection-id}}를 보라.
(역주: 의미가 다소 와닿지 않는데 추후 수정하자.)

보호된 패킷의 Version 필드는 현재 QUIC 버전이다.

Packet Number 필드는 패킷 보호가 적용된 뒤에 추가적인 기밀성 보호를 적용받은
패킷 번호를 담는다 (상세에 대해 {{QUIC-TLS}}를 보라). 패킷이 보내질 때마다
보호되기 전의 (underlying) 패킷 번호는 증가한다 (상세에 대해
{{packet-numbers}}를 보라).

페이로드는 인증암호화 (authenticated encryption)를 사용해 보호된다.
{{QUIC-TLS}}는 패킷 보호에 관한 상세를 설명한다. 복호화 후에, 평문
(plaintext)은 {{frames}}에 명시되었듯 일련의 프레임으로 이루어진다.
(역주: 인증암호화란 인증 정보 (보통 MAC)가 같이 붙은 암호문을 생성하는 것이다.)


## 패킷 통합 {#packet-coalesce}

송신측은 한 UDP 데이터그램에 (보통 암호학적 핸드셰이크 패킷과 보호된 패킷으로
이루어진) 여러 QUIC 패킷을 통합할 수 있다. 핸드셰이크 및 그 이후 과정에서
어플리케이션 데이터를 송신하는데 필요한 UDP 데이터그램의 수를 줄일 수 있다.
짧은 헤더를 갖는 패킷은 Length 필드를 가지지 않으므로, 해당 패킷이 UDP
데이터그램의 유일한 패킷이어야 한다.

\["MUST NOT" 송신측은  다른 QUIC 연결에 속한 QUIC 패킷을 하나의 UDP
데이터그램으로 통합해서는 안 된다.\]
(역주: 따라서 QUIC 연결을 여러 개 만들어 Multipath를 구성할 경우, 통합을 통한
메시지 수 줄이기가 불가능하다.)

한 UDP 데이터그램에 통합된 모든 QUIC 패킷은 분리되어 (separate) 있으며
완전하다. 비록 패킷 헤더에서 몇몇 필드 값이 중복될 수 있지만, 어떤 필드도
생략되지 않는다. \["MUST" 통합된 QUIC 패킷의 수신측은 각 QUIC 패킷을 개별적으로
처리해야만 하며\], 서로 다른 UDP 데이터그램의 페이로드로 수신된 것처럼 분리해서
확인응답해야 한다.


## 연결 ID {#connection-id}

연결 ID가 패킷의 항상적인 (consistent) 라우팅을 할 수 있도록 사용된다. 긴
헤더는 두 연결 ID를 포함하고 있다: 해당 패킷의 수신자에 의해 선택된 Destination
Connection ID가 항상적인 라우팅을 제공하기 위해 사용된다; Source Connection
ID는 상대방이 Destination Connection ID를 설정하기 위해 사용된다.

핸드셰이크 과정에서, 긴 헤더를 가진 패킷은 각 엔드포인트가 사용하는 연결 ID를
설립하기 위해 사용된다. 각 엔드포인트는 받은 패킷의 Destination Connection ID
필드에 사용된 연결 ID를 Source Connection ID 필드에 기재하기 위해 사용한다.
패킷을 받으면, 각 엔드포인트는 보낼 패킷의 Destination Connection ID를 받은
Source Connection ID 값과 같도록 설정한다.

핸드셰이크 과정에서, 각 엔드포인트는 여러 개의 긴 헤더 패킷들을 받을 수 있고,
따라서 보내는 패킷을 위한 Destination Connection ID를 업데이트할 기회가 여럿
있을 수 있다. \["MUST" 클라이언트는 서버로부터 받는 각 타입 (재시도 또는
핸드셰이크의 패킷 중 가장 첫 번째 패킷의 응답으로만 '보낼 Destination
Connection ID'를 변경해야 한다\]; \["MUST" 서버는 초기화 패킷에 대해서만 값을
설정해야 한다.\] 어떠한 추가 변경도 허용되지 않는다; 만약 해당 타입의 후속
패킷이 다른 Source Connection ID를 포함하고 있으면, \["MUST" 그 패킷은
폐기되어야 한다.\] 이는 다른 연결 ID를 생성하는 여러 초기화 패킷을 무상태로
처리하는 경우 발생할 수 있는 문제들을 회피하기 위함이다.

짧은 헤더는 Destination Connection ID만을 포함하며 명시적인 길이를 생략한다.
Destination Connection ID 필드의 길이는 각 엔드포인트에서 이미 알고 있을 것으로
기대된다.

연결 ID 기반의 로드 밸런서를 사용하는 엔드포인트들은 연결 ID의 '고정 또는 최소
길이' 및 인코딩에 대해 로드 밸런서와 합의할 수 있을 것이다. 이러한 고정된
부분은 명시적인 길이를 인코딩할 수 있을 것이며, 이는 전체 연결 ID의 길이를
다양하게 하면서도 로드 밸런서를 사용할 수 있게 한다.
(역주: 실질적인 구현에 대해 고민해볼 필요가 있겠다.)

클라이언트가 보낸 제일 처음 패킷은 Destination Connection ID 필드에 랜덤 값을
포함시킨다. \["MUST" 같은 값이 해당 연결에 송신되는 모든 0-RTT 패킷들에
사용되어야만 한다 ({{packet-protected}}).\] 이 랜덤 값은 핸드셰이크 패킷 보호
키를 결정하기 위해 사용된다 ({{QUIC-TLS}}의 5.3.2절을 보라).

\["MUST" 버전 협상 ({{packet-version}}) 패킷은 클라이언트에 의해 선택된 두 연결
ID를 사용해야만 한다.\] 이 때 두 d연결 ID는 클라이언트에게 올바로 전달될 수
있도록 교환된다.

연결 ID는 연결이 살아있는 동안, 특히 연결 이전 ({{migration}}) 상황에서, 변경될
수 있다. NEW_CONNECTION_ID 프레임 ({{frame-new-connection-id}})가 새로운 연결
ID 값을 제공하기 위해 사용된다.

## 패킷 번호 {#packet-numbers}

패킷 번호는 0에서 2^62-1 사이의 정수이다. 패킷 암호화를 위한 암호학적 논스
(nonse)를 결정하는데 사용된다. 각 엔드포인트는 보내기용 패킷 번호와 받기용 패킷
번호를 구분해 유지한다. \["MUST" 보내기용 패킷 번호는 보내는 첫 패킷에서 0으로
시작해야만 한다.\] \["MUST" 또한 패킷을 보낸 뒤에 적어도 1 이상 증가하여야만
한다.\]

\["MUST NOT" QUIC 엔드포인트는 같은 연결 내에서 (즉 같은 암호키를 쓰는
연결에서) 패킷 번호를 재사용해서는 안된다.\] 만약 보내기용 패킷 번호가 2^62 -
1에 도달했다면, \["MUST" 송신측은 CONNECTION_CLOSE 프레임 또는 이후의 패킷을
보내지 않고 연결을 닫아야만 한다\]; \["MAY" 엔드포인트는 (이러한 상황에서) 받은
후속 패킷의 응답으로 무상태 리셋 ({{stateless-reset}})을 보낼 수 있다.\]

QUIC의 긴/짧은 패킷 헤더에서, 패킷 번호를 나타내기 위해 필요한 비트 수는 패킷
번호에서 변화가 있는 최하위 비트만을 포함함으로써 줄인다. {{pn-encodings}}에서
보여지듯이, 첫 옥텟에서 한 개 또는 두 개의 최상위 비트는 몇 비트의 패킷 번호가
주어졌는지를 나타낸다.

| First octet pattern | Encoded Length | Bits Present |
|:--------------------|:---------------|:-------------|
| 0b0xxxxxxx          | 1 octet        | 7            |
| 0b10xxxxxx          | 2              | 14           |
| 0b11xxxxxx          | 4              | 30           |
{: #pn-encodings title="Packet Number Encodings for Packet Headers"}

이 인코딩은 {{integer-encoding}}에서의 인코딩과 비슷하지만, 다른 값을
사용한다는 점을 주의하라.

인코딩된 패킷 번호는 5.6절 {{QUIC-TLS}}에 설명된 것과 같이 보호된다. 패킷
번호의 보호는 완전한 패킷 번호를 복구하기 전에 제거된다. 완전한 패킷 번호는
최상위 비트들의 수, 비트의 내용, 그리고 성공적으로 인증된 패킷에서 받아낸 가장
큰 패킷 번호를 기반으로 재구성된다. 완전한 패킷 번호를 복구하는 것은 성공적으로
패킷 보호를 제거하기 위해 필수적이다.

패킷 번호 보호가 제거되면, 패킷 번호는 다음에 올 것으로 기대되는 패킷에 가장
가까운 패킷 번호 값을 찾는 작업을 통해 디코딩된다. 다음에 올 것으로 기대되는
패킷이란 (성공적으로) 받은 패킷 번호의 최대값에 1을 더한 것이다. 예를 들어,
성공적으로 인증된 패킷들 중 가장 높은 패킷 번호가 0xaa82f30e였다면, 14 비트
값인 0x1f94를 담은 패킷은 0xaa831f94로 디코딩되어야 한다.
(역주: 0xaa82f30e + 1 = 0xaa82f30f이다. 그런데 1f94는 f30e에서 상위 2 비트를
제거한 330e와 비교해봐도 작은 값이므로, 순증가하는 패킷 번호의 특성 상
0xaa831f94로 디코딩되어야 함을 알 수 있다. 다만 다음 패킷 번호가 저렇게 크게
바뀌어도 되는가가 불분명하므로, QUIC 구현 등을 검토할 필요가 있다.)

\["MUST" 송신측은 반드시 '확인응답된 패킷 중 가장 큰 패킷 번호'와 '송신 중인
패킷 번호 간의 차보다 범위 상 2 배 이상 큰 패킷 번호 크기를 사용해야만 한다.\]
그래야 패킷을 받는 상대방 (peer)이 올바로 패킷 번호를 해독할 수 있을 것이다.
다만 전달이 지연되어 더 큰 패킷 번호를 갖는 패킷들이 도착한 뒤에야 도착한
경우엔 그렇지 않을 수도 있다. \["SHOULD" 엔드포인트는 후속 패킷이 보내진 뒤에
도착하는 패킷이 있더라도 패킷 번호를 복구할 수 있을 정도로 충분히 큰 패킷 번호
인코딩을 사용하여야 한다.\]
(역주: 다만 .. 있다 부분은 의역. 이 부분은 패킷 번호를 압축하고자 하는
프로토콜 설게자들의 의지를 느낄 수 있지만, 유효한지가 의문이다. deployability
측면에서 별로 좋지 않은 설계인 듯. out-of-order 패킷의 지연 시간 분포에
대해서도 추가적인 체크가 필요하다.)

결론적으로, 패킷 번호 인코딩의 크기는 새로운 패킷을 포함해 확인응답을 받지 못한
인접한 패킷 번호의 개수에 밑이 2인 로그를 취한 값보다 최소 1 이상 커야 한다.
(역주: 역시 번역이 다소 의심스럽다.)

예를 들어, 엔드포인트가 패킷 0x6afa2f에 대한 확인응답을 방금 받았는데, 패킷
번호가 0x6b2d79인 패킷을 전송하기 위해서는 14비트 이상의 패킷 번호 인코딩이
필요하다; 반면 패킷 번호가 0x6bc107인 패킷을 보내기 위해서는 30 비트 패킷 번호
인코딩이 필요하다.

버전 협상 패킷 ({{packet-version}})은 패킷 번호를 포함하지 않는다. 재시도 패킷
({{packet-retry}})은 Packet Number 필드를 기재하기 (populating) 위한 특별한
규칙이 있다.


# 프레임 및 프레임 타입 {#frames}

패킷 보호를 제거한 뒤에, 모든 패킷의 페이로드는 {{packet-frames}}에 나타나듯
일련의 프레임으로 구성된다. 버전 협상 패킷 및 무상태 리셋 패킷은 프레임을
포함하지 않는다.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Frame 1 (*)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Frame 2 (*)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                               ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Frame N (*)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #packet-frames title="Contents of Protected Payload"}

\["MUST" 보호된 페이로드는 적어도 하나의 프레임을 포함해야만 한다.] \["MAY"
또한 보호된 페이로드는 여러 프레임 및 여러 프레임 타입을 포함할 수도 있다.\]

\["MUST" 프레임은 단일 QUIC 패킷 안에 들어갈 수 있어야만 한다.\] \["MUST NOT"
프레임은 QUIC 패킷 경계를 걸쳐서는 (span) 안된다.\] 각 프레임은 타입을 나타내기
위해 Frame Type 바이트로 시작한 뒤 추가적인 Type-Dependent 필드가 이어진다:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Type (8)    |           Type-Dependent Fields (*)         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #frame-layout title="Generic Frame Layout"}

프레임 타입은 {{frame-types}}에 나열되어 있다. STREAM 프레임에서 Frame Type
바이트는 다른 프레임-특화된 플래그를 담기 위해 사용됨에 주의하라. 다른 모든
프레임들에서는, Frame Type 바이트는 단순히 프레임을 구분한다. 이 프레임들에
대해서는 본 문서 뒷부분에서 좀 더 자세히 설명한다.

| Type Value  | Frame Type Name   | Definition                  |
|:------------|:------------------|:----------------------------|
| 0x00        | PADDING           | {{frame-padding}}           |
| 0x01        | RST_STREAM        | {{frame-rst-stream}}        |
| 0x02        | CONNECTION_CLOSE  | {{frame-connection-close}}  |
| 0x03        | APPLICATION_CLOSE | {{frame-application-close}} |
| 0x04        | MAX_DATA          | {{frame-max-data}}          |
| 0x05        | MAX_STREAM_DATA   | {{frame-max-stream-data}}   |
| 0x06        | MAX_STREAM_ID     | {{frame-max-stream-id}}     |
| 0x07        | PING              | {{frame-ping}}              |
| 0x08        | BLOCKED           | {{frame-blocked}}           |
| 0x09        | STREAM_BLOCKED    | {{frame-stream-blocked}}    |
| 0x0a        | STREAM_ID_BLOCKED | {{frame-stream-id-blocked}} |
| 0x0b        | NEW_CONNECTION_ID | {{frame-new-connection-id}} |
| 0x0c        | STOP_SENDING      | {{frame-stop-sending}}      |
| 0x0d        | ACK               | {{frame-ack}}               |
| 0x0e        | PATH_CHALLENGE    | {{frame-path-challenge}}    |
| 0x0f        | PATH_RESPONSE     | {{frame-path-response}}     |
| 0x10 - 0x17 | STREAM            | {{frame-stream}}            |
{: #frame-types title="Frame Types"}

# 연결의 생애

QUIC 연결은 두 QUIC 엔드포인트 간의 단일 대화이다. QUIC의 연결 설립은
{{handshake}}에 설명되어 있듯이 버전 협상에 암호학적 핸드셰이크와 전송
핸드셰이크를 섞어 연결 설립 지연을 줄인다. 연결이 설립된 뒤에, 연결은
{{migration}}에 설명되어 있듯이 NAT 재바인딩이나 이동성으로 인해 각
엔드포인트에서 다른 IP나 포트로 이전할 수 있다. 최종적으로 연결은
{{termination}}에 설명되어 있듯이 각 엔드포인트에서 종료될 수 있다.

## 패킷을 연결에 매칭하기 {#packet-handling}

도착 패킷은 수신 과정 (receipt)에서 분류된다. 패킷은 기존 연결에 결부될
(associated) 수 있으며, 또는 패킷은 - 서버에서 - 잠재적으로 새 연결을 생성할 수
있다.

호스트는 기존 연결에 패킷을 결부시키려고 한다. 패킷이 패킷이 기존 연결에
대응하는 Destination Connection ID를 가진다면, QUIC은 해당 패킷을 적절히
처리한다. NEW_CONNECTION_ID 프레임 ({{frame-new-connection-id}})는 한 연결에
한 개 이상의 연결 ID를 결부할 수 있음을 주의하라.

패킷의 Destination Connection ID의 길이가 0이고, 해당 호스트가 가지고 있는
'연결 ID를 필요로 하지 않는 연결'의 주소/포트 튜플과 그 패킷이 매칭된다면,
QUIC은 그 패킷을 해당 연결의 일부로 처리한다. \["MUST" 엔드포인트는,
Destination Connection ID 필드의 길이가 0인 패킷이 어느 단일 연결에도 대응하지
않을 때, 그 패킷을 폐기해야만 한다.\]


### 클라이언트의 패킷 핸들링 {#client-pkt-handling}

클라이언트로 보내진 유효한 패킷은 언제나 '클라이언트가 선택한 값과 매칭되는
Destination Connection ID'를 포함한다. 길이가 0인 연결 ID를 받고자 하는
클라이언트는 연결을 특정하기 위해 주소/포트 튜플을 사용할 수 있다. \["MAY" 기존
연결과 매칭되지 않는 패킷은 폐기될 수 있다.\]

패킷 재정렬 또는 패킷 손실로 인해, 클라이언트는 아직 계산되지 않은 키로
암호화된 연결에 대한 패킷을 받을 수 있다. \["MAY" 클라이언트는 이러한 패킷을
폐기할 수 있거나\], 또는 \["MAY" 클라이언트는 키를 계산할 수 있게 하는 후속
패킷을 고대하며 이러한 패킷을 버퍼링할 수 있다.\]

클라이언트가 지원되지 않는 버전의 패킷을 받으면, \["MUST" 그 패킷은 반드시
폐기되어야 한다.\]


### 서버의 패킷 핸들링 {#server-pkt-handling}

서버가 지원되지 않는 버전의 패킷을 받았는데, 그 패킷이 서버에서 지원되는 어떤
버전의 초기화 패킷이 될만큼 충분한 길이이면, \["SHOULD" 서버는 {{send-vn}}에
설명되었듯이 버전 협상 패킷을 보내야 한다.\] \["MAY" 서버는 버전 협상 패킷의
폭풍(storm)을 피하기 위해 패킷의 전송률 (rate)을 제어할 것이다.\]

지원되지 않는 버전의 첫 패킷은 어떤 버전 특화된 필드에 대해 다른 시맨틱과
인코딩을 사용할 수 있다. 특히, 다른 패킷 보호 키가 다른 버전에서 사용될 것이다.
특정 버전을 지원하지 않는 서버는 패킷의 내용을 해독하는 것이 거의 어려울
것이다. \["SHOULD NOT" 서버는 모르는 버전의 패킷을 디코딩 또는 복호화하려고
시도해서는 안 되지만,\] 대신, 받은 패킷이 충분히 긴 경우에는 버전 협상 패킷은
보내도 된다.

\["MUST" 서버는 지원되지 않는 버전을 포함한 다른 패킷들을 폐기해야만 한다.\]

지원되는 버전의 패킷 또는 Version 필드가 없는 패킷은 {{packet-handling}}에
설명된 대로 어떤 연결에 매칭되어진다. 매칭되지 않는다면, 서버는 아래와 같이
한다.

패킷이 명세를 완전히 따르는 초기화 패킷이라면, 서버는 핸드셰이크
({{handshake}})를 진행한다. 이는 서버가 클라이언트가 선택한 버전을 택하게
(commit) 한다.

서버가 현재 어떤 새로운 연결도 수락하지 않는다면, \["SHOULD" 서버는 에러 코드
SERVER_BUSY와 함께 CONNECTION_CLOSE 프레임을 담은 핸드셰이크 패킷을 보내야
한다.\]

패킷이 0-RTT 패킷이면, \["MAY" 서버는 늦게 도착하는 초기화 패킷을 기대하며
유한 개의 패킷을 버퍼링할 것이다.\] 클라이언트는 서버 응답을 받기 전에
핸드셰이크 패킷을 보내는 것이 금지된다. 따라서 \["SHOULD" 서버는 그런 패킷을
무시해야 한다.\]

\["MUST" 서버는 이외의 모든 상황에 처한 도착 패킷을 반드시 폐기하여야만
한다.\] \["SHOULD" 도착 패킷의 헤더에 연결 ID가 있을 경우, 서버는 무상태 리셋
({{stateless-reset}})을 보내야 한다.\]

## 버전 협상

버전 협상은 클라이언트와 서버가 상호간에 지원하는 QUIC 버전의 사용을 동의하도록
한다. 서버는 새로운 연결을 시작하려는 각 패킷의 응답으로 버전 협상 패킷을
보내는데, 상세한 내용은 {{packet-handling}}을 보아라.

클라이언트가 보낸 첫 패킷의 크기는 서버가 버전 협상 패킷을 보낼지 말지 여부를
결정할 것이다. 여러 QUIC 버전을 지원하는 클라이언트는 각 버전의 '최소 초기화
패킷 사이즈' 중 최대값을 반영해 패딩을 넣어야 한다. 이는 상호간에 지원하는
버전이 하나라도 있으면 서버가 응답하도록 한다.

### 버전 협상 패킷 보내기 {#send-vn}

If the version selected by the client is not acceptable to the server, the
server responds with a Version Negotiation packet (see {{packet-version}}).
This includes a list of versions that the server will accept.

This system allows a server to process packets with unsupported versions without
retaining state.  Though either the Initial packet or the Version Negotiation
packet that is sent in response could be lost, the client will send new packets
until it successfully receives a response or it abandons the connection attempt.


### Handling Version Negotiation Packets {#handle-vn}

When the client receives a Version Negotiation packet, it first checks that the
Destination and Source Connection ID fields match the Source and Destination
Connection ID fields in a packet that the client sent.  If this check fails, the
packet MUST be discarded.

Once the Version Negotiation packet is determined to be valid, the client then
selects an acceptable protocol version from the list provided by the server.
The client then attempts to create a connection using that version.  Though the
contents of the Initial packet the client sends might not change in
response to version negotiation, a client MUST increase the packet number it
uses on every packet it sends.  Packets MUST continue to use long headers and
MUST include the new negotiated protocol version.

The client MUST use the long header format and include its selected version on
all packets until it has 1-RTT keys and it has received a packet from the server
which is not a Version Negotiation packet.

A client MUST NOT change the version it uses unless it is in response to a
Version Negotiation packet from the server.  Once a client receives a packet
from the server which is not a Version Negotiation packet, it MUST discard other
Version Negotiation packets on the same connection.  Similarly, a client MUST
ignore a Version Negotiation packet if it has already received and acted on a
Version Negotiation packet.

A client MUST ignore a Version Negotiation packet that lists the client's chosen
version.

Version negotiation packets have no cryptographic protection. The result of the
negotiation MUST be revalidated as part of the cryptographic handshake (see
{{version-validation}}).


### Using Reserved Versions

For a server to use a new version in the future, clients must correctly handle
unsupported versions. To help ensure this, a server SHOULD include a reserved
version (see {{versions}}) while generating a Version Negotiation packet.

The design of version negotiation permits a server to avoid maintaining state
for packets that it rejects in this fashion. The validation of version
negotiation (see {{version-validation}}) only validates the result of version
negotiation, which is the same no matter which reserved version was sent.
A server MAY therefore send different reserved version numbers in the Version
Negotiation Packet and in its transport parameters.

A client MAY send a packet using a reserved version number.  This can be used to
solicit a list of supported versions from a server.


## Cryptographic and Transport Handshake {#handshake}

QUIC relies on a combined cryptographic and transport handshake to minimize
connection establishment latency.  QUIC allocates stream 0 for the cryptographic
handshake.  Version 0x00000001 of QUIC uses TLS 1.3 as described in
{{QUIC-TLS}}; a different QUIC version number could indicate that a different
cryptographic handshake protocol is in use.

QUIC provides this stream with reliable, ordered delivery of data.  In return,
the cryptographic handshake provides QUIC with:

* authenticated key exchange, where

   * a server is always authenticated,

   * a client is optionally authenticated,

   * every connection produces distinct and unrelated keys,

   * keying material is usable for packet protection for both 0-RTT and 1-RTT
     packets, and

   * 1-RTT keys have forward secrecy

* authenticated values for the transport parameters of the peer (see
  {{transport-parameters}})

* authenticated confirmation of version negotiation (see {{version-validation}})

* authenticated negotiation of an application protocol (TLS uses ALPN
  {{?RFC7301}} for this purpose)

* for the server, the ability to carry data that provides assurance that the
  client can receive packets that are addressed with the transport address that
  is claimed by the client (see {{address-validation}})

The initial cryptographic handshake message MUST be sent in a single packet.
Any second attempt that is triggered by address validation MUST also be sent
within a single packet.  This avoids having to reassemble a message from
multiple packets.  Reassembling messages requires that a server maintain state
prior to establishing a connection, exposing the server to a denial of service
risk.

The first client packet of the cryptographic handshake protocol MUST fit within
a 1232 octet QUIC packet payload.  This includes overheads that reduce the space
available to the cryptographic handshake protocol.

Details of how TLS is integrated with QUIC is provided in more detail in
{{QUIC-TLS}}.


## Transport Parameters

During connection establishment, both endpoints make authenticated declarations
of their transport parameters.  These declarations are made unilaterally by each
endpoint.  Endpoints are required to comply with the restrictions implied by
these parameters; the description of each parameter includes rules for its
handling.

The format of the transport parameters is the TransportParameters struct from
{{figure-transport-parameters}}.  This is described using the presentation
language from Section 3 of {{!I-D.ietf-tls-tls13}}.

~~~
   uint32 QuicVersion;

   enum {
      initial_max_stream_data(0),
      initial_max_data(1),
      initial_max_bidi_streams(2),
      idle_timeout(3),
      preferred_address(4),
      max_packet_size(5),
      stateless_reset_token(6),
      ack_delay_exponent(7),
      initial_max_uni_streams(8),
      (65535)
   } TransportParameterId;

   struct {
      TransportParameterId parameter;
      opaque value<0..2^16-1>;
   } TransportParameter;

   struct {
      select (Handshake.msg_type) {
         case client_hello:
            QuicVersion initial_version;

         case encrypted_extensions:
            QuicVersion negotiated_version;
            QuicVersion supported_versions<4..2^8-4>;
      };
      TransportParameter parameters<22..2^16-1>;
   } TransportParameters;

   struct {
     enum { IPv4(4), IPv6(6), (15) } ipVersion;
     opaque ipAddress<4..2^8-1>;
     uint16 port;
     opaque connectionId<0..18>;
     opaque statelessResetToken[16];
   } PreferredAddress;
~~~
{: #figure-transport-parameters title="Definition of TransportParameters"}

The `extension_data` field of the quic_transport_parameters extension defined in
{{QUIC-TLS}} contains a TransportParameters value.  TLS encoding rules are
therefore used to encode the transport parameters.

QUIC encodes transport parameters into a sequence of octets, which are then
included in the cryptographic handshake.  Once the handshake completes, the
transport parameters declared by the peer are available.  Each endpoint
validates the value provided by its peer.  In particular, version negotiation
MUST be validated (see {{version-validation}}) before the connection
establishment is considered properly complete.

Definitions for each of the defined transport parameters are included in
{{transport-parameter-definitions}}.  Any given parameter MUST appear
at most once in a given transport parameters extension.  An endpoint MUST
treat receipt of duplicate transport parameters as a connection error of
type TRANSPORT_PARAMETER_ERROR.


### Transport Parameter Definitions

An endpoint MUST include the following parameters in its encoded
TransportParameters:

initial_max_stream_data (0x0000):

: The initial stream maximum data parameter contains the initial value for the
  maximum data that can be sent on any newly created stream.  This parameter is
  encoded as an unsigned 32-bit integer in units of octets.  This is equivalent
  to an implicit MAX_STREAM_DATA frame ({{frame-max-stream-data}}) being sent on
  all streams immediately after opening.

initial_max_data (0x0001):

: The initial maximum data parameter contains the initial value for the maximum
  amount of data that can be sent on the connection.  This parameter is encoded
  as an unsigned 32-bit integer in units of octets.  This is equivalent to
  sending a MAX_DATA ({{frame-max-data}}) for the connection immediately after
  completing the handshake.

idle_timeout (0x0003):

: The idle timeout is a value in seconds that is encoded as an unsigned 16-bit
  integer.  The maximum value is 600 seconds (10 minutes).

An endpoint MAY use the following transport parameters:

initial_max_bidi_streams (0x0002):

: The initial maximum bidirectional streams parameter contains the initial
  maximum number of application-owned bidirectional streams the peer may
  initiate, encoded as an unsigned 16-bit integer.  If this parameter is absent
  or zero, application-owned bidirectional streams cannot be created until a
  MAX_STREAM_ID frame is sent.  Note that a value of 0 does not prevent the
  cryptographic handshake stream (that is, stream 0) from being used. Setting
  this parameter is equivalent to sending a MAX_STREAM_ID
  ({{frame-max-stream-id}}) immediately after completing the handshake
  containing the corresponding Stream ID. For example, a value of 0x05 would be
  equivalent to receiving a MAX_STREAM_ID containing 20 when received by a
  client or 17 when received by a server.

initial_max_uni_streams (0x0008):

: The initial maximum unidirectional streams parameter contains the initial
  maximum number of application-owned unidirectional streams the peer may
  initiate, encoded as an unsigned 16-bit integer.  If this parameter is absent
  or zero, unidirectional streams cannot be created until a MAX_STREAM_ID frame
  is sent.  Setting this parameter is equivalent to sending a MAX_STREAM_ID
  ({{frame-max-stream-id}}) immediately after completing the handshake
  containing the corresponding Stream ID. For example, a value of 0x05 would be
  equivalent to receiving a MAX_STREAM_ID containing 18 when received by a
  client or 19 when received by a server.

max_packet_size (0x0005):

: The maximum packet size parameter places a limit on the size of packets that
  the endpoint is willing to receive, encoded as an unsigned 16-bit integer.
  This indicates that packets larger than this limit will be dropped.  The
  default for this parameter is the maximum permitted UDP payload of 65527.
  Values below 1200 are invalid.  This limit only applies to protected packets
  ({{packet-protected}}).

ack_delay_exponent (0x0007):

: An 8-bit unsigned integer value indicating an exponent used to decode the ACK
  Delay field in the ACK frame, see {{frame-ack}}.  If this value is absent, a
  default value of 3 is assumed (indicating a multiplier of 8).  The default
  value is also used for ACK frames that are sent in Initial, Handshake, and
  Retry packets.  Values above 20 are invalid.

A server MAY include the following transport parameters:

stateless_reset_token (0x0006):

: The Stateless Reset Token is used in verifying a stateless reset, see
  {{stateless-reset}}.  This parameter is a sequence of 16 octets.

preferred_address (0x0004):

: The server's Preferred Address is used to effect a change in server address at
  the end of the handshake, as described in {{preferred-address}}.

A client MUST NOT include a stateless reset token or a preferred address.  A
server MUST treat receipt of either transport parameter as a connection error of
type TRANSPORT_PARAMETER_ERROR.


### Values of Transport Parameters for 0-RTT {#zerortt-parameters}

A client that attempts to send 0-RTT data MUST remember the transport parameters
used by the server.  The transport parameters that the server advertises during
connection establishment apply to all connections that are resumed using the
keying material established during that handshake.  Remembered transport
parameters apply to the new connection until the handshake completes and new
transport parameters from the server can be provided.

A server can remember the transport parameters that it advertised, or store an
integrity-protected copy of the values in the ticket and recover the information
when accepting 0-RTT data.  A server uses the transport parameters in
determining whether to accept 0-RTT data.

A server MAY accept 0-RTT and subsequently provide different values for
transport parameters for use in the new connection.  If 0-RTT data is accepted
by the server, the server MUST NOT reduce any limits or alter any values that
might be violated by the client with its 0-RTT data.  In particular, a server
that accepts 0-RTT data MUST NOT set values for initial_max_data or
initial_max_stream_data that are smaller than the remembered value of those
parameters.  Similarly, a server MUST NOT reduce the value of
initial_max_bidi_streams or initial_max_uni_streams.

Omitting or setting a zero value for certain transport parameters can result in
0-RTT data being enabled, but not usable.  The following transport parameters
SHOULD be set to non-zero values for 0-RTT: initial_max_bidi_streams,
initial_max_uni_streams, initial_max_data, initial_max_stream_data.

The value of the server's previous preferred_address MUST NOT be used when
establishing a new connection; rather, the client should wait to observe the
server's new preferred_address value in the handshake.

A server MUST reject 0-RTT data or even abort a handshake if the implied values
for transport parameters cannot be supported.


### New Transport Parameters

New transport parameters can be used to negotiate new protocol behavior.  An
endpoint MUST ignore transport parameters that it does not support.  Absence of
a transport parameter therefore disables any optional protocol feature that is
negotiated using the parameter.

New transport parameters can be registered according to the rules in
{{iana-transport-parameters}}.


### Version Negotiation Validation {#version-validation}

Though the cryptographic handshake has integrity protection, two forms of QUIC
version downgrade are possible.  In the first, an attacker replaces the QUIC
version in the Initial packet.  In the second, a fake Version Negotiation packet
is sent by an attacker.  To protect against these attacks, the transport
parameters include three fields that encode version information.  These
parameters are used to retroactively authenticate the choice of version (see
{{version-negotiation}}).

The cryptographic handshake provides integrity protection for the negotiated
version as part of the transport parameters (see {{transport-parameters}}).  As
a result, attacks on version negotiation by an attacker can be detected.

The client includes the initial_version field in its transport parameters.  The
initial_version is the version that the client initially attempted to use.  If
the server did not send a Version Negotiation packet {{packet-version}}, this
will be identical to the negotiated_version field in the server transport
parameters.

A server that processes all packets in a stateful fashion can remember how
version negotiation was performed and validate the initial_version value.

A server that does not maintain state for every packet it receives (i.e., a
stateless server) uses a different process. If the initial_version matches the
version of QUIC that is in use, a stateless server can accept the value.

If the initial_version is different from the version of QUIC that is in use, a
stateless server MUST check that it would have sent a Version Negotiation packet
if it had received a packet with the indicated initial_version.  If a server
would have accepted the version included in the initial_version and the value
differs from the QUIC version that is in use, the server MUST terminate the
connection with a VERSION_NEGOTIATION_ERROR error.

The server includes both the version of QUIC that is in use and a list of the
QUIC versions that the server supports.

The negotiated_version field is the version that is in use.  This MUST be set by
the server to the value that is on the Initial packet that it accepts (not an
Initial packet that triggers a Retry or Version Negotiation packet).  A client
that receives a negotiated_version that does not match the version of QUIC that
is in use MUST terminate the connection with a VERSION_NEGOTIATION_ERROR error
code.

The server includes a list of versions that it would send in any version
negotiation packet ({{packet-version}}) in the supported_versions field.  The
server populates this field even if it did not send a version negotiation
packet.

The client validates that the negotiated_version is included in the
supported_versions list and - if version negotiation was performed - that it
would have selected the negotiated version.  A client MUST terminate the
connection with a VERSION_NEGOTIATION_ERROR error code if the current QUIC
version is not listed in the supported_versions list.  A client MUST terminate
with a VERSION_NEGOTIATION_ERROR error code if version negotiation occurred but
it would have selected a different version based on the value of the
supported_versions list.

When an endpoint accepts multiple QUIC versions, it can potentially interpret
transport parameters as they are defined by any of the QUIC versions it
supports.  The version field in the QUIC packet header is authenticated using
transport parameters.  The position and the format of the version fields in
transport parameters MUST either be identical across different QUIC versions, or
be unambiguously different to ensure no confusion about their interpretation.
One way that a new format could be introduced is to define a TLS extension with
a different codepoint.


## Stateless Retries {#stateless-retry}

A server can process an initial cryptographic handshake messages from a client
without committing any state. This allows a server to perform address validation
({{address-validation}}), or to defer connection establishment costs.

A server that generates a response to an initial packet without retaining
connection state MUST use the Retry packet ({{packet-retry}}).  This packet
causes a client to reset its transport state and to continue the connection
attempt with new connection state while maintaining the state of the
cryptographic handshake.

A server MUST NOT send multiple Retry packets in response to a client handshake
packet.  Thus, any cryptographic handshake message that is sent MUST fit within
a single packet.

In TLS, the Retry packet type is used to carry the HelloRetryRequest message.


## Proof of Source Address Ownership {#address-validation}

Transport protocols commonly spend a round trip checking that a client owns the
transport address (IP and port) that it claims.  Verifying that a client can
receive packets sent to its claimed transport address protects against spoofing
of this information by malicious clients.

This technique is used primarily to avoid QUIC from being used for traffic
amplification attack.  In such an attack, a packet is sent to a server with
spoofed source address information that identifies a victim.  If a server
generates more or larger packets in response to that packet, the attacker can
use the server to send more data toward the victim than it would be able to send
on its own.

Several methods are used in QUIC to mitigate this attack.  Firstly, the initial
handshake packet is padded to at least 1200 octets.  This allows a server to
send a similar amount of data without risking causing an amplification attack
toward an unproven remote address.

A server eventually confirms that a client has received its messages when the
cryptographic handshake successfully completes.  This might be insufficient,
either because the server wishes to avoid the computational cost of completing
the handshake, or it might be that the size of the packets that are sent during
the handshake is too large.  This is especially important for 0-RTT, where the
server might wish to provide application data traffic - such as a response to a
request - in response to the data carried in the early data from the client.

To send additional data prior to completing the cryptographic handshake, the
server then needs to validate that the client owns the address that it claims.

Source address validation is therefore performed during the establishment of a
connection.  TLS provides the tools that support the feature, but basic
validation is performed by the core transport protocol.

A different type of source address validation is performed after a connection
migration, see {{migrate-validate}}.


### Client Address Validation Procedure

QUIC uses token-based address validation.  Any time the server wishes to
validate a client address, it provides the client with a token.  As long as the
token cannot be easily guessed (see {{token-integrity}}), if the client is able
to return that token, it proves to the server that it received the token.

During the processing of the cryptographic handshake messages from a client, TLS
will request that QUIC make a decision about whether to proceed based on the
information it has.  TLS will provide QUIC with any token that was provided by
the client.  For an initial packet, QUIC can decide to abort the connection,
allow it to proceed, or request address validation.

If QUIC decides to request address validation, it provides the cryptographic
handshake with a token.  The contents of this token are consumed by the server
that generates the token, so there is no need for a single well-defined format.
A token could include information about the claimed client address (IP and
port), a timestamp, and any other supplementary information the server will need
to validate the token in the future.

The cryptographic handshake is responsible for enacting validation by sending
the address validation token to the client.  A legitimate client will include a
copy of the token when it attempts to continue the handshake.  The cryptographic
handshake extracts the token then asks QUIC a second time whether the token is
acceptable.  In response, QUIC can either abort the connection or permit it to
proceed.

A connection MAY be accepted without address validation - or with only limited
validation - but a server SHOULD limit the data it sends toward an unvalidated
address.  Successful completion of the cryptographic handshake implicitly
provides proof that the client has received packets from the server.


### Address Validation on Session Resumption

A server MAY provide clients with an address validation token during one
connection that can be used on a subsequent connection.  Address validation is
especially important with 0-RTT because a server potentially sends a significant
amount of data to a client in response to 0-RTT data.

A different type of token is needed when resuming.  Unlike the token that is
created during a handshake, there might be some time between when the token is
created and when the token is subsequently used.  Thus, a resumption token
SHOULD include an expiration time.  It is also unlikely that the client port
number is the same on two different connections; validating the port is
therefore unlikely to be successful.

This token can be provided to the cryptographic handshake immediately after
establishing a connection.  QUIC might also generate an updated token if
significant time passes or the client address changes for any reason (see
{{migration}}).  The cryptographic handshake is responsible for
providing the client with the token.  In TLS the token is included in the ticket
that is used for resumption and 0-RTT, which is carried in a NewSessionTicket
message.


### Address Validation Token Integrity {#token-integrity}

An address validation token MUST be difficult to guess.  Including a large
enough random value in the token would be sufficient, but this depends on the
server remembering the value it sends to clients.

A token-based scheme allows the server to offload any state associated with
validation to the client.  For this design to work, the token MUST be covered by
integrity protection against modification or falsification by clients.  Without
integrity protection, malicious clients could generate or guess values for
tokens that would be accepted by the server.  Only the server requires access to
the integrity protection key for tokens.

In TLS the address validation token is often bundled with the information that
TLS requires, such as the resumption secret.  In this case, adding integrity
protection can be delegated to the cryptographic handshake protocol, avoiding
redundant protection.  If integrity protection is delegated to the cryptographic
handshake, an integrity failure will result in immediate cryptographic handshake
failure.  If integrity protection is performed by QUIC, QUIC MUST abort the
connection if the integrity check fails with a PROTOCOL_VIOLATION error code.


## Path Validation {#migrate-validate}

Path validation is used by an endpoint to verify reachability of a peer over a
specific path.  That is, it tests reachability between a specific local address
and a specific peer address, where an address is the two-tuple of IP address and
port.  Path validation tests that packets can be both sent to and received from
a peer.

Path validation is used during connection migration (see {{migration}} and
{{preferred-address}}) by the migrating endpoint to verify reachability of a
peer from a new local address. Path validation is also used by the peer to
verify that the migrating endpoint is able to receive packets sent to the its
new address.  That is, that the packets received from the migrating endpoint do
not carry a spoofed source address.

Path validation can be used at any time by either endpoint.  For instance, an
endpoint might check that a peer is still in possession of its address after a
period of quiescence.

Path validation is not designed as a NAT traversal mechanism. Though the
mechanism described here might be effective for the creation of NAT bindings
that support NAT traversal, the expectation is that one or other peer is able to
receive packets without first having sent a packet on that path. Effective NAT
traversal needs additional synchronization mechanisms that are not provided
here.

An endpoint MAY bundle PATH_CHALLENGE and PATH_RESPONSE frames that are used for
path validation with other frames.  For instance, an endpoint may pad a packet
carrying a PATH_CHALLENGE for PMTU discovery, or an endpoint may bundle a
PATH_RESPONSE with its own PATH_CHALLENGE.


### Initiation

To initiate path validation, an endpoint sends a PATH_CHALLENGE frame containing
a random payload on the path to be validated.

An endpoint MAY send additional PATH_CHALLENGE frames to handle packet loss.  An
endpoint SHOULD NOT send a PATH_CHALLENGE more frequently than it would an
Initial packet, ensuring that connection migration is no more load on a new path
than establishing a new connection.

The endpoint MUST use fresh random data in every PATH_CHALLENGE frame so that it
can associate the peer's response with the causative PATH_CHALLENGE.


### Response

On receiving a PATH_CHALLENGE frame, an endpoint MUST respond immediately by
echoing the data contained in the PATH_CHALLENGE frame in a PATH_RESPONSE frame,
with the following stipulation.  Since a PATH_CHALLENGE might be sent from a
spoofed address, an endpoint MAY limit the rate at which it sends PATH_RESPONSE
frames and MAY silently discard PATH_CHALLENGE frames that would cause it to
respond at a higher rate.

To ensure that packets can be both sent to and received from the peer, the
PATH_RESPONSE MUST be sent on the same path as the triggering PATH_CHALLENGE:
from the same local address on which the PATH_CHALLENGE was received, to the
same remote address from which the PATH_CHALLENGE was received.


### Completion

A new address is considered valid when a PATH_RESPONSE frame is received
containing data that was sent in a previous PATH_CHALLENGE. Receipt of an
acknowledgment for a packet containing a PATH_CHALLENGE frame is not adequate
validation, since the acknowledgment can be spoofed by a malicious peer.

For path validation to be successful, a PATH_RESPONSE frame MUST be received
from the same remote address to which the corresponding PATH_CHALLENGE was
sent. If a PATH_RESPONSE frame is received from a different remote address than
the one to which the PATH_CHALLENGE was sent, path validation is considered to
have failed, even if the data matches that sent in the PATH_CHALLENGE.

Additionally, the PATH_RESPONSE frame MUST be received on the same local address
from which the corresponding PATH_CHALLENGE was sent.  If a PATH_RESPONSE frame
is received on a different local address than the one from which the
PATH_CHALLENGE was sent, path validation is considered to have failed, even if
the data matches that sent in the PATH_CHALLENGE.  Thus, the endpoint considers
the path to be valid when a PATH_RESPONSE frame is received on the same path
with the same payload as the PATH_CHALLENGE frame.


### Abandonment

An endpoint SHOULD abandon path validation after sending some number of
PATH_CHALLENGE frames or after some time has passed.  When setting this timer,
implementations are cautioned that the new path could have a longer round-trip
time than the original.

Note that the endpoint might receive packets containing other frames on the new
path, but a PATH_RESPONSE frame with appropriate data is required for path
validation to succeed.

If path validation fails, the path is deemed unusable.  This does not
necessarily imply a failure of the connection - endpoints can continue sending
packets over other paths as appropriate.  If no paths are available, an endpoint
can wait for a new path to become available or close the connection.

A path validation might be abandoned for other reasons besides
failure. Primarily, this happens if a connection migration to a new path is
initiated while a path validation on the old path is in progress.


## Connection Migration {#migration}

QUIC allows connections to survive changes to endpoint addresses (that is, IP
address and/or port), such as those caused by a endpoint migrating to a new
network.  This section describes the process by which an endpoint migrates to a
new address.

An endpoint MUST NOT initiate connection migration before the handshake is
finished and the endpoint has 1-RTT keys.

This document limits migration of connections to new client addresses, except as
described in {{preferred-address}}. Clients are responsible for initiating all
migrations.  Servers do not send non-probing packets (see {{probing}}) toward a
client address until it sees a non-probing packet from that address.  If a
client receives packets from an unknown server address, the client MAY discard
these packets.


### Probing a New Path {#probing}

An endpoint MAY probe for peer reachability from a new local address using path
validation {{migrate-validate}} prior to migrating the connection to the new
local address.  Failure of path validation simply means that the new path is not
usable for this connection.  Failure to validate a path does not cause the
connection to end unless there are no valid alternative paths available.

An endpoint uses a new connection ID for probes sent from a new local address,
see {{migration-linkability}} for further discussion.

Receiving a PATH_CHALLENGE frame from a peer indicates that the peer is probing
for reachability on a path. An endpoint sends a PATH_RESPONSE in response as per
{{migrate-validate}}.

PATH_CHALLENGE, PATH_RESPONSE, and PADDING frames are "probing frames", and all
other frames are "non-probing frames".  A packet containing only probing frames
is a "probing packet", and a packet containing any other frame is a "non-probing
packet".


### Initiating Connection Migration {#initiating-migration}

A endpoint can migrate a connection to a new local address by sending packets
containing frames other than probing frames from that address.

Each endpoint validates its peer's address during connection establishment.
Therefore, a migrating endpoint can send to its peer knowing that the peer is
willing to receive at the peer's current address. Thus an endpoint can migrate
to a new local address without first validating the peer's address.

When migrating, the new path might not support the endpoint's current sending
rate. Therefore, the endpoint resets its congestion controller, as described in
{{migration-cc}}.

Receiving acknowledgments for data sent on the new path serves as proof of the
peer's reachability from the new address.  Note that since acknowledgments may
be received on any path, return reachability on the new path is not
established. To establish return reachability on the new path, an endpoint MAY
concurrently initiate path validation {{migrate-validate}} on the new path.


### Responding to Connection Migration {#migration-response}

Receiving a packet from a new peer address containing a non-probing frame
indicates that the peer has migrated to that address.

In response to such a packet, an endpoint MUST start sending subsequent packets
to the new peer address and MUST initiate path validation ({{migrate-validate}})
to verify the peer's ownership of the unvalidated address.

An endpoint MAY send data to an unvalidated peer address, but it MUST protect
against potential attacks as described in {{address-spoofing}} and
{{on-path-spoofing}}.  An endpoint MAY skip validation of a peer address if that
address has been seen recently.

An endpoint only changes the address that it sends packets to in response to the
highest-numbered non-probing packet. This ensures that an endpoint does not send
packets to an old peer address in the case that it receives reordered packets.

After changing the address to which it sends non-probing packets, an endpoint
could abandon any path validation for other addresses.

Receiving a packet from a new peer address might be the result of a NAT
rebinding at the peer.

After verifying a new client address, the server SHOULD send new address
validation tokens ({{address-validation}}) to the client.


#### Handling Address Spoofing by a Peer {#address-spoofing}

It is possible that a peer is spoofing its source address to cause an endpoint
to send excessive amounts of data to an unwilling host.  If the endpoint sends
significantly more data than the spoofing peer, connection migration might be
used to amplify the volume of data that an attacker can generate toward a
victim.

As described in {{migration-response}}, an endpoint is required to validate a
peer's new address to confirm the peer's possession of the new address.  Until a
peer's address is deemed valid, an endpoint MUST limit the rate at which it
sends data to this address.  The endpoint MUST NOT send more than a minimum
congestion window's worth of data per estimated round-trip time (kMinimumWindow,
as defined in {{QUIC-RECOVERY}}).  In the absence of this limit, an endpoint
risks being used for a denial of service attack against an unsuspecting victim.
Note that since the endpoint will not have any round-trip time measurements to
this address, the estimate SHOULD be the default initial value (see
{{QUIC-RECOVERY}}).

If an endpoint skips validation of a peer address as described in
{{migration-response}}, it does not need to limit its sending rate.


#### Handling Address Spoofing by an On-path Attacker {#on-path-spoofing}

An on-path attacker could cause a spurious connection migration by copying and
forwarding a packet with a spoofed address such that it arrives before the
original packet.  The packet with the spoofed address will be seen to come from
a migrating connection, and the original packet will be seen as a duplicate and
dropped. After a spurious migration, validation of the source address will fail
because the entity at the source address does not have the necessary
cryptographic keys to read or respond to the PATH_CHALLENGE frame that is sent
to it even if it wanted to.

To protect the connection from failing due to such a spurious migration, an
endpoint MUST revert to using the last validated peer address when validation of
a new peer address fails.

If an endpoint has no state about the last validated peer address, it MUST close
the connection silently by discarding all connection state. This results in new
packets on the connection being handled generically. For instance, an endpoint
MAY send a stateless reset in response to any further incoming packets.

Note that receipt of packets with higher packet numbers from the legitimate peer
address will trigger another connection migration.  This will cause the
validation of the address of the spurious migration to be abandoned.


### Loss Detection and Congestion Control {#migration-cc}

The capacity available on the new path might not be the same as the old path.
Packets sent on the old path SHOULD NOT contribute to congestion control or RTT
estimation for the new path.

On confirming a peer's ownership of its new address, an endpoint SHOULD
immediately reset the congestion controller and round-trip time estimator for
the new path.

An endpoint MUST NOT return to the send rate used for the previous path unless
it is reasonably sure that the previous send rate is valid for the new path.
For instance, a change in the client's port number is likely indicative of a
rebinding in a middlebox and not a complete change in path.  This determination
likely depends on heuristics, which could be imperfect; if the new path capacity
is significantly reduced, ultimately this relies on the congestion controller
responding to congestion signals and reducing send rates appropriately.

There may be apparent reordering at the receiver when an endpoint sends data and
probes from/to multiple addresses during the migration period, since the two
resulting paths may have different round-trip times.  A receiver of packets on
multiple paths will still send ACK frames covering all received packets.

While multiple paths might be used during connection migration, a single
congestion control context and a single loss recovery context (as described in
{{QUIC-RECOVERY}}) may be adequate.  A sender can make exceptions for probe
packets so that their loss detection is independent and does not unduly cause
the congestion controller to reduce its sending rate.  An endpoint might arm a
separate alarm when a PATH_CHALLENGE is sent, which is disarmed when the
corresponding PATH_RESPONSE is received.  If the alarm fires before the
PATH_RESPONSE is received, the endpoint might send a new PATH_CHALLENGE, and
restart the alarm for a longer period of time.


### Privacy Implications of Connection Migration {#migration-linkability}

Using a stable connection ID on multiple network paths allows a passive observer
to correlate activity between those paths.  An endpoint that moves between
networks might not wish to have their activity correlated by any entity other
than their peer. The NEW_CONNECTION_ID message can be sent to provide an
unlinkable connection ID for use in case a peer wishes to explicitly break
linkability between two points of network attachment.

An endpoint that does not require the use of a connection ID should not request
that its peer use a connection ID.  Such an endpoint does not need to provide
new connection IDs using the NEW_CONNECTION_ID frame.

An endpoint might need to send packets on multiple networks without receiving
any response from its peer.  To ensure that the endpoint is not linkable across
each of these changes, a new connection ID is needed for each network.  To
support this, multiple NEW_CONNECTION_ID messages are needed.

Upon changing networks an endpoint MUST use a previously unused connection ID
provided by its peer.  This eliminates the use of the connection ID for linking
activity from the same connection on different networks.  Protection of packet
numbers ensures that packet numbers cannot be used to correlate activity.  This
does not prevent other properties of packets, such as timing and size, from
being used to correlate activity.

Clients MAY change connection ID at any time based on implementation-specific
concerns.  For example, after a period of network inactivity NAT rebinding might
occur when the client begins sending data again.

A client might wish to reduce linkability by employing a new connection ID and
source UDP port when sending traffic after a period of inactivity.  Changing the
UDP port from which it sends packets at the same time might cause the packet to
appear as a connection migration. This ensures that the mechanisms that support
migration are exercised even for clients that don't experience NAT rebindings or
genuine migrations.  Changing port number can cause a peer to reset its
congestion state (see {{migration-cc}}), so the port SHOULD only be changed
infrequently.

An endpoint that receives a successfully authenticated packet with a previously
unused connection ID MUST use the next available connection ID for any packets
it sends to that address.  To avoid changing connection IDs multiple times when
packets arrive out of order, endpoints MUST change only in response to a packet
that increases the largest received packet number.  Failing to do this could
allow for use of that connection ID to link activity on new paths.  There is no
need to move to a new connection ID if the address of a peer changes without
also changing the connection ID.


## Server's Preferred Address {#preferred-address}

QUIC allows servers to accept connections on one IP address and attempt to
transfer these connections to a more preferred address shortly after the
handshake.  This is particularly useful when clients initially connect to an
address shared by multiple servers but would prefer to use a unicast address to
ensure connection stability. This section describes the protocol for migrating a
connection to a preferred server address.

Migrating a connection to a new server address mid-connection is left for future
work. If a client receives packets from a new server address not indicated by
the preferred_address transport parameter, the client SHOULD discard these
packets.

### Communicating A Preferred Address

A server conveys a preferred address by including the preferred_address
transport parameter in the TLS handshake.

Once the handshake is finished, the client SHOULD initiate path validation (see
{{migrate-validate}}) of the server's preferred address using the connection ID
provided in the preferred_address transport parameter.

If path validation succeeds, the client SHOULD immediately begin sending all
future packets to the new server address using the new connection ID and
discontinue use of the old server address.  If path validation fails, the client
MUST continue sending all future packets to the server's original IP address.


### Responding to Connection Migration

A server might receive a packet addressed to its preferred IP address at any
time after the handshake is completed.  If this packet contains a PATH_CHALLENGE
frame, the server sends a PATH_RESPONSE frame as per {{migrate-validate}}, but
the server MUST continue sending all other packets from its original IP address.

The server SHOULD also initiate path validation of the client using its
preferred address and the address from which it received the client probe.  This
helps to guard against spurious migration initiated by an attacker.

Once the server has completed its path validation and has received a non-probing
packet with a new largest packet number on its preferred address, the server
begins sending to the client exclusively from its preferred IP address.  It
SHOULD drop packets for this connection received on the old IP address, but MAY
continue to process delayed packets.


### Interaction of Client Migration and Preferred Address

A client might need to perform a connection migration before it has migrated to
the server's preferred address.  In this case, the client SHOULD perform path
validation to both the original and preferred server address from the client's
new address concurrently.

If path validation of the server's preferred address succeeds, the client MUST
abandon validation of the original address and migrate to using the server's
preferred address.  If path validation of the server's preferred address fails,
but validation of the server's original address succeeds, the client MAY migrate
to using the original address from the client's new address.

If the connection to the server's preferred address is not from the same client
address, the server MUST protect against potential attacks as described in
{{address-spoofing}} and {{on-path-spoofing}}.  In addition to intentional
simultaneous migration, this might also occur because the client's access
network used a different NAT binding for the server's preferred address.

Servers SHOULD initiate path validation to the client's new address upon
receiving a probe packet from a different address.  Servers MUST NOT send more
than a minimum congestion window's worth of non-probing packets to the new
address before path validation is complete.


## Connection Termination {#termination}

Connections should remain open until they become idle for a pre-negotiated
period of time.  A QUIC connection, once established, can be terminated in one
of three ways:

* idle timeout ({{idle-timeout}})
* immediate close ({{immediate-close}})
* stateless reset ({{stateless-reset}})


### Closing and Draining Connection States {#draining}

The closing and draining connection states exist to ensure that connections
close cleanly and that delayed or reordered packets are properly discarded.
These states SHOULD persist for three times the current Retransmission Timeout
(RTO) interval as defined in {{QUIC-RECOVERY}}.

An endpoint enters a closing period after initiating an immediate close
({{immediate-close}}).  While closing, an endpoint MUST NOT send packets unless
they contain a CONNECTION_CLOSE or APPLICATION_CLOSE frame (see
{{immediate-close}} for details).

In the closing state, only a packet containing a closing frame can be sent.  An
endpoint retains only enough information to generate a packet containing a
closing frame and to identify packets as belonging to the connection.  The
connection ID and QUIC version is sufficient information to identify packets for
a closing connection; an endpoint can discard all other connection state.  An
endpoint MAY retain packet protection keys for incoming packets to allow it to
read and process a closing frame.

The draining state is entered once an endpoint receives a signal that its peer
is closing or draining.  While otherwise identical to the closing state, an
endpoint in the draining state MUST NOT send any packets.  Retaining packet
protection keys is unnecessary once a connection is in the draining state.

An endpoint MAY transition from the closing period to the draining period if it
can confirm that its peer is also closing or draining.  Receiving a closing
frame is sufficient confirmation, as is receiving a stateless reset.  The
draining period SHOULD end when the closing period would have ended.  In other
words, the endpoint can use the same end time, but cease retransmission of the
closing packet.

Disposing of connection state prior to the end of the closing or draining period
could cause delayed or reordered packets to be handled poorly.  Endpoints that
have some alternative means to ensure that late-arriving packets on the
connection do not create QUIC state, such as those that are able to close the
UDP socket, MAY use an abbreviated draining period which can allow for faster
resource recovery.  Servers that retain an open socket for accepting new
connections SHOULD NOT exit the closing or draining period early.

Once the closing or draining period has ended, an endpoint SHOULD discard all
connection state.  This results in new packets on the connection being handled
generically.  For instance, an endpoint MAY send a stateless reset in response
to any further incoming packets.

The draining and closing periods do not apply when a stateless reset
({{stateless-reset}}) is sent.

An endpoint is not expected to handle key updates when it is closing or
draining.  A key update might prevent the endpoint from moving from the closing
state to draining, but it otherwise has no impact.

An endpoint could receive packets from a new source address, indicating a client
connection migration ({{migration}}), while in the closing period. An endpoint
in the closing state MUST strictly limit the number of packets it sends to this
new address until the address is validated (see {{migrate-validate}}). A server
in the closing state MAY instead choose to discard packets received from a new
source address.


### Idle Timeout

A connection that remains idle for longer than the idle timeout (see
{{transport-parameter-definitions}}) is closed.  A connection enters the
draining state when the idle timeout expires.

The time at which an idle timeout takes effect won't be perfectly synchronized
on both endpoints.  An endpoint that sends packets near the end of an idle
period could have those packets discarded if its peer enters the draining state
before the packet is received.


### Immediate Close

An endpoint sends a closing frame, either CONNECTION_CLOSE or APPLICATION_CLOSE,
to terminate the connection immediately.  Either closing frame causes all
streams to immediately become closed; open streams can be assumed to be
implicitly reset.

After sending a closing frame, endpoints immediately enter the closing state.
During the closing period, an endpoint that sends a closing frame SHOULD respond
to any packet that it receives with another packet containing a closing frame.
To minimize the state that an endpoint maintains for a closing connection,
endpoints MAY send the exact same packet.  However, endpoints SHOULD limit the
number of packets they generate containing a closing frame.  For instance, an
endpoint could progressively increase the number of packets that it receives
before sending additional packets or increase the time between packets.

Note:

: Allowing retransmission of a packet contradicts other advice in this document
  that recommends the creation of new packet numbers for every packet.  Sending
  new packet numbers is primarily of advantage to loss recovery and congestion
  control, which are not expected to be relevant for a closed connection.
  Retransmitting the final packet requires less state.

After receiving a closing frame, endpoints enter the draining state.  An
endpoint that receives a closing frame MAY send a single packet containing a
closing frame before entering the draining state, using a CONNECTION_CLOSE frame
and a NO_ERROR code if appropriate.  An endpoint MUST NOT send further packets,
which could result in a constant exchange of closing frames until the closing
period on either peer ended.

An immediate close can be used after an application protocol has arranged to
close a connection.  This might be after the application protocols negotiates a
graceful shutdown.  The application protocol exchanges whatever messages that
are needed to cause both endpoints to agree to close the connection, after which
the application requests that the connection be closed.  The application
protocol can use an APPLICATION_CLOSE message with an appropriate error code to
signal closure.


### Stateless Reset {#stateless-reset}

A stateless reset is provided as an option of last resort for an endpoint that
does not have access to the state of a connection.  A crash or outage might
result in peers continuing to send data to an endpoint that is unable to
properly continue the connection.  An endpoint that wishes to communicate a
fatal connection error MUST use a closing frame if it has sufficient state to do
so.

To support this process, a token is sent by endpoints.  The token is carried in
the NEW_CONNECTION_ID frame sent by either peer, and servers can specify the
stateless_reset_token transport parameter during the handshake (clients cannot
because their transport parameters don't have confidentiality protection).  This
value is protected by encryption, so only client and server know this value.

An endpoint that receives packets that it cannot process sends a packet in the
following layout:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
|0|K|1|1|0|0|0|0|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Random Octets (160..)                  ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                   Stateless Reset Token (128)                 +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

This design ensures that a stateless reset packet is - to the extent possible -
indistinguishable from a regular packet with a short header.

The message consists of a header octet, followed by random octets of arbitrary
length, followed by a Stateless Reset Token.

A stateless reset will be interpreted by a recipient as a packet with a short
header.  For the packet to appear as valid, the Random Octets field needs to
include at least 20 octets of random or unpredictable values.  This is intended
to allow for a destination connection ID of the maximum length permitted, a
packet number, and minimal payload.  The Stateless Reset Token corresponds to
the minimum expansion of the packet protection AEAD.  More random octets might
be necessary if the endpoint could have negotiated a packet protection scheme
with a larger minimum AEAD expansion.

An endpoint SHOULD NOT send a stateless reset that is significantly larger than
the packet it receives.  Endpoints MUST discard packets that are too small to be
valid QUIC packets.  With the set of AEAD functions defined in {{QUIC-TLS}},
packets less than 19 octets long are never valid.

An endpoint cannot determine the Source Connection ID from a packet with a short
header, therefore it cannot set the Destination Connection ID in the stateless
reset packet.  The destination connection ID will therefore differ from the
value used in previous packets.  A random Destination Connection ID makes the
connection ID appear to be the result of moving to a new connection ID that was
provided using a NEW_CONNECTION_ID frame ({{frame-new-connection-id}}).

Using a randomized connection ID results in two problems:

* The packet might not reach the peer.  If the Destination Connection ID is
  critical for routing toward the peer, then this packet could be incorrectly
  routed.  This causes the stateless reset to be ineffective in causing errors
  to be quickly detected and recovered.  In this case, endpoints will need to
  rely on other methods - such as timers - to detect that the connection has
  failed.

* The randomly generated connection ID can be used by entities other than the
  peer to identify this as a potential stateless reset.  An endpoint that
  occasionally uses different connection IDs might introduce some uncertainty
  about this.

Finally, the last 16 octets of the packet are set to the value of the Stateless
Reset Token.

A stateless reset is not appropriate for signaling error conditions.  An
endpoint that wishes to communicate a fatal connection error MUST use a
CONNECTION_CLOSE or APPLICATION_CLOSE frame if it has sufficient state to do so.

This stateless reset design is specific to QUIC version 1.  An endpoint that
supports multiple versions of QUIC needs to generate a stateless reset that will
be accepted by peers that support any version that the endpoint might support
(or might have supported prior to losing state).  Designers of new versions of
QUIC need to be aware of this and either reuse this design, or use a portion of
the packet other than the last 16 octets for carrying data.


#### Detecting a Stateless Reset

An endpoint detects a potential stateless reset when a packet with a short
header either cannot be decrypted or is marked as a duplicate packet.  The
endpoint then compares the last 16 octets of the packet with the Stateless Reset
Token provided by its peer, either in a NEW_CONNECTION_ID frame or the server's
transport parameters.  If these values are identical, the endpoint MUST enter
the draining period and not send any further packets on this connection.  If the
comparison fails, the packet can be discarded.


#### Calculating a Stateless Reset Token

The stateless reset token MUST be difficult to guess.  In order to create a
Stateless Reset Token, an endpoint could randomly generate {{!RFC4086}} a secret
for every connection that it creates.  However, this presents a coordination
problem when there are multiple instances in a cluster or a storage problem for
a endpoint that might lose state.  Stateless reset specifically exists to handle
the case where state is lost, so this approach is suboptimal.

A single static key can be used across all connections to the same endpoint by
generating the proof using a second iteration of a preimage-resistant function
that takes three inputs: the static key, the connection ID chosen by the
endpoint (see {{connection-id}}), and an instance identifier.  An endpoint could
use HMAC {{?RFC2104}} (for example, HMAC(static_key, instance_id ||
connection_id)) or HKDF {{?RFC5869}} (for example, using the static key as input
keying material, with instance and connection identifiers as salt).  The output
of this function is truncated to 16 octets to produce the Stateless Reset Token
for that connection.

An endpoint that loses state can use the same method to generate a valid
Stateless Reset Token.  The connection ID comes from the packet that the
endpoint receives.  An instance that receives a packet for another instance
might be able to recover the instance identifier using the connection ID.
Alternatively, the instance identifier might be omitted from the calculation of
the Stateless Reset Token so that all instances are equally able to generate a
stateless reset.

This design relies on the peer always sending a connection ID in its packets so
that the endpoint can use the connection ID from a packet to reset the
connection.  An endpoint that uses this design cannot allow its peers to send
packets with a zero-length destination connection ID.

Revealing the Stateless Reset Token allows any entity to terminate the
connection, so a value can only be used once.  This method for choosing the
Stateless Reset Token means that the combination of instance, connection ID, and
static key cannot occur for another connection.  A connection ID from a
connection that is reset by revealing the Stateless Reset Token cannot be reused
for new connections at the same instance without first changing to use a
different static key or instance identifier.

Note that Stateless Reset messages do not have any cryptographic protection.


# Frame Types and Formats

As described in {{frames}}, packets contain one or more frames. This section
describes the format and semantics of the core QUIC frame types.


## Variable-Length Integer Encoding {#integer-encoding}

QUIC frames commonly use a variable-length encoding for non-negative integer
values.  This encoding ensures that smaller integer values need fewer octets to
encode.

The QUIC variable-length integer encoding reserves the two most significant bits
of the first octet to encode the base 2 logarithm of the integer encoding length
in octets.  The integer value is encoded on the remaining bits, in network byte
order.

This means that integers are encoded on 1, 2, 4, or 8 octets and can encode 6,
14, 30, or 62 bit values respectively.  {{integer-summary}} summarizes the
encoding properties.

| 2Bit | Length | Usable Bits | Range                 |
|:-----|:-------|:------------|:----------------------|
| 00   | 1      | 6           | 0-63                  |
| 01   | 2      | 14          | 0-16383               |
| 10   | 4      | 30          | 0-1073741823          |
| 11   | 8      | 62          | 0-4611686018427387903 |
{: #integer-summary title="Summary of Integer Encodings"}

For example, the eight octet sequence c2 19 7c 5e ff 14 e8 8c (in hexadecimal)
decodes to the decimal value 151288809941952652; the four octet sequence 9d 7f
3e 7d decodes to 494878333; the two octet sequence 7b bd decodes to 15293; and
the single octet 25 decodes to 37 (as does the two octet sequence 40 25).

Error codes ({{error-codes}}) are described using integers, but do not use this
encoding.


## PADDING Frame {#frame-padding}

The PADDING frame (type=0x00) has no semantic value.  PADDING frames can be used
to increase the size of a packet.  Padding can be used to increase an initial
client packet to the minimum required size, or to provide protection against
traffic analysis for protected packets.

A PADDING frame has no content.  That is, a PADDING frame consists of the single
octet that identifies the frame as a PADDING frame.


## RST_STREAM Frame {#frame-rst-stream}

An endpoint may use a RST_STREAM frame (type=0x01) to abruptly terminate a
stream.

After sending a RST_STREAM, an endpoint ceases transmission and retransmission
of STREAM frames on the identified stream.  A receiver of RST_STREAM can discard
any data that it already received on that stream.

An endpoint that receives a RST_STREAM frame for a send-only stream MUST
terminate the connection with error PROTOCOL_VIOLATION.

The RST_STREAM frame is as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Stream ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Application Error Code (16)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Final Offset (i)                     ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The fields are:

Stream ID:

: A variable-length integer encoding of the Stream ID of the stream being
  terminated.

Application Protocol Error Code:

: A 16-bit application protocol error code (see {{app-error-codes}}) which
  indicates why the stream is being closed.

Final Offset:

: A variable-length integer indicating the absolute byte offset of the end of
  data written on this stream by the RST_STREAM sender.


## CONNECTION_CLOSE frame {#frame-connection-close}

An endpoint sends a CONNECTION_CLOSE frame (type=0x02) to notify its peer that
the connection is being closed.  CONNECTION_CLOSE is used to signal errors at
the QUIC layer, or the absence of errors (with the NO_ERROR code).

If there are open streams that haven't been explicitly closed, they are
implicitly closed when the connection is closed.

The CONNECTION_CLOSE frame is as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Error Code (16)     |   Reason Phrase Length (i)  ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Reason Phrase (*)                    ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The fields of a CONNECTION_CLOSE frame are as follows:

Error Code:

: A 16-bit error code which indicates the reason for closing this connection.
  CONNECTION_CLOSE uses codes from the space defined in {{error-codes}}
  (APPLICATION_CLOSE uses codes from the application protocol error code space,
  see {{app-error-codes}}).

Reason Phrase Length:

: A variable-length integer specifying the length of the reason phrase in bytes.
  Note that a CONNECTION_CLOSE frame cannot be split between packets, so in
  practice any limits on packet size will also limit the space available for a
  reason phrase.

Reason Phrase:

: A human-readable explanation for why the connection was closed.  This can be
  zero length if the sender chooses to not give details beyond the Error Code.
  This SHOULD be a UTF-8 encoded string {{!RFC3629}}.


## APPLICATION_CLOSE frame {#frame-application-close}

An APPLICATION_CLOSE frame (type=0x03) uses the same format as the
CONNECTION_CLOSE frame ({{frame-connection-close}}), except that it uses error
codes from the application protocol error code space ({{app-error-codes}})
instead of the transport error code space.

Other than the error code space, the format and semantics of the
APPLICATION_CLOSE frame are identical to the CONNECTION_CLOSE frame.


## MAX_DATA Frame {#frame-max-data}

The MAX_DATA frame (type=0x04) is used in flow control to inform the peer of
the maximum amount of data that can be sent on the connection as a whole.

The frame is as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Maximum Data (i)                     ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The fields in the MAX_DATA frame are as follows:

Maximum Data:

: A variable-length integer indicating the maximum amount of data that can be
  sent on the entire connection, in units of octets.

All data sent in STREAM frames counts toward this limit, with the exception of
data on stream 0.  The sum of the largest received offsets on all streams -
including streams in terminal states, but excluding stream 0 - MUST NOT exceed
the value advertised by a receiver.  An endpoint MUST terminate a connection
with a QUIC_FLOW_CONTROL_RECEIVED_TOO_MUCH_DATA error if it receives more data
than the maximum data value that it has sent, unless this is a result of a
change in the initial limits (see {{zerortt-parameters}}).


## MAX_STREAM_DATA Frame {#frame-max-stream-data}

The MAX_STREAM_DATA frame (type=0x05) is used in flow control to inform a peer
of the maximum amount of data that can be sent on a stream.

An endpoint that receives a MAX_STREAM_DATA frame for a receive-only stream
MUST terminate the connection with error PROTOCOL_VIOLATION.

An endpoint that receives a MAX_STREAM_DATA frame for a send-only stream
it has not opened MUST terminate the connection with error PROTOCOL_VIOLATION.

Note that an endpoint may legally receive a MAX_STREAM_DATA frame on a
bidirectional stream it has not opened.

The frame is as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Stream ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Maximum Stream Data (i)                  ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The fields in the MAX_STREAM_DATA frame are as follows:

Stream ID:

: The stream ID of the stream that is affected encoded as a variable-length
  integer.

Maximum Stream Data:

: A variable-length integer indicating the maximum amount of data that can be
  sent on the identified stream, in units of octets.

When counting data toward this limit, an endpoint accounts for the largest
received offset of data that is sent or received on the stream.  Loss or
reordering can mean that the largest received offset on a stream can be greater
than the total size of data received on that stream.  Receiving STREAM frames
might not increase the largest received offset.

The data sent on a stream MUST NOT exceed the largest maximum stream data value
advertised by the receiver.  An endpoint MUST terminate a connection with a
FLOW_CONTROL_ERROR error if it receives more data than the largest maximum
stream data that it has sent for the affected stream, unless this is a result of
a change in the initial limits (see {{zerortt-parameters}}).


## MAX_STREAM_ID Frame {#frame-max-stream-id}

The MAX_STREAM_ID frame (type=0x06) informs the peer of the maximum stream ID
that they are permitted to open.

The frame is as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Maximum Stream ID (i)                    ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The fields in the MAX_STREAM_ID frame are as follows:

Maximum Stream ID:
: ID of the maximum unidirectional or bidirectional peer-initiated stream ID for
  the connection encoded as a variable-length integer. The limit applies to
  unidirectional steams if the second least signification bit of the stream ID
  is 1, and applies to bidirectional streams if it is 0.

Loss or reordering can mean that a MAX_STREAM_ID frame can be received which
states a lower stream limit than the client has previously received.
MAX_STREAM_ID frames which do not increase the maximum stream ID MUST be
ignored.

A peer MUST NOT initiate a stream with a higher stream ID than the greatest
maximum stream ID it has received.  An endpoint MUST terminate a connection with
a STREAM_ID_ERROR error if a peer initiates a stream with a higher stream ID
than it has sent, unless this is a result of a change in the initial limits (see
{{zerortt-parameters}}).


## PING Frame {#frame-ping}

Endpoints can use PING frames (type=0x07) to verify that their peers are still
alive or to check reachability to the peer. The PING frame contains no
additional fields.

The receiver of a PING frame simply needs to acknowledge the packet containing
this frame.

The PING frame can be used to keep a connection alive when an application or
application protocol wishes to prevent the connection from timing out. An
application protocol SHOULD provide guidance about the conditions under which
generating a PING is recommended.  This guidance SHOULD indicate whether it is
the client or the server that is expected to send the PING.  Having both
endpoints send PING frames without coordination can produce an excessive number
of packets and poor performance.

A connection will time out if no packets are sent or received for a period
longer than the time specified in the idle_timeout transport parameter (see
{{termination}}).  However, state in middleboxes might time out earlier than
that.  Though REQ-5 in {{?RFC4787}} recommends a 2 minute timeout interval,
experience shows that sending packets every 15 to 30 seconds is necessary to
prevent the majority of middleboxes from losing state for UDP flows.


## BLOCKED Frame {#frame-blocked}

A sender SHOULD send a BLOCKED frame (type=0x08) when it wishes to send data,
but is unable to due to connection-level flow control (see {{blocking}}).
BLOCKED frames can be used as input to tuning of flow control algorithms (see
{{fc-credit}}).

The BLOCKED frame is as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Offset (i)                         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The BLOCKED frame contains a single field.

Offset:

: A variable-length integer indicating the connection-level offset at which
  the blocking occurred.


## STREAM_BLOCKED Frame {#frame-stream-blocked}

A sender SHOULD send a STREAM_BLOCKED frame (type=0x09) when it wishes to send
data, but is unable to due to stream-level flow control.  This frame is
analogous to BLOCKED ({{frame-blocked}}).

An endpoint that receives a STREAM_BLOCKED frame for a send-only stream MUST
terminate the connection with error PROTOCOL_VIOLATION.

The STREAM_BLOCKED frame is as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Stream ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Offset (i)                          ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The STREAM_BLOCKED frame contains two fields:

Stream ID:

: A variable-length integer indicating the stream which is flow control blocked.

Offset:

: A variable-length integer indicating the offset of the stream at which the
  blocking occurred.


## STREAM_ID_BLOCKED Frame {#frame-stream-id-blocked}

A sender MAY send a STREAM_ID_BLOCKED frame (type=0x0a) when it wishes to open a
stream, but is unable to due to the maximum stream ID limit set by its peer (see
{{frame-max-stream-id}}).  This does not open the stream, but informs the peer
that a new stream was needed, but the stream limit prevented the creation of the
stream.

The STREAM_ID_BLOCKED frame is as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Stream ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The STREAM_ID_BLOCKED frame contains a single field.

Stream ID:

: A variable-length integer indicating the highest stream ID that the sender
  was permitted to open.

## NEW_CONNECTION_ID Frame {#frame-new-connection-id}

An endpoint sends a NEW_CONNECTION_ID frame (type=0x0b) to provide its peer with
alternative connection IDs that can be used to break linkability when migrating
connections (see {{migration-linkability}}).

The NEW_CONNECTION_ID is as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Length (8)  |          Connection ID (32..144)            ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                   Stateless Reset Token (128)                 +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The fields are:

Length:

: An 8-bit unsigned integer containing the length of the connection ID.  Values
  less than 4 and greater than 18 are invalid and MUST be treated as a
  connection error of type PROTOCOL_VIOLATION.

Connection ID:

: A connection ID of the specified length.

Stateless Reset Token:

: A 128-bit value that will be used to for a stateless reset when the associated
  connection ID is used (see {{stateless-reset}}).

An endpoint MUST NOT send this frame if it currently requires that its peer send
packets with a zero-length Destination Connection ID.  Changing the length of a
connection ID to or from zero-length makes it difficult to identify when the
value of the connection ID changed.  An endpoint that is sending packets with a
zero-length Destination Connection ID MUST treat receipt of a NEW_CONNECTION_ID
frame as a connection error of type PROTOCOL_VIOLATION.


## STOP_SENDING Frame {#frame-stop-sending}

An endpoint may use a STOP_SENDING frame (type=0x0c) to communicate that
incoming data is being discarded on receipt at application request.  This
signals a peer to abruptly terminate transmission on a stream.

Receipt of a STOP_SENDING frame is only valid for a send stream that exists and
is not in the "Ready" state (see {{stream-send-states}}).  Receiving a
STOP_SENDING frame for a send stream that is "Ready" or non-existent MUST be
treated as a connection error of type PROTOCOL_VIOLATION.  An endpoint that
receives a STOP_SENDING frame for a receive-only stream MUST terminate the
connection with error PROTOCOL_VIOLATION.

The STOP_SENDING frame is as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Stream ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Application Error Code (16)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The fields are:

Stream ID:

: A variable-length integer carrying the Stream ID of the stream being ignored.

Application Error Code:

: A 16-bit, application-specified reason the sender is ignoring the stream (see
  {{app-error-codes}}).


## ACK Frame {#frame-ack}

Receivers send ACK frames (type=0x0d) to inform senders which packets they have
received and processed. The ACK frame contains any number of ACK blocks.
ACK blocks are ranges of acknowledged packets.

QUIC acknowledgements are irrevocable.  Once acknowledged, a packet remains
acknowledged, even if it does not appear in a future ACK frame.  This is unlike
TCP SACKs ({{?RFC2018}}).

A client MUST NOT acknowledge Retry packets.  Retry packets include the packet
number from the Initial packet it responds to.  Version Negotiation packets
cannot be acknowledged because they do not contain a packet number.  Rather than
relying on ACK frames, these packets are implicitly acknowledged by the next
Initial packet sent by the client.

An ACK frame is shown below.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Largest Acknowledged (i)                ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          ACK Delay (i)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       ACK Block Count (i)                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          ACK Blocks (*)                     ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #ack-format title="ACK Frame Format"}

The fields in the ACK frame are as follows:

Largest Acknowledged:

: A variable-length integer representing the largest packet number the peer is
  acknowledging; this is usually the largest packet number that the peer has
  received prior to generating the ACK frame.  Unlike the packet number in the
  QUIC long or short header, the value in an ACK frame is not truncated.

ACK Delay:

: A variable-length integer including the time in microseconds that the largest
  acknowledged packet, as indicated in the Largest Acknowledged field, was
  received by this peer to when this ACK was sent.  The value of the ACK Delay
  field is scaled by multiplying the encoded value by the 2 to the power of the
  value of the `ack_delay_exponent` transport parameter set by the sender of the
  ACK frame.  The `ack_delay_exponent` defaults to 3, or a multiplier of 8 (see
  {{transport-parameter-definitions}}).  Scaling in this fashion allows for a
  larger range of values with a shorter encoding at the cost of lower
  resolution.

ACK Block Count:

: The number of Additional ACK Block (and Gap) fields after the First ACK Block.

ACK Blocks:

: Contains one or more blocks of packet numbers which have been successfully
  received, see {{ack-block-section}}.


### ACK Block Section {#ack-block-section}

The ACK Block Section consists of alternating Gap and ACK Block fields in
descending packet number order.  A First Ack Block field is followed by a
variable number of alternating Gap and Additional ACK Blocks.  The number of Gap
and Additional ACK Block fields is determined by the ACK Block Count field.

Gap and ACK Block fields use a relative integer encoding for efficiency.  Though
each encoded value is positive, the values are subtracted, so that each ACK
Block describes progressively lower-numbered packets.  As long as contiguous
ranges of packets are small, the variable-length integer encoding ensures that
each range can be expressed in a small number of octets.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      First ACK Block (i)                    ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Gap (i)                         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Additional ACK Block (i)                 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Gap (i)                         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Additional ACK Block (i)                 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                               ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Gap (i)                         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Additional ACK Block (i)                 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #ack-block-format title="ACK Block Section"}

Each ACK Block acknowledges a contiguous range of packets by indicating the
number of acknowledged packets that precede the largest packet number in that
block.  A value of zero indicates that only the largest packet number is
acknowledged.  Larger ACK Block values indicate a larger range, with
corresponding lower values for the smallest packet number in the range.  Thus,
given a largest packet number for the ACK, the smallest value is determined by
the formula:

~~~
   smallest = largest - ack_block
~~~

The range of packets that are acknowledged by the ACK block include the range
from the smallest packet number to the largest, inclusive.

The largest value for the First ACK Block is determined by the Largest
Acknowledged field; the largest for Additional ACK Blocks is determined by
cumulatively subtracting the size of all preceding ACK Blocks and Gaps.

Each Gap indicates a range of packets that are not being acknowledged.  The
number of packets in the gap is one higher than the encoded value of the Gap
Field.

The value of the Gap field establishes the largest packet number value for the
ACK block that follows the gap using the following formula:

~~~
  largest = previous_smallest - gap - 2
~~~

If the calculated value for largest or smallest packet number for any ACK Block
is negative, an endpoint MUST generate a connection error of type FRAME_ERROR
indicating an error in an ACK frame (that is, 0x10d).

The fields in the ACK Block Section are:

First ACK Block:

: A variable-length integer indicating the number of contiguous packets
  preceding the Largest Acknowledged that are being acknowledged.

Gap (repeated):

: A variable-length integer indicating the number of contiguous unacknowledged
  packets preceding the packet number one lower than the smallest in the
  preceding ACK Block.

ACK Block (repeated):

: A variable-length integer indicating the number of contiguous acknowledged
  packets preceding the largest packet number, as determined by the
  preceding Gap.

### Sending ACK Frames

Implementations MUST NOT generate packets that only contain ACK frames in
response to packets which only contain ACK frames. However, they MUST
acknowledge packets containing only ACK frames when sending ACK frames in
response to other packets.  Implementations MUST NOT send more than one packet
containing only ACK frames per received packet that contains frames other than
ACK frames.  Packets containing non-ACK frames MUST be acknowledged immediately
or when a delayed ack timer expires.

To limit ACK blocks to those that have not yet been received by the sender, the
receiver SHOULD track which ACK frames have been acknowledged by its peer.  Once
an ACK frame has been acknowledged, the packets it acknowledges SHOULD NOT be
acknowledged again.

Because ACK frames are not sent in response to ACK-only packets, a receiver that
is only sending ACK frames will only receive acknowledgements for its packets
if the sender includes them in packets with non-ACK frames.  A sender SHOULD
bundle ACK frames with other frames when possible.

To limit receiver state or the size of ACK frames, a receiver MAY limit the
number of ACK blocks it sends.  A receiver can do this even without receiving
acknowledgment of its ACK frames, with the knowledge this could cause the sender
to unnecessarily retransmit some data.  Standard QUIC {{QUIC-RECOVERY}}
algorithms declare packets lost after sufficiently newer packets are
acknowledged.  Therefore, the receiver SHOULD repeatedly acknowledge newly
received packets in preference to packets received in the past.

### ACK Frames and Packet Protection

ACK frames that acknowledge protected packets MUST be carried in a packet that
has an equivalent or greater level of packet protection.

Packets that are protected with 1-RTT keys MUST be acknowledged in packets that
are also protected with 1-RTT keys.

A packet that is not protected and claims to acknowledge a packet number that
was sent with packet protection is not valid.  An unprotected packet that
carries acknowledgments for protected packets MUST be discarded in its entirety.

Packets that a client sends with 0-RTT packet protection MUST be acknowledged by
the server in packets protected by 1-RTT keys.  This can mean that the client is
unable to use these acknowledgments if the server cryptographic handshake
messages are delayed or lost.  Note that the same limitation applies to other
data sent by the server protected by the 1-RTT keys.

Unprotected packets, such as those that carry the initial cryptographic
handshake messages, MAY be acknowledged in unprotected packets.  Unprotected
packets are vulnerable to falsification or modification.  Unprotected packets
can be acknowledged along with protected packets in a protected packet.

An endpoint SHOULD acknowledge packets containing cryptographic handshake
messages in the next unprotected packet that it sends, unless it is able to
acknowledge those packets in later packets protected by 1-RTT keys.  At the
completion of the cryptographic handshake, both peers send unprotected packets
containing cryptographic handshake messages followed by packets protected by
1-RTT keys. An endpoint SHOULD acknowledge the unprotected packets that complete
the cryptographic handshake in a protected packet, because its peer is
guaranteed to have access to 1-RTT packet protection keys.

For instance, a server acknowledges a TLS ClientHello in the packet that carries
the TLS ServerHello; similarly, a client can acknowledge a TLS HelloRetryRequest
in the packet containing a second TLS ClientHello.  The complete set of server
handshake messages (TLS ServerHello through to Finished) might be acknowledged
by a client in protected packets, because it is certain that the server is able
to decipher the packet.


## PATH_CHALLENGE Frame {#frame-path-challenge}

Endpoints can use PATH_CHALLENGE frames (type=0x0e) to check reachability to the
peer and for path validation during connection establishment and connection
migration.

PATH_CHALLENGE frames contain an 8-byte payload.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                            Data (8)                           +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

~~~

Data:

: This 8-byte field contains arbitrary data.

A PATH_CHALLENGE frame containing 8 octets that are hard to guess is sufficient
to ensure that it is easier to receive the packet than it is to guess the value
correctly.

The recipient of this frame MUST generate a PATH_RESPONSE frame
({{frame-path-response}}) containing the same Data.


## PATH_RESPONSE Frame {#frame-path-response}

The PATH_RESPONSE frame (type=0x0f) is sent in response to a PATH_CHALLENGE
frame.  Its format is identical to the PATH_CHALLENGE frame
({{frame-path-challenge}}).

If the content of a PATH_RESPONSE frame does not match the content of a
PATH_CHALLENGE frame previously sent by the endpoint, the endpoint MAY generate
a connection error of type UNSOLICITED_PATH_RESPONSE.


## STREAM Frames {#frame-stream}

STREAM frames implicitly create a stream and carry stream data.  The STREAM
frame takes the form 0b00010XXX (or the set of values from 0x10 to 0x17).  The
value of the three low-order bits of the frame type determine the fields that
are present in the frame.

* The OFF bit (0x04) in the frame type is set to indicate that there is an
  Offset field present.  When set to 1, the Offset field is present; when set to
  0, the Offset field is absent and the Stream Data starts at an offset of 0
  (that is, the frame contains the first octets of the stream, or the end of a
  stream that includes no data).

* The LEN bit (0x02) in the frame type is set to indicate that there is a Length
  field present.  If this bit is set to 0, the Length field is absent and the
  Stream Data field extends to the end of the packet.  If this bit is set to 1,
  the Length field is present.

* The FIN bit (0x01) of the frame type is set only on frames that contain the
  final offset of the stream.  Setting this bit indicates that the frame
  marks the end of the stream.

An endpoint that receives a STREAM frame for a send-only stream MUST terminate
the connection with error PROTOCOL_VIOLATION.

A STREAM frame is shown below.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Stream ID (i)                       ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         [Offset (i)]                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         [Length (i)]                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Stream Data (*)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #stream-format title="STREAM Frame Format"}

The STREAM frame contains the following fields:

Stream ID:

: A variable-length integer indicating the stream ID of the stream (see
  {{stream-id}}).

Offset:

: A variable-length integer specifying the byte offset in the stream for the
  data in this STREAM frame.  This field is present when the OFF bit is set to
  1.  When the Offset field is absent, the offset is 0.

Length:

: A variable-length integer specifying the length of the Stream Data field in
  this STREAM frame.  This field is present when the LEN bit is set to 1.  When
  the LEN bit is set to 0, the Stream Data field consumes all the remaining
  octets in the packet.

Stream Data:

: The bytes from the designated stream to be delivered.

When a Stream Data field has a length of 0, the offset in the STREAM frame is
the offset of the next byte that would be sent.

The first byte in the stream has an offset of 0.  The largest offset delivered
on a stream - the sum of the re-constructed offset and data length - MUST be
less than 2^62.

Stream multiplexing is achieved by interleaving STREAM frames from multiple
streams into one or more QUIC packets.  A single QUIC packet can include
multiple STREAM frames from one or more streams.

Implementation note: One of the benefits of QUIC is avoidance of head-of-line
blocking across multiple streams.  When a packet loss occurs, only streams with
data in that packet are blocked waiting for a retransmission to be received,
while other streams can continue making progress.  Note that when data from
multiple streams is bundled into a single QUIC packet, loss of that packet
blocks all those streams from making progress.  An implementation is therefore
advised to bundle as few streams as necessary in outgoing packets without losing
transmission efficiency to underfilled packets.


# Packetization and Reliability {#packetization}

A sender bundles one or more frames in a QUIC packet (see {{frames}}).

A sender SHOULD minimize per-packet bandwidth and computational costs by
bundling as many frames as possible within a QUIC packet.  A sender MAY wait for
a short period of time to bundle multiple frames before sending a packet that is
not maximally packed, to avoid sending out large numbers of small packets.  An
implementation may use knowledge about application sending behavior or
heuristics to determine whether and for how long to wait.  This waiting period
is an implementation decision, and an implementation should be careful to delay
conservatively, since any delay is likely to increase application-visible
latency.


## Packet Processing and Acknowledgment

A packet MUST NOT be acknowledged until packet protection has been successfully
removed and all frames contained in the packet have been processed.  Any stream
state transitions triggered by the frame MUST have occurred.  For STREAM frames,
this means the data has been enqueued in preparation to be received by the
application protocol, but it does not require that data is delivered and
consumed.

Once the packet has been fully processed, a receiver acknowledges receipt by
sending one or more ACK frames containing the packet number of the received
packet.  To avoid creating an indefinite feedback loop, an endpoint MUST NOT
send an ACK frame in response to a packet containing only ACK or PADDING frames,
even if there are packet gaps which precede the received packet.  The endpoint
MUST acknowledge packets containing only ACK or PADDING frames in the next ACK
frame that it sends.

Strategies and implications of the frequency of generating acknowledgments are
discussed in more detail in {{QUIC-RECOVERY}}.


## Retransmission of Information

QUIC packets that are determined to be lost are not retransmitted whole. The
same applies to the frames that are contained within lost packets. Instead, the
information that might be carried in frames is sent again in new frames as
needed.

New frames and packets are used to carry information that is determined to have
been lost.  In general, information is sent again when a packet containing that
information is determined to be lost and sending ceases when a packet
containing that information is acknowledged.

* Application data sent in STREAM frames is retransmitted in new STREAM frames
  unless the endpoint has sent a RST_STREAM for that stream.  Once an endpoint
  sends a RST_STREAM frame, no further STREAM frames are needed.

* The most recent set of acknowledgments are sent in ACK frames.  An ACK frame
  SHOULD contain all unacknowledged acknowledgments, as described in
  {{sending-ack-frames}}.

* Cancellation of stream transmission, as carried in a RST_STREAM frame, is
  sent until acknowledged or until all stream data is acknowledged by the peer
  (that is, either the "Reset Recvd" or "Data Recvd" state is reached on the
  send stream). The content of a RST_STREAM frame MUST NOT change when it is
  sent again.

* Similarly, a request to cancel stream transmission, as encoded in a
  STOP_SENDING frame, is sent until the receive stream enters either a "Data
  Recvd" or "Reset Recvd" state, see {{solicited-state-transitions}}.

* Connection close signals, including those that use CONNECTION_CLOSE and
  APPLICATION_CLOSE frames, are not sent again when packet loss is detected, but
  as described in {{termination}}.

* The current connection maximum data is sent in MAX_DATA frames. An updated
  value is sent in a MAX_DATA frame if the packet containing the most recently
  sent MAX_DATA frame is declared lost, or when the endpoint decides to update
  the limit.  Care is necessary to avoid sending this frame too often as the
  limit can increase frequently and cause an unnecessarily large number of
  MAX_DATA frames to be sent.

* The current maximum stream data offset is sent in MAX_STREAM_DATA frames.
  Like MAX_DATA, an updated value is sent when the packet containing
  the most recent MAX_STREAM_DATA frame for a stream is lost or when the limit
  is updated, with care taken to prevent the frame from being sent too often. An
  endpoint SHOULD stop sending MAX_STREAM_DATA frames when the receive stream
  enters a "Size Known" state.

* The maximum stream ID for a stream of a given type is sent in MAX_STREAM_ID
  frames.  Like MAX_DATA, an updated value is sent when a packet containing the
  most recent MAX_STREAM_ID for a stream type frame is declared lost or when
  the limit is updated, with care taken to prevent the frame from being sent
  too often.

* Blocked signals are carried in BLOCKED, STREAM_BLOCKED, and STREAM_ID_BLOCKED
  frames. BLOCKED streams have connection scope, STREAM_BLOCKED frames have
  stream scope, and STREAM_ID_BLOCKED frames are scoped to a specific stream
  type. New frames are sent if packets containing the most recent frame for a
  scope is lost, but only while the endpoint is blocked on the corresponding
  limit. These frames always include the limit that is causing blocking at the
  time that they are transmitted.

* A liveness or path validation check using PATH_CHALLENGE frames is sent
  periodically until a matching PATH_RESPONSE frame is received or until there
  is no remaining need for liveness or path validation checking. PATH_CHALLENGE
  frames include a different payload each time they are sent.

* Responses to path validation using PATH_RESPONSE frames are sent just once.
  A new PATH_CHALLENGE frame will be sent if another PATH_RESPONSE frame is
  needed.

* New connection IDs are sent in NEW_CONNECTION_ID frames and retransmitted if
  the packet containing them is lost.

* PADDING frames contain no information, so lost PADDING frames do not require
  repair.

Upon detecting losses, a sender MUST take appropriate congestion control action.
The details of loss detection and congestion control are described in
{{QUIC-RECOVERY}}.


## Packet Size {#packet-size}

The QUIC packet size includes the QUIC header and integrity check, but not the
UDP or IP header.

Clients MUST pad any Initial packet it sends to have a QUIC packet size of at
least 1200 octets. Sending an Initial packet of this size ensures that the
network path supports a reasonably sized packet, and helps reduce the amplitude
of amplification attacks caused by server responses toward an unverified client
address.

An Initial packet MAY exceed 1200 octets if the client knows that the Path
Maximum Transmission Unit (PMTU) supports the size that it chooses.

A server MAY send a CONNECTION_CLOSE frame with error code PROTOCOL_VIOLATION in
response to an Initial packet smaller than 1200 octets. It MUST NOT send any
other frame type in response, or otherwise behave as if any part of the
offending packet was processed as valid.

## Path Maximum Transmission Unit

The Path Maximum Transmission Unit (PMTU) is the maximum size of the entire IP
header, UDP header, and UDP payload. The UDP payload includes the QUIC packet
header, protected payload, and any authentication fields.

All QUIC packets SHOULD be sized to fit within the estimated PMTU to avoid IP
fragmentation or packet drops. To optimize bandwidth efficiency, endpoints
SHOULD use Packetization Layer PMTU Discovery ({{!PLPMTUD=RFC4821}}).  Endpoints
MAY use PMTU Discovery ({{!PMTUDv4=RFC1191}}, {{!PMTUDv6=RFC8201}}) for
detecting the PMTU, setting the PMTU appropriately, and storing the result of
previous PMTU determinations.

In the absence of these mechanisms, QUIC endpoints SHOULD NOT send IP packets
larger than 1280 octets. Assuming the minimum IP header size, this results in
a QUIC packet size of 1232 octets for IPv6 and 1252 octets for IPv4. Some
QUIC implementations MAY wish to be more conservative in computing allowed
QUIC packet size given unknown tunneling overheads or IP header options.

QUIC endpoints that implement any kind of PMTU discovery SHOULD maintain an
estimate for each combination of local and remote IP addresses.  Each pairing of
local and remote addresses could have a different maximum MTU in the path.

QUIC depends on the network path supporting a MTU of at least 1280 octets. This
is the IPv6 minimum MTU and therefore also supported by most modern IPv4
networks.  An endpoint MUST NOT reduce its MTU below this number, even if it
receives signals that indicate a smaller limit might exist.

If a QUIC endpoint determines that the PMTU between any pair of local and remote
IP addresses has fallen below 1280 octets, it MUST immediately cease sending
QUIC packets on the affected path.  This could result in termination of the
connection if an alternative path cannot be found.


### IPv4 PMTU Discovery {#v4-pmtud}

Traditional ICMP-based path MTU discovery in IPv4 {{!PMTUDv4}} is potentially
vulnerable to off-path attacks that successfully guess the IP/port 4-tuple and
reduce the MTU to a bandwidth-inefficient value. TCP connections mitigate this
risk by using the (at minimum) 8 bytes of transport header echoed in the ICMP
message to validate the TCP sequence number as valid for the current
connection. However, as QUIC operates over UDP, in IPv4 the echoed information
could consist only of the IP and UDP headers, which usually has insufficient
entropy to mitigate off-path attacks.

As a result, endpoints that implement PMTUD in IPv4 SHOULD take steps to
mitigate this risk. For instance, an application could:

* Set the IPv4 Don't Fragment (DF) bit on a small proportion of packets, so that
most invalid ICMP messages arrive when there are no DF packets outstanding, and
can therefore be identified as spurious.

* Store additional information from the IP or UDP headers from DF packets (for
example, the IP ID or UDP checksum) to further authenticate incoming Datagram
Too Big messages.

* Any reduction in PMTU due to a report contained in an ICMP packet is
provisional until QUIC's loss detection algorithm determines that the packet is
actually lost.


### Special Considerations for Packetization Layer PMTU Discovery


The PADDING frame provides a useful option for PMTU probe packets. PADDING
frames generate acknowledgements, but they need not be delivered reliably. As a
result, the loss of PADDING frames in probe packets does not require
delay-inducing retransmission. However, PADDING frames do consume congestion
window, which may delay the transmission of subsequent application data.

When implementing the algorithm in Section 7.2 of {{!PLPMTUD}}, the initial
value of search_low SHOULD be consistent with the IPv6 minimum packet size.
Paths that do not support this size cannot deliver Initial packets, and
therefore are not QUIC-compliant.

Section 7.3 of {{!PLPMTUD}} discusses tradeoffs between small and large
increases in the size of probe packets. As QUIC probe packets need not contain
application data, aggressive increases in probe size carry fewer consequences.


# Streams: QUIC's Data Structuring Abstraction {#streams}

Streams in QUIC provide a lightweight, ordered byte-stream abstraction.

There are two basic types of stream in QUIC.  Unidirectional streams carry data
in one direction only; bidirectional streams allow for data to be sent in both
directions.  Different stream identifiers are used to distinguish between
unidirectional and bidirectional streams, as well as to create a separation
between streams that are initiated by the client and server (see {{stream-id}}).

Either type of stream can be created by either endpoint, can concurrently send
data interleaved with other streams, and can be cancelled.

Stream offsets allow for the octets on a stream to be placed in order.  An
endpoint MUST be capable of delivering data received on a stream in order.
Implementations MAY choose to offer the ability to deliver data out of order.
There is no means of ensuring ordering between octets on different streams.

The creation and destruction of streams are expected to have minimal bandwidth
and computational cost.  A single STREAM frame may create, carry data for, and
terminate a stream, or a stream may last the entire duration of a connection.

Streams are individually flow controlled, allowing an endpoint to limit memory
commitment and to apply back pressure.  The creation of streams is also flow
controlled, with each peer declaring the maximum stream ID it is willing to
accept at a given time.

An alternative view of QUIC streams is as an elastic "message" abstraction,
similar to the way ephemeral streams are used in SST
{{?SST=DOI.10.1145/1282427.1282421}}, which may be a more appealing description
for some applications.


## Stream Identifiers {#stream-id}

Streams are identified by an unsigned 62-bit integer, referred to as the Stream
ID.  The least significant two bits of the Stream ID are used to identify the
type of stream (unidirectional or bidirectional) and the initiator of the
stream.

The least significant bit (0x1) of the Stream ID identifies the initiator of the
stream.  Clients initiate even-numbered streams (those with the least
significant bit set to 0); servers initiate odd-numbered streams (with the bit
set to 1).  Separation of the stream identifiers ensures that client and server
are able to open streams without the latency imposed by negotiating for an
identifier.

If an endpoint receives a frame for a stream that it expects to initiate (i.e.,
odd-numbered for the client or even-numbered for the server), but which it has
not yet opened, it MUST close the connection with error code STREAM_STATE_ERROR.

The second least significant bit (0x2) of the Stream ID differentiates between
unidirectional streams and bidirectional streams. Unidirectional streams always
have this bit set to 1 and bidirectional streams have this bit set to 0.

The two type bits from a Stream ID therefore identify streams as summarized in
{{stream-id-types}}.

| Low Bits | Stream Type                      |
|:---------|:---------------------------------|
| 0x0      | Client-Initiated, Bidirectional  |
| 0x1      | Server-Initiated, Bidirectional  |
| 0x2      | Client-Initiated, Unidirectional |
| 0x3      | Server-Initiated, Unidirectional |
{: #stream-id-types title="Stream ID Types"}

Stream ID 0 (0x0) is a client-initiated, bidirectional stream that is used for
the cryptographic handshake.  Stream 0 MUST NOT be used for application data.

A QUIC endpoint MUST NOT reuse a Stream ID.  Streams can be used in any order.
Streams that are used out of order result in opening all lower-numbered streams
of the same type in the same direction.

Stream IDs are encoded as a variable-length integer (see {{integer-encoding}}).


## Stream States {#stream-states}

This section describes the two types of QUIC stream in terms of the states of
their send or receive components.  Two state machines are described: one for
streams on which an endpoint transmits data ({{stream-send-states}}); another
for streams from which an endpoint receives data ({{stream-recv-states}}).

Unidirectional streams use the applicable state machine directly.  Bidirectional
streams use both state machines.  For the most part, the use of these state
machines is the same whether the stream is unidirectional or bidirectional.  The
conditions for opening a stream are slightly more complex for a bidirectional
stream because the opening of either send or receive sides causes the stream
to open in both directions.

An endpoint can open streams up to its maximum stream limit in any order,
however endpoints SHOULD open the send side of streams for each type in order.

Note:

: These states are largely informative.  This document uses stream states to
  describe rules for when and how different types of frames can be sent and the
  reactions that are expected when different types of frames are received.
  Though these state machines are intended to be useful in implementing QUIC,
  these states aren't intended to constrain implementations.  An implementation
  can define a different state machine as long as its behavior is consistent
  with an implementation that implements these states.


### Send Stream States {#stream-send-states}

{{fig-stream-send-states}} shows the states for the part of a stream that sends
data to a peer.

~~~
       o
       | Create Stream (Sending)
       | Create Bidirectional Stream (Receiving)
       v
   +-------+
   | Ready | Send RST_STREAM
   |       |-----------------------.
   +-------+                       |
       |                           |
       | Send STREAM /             |
       |      STREAM_BLOCKED       |
       v                           |
   +-------+                       |
   | Send  | Send RST_STREAM       |
   |       |---------------------->|
   +-------+                       |
       |                           |
       | Send STREAM + FIN         |
       v                           v
   +-------+                   +-------+
   | Data  | Send RST_STREAM   | Reset |
   | Sent  +------------------>| Sent  |
   +-------+                   +-------+
       |                           |
       | Recv All ACKs             | Recv ACK
       v                           v
   +-------+                   +-------+
   | Data  |                   | Reset |
   | Recvd |                   | Recvd |
   +-------+                   +-------+
~~~
{: #fig-stream-send-states title="States for Send Streams"}

The sending part of stream that the endpoint initiates (types 0 and 2 for
clients, 1 and 3 for servers) is opened by the application or application
protocol.  The "Ready" state represents a newly created stream that is able to
accept data from the application.  Stream data might be buffered in this state
in preparation for sending.

The sending part of a bidirectional stream initiated by a peer (type 0 for a
server, type 1 for a client) enters the "Ready" state if the receiving part
enters the "Recv" state.

Sending the first STREAM or STREAM_BLOCKED frame causes a send stream to enter
the "Send" state.  An implementation might choose to defer allocating a Stream
ID to a send stream until it sends the first frame and enters this state, which
can allow for better stream prioritization.

In the "Send" state, an endpoint transmits - and retransmits as necessary - data
in STREAM frames.  The endpoint respects the flow control limits of its peer,
accepting MAX_STREAM_DATA frames.  An endpoint in the "Send" state generates
STREAM_BLOCKED frames if it encounters flow control limits.

After the application indicates that stream data is complete and a STREAM frame
containing the FIN bit is sent, the send stream enters the "Data Sent" state.
From this state, the endpoint only retransmits stream data as necessary.  The
endpoint no longer needs to track flow control limits or send STREAM_BLOCKED
frames for a send stream in this state.  The endpoint can ignore any
MAX_STREAM_DATA frames it receives from its peer in this state; MAX_STREAM_DATA
frames might be received until the peer receives the final stream offset.

Once all stream data has been successfully acknowledged, the send stream enters
the "Data Recvd" state, which is a terminal state.

From any of the "Ready", "Send", or "Data Sent" states, an application can
signal that it wishes to abandon transmission of stream data.  Similarly, the
endpoint might receive a STOP_SENDING frame from its peer.  In either case, the
endpoint sends a RST_STREAM frame, which causes the stream to enter the "Reset
Sent" state.

An endpoint MAY send a RST_STREAM as the first frame on a send stream; this
causes the send stream to open and then immediately transition to the "Reset
Sent" state.

Once a packet containing a RST_STREAM has been acknowledged, the send stream
enters the "Reset Recvd" state, which is a terminal state.


### Receive Stream States {#stream-recv-states}

{{fig-stream-recv-states}} shows the states for the part of a stream that
receives data from a peer.  The states for a receive stream mirror only some of
the states of the send stream at the peer.  A receive stream doesn't track
states on the send stream that cannot be observed, such as the "Ready" state;
instead, receive streams track the delivery of data to the application or
application protocol some of which cannot be observed by the sender.

~~~
       o
       | Recv STREAM / STREAM_BLOCKED / RST_STREAM
       | Create Bidirectional Stream (Sending)
       | Recv MAX_STREAM_DATA
       v
   +-------+
   | Recv  | Recv RST_STREAM
   |       |-----------------------.
   +-------+                       |
       |                           |
       | Recv STREAM + FIN         |
       v                           |
   +-------+                       |
   | Size  | Recv RST_STREAM       |
   | Known +---------------------->|
   +-------+                       |
       |                           |
       | Recv All Data             |
       v                           v
   +-------+                   +-------+
   | Data  | Recv RST_STREAM   | Reset |
   | Recvd +<-- (optional) --->| Recvd |
   +-------+                   +-------+
       |                           |
       | App Read All Data         | App Read RST
       v                           v
   +-------+                   +-------+
   | Data  |                   | Reset |
   | Read  |                   | Read  |
   +-------+                   +-------+
~~~
{: #fig-stream-recv-states title="States for Receive Streams"}

The receiving part of a stream initiated by a peer (types 1 and 3 for a client,
or 0 and 2 for a server) are created when the first STREAM, STREAM_BLOCKED,
RST_STREAM, or MAX_STREAM_DATA (bidirectional only, see below) is received for
that stream.  The initial state for a receive stream is "Recv".  Receiving a
RST_STREAM frame causes the receive stream to immediately transition to the
"Reset Recvd".

The receive stream enters the "Recv" state when the sending part of a
bidirectional stream initiated by the endpoint (type 0 for a client, type 1 for
a server) enters the "Ready" state.

A bidirectional stream also opens when a MAX_STREAM_DATA frame is received.
Receiving a MAX_STREAM_DATA frame implies that the remote peer has opened the
stream and is providing flow control credit.  A MAX_STREAM_DATA frame might
arrive before a STREAM or STREAM_BLOCKED frame if packets are lost or reordered.

In the "Recv" state, the endpoint receives STREAM and STREAM_BLOCKED frames.
Incoming data is buffered and can be reassembled into the correct order for
delivery to the application.  As data is consumed by the application and buffer
space becomes available, the endpoint sends MAX_STREAM_DATA frames to allow the
peer to send more data.

When a STREAM frame with a FIN bit is received, the final offset (see
{{final-offset}}) is known.  The receive stream enters the "Size Known" state.
In this state, the endpoint no longer needs to send MAX_STREAM_DATA frames, it
only receives any retransmissions of stream data.

Once all data for the stream has been received, the receive stream enters the
"Data Recvd" state.  This might happen as a result of receiving the same STREAM
frame that causes the transition to "Size Known".  In this state, the endpoint
has all stream data.  Any STREAM or STREAM_BLOCKED frames it receives for the
stream can be discarded.

The "Data Recvd" state persists until stream data has been delivered to the
application or application protocol.  Once stream data has been delivered, the
stream enters the "Data Read" state, which is a terminal state.

Receiving a RST_STREAM frame in the "Recv" or "Size Known" states causes the
stream to enter the "Reset Recvd" state.  This might cause the delivery of
stream data to the application to be interrupted.

It is possible that all stream data is received when a RST_STREAM is received
(that is, from the "Data Recvd" state).  Similarly, it is possible for remaining
stream data to arrive after receiving a RST_STREAM frame (the "Reset Recvd"
state).  An implementation is able to manage this situation as they choose.
Sending RST_STREAM means that an endpoint cannot guarantee delivery of stream
data; however there is no requirement that stream data not be delivered if a
RST_STREAM is received.  An implementation MAY interrupt delivery of stream
data, discard any data that was not consumed, and signal the existence of the
RST_STREAM immediately.  Alternatively, the RST_STREAM signal might be
suppressed or withheld if stream data is completely received.  In the latter
case, the receive stream effectively transitions to "Data Recvd" from "Reset
Recvd".

Once the application has been delivered the signal indicating that the receive
stream was reset, the receive stream transitions to the "Reset Read" state,
which is a terminal state.


### Permitted Frame Types

The sender of a stream sends just three frame types that affect the state of a
stream at either sender or receiver: STREAM ({{frame-stream}}), STREAM_BLOCKED
({{frame-stream-blocked}}), and RST_STREAM ({{frame-rst-stream}}).

A sender MUST NOT send any of these frames from a terminal state ("Data Recvd"
or "Reset Recvd").  A sender MUST NOT send STREAM or STREAM_BLOCKED after
sending a RST_STREAM; that is, in the "Reset Sent" state in addition to the
terminal states.  A receiver could receive any of these frames in any state, but
only due to the possibility of delayed delivery of packets carrying them.

The receiver of a stream sends MAX_STREAM_DATA ({{frame-max-stream-data}}) and
STOP_SENDING frames ({{frame-stop-sending}}).

The receiver only sends MAX_STREAM_DATA in the "Recv" state.  A receiver can
send STOP_SENDING in any state where it has not received a RST_STREAM frame;
that is states other than "Reset Recvd" or "Reset Read".  However there is
little value in sending a STOP_SENDING frame after all stream data has been
received in the "Data Recvd" state.  A sender could receive these frames in any
state as a result of delayed delivery of packets.


### Bidirectional Stream States {#stream-bidi-states}

A bidirectional stream is composed of a send stream and a receive stream.
Implementations may represent states of the bidirectional stream as composites
of send and receive stream states.  The simplest model presents the stream as
"open" when either send or receive stream is in a non-terminal state and
"closed" when both send and receive streams are in a terminal state.

{{stream-bidi-mapping}} shows a more complex mapping of bidirectional stream
states that loosely correspond to the stream states in HTTP/2
{{?HTTP2=RFC7540}}.  This shows that multiple states on send or receive streams
are mapped to the same composite state.  Note that this is just one possibility
for such a mapping; this mapping requires that data is acknowledged before the
transition to a "closed" or "half-closed" state.

| Send Stream            | Receive Stream         | Composite State      |
|:-----------------------|:-----------------------|:---------------------|
| No Stream/Ready        | No Stream/Recv *1      | idle                 |
| Ready/Send/Data Sent   | Recv/Size Known        | open                 |
| Ready/Send/Data Sent   | Data Recvd/Data Read   | half-closed (remote) |
| Ready/Send/Data Sent   | Reset Recvd/Reset Read | half-closed (remote) |
| Data Recvd             | Recv/Size Known        | half-closed (local)  |
| Reset Sent/Reset Recvd | Recv/Size Known        | half-closed (local)  |
| Data Recvd             | Recv/Size Known        | half-closed (local)  |
| Reset Sent/Reset Recvd | Data Recvd/Data Read   | closed               |
| Reset Sent/Reset Recvd | Reset Recvd/Reset Read | closed               |
| Data Recvd             | Data Recvd/Data Read   | closed               |
| Data Recvd             | Reset Recvd/Reset Read | closed               |
{: #stream-bidi-mapping title="Possible Mapping of Stream States to HTTP/2"}

Note (*1):

: A stream is considered "idle" if it has not yet been created, or if the
  receive stream is in the "Recv" state without yet having received any frames.


## Solicited State Transitions

If an endpoint is no longer interested in the data it is receiving on a stream,
it MAY send a STOP_SENDING frame identifying that stream to prompt closure of
the stream in the opposite direction.  This typically indicates that the
receiving application is no longer reading data it receives from the stream, but
is not a guarantee that incoming data will be ignored.

STREAM frames received after sending STOP_SENDING are still counted toward the
connection and stream flow-control windows, even though these frames will be
discarded upon receipt.  This avoids potential ambiguity about which STREAM
frames count toward flow control.

A STOP_SENDING frame requests that the receiving endpoint send a RST_STREAM
frame.  An endpoint that receives a STOP_SENDING frame MUST send a RST_STREAM
frame for that stream, and can use an error code of STOPPING.  If the
STOP_SENDING frame is received on a send stream that is already in the "Data
Sent" state, a RST_STREAM frame MAY still be sent in order to cancel
retransmission of previously-sent STREAM frames.

STOP_SENDING SHOULD only be sent for a receive stream that has not been
reset. STOP_SENDING is most useful for streams in the "Recv" or "Size Known"
states.

An endpoint is expected to send another STOP_SENDING frame if a packet
containing a previous STOP_SENDING is lost.  However, once either all stream
data or a RST_STREAM frame has been received for the stream - that is, the
stream is in any state other than "Recv" or "Size Known" - sending a
STOP_SENDING frame is unnecessary.


## Stream Concurrency {#stream-concurrency}

An endpoint limits the number of concurrently active incoming streams by
adjusting the maximum stream ID.  An initial value is set in the transport
parameters (see {{transport-parameter-definitions}}) and is subsequently
increased by MAX_STREAM_ID frames (see {{frame-max-stream-id}}).

The maximum stream ID is specific to each endpoint and applies only to the peer
that receives the setting. That is, clients specify the maximum stream ID the
server can initiate, and servers specify the maximum stream ID the client can
initiate.  Each endpoint may respond on streams initiated by the other peer,
regardless of whether it is permitted to initiated new streams.

Endpoints MUST NOT exceed the limit set by their peer.  An endpoint that
receives a STREAM frame with an ID greater than the limit it has sent MUST treat
this as a stream error of type STREAM_ID_ERROR ({{error-handling}}), unless this
is a result of a change in the initial offsets (see {{zerortt-parameters}}).

A receiver MUST NOT renege on an advertisement; that is, once a receiver
advertises a stream ID via a MAX_STREAM_ID frame, it MUST NOT subsequently
advertise a smaller maximum ID.  A sender may receive MAX_STREAM_ID frames out
of order; a sender MUST therefore ignore any MAX_STREAM_ID that does not
increase the maximum.

## Sending and Receiving Data

Once a stream is created, endpoints may use the stream to send and receive data.
Each endpoint may send a series of STREAM frames encapsulating data on a stream
until the stream is terminated in that direction.  Streams are an ordered
byte-stream abstraction, and they have no other structure within them.  STREAM
frame boundaries are not expected to be preserved in retransmissions from the
sender or during delivery to the application at the receiver.

When new data is to be sent on a stream, a sender MUST set the encapsulating
STREAM frame's offset field to the stream offset of the first byte of this new
data.  The first octet of data on a stream has an offset of 0.  An endpoint is
expected to send every stream octet.  The largest offset delivered on a stream
MUST be less than 2^62.

QUIC makes no specific allowances for partial reliability or delivery of stream
data out of order.  Endpoints MUST be able to deliver stream data to an
application as an ordered byte-stream.  Delivering an ordered byte-stream
requires that an endpoint buffer any data that is received out of order, up to
the advertised flow control limit.

An endpoint could receive the same octets multiple times; octets that have
already been received can be discarded.  The value for a given octet MUST NOT
change if it is sent multiple times; an endpoint MAY treat receipt of a changed
octet as a connection error of type PROTOCOL_VIOLATION.

An endpoint MUST NOT send data on any stream without ensuring that it is within
the data limits set by its peer.  The cryptographic handshake stream, Stream 0,
is exempt from the connection-level data limits established by MAX_DATA. Data on
stream 0 other than the initial cryptographic handshake message is still subject
to stream-level data limits and MAX_STREAM_DATA. This message is exempt from
flow control because it needs to be sent in a single packet regardless of the
server's flow control state. This rule applies even for 0-RTT handshakes where
the remembered value of MAX_STREAM_DATA would not permit sending a full initial
cryptographic handshake message.

Flow control is described in detail in {{flow-control}}, and congestion control
is described in the companion document {{QUIC-RECOVERY}}.


## Stream Prioritization

Stream multiplexing has a significant effect on application performance if
resources allocated to streams are correctly prioritized.  Experience with other
multiplexed protocols, such as HTTP/2 {{?HTTP2}}, shows that effective
prioritization strategies have a significant positive impact on performance.

QUIC does not provide frames for exchanging prioritization information.  Instead
it relies on receiving priority information from the application that uses QUIC.
Protocols that use QUIC are able to define any prioritization scheme that suits
their application semantics.  A protocol might define explicit messages for
signaling priority, such as those defined in HTTP/2; it could define rules that
allow an endpoint to determine priority based on context; or it could leave the
determination to the application.

A QUIC implementation SHOULD provide ways in which an application can indicate
the relative priority of streams.  When deciding which streams to dedicate
resources to, QUIC SHOULD use the information provided by the application.
Failure to account for priority of streams can result in suboptimal performance.

Stream priority is most relevant when deciding which stream data will be
transmitted.  Often, there will be limits on what can be transmitted as a result
of connection flow control or the current congestion controller state.

Giving preference to the transmission of its own management frames ensures that
the protocol functions efficiently.  That is, prioritizing frames other than
STREAM frames ensures that loss recovery, congestion control, and flow control
operate effectively.

Stream 0 MUST be prioritized over other streams prior to the completion of the
cryptographic handshake.  This includes the retransmission of the second flight
of client handshake messages, that is, the TLS Finished and any client
authentication messages.

STREAM data in frames determined to be lost SHOULD be retransmitted before
sending new data, unless application priorities indicate otherwise.
Retransmitting lost stream data can fill in gaps, which allows the peer to
consume already received data and free up flow control window.


# Flow Control {#flow-control}

It is necessary to limit the amount of data that a sender may have outstanding
at any time, so as to prevent a fast sender from overwhelming a slow receiver,
or to prevent a malicious sender from consuming significant resources at a
receiver.  This section describes QUIC's flow-control mechanisms.

QUIC employs a credit-based flow-control scheme similar to HTTP/2's flow control
{{?HTTP2}}.  A receiver advertises the number of octets it is prepared to
receive on a given stream and for the entire connection.  This leads to two
levels of flow control in QUIC: (i) Connection flow control, which prevents
senders from exceeding a receiver's buffer capacity for the connection, and (ii)
Stream flow control, which prevents a single stream from consuming the entire
receive buffer for a connection.

A data receiver sends MAX_STREAM_DATA or MAX_DATA frames to the sender
to advertise additional credit. MAX_STREAM_DATA frames send the the
maximum absolute byte offset of a stream, while MAX_DATA sends the
maximum sum of the absolute byte offsets of all streams other than
stream 0.

A receiver MAY advertise a larger offset at any point by sending MAX_DATA or
MAX_STREAM_DATA frames.  A receiver MUST NOT renege on an advertisement; that
is, once a receiver advertises an offset, it MUST NOT subsequently advertise a
smaller offset.  A sender could receive MAX_DATA or MAX_STREAM_DATA frames out
of order; a sender MUST therefore ignore any flow control offset that does not
move the window forward.

A receiver MUST close the connection with a FLOW_CONTROL_ERROR error
({{error-handling}}) if the peer violates the advertised connection or stream
data limits.

A sender SHOULD send BLOCKED or STREAM_BLOCKED frames to indicate it has data to
write but is blocked by flow control limits.  These frames are expected to be
sent infrequently in common cases, but they are considered useful for debugging
and monitoring purposes.

A receiver advertises credit for a stream by sending a MAX_STREAM_DATA frame
with the Stream ID set appropriately. A receiver could use the current offset of
data consumed to determine the flow control offset to be advertised.  A receiver
MAY send MAX_STREAM_DATA frames in multiple packets in order to make sure that
the sender receives an update before running out of flow control credit, even if
one of the packets is lost.

Connection flow control is a limit to the total bytes of stream data sent in
STREAM frames on all streams except stream 0.  A receiver advertises credit for
a connection by sending a MAX_DATA frame.  A receiver maintains a cumulative sum
of bytes received on all contributing streams, which are used to check for flow
control violations. A receiver might use a sum of bytes consumed on all
contributing streams to determine the maximum data limit to be advertised.

## Edge Cases and Other Considerations

There are some edge cases which must be considered when dealing with stream and
connection level flow control.  Given enough time, both endpoints must agree on
flow control state.  If one end believes it can send more than the other end is
willing to receive, the connection will be torn down when too much data arrives.

Conversely if a sender believes it is blocked, while endpoint B expects more
data can be received, then the connection can be in a deadlock, with the sender
waiting for a MAX_DATA or MAX_STREAM_DATA frame which will never come.

On receipt of a RST_STREAM frame, an endpoint will tear down state for the
matching stream and ignore further data arriving on that stream.  This could
result in the endpoints getting out of sync, since the RST_STREAM frame may have
arrived out of order and there may be further bytes in flight.  The data sender
would have counted the data against its connection level flow control budget,
but a receiver that has not received these bytes would not know to include them
as well.  The receiver must learn the number of bytes that were sent on the
stream to make the same adjustment in its connection flow controller.

To avoid this de-synchronization, a RST_STREAM sender MUST include the final
byte offset sent on the stream in the RST_STREAM frame.  On receiving a
RST_STREAM frame, a receiver definitively knows how many bytes were sent on that
stream before the RST_STREAM frame, and the receiver MUST use the final offset
to account for all bytes sent on the stream in its connection level flow
controller.

### Response to a RST_STREAM

RST_STREAM terminates one direction of a stream abruptly.  Whether any action or
response can or should be taken on the data already received is an
application-specific issue, but it will often be the case that upon receipt of a
RST_STREAM an endpoint will choose to stop sending data in its own direction. If
the sender of a RST_STREAM wishes to explicitly state that no future data will
be processed, that endpoint MAY send a STOP_SENDING frame at the same time.

### Data Limit Increments {#fc-credit}

This document leaves when and how many bytes to advertise in a MAX_DATA or
MAX_STREAM_DATA to implementations, but offers a few considerations.  These
frames contribute to connection overhead.  Therefore frequently sending frames
with small changes is undesirable.  At the same time, infrequent updates require
larger increments to limits if blocking is to be avoided.  Thus, larger updates
require a receiver to commit to larger resource commitments.  Thus there is a
tradeoff between resource commitment and overhead when determining how large a
limit is advertised.

A receiver MAY use an autotuning mechanism to tune the frequency and amount that
it increases data limits based on a round-trip time estimate and the rate at
which the receiving application consumes data, similar to common TCP
implementations.

### Handshake Exemption

During the initial handshake, an endpoint could need to send a larger message on
stream 0 than would ordinarily be permitted by the peer's initial stream flow
control window. Since MAX_STREAM_DATA frames are not permitted in these early
packets, the peer cannot provide additional flow control window in order to
complete the handshake.

Endpoints MAY exceed the flow control limits on stream 0 prior to the completion
of the cryptographic handshake.  (That is, in Initial, Retry, and Handshake
packets.)  However, once the handshake is complete, endpoints MUST NOT send
additional data beyond the peer's permitted offset.  If the amount of data sent
during the handshake exceeds the peer's maximum offset, the endpoint cannot send
additional data on stream 0 until the peer has sent a MAX_STREAM_DATA frame
indicating a larger maximum offset.

## Stream Limit Increment

As with flow control, this document leaves when and how many streams to make
available to a peer via MAX_STREAM_ID to implementations, but offers a few
considerations.  MAX_STREAM_ID frames constitute minimal overhead, while
withholding MAX_STREAM_ID frames can prevent the peer from using the available
parallelism.

Implementations will likely want to increase the maximum stream ID as
peer-initiated streams close.  A receiver MAY also advance the maximum stream ID
based on current activity, system conditions, and other environmental factors.


### Blocking on Flow Control {#blocking}

If a sender does not receive a MAX_DATA or MAX_STREAM_DATA frame when it has run
out of flow control credit, the sender will be blocked and SHOULD send a BLOCKED
or STREAM_BLOCKED frame.  These frames are expected to be useful for debugging
at the receiver; they do not require any other action.  A receiver SHOULD NOT
wait for a BLOCKED or STREAM_BLOCKED frame before sending MAX_DATA or
MAX_STREAM_DATA, since doing so will mean that a sender is unable to send for an
entire round trip.

For smooth operation of the congestion controller, it is generally considered
best to not let the sender go into quiescence if avoidable.  To avoid blocking a
sender, and to reasonably account for the possibiity of loss, a receiver should
send a MAX_DATA or MAX_STREAM_DATA frame at least two round trips before it
expects the sender to get blocked.

A sender sends a single BLOCKED or STREAM_BLOCKED frame only once when it
reaches a data limit.  A sender SHOULD NOT send multiple BLOCKED or
STREAM_BLOCKED frames for the same data limit, unless the original frame is
determined to be lost.  Another BLOCKED or STREAM_BLOCKED frame can be sent
after the data limit is increased.


## Stream Final Offset {#final-offset}

The final offset is the count of the number of octets that are transmitted on a
stream.  For a stream that is reset, the final offset is carried explicitly in
a RST_STREAM frame.  Otherwise, the final offset is the offset of the end of the
data carried in a STREAM frame marked with a FIN flag, or 0 in the case of
incoming unidirectional streams.

An endpoint will know the final offset for a stream when the receive stream
enters the "Size Known" or "Reset Recvd" state.

An endpoint MUST NOT send data on a stream at or beyond the final offset.

Once a final offset for a stream is known, it cannot change.  If a RST_STREAM or
STREAM frame causes the final offset to change for a stream, an endpoint SHOULD
respond with a FINAL_OFFSET_ERROR error (see {{error-handling}}).  A receiver
SHOULD treat receipt of data at or beyond the final offset as a
FINAL_OFFSET_ERROR error, even after a stream is closed.  Generating these
errors is not mandatory, but only because requiring that an endpoint generate
these errors also means that the endpoint needs to maintain the final offset
state for closed streams, which could mean a significant state commitment.


# Error Handling

An endpoint that detects an error SHOULD signal the existence of that error to
its peer.  Both transport-level and application-level errors can affect an
entire connection (see {{connection-errors}}), while only application-level
errors can be isolated to a single stream (see {{stream-errors}}).

The most appropriate error code ({{error-codes}}) SHOULD be included in the
frame that signals the error.  Where this specification identifies error
conditions, it also identifies the error code that is used.

A stateless reset ({{stateless-reset}}) is not suitable for any error that can
be signaled with a CONNECTION_CLOSE, APPLICATION_CLOSE, or RST_STREAM frame.  A
stateless reset MUST NOT be used by an endpoint that has the state necessary to
send a frame on the connection.


## Connection Errors

Errors that result in the connection being unusable, such as an obvious
violation of protocol semantics or corruption of state that affects an entire
connection, MUST be signaled using a CONNECTION_CLOSE or APPLICATION_CLOSE frame
({{frame-connection-close}}, {{frame-application-close}}). An endpoint MAY close
the connection in this manner even if the error only affects a single stream.

Application protocols can signal application-specific protocol errors using the
APPLICATION_CLOSE frame.  Errors that are specific to the transport, including
all those described in this document, are carried in a CONNECTION_CLOSE frame.
Other than the type of error code they carry, these frames are identical in
format and semantics.

A CONNECTION_CLOSE or APPLICATION_CLOSE frame could be sent in a packet that is
lost.  An endpoint SHOULD be prepared to retransmit a packet containing either
frame type if it receives more packets on a terminated connection.  Limiting the
number of retransmissions and the time over which this final packet is sent
limits the effort expended on terminated connections.

An endpoint that chooses not to retransmit packets containing CONNECTION_CLOSE
or APPLICATION_CLOSE risks a peer missing the first such packet.  The only
mechanism available to an endpoint that continues to receive data for a
terminated connection is to use the stateless reset process
({{stateless-reset}}).

An endpoint that receives an invalid CONNECTION_CLOSE or APPLICATION_CLOSE frame
MUST NOT signal the existence of the error to its peer.


## Stream Errors

If an application-level error affects a single stream, but otherwise leaves the
connection in a recoverable state, the endpoint can send a RST_STREAM frame
({{frame-rst-stream}}) with an appropriate error code to terminate just the
affected stream.

Stream 0 is critical to the functioning of the entire connection.  If stream 0
is closed with either a RST_STREAM or STREAM frame bearing the FIN flag, an
endpoint MUST generate a connection error of type PROTOCOL_VIOLATION.

Other than STOPPING ({{solicited-state-transitions}}), RST_STREAM MUST be
instigated by the application and MUST carry an application error code.
Resetting a stream without knowledge of the application protocol could cause the
protocol to enter an unrecoverable state.  Application protocols might require
certain streams to be reliably delivered in order to guarantee consistent state
between endpoints.


## Transport Error Codes {#error-codes}

QUIC error codes are 16-bit unsigned integers.

This section lists the defined QUIC transport error codes that may be used in a
CONNECTION_CLOSE frame.  These errors apply to the entire connection.

NO_ERROR (0x0):

: An endpoint uses this with CONNECTION_CLOSE to signal that the connection is
  being closed abruptly in the absence of any error.

INTERNAL_ERROR (0x1):

: The endpoint encountered an internal error and cannot continue with the
  connection.

SERVER_BUSY (0x2):

: The server is currently busy and does not accept any new connections.

FLOW_CONTROL_ERROR (0x3):

: An endpoint received more data than it permitted in its advertised data limits
  (see {{flow-control}}).

STREAM_ID_ERROR (0x4):

: An endpoint received a frame for a stream identifier that exceeded its
  advertised maximum stream ID.

STREAM_STATE_ERROR (0x5):

: An endpoint received a frame for a stream that was not in a state that
  permitted that frame (see {{stream-states}}).

FINAL_OFFSET_ERROR (0x6):

: An endpoint received a STREAM frame containing data that exceeded the
  previously established final offset.  Or an endpoint received a RST_STREAM
  frame containing a final offset that was lower than the maximum offset of data
  that was already received.  Or an endpoint received a RST_STREAM frame
  containing a different final offset to the one already established.

FRAME_FORMAT_ERROR (0x7):

: An endpoint received a frame that was badly formatted.  For instance, an empty
  STREAM frame that omitted the FIN flag, or an ACK frame that has more
  acknowledgment ranges than the remainder of the packet could carry.  This is a
  generic error code; an endpoint SHOULD use the more specific frame format
  error codes (0x1XX) if possible.

TRANSPORT_PARAMETER_ERROR (0x8):

: An endpoint received transport parameters that were badly formatted, included
  an invalid value, was absent even though it is mandatory, was present though
  it is forbidden, or is otherwise in error.

VERSION_NEGOTIATION_ERROR (0x9):

: An endpoint received transport parameters that contained version negotiation
  parameters that disagreed with the version negotiation that it performed.
  This error code indicates a potential version downgrade attack.

PROTOCOL_VIOLATION (0xA):

: An endpoint detected an error with protocol compliance that was not covered by
  more specific error codes.

UNSOLICITED_PATH_RESPONSE (0xB):

: An endpoint received a PATH_RESPONSE frame that did not correspond to any
  PATH_CHALLENGE frame that it previously sent.

FRAME_ERROR (0x1XX):

: An endpoint detected an error in a specific frame type.  The frame type is
  included as the last octet of the error code.  For example, an error in a
  MAX_STREAM_ID frame would be indicated with the code (0x106).

Codes for errors occuring when TLS is used for the crypto handshake are defined
in Section 11 of {{QUIC-TLS}}. See {{iana-error-codes}} for details of
registering new error codes.


## Application Protocol Error Codes {#app-error-codes}

Application protocol error codes are 16-bit unsigned integers, but the
management of application error codes are left to application protocols.
Application protocol error codes are used for the RST_STREAM
({{frame-rst-stream}}) and APPLICATION_CLOSE ({{frame-application-close}})
frames.

There is no restriction on the use of the 16-bit error code space for
application protocols.  However, QUIC reserves the error code with a value of 0
to mean STOPPING.  The application error code of STOPPING (0) is used by the
transport to cancel a stream in response to receipt of a STOP_SENDING frame.


# Security Considerations

## Handshake Denial of Service

As an encrypted and authenticated transport QUIC provides a range of protections
against denial of service.  Once the cryptographic handshake is complete, QUIC
endpoints discard most packets that are not authenticated, greatly limiting the
ability of an attacker to interfere with existing connections.

Once a connection is established QUIC endpoints might accept some
unauthenticated ICMP packets (see {{v4-pmtud}}), but the use of these packets is
extremely limited.  The only other type of packet that an endpoint might accept
is a stateless reset ({{stateless-reset}}) which relies on the token being kept
secret until it is used.

During the creation of a connection, QUIC only provides protection against
attack from off the network path.  All QUIC packets contain proof that the
recipient saw a preceding packet from its peer.

The first mechanism used is the source and destination connection IDs, which are
required to match those set by a peer.  Except for an Initial and stateless
reset packets, an endpoint only accepts packets that include a destination
connection that matches a connection ID the endpoint previously chose.  This is
the only protection offered for Version Negotiation packets.

The destination connection ID in an Initial packet is selected by a client to be
unpredictable, which serves an additional purpose.  The packets that carry the
cryptographic handshake are protected with a key that is derived from this
connection ID and salt specific to the QUIC version.  This allows endpoints to
use the same process for authenticating packets that they receive as they use
after the cryptographic handshake completes.  Packets that cannot be
authenticated are discarded.  Protecting packets in this fashion provides a
strong assurance that the sender of the packet saw the Initial packet and
understood it.

These protections are not intended to be effective against an attacker that is
able to receive QUIC packets prior to the connection being established.  Such an
attacker can potentially send packets that will be accepted by QUIC endpoints.
This version of QUIC attempts to detect this sort of attack, but it expects that
endpoints will fail to establish a connection rather than recovering.  For the
most part, the cryptographic handshake protocol {{QUIC-TLS}} is responsible for
detecting tampering during the handshake, though additional validation is
required for version negotiation (see {{version-validation}}).

Endpoints are permitted to use other methods to detect and attempt to recover
from interference with the handshake.  Invalid packets may be identified and
discarded using other methods, but no specific method is mandated in this
document.


## Spoofed ACK Attack

An attacker might be able to receive an address validation token
({{address-validation}}) from the server and then release the IP address it
used to acquire that token.  The attacker may, in the future, spoof this same
address (which now presumably addresses a different endpoint), and initiate a
0-RTT connection with a server on the victim's behalf.  The attacker can then
spoof ACK frames to the server which cause the server to send excessive amounts
of data toward the new owner of the IP address.

There are two possible mitigations to this attack.  The simplest one is that a
server can unilaterally create a gap in packet-number space.  In the non-attack
scenario, the client will send an ACK frame with the larger value for largest
acknowledged.  In the attack scenario, the attacker could acknowledge a packet
in the gap.  If the server sees an acknowledgment for a packet that was never
sent, the connection can be aborted.

The second mitigation is that the server can require that acknowledgments for
sent packets match the encryption level of the sent packet.  This mitigation is
useful if the connection has an ephemeral forward-secure key that is generated
and used for every new connection.  If a packet sent is protected with a
forward-secure key, then any acknowledgments that are received for them MUST
also be forward-secure protected.  Since the attacker will not have the forward
secure key, the attacker will not be able to generate forward-secure protected
packets with ACK frames.


## Optimistic ACK Attack

An endpoint that acknowledges packets it has not received might cause a
congestion controller to permit sending at rates beyond what the network
supports.  An endpoint MAY skip packet numbers when sending packets to detect
this behavior.  An endpoint can then immediately close the connection with a
connection error of type PROTOCOL_VIOLATION (see {{immediate-close}}).


## Slowloris Attacks

The attacks commonly known as Slowloris {{SLOWLORIS}} try to keep many
connections to the target endpoint open and hold them open as long as possible.
These attacks can be executed against a QUIC endpoint by generating the minimum
amount of activity necessary to avoid being closed for inactivity.  This might
involve sending small amounts of data, gradually opening flow control windows in
order to control the sender rate, or manufacturing ACK frames that simulate a
high loss rate.

QUIC deployments SHOULD provide mitigations for the Slowloris attacks, such as
increasing the maximum number of clients the server will allow, limiting the
number of connections a single IP address is allowed to make, imposing
restrictions on the minimum transfer speed a connection is allowed to have, and
restricting the length of time an endpoint is allowed to stay connected.


## Stream Fragmentation and Reassembly Attacks

An adversarial endpoint might intentionally fragment the data on stream buffers
in order to cause disproportionate memory commitment.  An adversarial endpoint
could open a stream and send some STREAM frames containing arbitrary fragments
of the stream content.

The attack is mitigated if flow control windows correspond to available
memory.  However, some receivers will over-commit memory and advertise flow
control offsets in the aggregate that exceed actual available memory.  The
over-commitment strategy can lead to better performance when endpoints are well
behaved, but renders endpoints vulnerable to the stream fragmentation attack.

QUIC deployments SHOULD provide mitigations against the stream fragmentation
attack.  Mitigations could consist of avoiding over-committing memory, delaying
reassembly of STREAM frames, implementing heuristics based on the age and
duration of reassembly holes, or some combination.


## Stream Commitment Attack

An adversarial endpoint can open lots of streams, exhausting state on an
endpoint.  The adversarial endpoint could repeat the process on a large number
of connections, in a manner similar to SYN flooding attacks in TCP.

Normally, clients will open streams sequentially, as explained in {{stream-id}}.
However, when several streams are initiated at short intervals, transmission
error may cause STREAM DATA frames opening streams to be received out of
sequence.  A receiver is obligated to open intervening streams if a
higher-numbered stream ID is received.  Thus, on a new connection, opening
stream 2000001 opens 1 million streams, as required by the specification.

The number of active streams is limited by the concurrent stream limit transport
parameter, as explained in {{stream-concurrency}}.  If chosen judisciously, this
limit mitigates the effect of the stream commitment attack.  However, setting
the limit too low could affect performance when applications expect to open
large number of streams.


# IANA Considerations

## QUIC Transport Parameter Registry {#iana-transport-parameters}

IANA \[SHALL add/has added] a registry for "QUIC Transport Parameters" under a
"QUIC Protocol" heading.

The "QUIC Transport Parameters" registry governs a 16-bit space.  This space is
split into two spaces that are governed by different policies.  Values with the
first byte in the range 0x00 to 0xfe (in hexadecimal) are assigned via the
Specification Required policy {{!RFC8126}}.  Values with the first byte 0xff are
reserved for Private Use {{!RFC8126}}.

Registrations MUST include the following fields:

Value:

: The numeric value of the assignment (registrations will be between 0x0000 and
  0xfeff).

Parameter Name:

: A short mnemonic for the parameter.

Specification:

: A reference to a publicly available specification for the value.


The nominated expert(s) verify that a specification exists and is readily
accessible.  The expert(s) are encouraged to be biased towards approving
registrations unless they are abusive, frivolous, or actively harmful (not
merely aesthetically displeasing, or architecturally dubious).

The initial contents of this registry are shown in {{iana-tp-table}}.

| Value  | Parameter Name             | Specification                       |
|:-------|:---------------------------|:------------------------------------|
| 0x0000 | initial_max_stream_data    | {{transport-parameter-definitions}} |
| 0x0001 | initial_max_data           | {{transport-parameter-definitions}} |
| 0x0002 | initial_max_bidi_streams   | {{transport-parameter-definitions}} |
| 0x0003 | idle_timeout               | {{transport-parameter-definitions}} |
| 0x0004 | preferred_address          | {{transport-parameter-definitions}} |
| 0x0005 | max_packet_size            | {{transport-parameter-definitions}} |
| 0x0006 | stateless_reset_token      | {{transport-parameter-definitions}} |
| 0x0007 | ack_delay_exponent         | {{transport-parameter-definitions}} |
| 0x0008 | initial_max_uni_streams    | {{transport-parameter-definitions}} |
{: #iana-tp-table title="Initial QUIC Transport Parameters Entries"}


## QUIC Transport Error Codes Registry {#iana-error-codes}

IANA \[SHALL add/has added] a registry for "QUIC Transport Error Codes" under a
"QUIC Protocol" heading.

The "QUIC Transport Error Codes" registry governs a 16-bit space.  This space is
split into two spaces that are governed by different policies.  Values with the
first byte in the range 0x00 to 0xfe (in hexadecimal) are assigned via the
Specification Required policy {{!RFC8126}}.  Values with the first byte 0xff are
reserved for Private Use {{!RFC8126}}.

Registrations MUST include the following fields:

Value:

: The numeric value of the assignment (registrations will be between 0x0000 and
  0xfeff).

Code:

: A short mnemonic for the parameter.

Description:

: A brief description of the error code semantics, which MAY be a summary if a
  specification reference is provided.

Specification:

: A reference to a publicly available specification for the value.

The initial contents of this registry are shown in {{iana-error-table}}.  Note
that FRAME_ERROR takes the range from 0x100 to 0x1FF and private use occupies
the range from 0xFE00 to 0xFFFF.

| Value       | Error                     | Description                   | Specification   |
|:------------|:--------------------------|:------------------------------|:----------------|
| 0x0         | NO_ERROR                  | No error                      | {{error-codes}} |
| 0x1         | INTERNAL_ERROR            | Implementation error          | {{error-codes}} |
| 0x2         | SERVER_BUSY               | Server currently busy         | {{error-codes}} |
| 0x3         | FLOW_CONTROL_ERROR        | Flow control error            | {{error-codes}} |
| 0x4         | STREAM_ID_ERROR           | Invalid stream ID             | {{error-codes}} |
| 0x5         | STREAM_STATE_ERROR        | Frame received in invalid stream state | {{error-codes}} |
| 0x6         | FINAL_OFFSET_ERROR        | Change to final stream offset | {{error-codes}} |
| 0x7         | FRAME_FORMAT_ERROR        | Generic frame format error    | {{error-codes}} |
| 0x8         | TRANSPORT_PARAMETER_ERROR | Error in transport parameters | {{error-codes}} |
| 0x9         | VERSION_NEGOTIATION_ERROR | Version negotiation failure   | {{error-codes}} |
| 0xA         | PROTOCOL_VIOLATION        | Generic protocol violation    | {{error-codes}} |
| 0xB         | UNSOLICITED_PATH_RESPONSE | Unsolicited PATH_RESPONSE frame | {{error-codes}} |
| 0x100-0x1FF | FRAME_ERROR               | Specific frame format error   | {{error-codes}} |
{: #iana-error-table title="Initial QUIC Transport Error Codes Entries"}


--- back

# 변경 내역

> **RFC 편집자의 노트:** 본 문서의 최종 버전 출판 전에 본 절을 삭제하시오.

문제와 pull 요청 번호가 넘버 기호 (octothorp)와 함께 나열되었다.

## Since draft-ietf-quic-transport-11

- Enable server to transition connections to a preferred address (#560, #1251)
- Packet numbers are encrypted (#1174, #1043, #1048, #1034, #850, #990, #734,
  #1079)
- Packet numbers use a variable-length encoding (#989, #1334)
- STREAM frames can now be empty (#1350)

## Since draft-ietf-quic-transport-10

- Swap payload length and packed number fields in long header (#1294)
- Clarified that CONNECTION_CLOSE is allowed in Handshake packet (#1274)
- Spin bit reserved (#1283)
- Coalescing multiple QUIC packets in a UDP datagram (#1262, #1285)
- A more complete connection migration (#1249)
- Refine opportunistic ACK defense text (#305, #1030, #1185)
- A Stateless Reset Token isn't mandatory (#818, #1191)
- Removed implicit stream opening (#896, #1193)
- An empty STREAM frame can be used to open a stream without sending data (#901,
  #1194)
- Define stream counts in transport parameters rather than a maximum stream ID
  (#1023, #1065)
- STOP_SENDING is now prohibited before streams are used (#1050)
- Recommend including ACK in Retry packets and allow PADDING (#1067, #882)
- Endpoints now become closing after an idle timeout (#1178, #1179)
- Remove implication that Version Negotiation is sent when a packet of the wrong
  version is received (#1197)

## Since draft-ietf-quic-transport-09

- Added PATH_CHALLENGE and PATH_RESPONSE frames to replace PING with Data and
  PONG frame. Changed ACK frame type from 0x0e to 0x0d. (#1091, #725, #1086)
- A server can now only send 3 packets without validating the client address
  (#38, #1090)
- Delivery order of stream data is no longer strongly specified (#252, #1070)
- Rework of packet handling and version negotiation (#1038)
- Stream 0 is now exempt from flow control until the handshake completes (#1074,
  #725, #825, #1082)
- Improved retransmission rules for all frame types: information is
  retransmitted, not packets or frames (#463, #765, #1095, #1053)
- Added an error code for server busy signals (#1137)

- Endpoints now set the connection ID that their peer uses.  Connection IDs are
  variable length.  Removed the omit_connection_id transport parameter and the
  corresponding short header flag. (#1089, #1052, #1146, #821, #745, #821,
  #1166, #1151)

## Since draft-ietf-quic-transport-08

- Clarified requirements for BLOCKED usage (#65,  #924)
- BLOCKED frame now includes reason for blocking (#452, #924, #927, #928)
- GAP limitation in ACK Frame (#613)
- Improved PMTUD description (#614, #1036)
- Clarified stream state machine (#634, #662, #743, #894)
- Reserved versions don't need to be generated deterministically (#831, #931)
- You don't always need the draining period (#871)
- Stateless reset clarified as version-specific (#930, #986)
- initial_max_stream_id_x transport parameters are optional (#970, #971)
- Ack Delay assumes a default value during the handshake (#1007, #1009)
- Removed transport parameters from NewSessionTicket (#1015)

## Since draft-ietf-quic-transport-07

- The long header now has version before packet number (#926, #939)
- Rename and consolidate packet types (#846, #822, #847)
- Packet types are assigned new codepoints and the Connection ID Flag is
  inverted (#426, #956)
- Removed type for Version Negotiation and use Version 0 (#963, #968)
- Streams are split into unidirectional and bidirectional (#643, #656, #720,
  #872, #175, #885)
  * Stream limits now have separate uni- and bi-directinal transport parameters
    (#909, #958)
  * Stream limit transport parameters are now optional and default to 0 (#970,
    #971)
- The stream state machine has been split into read and write (#634, #894)
- Employ variable-length integer encodings throughout (#595)
- Improvements to connection close
  * Added distinct closing and draining states (#899, #871)
  * Draining period can terminate early (#869, #870)
  * Clarifications about stateless reset (#889, #890)
- Address validation for connection migration (#161, #732, #878)
- Clearly defined retransmission rules for BLOCKED (#452, #65, #924)
- negotiated_version is sent in server transport parameters (#710, #959)
- Increased the range over which packet numbers are randomized (#864, #850,
  #964)

## Since draft-ietf-quic-transport-06

- Replaced FNV-1a with AES-GCM for all "Cleartext" packets (#554)
- Split error code space between application and transport (#485)
- Stateless reset token moved to end (#820)
- 1-RTT-protected long header types removed (#848)
- No acknowledgments during draining period (#852)
- Remove "application close" as a separate close type (#854)
- Remove timestamps from the ACK frame (#841)
- Require transport parameters to only appear once (#792)

## Since draft-ietf-quic-transport-05

- Stateless token is server-only (#726)
- Refactor section on connection termination (#733, #748, #328, #177)
- Limit size of Version Negotiation packet (#585)
- Clarify when and what to ack (#736)
- Renamed STREAM_ID_NEEDED to STREAM_ID_BLOCKED
- Clarify Keep-alive requirements (#729)

## Since draft-ietf-quic-transport-04

- Introduce STOP_SENDING frame, RST_STREAM only resets in one direction (#165)
- Removed GOAWAY; application protocols are responsible for graceful shutdown
  (#696)
- Reduced the number of error codes (#96, #177, #184, #211)
- Version validation fields can't move or change (#121)
- Removed versions from the transport parameters in a NewSessionTicket message
  (#547)
- Clarify the meaning of "bytes in flight" (#550)
- Public reset is now stateless reset and not visible to the path (#215)
- Reordered bits and fields in STREAM frame (#620)
- Clarifications to the stream state machine (#572, #571)
- Increased the maximum length of the Largest Acknowledged field in ACK frames
  to 64 bits (#629)
- truncate_connection_id is renamed to omit_connection_id (#659)
- CONNECTION_CLOSE terminates the connection like TCP RST (#330, #328)
- Update labels used in HKDF-Expand-Label to match TLS 1.3 (#642)

## Since draft-ietf-quic-transport-03

- Change STREAM and RST_STREAM layout
- Add MAX_STREAM_ID settings

## Since draft-ietf-quic-transport-02

- The size of the initial packet payload has a fixed minimum (#267, #472)
- Define when Version Negotiation packets are ignored (#284, #294, #241, #143,
  #474)
- The 64-bit FNV-1a algorithm is used for integrity protection of unprotected
  packets (#167, #480, #481, #517)
- Rework initial packet types to change how the connection ID is chosen (#482,
  #442, #493)
- No timestamps are forbidden in unprotected packets (#542, #429)
- Cryptographic handshake is now on stream 0 (#456)
- Remove congestion control exemption for cryptographic handshake (#248, #476)
- Version 1 of QUIC uses TLS; a new version is needed to use a different
  handshake protocol (#516)
- STREAM frames have a reduced number of offset lengths (#543, #430)
- Split some frames into separate connection- and stream- level frames
  (#443)
  - WINDOW_UPDATE split into MAX_DATA and MAX_STREAM_DATA (#450)
  - BLOCKED split to match WINDOW_UPDATE split (#454)
  - Define STREAM_ID_NEEDED frame (#455)
- A NEW_CONNECTION_ID frame supports connection migration without linkability
  (#232, #491, #496)
- Transport parameters for 0-RTT are retained from a previous connection (#405,
  #513, #512)
  - A client in 0-RTT no longer required to reset excess streams (#425, #479)
- Expanded security considerations (#440, #444, #445, #448)


## Since draft-ietf-quic-transport-01

- Defined short and long packet headers (#40, #148, #361)
- Defined a versioning scheme and stable fields (#51, #361)
- Define reserved version values for "greasing" negotiation (#112, #278)
- The initial packet number is randomized (#35, #283)
- Narrow the packet number encoding range requirement (#67, #286, #299, #323,
  #356)

- Defined client address validation (#52, #118, #120, #275)
- Define transport parameters as a TLS extension (#49, #122)
- SCUP and COPT parameters are no longer valid (#116, #117)
- Transport parameters for 0-RTT are either remembered from before, or assume
  default values (#126)
- The server chooses connection IDs in its final flight (#119, #349, #361)
- The server echoes the Connection ID and packet number fields when sending a
  Version Negotiation packet (#133, #295, #244)

- Defined a minimum packet size for the initial handshake packet from the client
  (#69, #136, #139, #164)
- Path MTU Discovery (#64, #106)
- The initial handshake packet from the client needs to fit in a single packet
  (#338)

- Forbid acknowledgment of packets containing only ACK and PADDING (#291)
- Require that frames are processed when packets are acknowledged (#381, #341)
- Removed the STOP_WAITING frame (#66)
- Don't require retransmission of old timestamps for lost ACK frames (#308)
- Clarified that frames are not retransmitted, but the information in them can
  be (#157, #298)

- Error handling definitions (#335)
- Split error codes into four sections (#74)
- Forbid the use of Public Reset where CONNECTION_CLOSE is possible (#289)

- Define packet protection rules (#336)

- Require that stream be entirely delivered or reset, including acknowledgment
  of all STREAM frames or the RST_STREAM, before it closes (#381)
- Remove stream reservation from state machine (#174, #280)
- Only stream 1 does not contribute to connection-level flow control (#204)
- Stream 1 counts towards the maximum concurrent stream limit (#201, #282)
- Remove connection-level flow control exclusion for some streams (except 1)
  (#246)
- RST_STREAM affects connection-level flow control (#162, #163)
- Flow control accounting uses the maximum data offset on each stream, rather
  than bytes received (#378)

- Moved length-determining fields to the start of STREAM and ACK (#168, #277)
- Added the ability to pad between frames (#158, #276)
- Remove error code and reason phrase from GOAWAY (#352, #355)
- GOAWAY includes a final stream number for both directions (#347)
- Error codes for RST_STREAM and CONNECTION_CLOSE are now at a consistent offset
  (#249)

- Defined priority as the responsibility of the application protocol (#104,
  #303)


## Since draft-ietf-quic-transport-00

- Replaced DIVERSIFICATION_NONCE flag with KEY_PHASE flag
- Defined versioning
- Reworked description of packet and frame layout
- Error code space is divided into regions for each component
- Use big endian for all numeric values


## Since draft-hamilton-quic-transport-protocol-01

- Adopted as base for draft-ietf-quic-tls
- Updated authors/editors list
- Added IANA Considerations section
- Moved Contributors and Acknowledgments to appendices


# 사사 (Acknowledgements)
{:numbered="false"}

IETF 전의 QUIC을 구성하고 이를 배포하는데 도움을 준 다음 사람들에게 특히
감사한다: Chris Bentzel, Misha Efimov, Roberto Peon, Alistair Riddoch,
Siddharth Vijayakrishnan, 그리고 Assar Westerlund.

이 문서는 여러 개별 토론과 quic@ietf.org 및 proto-quic@chromium.org 메일링
리스트에서 공개적인 토론에서 혜택을 보았다. 모든 이에게 감사한다.


# 기여자
{:numbered="false"}

이 명세의 원 저자는 Ryan Hamilton, Jana Iyengar, Ian Swett, 그리고 Alyssa
Wilk이다.

이 프로토콜의 원 설계와 프로토콜 설계 근거는 Jim Roskind의 작업
{{EARLY-DESIGN}}에서 상당 부분 도출되었다. 알파벳 순서로, 구글에서 진행된
IETF 전의 QUIC 프로젝트의 기여자는 다음과 같다.: Britt Cyr, Jeremy Dorfman,
Ryan Hamilton, Jana Iyengar, Fedor Kouranov, Charles Krasic, Jo Kulik, Adam
Langley, Jim Roskind, Robbie Shade, Satyam Shekhar, Cherie Shi, Ian Swett,
Raman Tenneti, Victor Vasiliev, Antonio Vicente, Patrik Westin, Alyssa Wilk,
Dale Worley, Fan Yang, Dan Zhang, Daniel Ziegler.
