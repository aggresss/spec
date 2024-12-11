# Multipath RTP (MPRTP)

> Origin [https://datatracker.ietf.org/doc/html/draft-ietf-avtcore-mprtp-03](https://datatracker.ietf.org/doc/html/draft-ietf-avtcore-mprtp-03)

## Abstract

The Real-time Transport Protocol (RTP) is used to deliver real-time content and, along with the RTP Control Protocol (RTCP), forms the control channel between the sender and receiver.  However, RTP and RTCP assume a single delivery path between the sender and receiver and make decisions based on the measured characteristics of this single path.  Increasingly, endpoints are becoming multi-homed, which means that they are connected via multiple Internet paths.  Network utilization can be improved when endpoints use multiple parallel paths for communication.  The resulting increase in reliability and throughput can also enhance the user experience.  This document extends the Real-time Transport Protocol (RTP) so that a single session can take advantage of the availability of multiple paths between two endpoints.

## Status of This Memo

This Internet-Draft is submitted in full conformance with the provisions of BCP 78 and BCP 79.

Internet-Drafts are working documents of the Internet Engineering Task Force (IETF).  Note that other groups may also distribute working documents as Internet-Drafts.  The list of current Internet-Drafts is at http://datatracker.ietf.org/drafts/current/.

Internet-Drafts are draft documents valid for a maximum of six months and may be updated, replaced, or obsoleted by other documents at any time.  It is inappropriate to use Internet-Drafts as reference material or to cite them other than as "work in progress."

This Internet-Draft will expire on January 9, 2017.

## Copyright Notice

Copyright (c) 2016 IETF Trust and the persons identified as the document authors.  All rights reserved.

This document is subject to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF Documents (http://trustee.ietf.org/license-info) in effect on the date of publication of this document.  Please review these documents carefully, as they describe your rights and restrictions with respect to this document.  Code Components extracted from this document must include Simplified BSD License text as described in Section 4.e of the Trust Legal Provisions and are provided without warranty as described in the Simplified BSD License.

## Table of Contents

```
   1.  Introduction  . . . . . . . . . . . . . . . . . . . . . . . .  40
     1.1.  Requirements Language . . . . . . . . . . . . . . . . . .   4
     1.2.  Terminology . . . . . . . . . . . . . . . . . . . . . . .   4
     1.3.  Use-cases . . . . . . . . . . . . . . . . . . . . . . . .   5
   2.  Goals . . . . . . . . . . . . . . . . . . . . . . . . . . . .   6
     2.1.  Functional goals  . . . . . . . . . . . . . . . . . . . .   6
     2.2.  Compatibility goals . . . . . . . . . . . . . . . . . . .   6
   3.  RTP Topologies  . . . . . . . . . . . . . . . . . . . . . . .   7
   4.  MPRTP Architecture  . . . . . . . . . . . . . . . . . . . . .   7
   5.  Example Media Flow Diagrams . . . . . . . . . . . . . . . . .   8
     5.1.  Streaming use-case  . . . . . . . . . . . . . . . . . . .   8
     5.2.  Conversational use-case . . . . . . . . . . . . . . . . .   9
   6.  MPRTP Functional Blocks . . . . . . . . . . . . . . . . . . .  10
   7.  Available Mechanisms within the Functional Blocks . . . . . .  11
     7.1.  Session Setup . . . . . . . . . . . . . . . . . . . . . .  11
       7.1.1.  Connectivity Checks . . . . . . . . . . . . . . . . .  11
       7.1.2.  Adding New or Updating Interfaces . . . . . . . . . .  12
       7.1.3.  In-band vs. Out-of-band Signaling . . . . . . . . . .  12
     7.2.  Expanding RTP . . . . . . . . . . . . . . . . . . . . . .  13
     7.3.  Expanding RTCP  . . . . . . . . . . . . . . . . . . . . .  13
     7.4.  Failure Handling and Teardown . . . . . . . . . . . . . .  14
   8.  MPRTP Protocol  . . . . . . . . . . . . . . . . . . . . . . .  14
     8.1.  Overview  . . . . . . . . . . . . . . . . . . . . . . . .  15
       8.1.1.  Gather or Discovering Candidates  . . . . . . . . . .  15
       8.1.2.  NAT Traversal . . . . . . . . . . . . . . . . . . . .  15
       8.1.3.  Choosing between In-band (in RTCP) and Out-of-band
               (in SDP) Interface Advertisement  . . . . . . . . . .  16
       8.1.4.  In-band Interface Advertisement . . . . . . . . . . .  16
       8.1.5.  Subflow ID Assignment . . . . . . . . . . . . . . . .  17
       8.1.6.  Active and Passive Subflows . . . . . . . . . . . . .  17
     8.2.  RTP Transmission  . . . . . . . . . . . . . . . . . . . .  18
     8.3.  Playout Considerations at the Receiver  . . . . . . . . .  18
     8.4.  Subflow-specific RTCP Statistics and RTCP Aggregation . .  18
     8.5.  RTCP Transmission . . . . . . . . . . . . . . . . . . . .  19
   9.  Packet Formats  . . . . . . . . . . . . . . . . . . . . . . .  19
     9.1.  RTP Header Extension for MPRTP  . . . . . . . . . . . . .  19
       9.1.1.  MPRTP RTP Extension for a Subflow . . . . . . . . . .  21
     9.2.  RTCP Extension for MPRTP (MPRTCP) . . . . . . . . . . . .  21
       9.2.1.  MPRTCP Extension for Subflow Reporting  . . . . . . .  23
         9.2.1.1.  MPRTCP for Subflow-specific SR, RR and XR . . . .  25
     9.3.  MPRTCP Extension for Interface advertisement  . . . . . .  27
   10. RTCP Timing reconsiderations for MPRTCP . . . . . . . . . . .  28
   11. SDP Considerations  . . . . . . . . . . . . . . . . . . . . .  29
     11.1.  Signaling MPRTP Header Extension in SDP  . . . . . . . .  29
     11.2.  Signaling MPRTP capability in SDP  . . . . . . . . . . .  29
     11.3.  MPRTP with ICE . . . . . . . . . . . . . . . . . . . . .  30
     11.4.  Increased Throughput . . . . . . . . . . . . . . . . . .  30
     11.5.  Offer/Answer . . . . . . . . . . . . . . . . . . . . . .  31
       11.5.1.  In-band Signaling Example  . . . . . . . . . . . . .  31
   12. IANA Considerations . . . . . . . . . . . . . . . . . . . . .  32
     12.1.  MPRTP Header Extension . . . . . . . . . . . . . . . . .  32
     12.2.  MPRTCP Packet Type . . . . . . . . . . . . . . . . . . .  32
     12.3.  SDP Attributes . . . . . . . . . . . . . . . . . . . . .  33
       12.3.1.  "mprtp" attribute  . . . . . . . . . . . . . . . . .  33
   13. Security Considerations . . . . . . . . . . . . . . . . . . .  34
   14. Acknowledgements  . . . . . . . . . . . . . . . . . . . . . .  34
   15. References  . . . . . . . . . . . . . . . . . . . . . . . . .  34
     15.1.  Normative References . . . . . . . . . . . . . . . . . .  35
     15.2.  Informative References . . . . . . . . . . . . . . . . .  36
   Appendix A.  Interoperating with Legacy Applications  . . . . . .  37
   Appendix B.  Change Log . . . . . . . . . . . . . . . . . . . . .  38
     B.1.  Changes in draft-ietf-avtcore-mprtp-03  . . . . . . . . .  38
     B.2.  Changes in draft-ietf-avtcore-mprtp-01, and -02 . . . . .  38
     B.3.  Changes in draft-ietf-avtcore-mprtp-00  . . . . . . . . .  38
     B.4.  Changes in draft-singh-avtcore-mprtp-10 . . . . . . . . .  38
     B.5.  Changes in draft-singh-avtcore-mprtp-09 . . . . . . . . .  38
     B.6.  Changes in draft-singh-avtcore-mprtp-08 . . . . . . . . .  38
     B.7.  Changes in draft-singh-avtcore-mprtp-06 and -07 . . . . .  39
     B.8.  Changes in draft-singh-avtcore-mprtp-05 . . . . . . . . .  39
     B.9.  Changes in draft-singh-avtcore-mprtp-04 . . . . . . . . .  39
     B.10. Changes in draft-singh-avtcore-mprtp-03 . . . . . . . . .  39
     B.11. Changes in draft-singh-avtcore-mprtp-02 . . . . . . . . .  40
     B.12. Changes in draft-singh-avtcore-mprtp-01 . . . . . . . . .  40
   Authors' Addresses  . . . . . . . . . . . . . . . . . . . . . . .  40
```

## 1. Introduction

Multi-homed endpoints are becoming common in today's Internet, e.g., devices that support multiple wireless access technologies such as 3G and Wireless LAN.  This means that there is often more than one network path available between two endpoints.  Transport protocols, such as RTP, have not been designed to take advantage of the availability of multiple concurrent paths and therefore cannot benefit from the increased capacity and reliability that can be achieved by pooling their respective capacities.

Multipath RTP (MPRTP) is an extension to RTP [RFC3550] which splits a single RTP Stream [RFC7656] (At any given point in time, an RTP stream can have one and only one SSRC) into multiple subflows that are transmitted over different network interfaces paths (i.e., 2- out of the 5-tuples are different).  In effect, this pools the resource capacity of multiple paths.  Multipath RTCP (MPRTCP) is an extension to RTCP, it is used along with MPRTP to report sender and receiver characteristics for each subflow.

Other IETF transport protocols that are capable of using multiple paths include SCTP [RFC4960], MPTCP [RFC6182] and SHIM6 [RFC5533]. However, these protocols are not suitable for real-time communications.

### 1.1. Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119].

### 1.2. Terminology

- **Endpoint**: corresponds to a single host either initiating or terminating a communication session [RFC7656].
- **Interface**: logical or physical component that is capable of acquiring a unique IP address.
- **Path**: sequence of links between a sender and a receiver. Typically, defined by a set of source and destination addresses. In the context of RTP taxonomy [RFC7656] it best matches a media transports.
- **Multipath RTP Stream**: An RTP Stream that makes use of one or more paths, i.e., simultaneously makes use of multiple media transports to send and receive from a particular RTP Stream.  In cases, where a single path is available, an MPRTP Stream will appear to be an RTP Stream.
- **MPRTP Subflow**: The RTP packets from a single RTP Stream that are sent or received over a particular media transport are identified as a subflow.  Therefore, an MPRTP Stream has several subflows, each subflow is associated to a particular media transport.
- **Subflow Identifier**: within a communication session, the subflow identifier denotes the RTP packets belonging to a RTP Stream sent over a particular media transport.  There are two design choices:
    - Each media transport at an endpoint chooses a unique identifier for itself, and the Subflow RTP Streams use the chosen identifier.
    - The RTP Stream chooses distinct identifiers for each Subflow RTP stream

### 1.3. Use-cases

The primary use-case for MPRTP is transporting high bit-rate streaming multimedia content between endpoints, where at least one is multi-homed.  Such endpoints could be residential IPTV devices that connect to the Internet through two different Internet service providers (ISPs), or mobile devices that connect to the Internet through 3G and WLAN interfaces.  By allowing a single RTP Stream to use multiple paths for transmission, the following gains can be achieved:
- Higher quality: Pooling the resource capacity of multiple Internet paths allows higher bit-rate and higher quality codecs to be used. From the application perspective, the available bandwidth between the two endpoints increases.
- Load balancing: Transmitting a single RTP stream (or several RTP streams) over multiple paths reduces the bandwidth usage on each path, which in turn reduces the impact of the media stream(s) on other traffic on those paths.
- Fault tolerance: When multiple paths are used in conjunction with redundancy mechanisms (FEC, re-transmissions, etc.), outages on one path have less impact on the overall perceived quality of the media stream.

A secondary use-case for MPRTP is transporting Voice over IP (VoIP) calls to a device with multiple interfaces.  Again, such an endpoint could be a mobile device with multiple wireless interfaces.  In this case, little is to be gained from resource pooling, i.e., higher capacity or load balancing, because a single path should be easily capable of handling the required load.  However, using multiple concurrent subflows can improve fault tolerance, because traffic can shift between the subflows when path outages occur.  This results in very fast transport-layer handovers that do not require support from signaling.

## 2. Goals

This section outlines the basic goals that multipath RTP aims to meet.  These are broadly classified as Functional goals and Compatibility goals.

### 2.1.  Functional goals

Allow an unicast RTP Stream to be split into multiple subflows in order to be carried over multiple paths.  This may prove beneficial in case of video streaming.

- Increased Throughput: Cumulative capacity of the two paths may meet the requirements of the multimedia session.  Therefore, MPRTP MUST support concurrent use of the multiple paths.
- Improved Reliability: MPRTP SHOULD be able to send redundant packets or re-transmit packets along any available path to increase reliability.

The protocol SHOULD be able to open new subflows for an existing multimedia session when new paths appear and MUST be able to close subflows when paths disappear.

### 2.2.  Compatibility goals

Multipath RTP (MPRTP) MUST be backwards compatible; an MPRTP Stream needs to fall back to be compatible with legacy RTP stacks if MPRTP support is not successfully negotiated.

- **Application Compatibility**: MPRTP service model MUST be backwards compatible with existing RTP applications, i.e., an MPRTP stack MUST be able to work with legacy RTP applications and not require changes to them.  Therefore, the basic RTP APIs MUST remain unchanged, but an MPRTP stack MAY provide extended APIs so that the application can configure any additional features provided by the MPRTP stack.
- **Network Compatibility**: individual MPRTP subflows MUST themselves be flows of well-formed RTP packets, so that they are able to traverse NATs and firewalls.  This MUST be the case even when interfaces appear after session initiation.  Interactive Connectivity Establishment (ICE) [RFC5245] MAY be used for discovering new interfaces or performing connectivity checks.

## 3. RTP Topologies

RFC 5117 [RFC5117] describes a number of scenarios using mixers and translators in single-party (point-to-point), and multi-party (point-to-multipoint) scenarios.  RFC 3550 [RFC3550] (Section 2.3 and 7.x) discuss in detail the impact of mixers and translators on RTP and RTCP packets.  MPRTP assumes that if a mixer or translator exists in the network, then either all of the multiple paths or none of the multiple paths go via this component.

## 4.  MPRTP Architecture

In a typical scenario, an RTP Stream uses a single path.  In an MPRTP scenario, an RTP Stream uses multiple subflows, those subflows may each use a different path.  Figure 1 shows the difference.

```
   +--------------+                Signaling            +--------------+
   |              |------------------------------------>|              |
   |    Client    |<------------------------------------|    Server    |
   |              |          Single RTP Stream          |              |
   +--------------+                                     +--------------+

   +--------------+              Signaling              +--------------+
   |              |------------------------------------>|              |
   |    Client    |<------------------------------------|    Server    |
   |              |<------------------------------------|              |
   +--------------+            MPRTP subflows           +--------------+

     Figure 1: Comparison between traditional RTP streaming and MPRTP

```

```
   +-----------------------+         +-------------------------------+
   |      Application      |         |          Application          |
   +-----------------------+         +-------------------------------+
   |                       |         |              RTP              |
   +          RTP          +         +- - - - - - - -+- - - - - - - -+
   |                       |         | MPRTP subflow | MPRTP subflow |
   +-----------------------+         +---------------+---------------+
   |          UDP          |         |      UDP      |      UDP      |
   +-----------------------+         +---------------+---------------+
   |           IP          |         |       IP      |       IP      |
   +-----------------------+         +---------------+---------------+

                       Figure 2: MPRTP Architecture
```

Figure 2 illustrates the differences between the standard RTP stack and the MPRTP stack.  MPRTP receives a normal RTP Stream from the application and splits it into multiple RTP subflows.  Each subflow is then sent along a different path to the receiver.  To the network, each subflow appears as an independent flow of well-formed RTP packets.  At the receiver, the subflows are combined to recreate the original RTP Stream.  The MPRTP layer performs the following functions:
- Path Management: The layer is aware of alternate paths to the other host, which may, for example, be the peer's multiple interfaces.  This enables the endpoint to transmit differently marked packets along separate paths.  MPRTP also selects interfaces to send and receive data.  Furthermore, it manages the port and IP address pair bindings for each subflow.
- Packet Scheduling: the layer splits a single RTP Stream into multiple subflows and sends them across multiple interfaces (paths).  The splitting MAY BE done using different path characteristics.
- Subflow recombination: the layer creates the original RTP stream by recombining the independent subflows associated with the RTP Stream.  Therefore, the subflows are eventually presented as a single RTP stream to the applications.

## 5. Example Media Flow Diagrams

There may be many complex technical scenarios for MPRTP, however, this memo only considers the following two scenarios:
1. a unidirectional media stream that represents the streaming use-case, and
2. a bidirectional media stream that represents a conversational use-case.

### 5.1.  Streaming use-case

In the unidirectional scenario, the receiver (client) initiates a multimedia session with the sender (server).  The receiver or the sender may have multiple interfaces and both endpoints are MPRTP- capable.  Figure 3 shows this scenario.  In this case, host A has multiple interfaces.  Host B performs connectivity checks on host A's multiple interfaces.  If the interfaces are reachable, then host B streams multimedia data along multiple paths to host A.  Moreover, host B also Sends RTCP Sender Reports (SR) for the overall session and for each subflow and host A responds with a normal RTCP Receiver Report (RR) for the overall session as well as the receiver statistics for each subflow.  Host B distributes the packets across the subflows based on the individually measured path characteristics.

Alternatively, to reduce media startup time, host B may start streaming multimedia data to host A's initiating interface and then perform connectivity checks for the other interfaces.  This method of updating a single path session to a multipath session is called "multipath session upgrade".

```
           Host A                           Host B
   -----------------------                ----------
   Interface A1   Interface A2            Interface B1
   -----------------------                ----------
       |                                      |
       |------------------------------------->| Session setup with
       |<-------------------------------------| multiple interfaces
       |            |                         |
       |            |                         |
       |       (RTP data B1->A1, B1->A2)      |
       |<=====================================|
       |            |<========================|
       |            |                         |
       Note: there may be more scenarios.

                    Figure 3: Unidirectional media flow
```

### 5.2.  Conversational use-case

In the bidirectional scenario, multimedia data flows in both directions.  The two hosts exchange their lists of interfaces with each other and perform connectivity checks.  Communication begins after each host finds suitable address, port pairs.  Interfaces that receive data send back RTCP receiver statistics for that path (based on the 5-tuple).  The hosts balance their multimedia stream across multiple paths based on the per path reception statistics and its own volume of traffic.  Figure 4 describes an example of a bidirectional flow.

```
           Host A                             Host B
   ---------------------------       ---------------------------
   Interface A1   Interface A2       Interface B1   Interface B2
   ---------------------------       ---------------------------
     |            |                       |            |
     |            |                       |            |Session setup
     |----------------------------------->|            |with multiple
     |<-----------------------------------|            |interfaces
     |            |                       |            |
     |            |                       |            |
     |    (RTP data B1<->A1, B2<->A2)     |            |
     |<===================================|            |
     |            |<===================================|
     |===================================>|            |
     |            |===================================>|
     |            |                       |            |
       Note: the address pairs may have other permutations,
       and they may be symmetric or asymmetric combinations.

                       Figure 4: Bidirectional flow
```

## 6.  MPRTP Functional Blocks

This section describes some of the functional blocks needed for MPRTP.  We then investigate each block and consider available mechanisms in the next section.

1. Session Setup: Interfaces may appear or disappear at anytime during the communication session.  To preserve backward compatibility with legacy applications, a MPRTP session MUST look like a bundle of individual RTP sessions.  A MPRTP session may start using a single path, later start using multiple paths as and when new interfaces appear or disappear.  However, it is also possible to setup a MPRTP session from the beginning, if multiple interfaces are available at the start of the multimedia session.
2. Expanding RTP: For a MPRTP Stream, each subflow MUST look like an independent flow of RTP packets, so that individual RTCP messages can be generated for each subflow.  Consequently, MPRTP may split the RTP Stream into multiple subflows based on path characteristics (e.g.  RTT, fractional losses, receiver rate, bandwidth-delay product, etc.), i.e., dynamically adjusts the load on each subflow.
3. Adding Interfaces: Interfaces on the host need to be regularly discovered and advertised.  This can be done when the MPRTP session is set up and/or during the communication session. Discovering interfaces is outside the scope of this document.
4. Expanding RTCP: MPRTP MUST provide per path RTCP reports for monitoring the quality of the path, for load balancing, or for congestion control, etc.  To maintain backward compatibility with legacy applications, the receiver MUST also send aggregate RTCP reports along with the per-path reports.
5. Maintenance and Failure Handling: In a multi-homed endpoint interfaces may appear and disappear.  If this occurs at the sender, it has to re-adjust the load on the available paths.  On the other hand, if this occurs at the receiver, then the multimedia data transmitted by the sender to those interfaces is lost.  This data may be re-transmitted along a different path i.e., to a different interface on the receiver.  Furthermore, the endpoint has to either explicitly signal the disappearance of an interface, or the other endpoint has to detect it (by lack of media packets, RTCP feedback, or keep-alive packets).
6. Teardown: The MPRTP layer releases the occupied ports on the interfaces.

## 7. Available Mechanisms within the Functional Blocks

This section discusses some of the possible alternatives for each functional block mentioned in the previous section.

### 7.1.  Session Setup

MPRTP session can be set up in many possible ways e.g., during handshake, or upgraded mid-session.  The capability exchange may be done using out-of-band signaling (e.g., Session Description Protocol (SDP) [RFC3264] in Session Initiation Protocol (SIP) [RFC3261], Real- Time Streaming Protocol (RTSP) [I-D.ietf-mmusic-rfc2326bis]) or in- band signaling (e.g., RTP/RTCP header extension, Session Traversal Utilities for NAT (STUN) messages).

> [Editor's Note: , with continuous ICE Nomination.]

#### 7.1.1.  Connectivity Checks

The endpoint SHOULD be capable of performing connectivity checks (e.g., using ICE [RFC5245]).  If the IP addresses of the endpoints are reachable (e.g., globally addressable, same network etc) then connectivity checks are not needed.

#### 7.1.2.  Adding New or Updating Interfaces

Interfaces can appear and disappear during a MPRTP session, the endpoint MUST be capable of advertising the changes in its set of usable interfaces.  Additionally, the application or OS may define a policy on when and/or what interfaces are usable.  However, MPRTP requires a method to advertise or notify the other endpoint about the updated set of usable interfaces.

#### 7.1.3.  In-band vs. Out-of-band Signaling

MPRTP nodes will generally use a signaling protocol to establish their MPRTP session.  With the existence of such a signaling relationship, two alternatives become available to exchange information about the available interfaces on each side for extending RTP sessions to MPRTP and for modifying MPRTP sessions: in-band and out-of-band signaling.

In-band signaling refers to using mechanisms of RTP/RTCP itself to communicate interface addresses, e.g., a dedicated RTCP extensions along the lines of the one defined to communicate information about the feedback target for RTP over SSM [RFC5760].  In-band signaling does not rely on the availability of a separate signaling connection and the information flows along the same path as the media streams, thus minimizing dependencies.  Moreover, if the media channel is secured (e.g., by means of SRTP), the signaling is implicitly protected as well if SRTCP encryption and authentication are chosen. In-band signaling is also expected to take a direct path to the peer, avoiding any signaling overlay-induced indirections and associated processing overheads in signaling elements -- avoiding such may be especially worthwhile if frequent updates may occur as in the case of mobile users.  Finally, RTCP is usually sent sufficiently frequently (in point-to-point settings) to provide enough opportunities for transmission and (in case of loss) retransmission of the corresponding RTCP packets.

Examples for in-band signaling include RTCP extensions as noted above or suitable extensions to STUN [I-D.wing-mmusic-ice-mobility].

Out-of-band signaling refers to using a separate signaling connection (via SIP, RTSP, or HTTP) to exchange interface information, e.g., expressed in SDP.  Clear benefits are that signaling occurs at setup time anyway and that experience and SDP syntax (and procedures) are available that can be re-used or easily adapted to provide the necessary capabilities.  In contrast to RTCP, SDP offers a reliable communication channel so that no separate retransmissions logic is necessary.  In SDP, especially when combined with ICE, connectivity check mechanisms including sophisticated rules are readily available.

While SDP is not inherently protected, suitable security may need to be applied anyway to the basic session setup.

Examples for out-of-band signaling are dedicated extensions to SDP; those may be combined with ICE.

Both types of mechanisms have their pros and cons for middleboxes. With in-band signaling, control packets take the same path as the media packets and they can be directly inspected by middleboxes so that the elements operating on the signaling channel do not need to understand new SDP.  With out-of-band signaling, only the middleboxes processing the signaling need to be modified and those on the data forwarding path can remain untouched.

Overall, it may appear sensible to provide a combination of both mechanisms: out-of-band signaling for session setup and initial interface negotiation and in-band signaling to deal with frequent changes in interface state (and for the potential case, albeit rather theoretical case of MPRTP communication without a signaling channel).

In its present version, this document explores both options to provide a broad understanding of how the corresponding mechanisms would look like.

### 7.2.  Expanding RTP

RTCP [RFC3550] is generated per media session.  However, with MPRTP, the media sender spreads the RTP load across several interfaces.  The media sender SHOULD make the path selection, load balancing and fault tolerance decisions based on the characteristics of each path. Therefore, apart from normal RTP sequence numbers defined in [RFC3550], the MPRTP sender MUST add the following within the SSRC scope, subflow-specific sequence numbers to help calculate fractional losses, jitter, RTT, playout time, etc., for each subflow, and a subflow identifier to associate the characteristics with a path.  The RTP header extension for MPRTP is shown in Section 9.1.

### 7.3.  Expanding RTCP

To provide accurate per path information an MPRTP endpoint MUST send (SR/RR) report for each unique subflow along with the overall session RTCP report.  Therefore, the additional subflow reporting affects the RTCP bandwidth and the RTCP reporting interval.  RTCP report scheduling for each subflow may cause a problem for RTCP recombination and reconstruction in cases when
1. RTCP for a subflow is lost, and
2. RTCP for a subflow arrives later than the other subflows.  (There may be other cases as well.)

The sender distributes the media across different paths using the per path RTCP reports.  However, this document doesn't cover algorithms for congestion control or load balancing.

### 7.4.  Failure Handling and Teardown

An MPRTP endpoint MUST keep alive subflows that have been negotiated but no media is sent on them.  Moreover, using the information in the subflow reports, a sender can monitor an active subflow for failure (errors, unreachability, congestion) and decide not to use (make the active subflow passive), or teardown the subflow.

If an interface disappears, the endpoint MUST send an updated interface advertisement without the interface and release the associated subflows.

## 8.  MPRTP Protocol

```
           Host A                                   Host B
   -----------------------                  -----------------------
   Interface A1   Interface A2            Interface B1   Interface B2
   -----------------------                  -----------------------
       |            |                           |             |
       |            |      (1)                  |             |
       |--------------------------------------->|             |
       |<---------------------------------------|             |
       |            |      (2)                  |             |
       |<=======================================|             |
       |<=======================================|     (3)     |
       |            |      (4)                  |             |
       |<- - - - - - - - - - - - - - - - - - - -|             |
       |<- - - - - - - - - - - - - - - - - - - -|             |
       |<- - - - - - - - - - - - - - - - - - - -|             |
       |            |      (5)                  |             |
       |- - - - - - - - - - - - - - - - - - - ->|             |
       |<=======================================|     (6)     |
       |<=======================================|             |
       |            |<========================================|
       |<=======================================|             |
       |            |<========================================|

   Key:
   |  Interface
   ---> Signaling Protocol
   <=== RTP Packets
   - -> RTCP Packet

                       Figure 5: MPRTP New Interface
```

### 8.1.  Overview

The bullet points explain the different steps shown in Figure 5 for upgrading a single path multimedia session to multipath session.

1. The first two interactions between the hosts represents the establishment of a normal RTP session.  This may performed e.g. using SIP or RTSP.
2. When the RTP session has been established, host B streams media using its interface B1 to host A at interface A1.
3. Host B supports sending media using MPRTP and becomes aware of an additional interface B2.
4. Host B advertises the multiple interface addresses.
5. Host A supports receiving media using MPRTP and becomes aware of an additional interface A2. Side note, even if an MPRTP-capable host has only one interface, it MUST respond to the advertisement with its single interface.
6. Each host receives information about the additional interfaces and the appropriate endpoints starts to stream the multimedia content using the additional paths.

If needed, each endpoint will need to independently perform connectivity checks (not shown in figure) and ascertain reachability before using the paths.

#### 8.1.1.  Gather or Discovering Candidates

The endpoint periodically polls the operating system or is notified when an additional interface appears.  If the endpoint wants to use the additional interface for MPRTP it MUST advertise it to the other peers.  The endpoint may also use ICE [RFC5245] to gather additional candidates.

#### 8.1.2.  NAT Traversal

After gathering their interface candidates, the endpoints decide internally if they wish to perform connectivity checks.

> [note-iceornot]

If the endpoint chooses to perform connectivity checks then it MUST first advertise the gathered candidates as ICE candidates in SDP during session setup and let ICE perform the connectivity checks.  As soon as a sufficient number of connectivity checks succeed, the endpoint can use the successful candidates to advertise its MPRTP interface candidates.

Alternatively, if the endpoint supports mobility extensions for ICE [I-D.wing-mmusic-ice-mobility], it can let the ICE agent gather and perform the connectivity checks.  When the connectivity checks succeed, the ICE agent should notify the MPRTP layer of these new paths (5-tuples), these new paths are then used by MPRTP to send media packets depending on the scheduling algorithm.

If an endpoint supports Interface selection via PCP Flow Extension [I-D.reddy-mmusic-ice-best-interface-pcp], it will receive notifications when new interfaces become available and additionally when the link quality of a currently available interface changes. The application can advertise and perform connectivity checks with the new interface and in the case of changes in link quality pass the information to the scheduling algorithm for better application performance.

> [Editors note: The interaction between the RTP agent and ICE agent is needed for implementing a scheduling algorithm or congestion control. See details of a scheduling algorithm in [ACM-MPRTP].]

#### 8.1.3.  Choosing between In-band (in RTCP) and Out-of-band (in SDP) Interface Advertisement

If there is no media flowing at the moment and the application wants to use the interfaces from the start of the communication session, it should advertise them in SDP (See [I-D.singh-mmusic-mprtp-sdp-extension]).  Alternatively, the endpoint can setup the session as a single path communication session and upgrade the session to multipath by advertising the session in-band (See Section 8.1.4 or [I-D.wing-mmusic-ice-mobility]).  Moreover, if an interface appears and disappears, the endpoint SHOULD prefer to advertise it in-band because the endpoint would not have to wait for a response from the other endpoint before starting to use the interface.  However, if there is a conflict between an in-band and out-of-band advertisement, i.e., the endpoint receives an in-band advertisement while it has a pending out-of-band advertisement, or vice versa then the session is setup using out-of-band mechanisms.

#### 8.1.4.  In-band Interface Advertisement

To advertise the multiple interfaces in RTCP, an MPRTP-capable endpoint SHOULD add the MPRTP Interface Advertisement defined in Figure 13 with an RTCP Sender Report (SR) or RTCP Receiver Report (RR).  If reduced-size [RFC5506] RTCP is used, the interface advertisements can be sent separately depending on the RTCP timing rules.

Each unique address is encapsulated in an Interface Advertisement block and contains the IP address, RTP port addresses.  The Interface Advertisement blocks are ordered based on a decreasing priority level.  On receiving the MPRTP Interface Advertisement, an MPRTP- capable receiver MUST respond with the set of interfaces (subset or all available) it wants to use.

If the sender and receiver have only one interface, then the endpoints MUST indicate the negotiated single path IP, RTP port addresses.

> [Editor: Do we still need this: could use passive/aggressive nomination? or perform an ice restart?]

#### 8.1.5.  Subflow ID Assignment

After interface advertisements have been exchanged, the endpoint MUST associate a Subflow ID to each unique subflow.  Each combination of sender and receiver IP addresses and port pairs (5-tuple) is a unique subflow.  If the connectivity checks have been performed then the endpoint MUST only use the subflows for which the connectivity checks have succeeded.

#### 8.1.6.  Active and Passive Subflows

To send and receive data an endpoint MAY use any number of subflows from the set of available subflows.  The subflows that carry media data are called active subflows, while those subflows that don't send any media packets (fallback paths) are called passive subflows.

An endpoint MUST multiplex the subflow specific RTP and RTCP packets on the same port to keep the NAT bindings alive.  This is inline with the recommendation made in RFC6263[RFC6263].  Moreover, if an endpoint uses ICE, multiplexing RTP and RTCP reduces the number of components from 2 to 1 per media stream.  If no MPRTP or MPRTCP packets are received on a particular subflow at a receiver, the receiver SHOULD remove the subflow from active and passive lists and not send any MPRTCP reports for that subflow.  It may keep the bindings in a dead-pool, so that if the 5-tuple or subflow reappears, it can quickly reallocate it based on past history.

### 8.2.  RTP Transmission

If both endpoints are MPRTP-capable and if they want to use their multiple interfaces for sending the media stream then they MUST use the MPRTP header extensions.  They MAY use normal RTP with legacy endpoints (see Appendix A).

An MPRTP endpoint sends RTP packets with an MPRTP extension that maps the media packet to a specific subflow (see Figure 8).  The MPRTP layer SHOULD associate an RTP packet with a subflow based on a scheduling strategy.  The scheduling strategy may either choose to augment the paths to create higher throughput or use the alternate paths for enhancing resilience or error-repair.  Due to the changes in path characteristics, the endpoint should be able to change its scheduling strategy during an ongoing session.  The MPRTP sender MUST also populate the subflow specific fields described in the MPRTP extension header (see Section 9.1.1).

[ACM-MPRTP] describes and evaluates a scheduling algorithm for video over multiple interfaces.  The video is encoded at a single target bit rate and is evaluated in various network scenarios, as discussed in [I-D.ietf-rmcat-eval-criteria].

### 8.3.  Playout Considerations at the Receiver

A media receiver, irrespective of MPRTP support or not, should be able to playback the media stream because the received RTP packets are compliant to [RFC3550], i.e., a non-MPRTP receiver will ignore the MPRTP header and still be able to playback the RTP packets. However, the variation of jitter and loss per path may affect proper playout.  The receiver can compensate for the jitter by modifying the playout delay (i.e., by calculating skew across all paths) of the received RTP packets.  For example, an adaptive playout algorithm is discussed in [ACM-MPRTP].

### 8.4.  Subflow-specific RTCP Statistics and RTCP Aggregation

Aggregate RTCP provides the overall media statistics and follows the normal RTCP defined in RFC3550 [RFC3550].  However, subflow specific RTCP provides the per path media statistics because the aggregate RTCP report may not provide sufficient per path information to an MPRTP scheduler.  Specifically, the scheduler should be aware of each path's RTT and loss-rate, which an aggregate RTCP cannot provide.

The aggregate RTCP report (i.e., the regularly scheduled RTCP report) MUST be sent compounded as described in [RFC5506], however, the subflow-specific RTCP reports MAY be sent non-compounded (reduced-size).  Further, each subflow and the aggregate RTCP report MUST follow the timing rules defined in [RFC4585].

The RTCP reporting interval is locally implemented and the scheduling of the RTCP reports may depend on the behavior of each path.  For instance, the RTCP interval may be different for a passive path than an active path to keep port bindings alive.  Additionally, an endpoint may decide to share the RTCP reporting bit rate equally across all its paths or schedule based on the receiver rate on each path.

### 8.5.  RTCP Transmission

The sender sends an RTCP Subflow SR (SSR) on each active path.  For each SR the receiver gets, it echoes one back to the same IP address-port pair that sent the SR.  The receiver tries to choose the symmetric path and if the routing is symmetric then the per-path RTT calculations will work out correctly.  However, even if the paths are not symmetric, the sender would at maximum, under-estimate the RTT of the path by a factor of half of the actual path RTT.

## 9.  Packet Formats

In this section we define the protocol structures described in the previous sections.

### 9.1.  RTP Header Extension for MPRTP

The MPRTP header extension is used to distribute a single RTP stream over multiple subflows.

The header conforms to the one-byte RTP header extension defined in [RFC5285].  The header extension contains a 16-bit length field that counts the number of 32-bit words in the extension, excluding the four-octet extension header (therefore zero is a valid length, see Section 5.3.1 of [RFC3550] for details).

The RTP header for each subflow is defined below:

```
       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |V=2|P|1|  CC   |M|     PT      |       sequence number         |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                           timestamp                           |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |           synchronization source (SSRC) identifier            |
      +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
      |       0xBE    |    0xDE       |           length=N-1          |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |   ID  |  LEN  |  MPID |LENGTH |       MPRTP header data       |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
      |                             ....                              |
      +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
      |                         RTP payload                           |
      |                             ....                              |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

                 Figure 6: Generic MPRTP header extension
```

- MPID: The MPID field corresponds to the type of MPRTP packet. Namely:

```
    +---------------+--------------------------------------------------+
    |    MPID ID    | Use                                              |
    |     Value     |                                                  |
    +---------------+--------------------------------------------------+
    |      0x0      | Subflow RTP Header. For this case the Length is  |
    |               | set to 4                                         |
    +---------------+--------------------------------------------------+

        Figure 7: RTP header extension values for MPRTP (H-Ext ID)
```

- **Length**: The 4-bit length field is the length of extension data in bytes not including the H-Ext ID and length fields.  The value zero indicates there is no data following.
- **MPRTP header data**: Carries the MPID specific data as described in the following sub-sections.

#### 9.1.1.  MPRTP RTP Extension for a Subflow

The RTP header for each subflow is defined below:

```
       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |V=2|P|1|  CC   |M|     PT      |       sequence number         |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                           timestamp                           |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |           synchronization source (SSRC) identifier            |
      +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
      |       0xBE    |    0xDE       |           length=2            |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |   ID  | LEN=4 |  0x0  | LEN=4 |          Subflow ID           |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |  Subflow-specific Seq Number  |    Pad (0)    |   Pad (0)     |
      +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
      |                         RTP payload                           |
      |                             ....                              |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

                    Figure 8: MPRTP header for subflow
```

- **MP ID** = 0x0 Indicates that the MPRTP header extension carries subflow specific information.
- **Length** = 4
- **Subflow ID**: Identifier of the subflow.  Every RTP packet belonging to the same subflow carries the same unique subflow identifier.
- **Flow-Specific Sequence Number (FSSN)**: Sequence of the packet in the subflow.  Each subflow has its own strictly monotonically increasing sequence number space.

### 9.2.  RTCP Extension for MPRTP (MPRTCP)

The MPRTP RTCP header extension is used to
1. provide RTCP feedback per subflow to determine the characteristics of each path, and
2. advertise each interface.

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P|reserved | PT=MPRTCP=211 |         encaps_length         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                     SSRC of packet sender                     |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |                 SSRC_1 (SSRC of first source)                 |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  MPRTCP_Type  |  block length |                               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+         MPRTCP Reports        |
   |                             ...                               |
   |                             ...                               |
   |                             ...                               |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

     Figure 9: Generic RTCP Extension for MPRTP (MPRTCP) [appended to
                               normal SR/RR]
```

- **MPRTCP**: 8 bits, Contains the constant 211 to identify this as an Multipath RTCP packet.
- **encaps_length**: 16 bits As described for the RTCP packet (see Section 6.4.1 of the RTP specification [RFC3550]), the length of this is in 32-bit words minus one, including the header and any padding. The encaps_length covers all MPRTCP_Type report blocks included within this report.
- **MPRTCP_Type**: 8-bits, The MPRTCP_Type field corresponds to the type of MPRTP RTCP packet.  Namely:

```
    +---------------+--------------------------------------------------+
    |  MPRTCP_Type  | Use                                              |
    |     Value     |                                                  |
    +---------------+--------------------------------------------------+
    |       0       | Subflow Specific Report                          |
    |       1       | Interface Advertisement (IPv4 Address)           |
    |       2       | Interface Advertisement (IPv4 Address)           |
    |       3       | Interface Advertisement (DNS Address)            |
    +---------------+--------------------------------------------------+

      Figure 10: RTP header extension values for MPRTP (MPRTCP_Type)
```

- **block length**: 8-bits, The 8-bit length field is the length of the encapsulated MPRTCP reports in 32-bit word length (i.e., length * 4 bytes) including the MPRTCP_Type and length fields.  The value zero is invalid and the report block MUST be ignored.
- **MPRTCP Reports**: variable size, Defined later in 9.2.1 and 9.3.

#### 9.2.1.  MPRTCP Extension for Subflow Reporting

When sending a report for a specific subflow the sender or receiver MUST add only the reports associated with that 5-tuple.  Each subflow is reported independently using the following MPRTCP Feedback header.

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P|reserved | PT=MPRTCP=211 |         encaps_length         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                     SSRC of packet sender                     |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |                 SSRC_1 (SSRC of first source)                 |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   | MPRTCP_Type=0 |  block length |         Subflow ID #1         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Subflow-specific reports                   |
   |                             ....                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   | MPRTCP_Type=0 |  block length |         Subflow ID #2         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Subflow-specific reports                   |
   |                             ....                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

                Figure 11: MPRTCP Subflow Reporting Header
```

- **MPRTCP_Type**: 0, The value indicates that the encapsulated block is a subflow report.
- **block length**: 8-bits, The 8-bit length field is the length of the encapsulated subflow-specific reports in 32-bit word length not including the MPRTCP_Type and length fields.
- **Subflow ID**: 16 bits, Subflow identifier is the value associated with the subflow the endpoint is reporting about.  If it is a sender it MUST use the Subflow ID associated with the 5-tuple.  If it is a receiver it MUST use the Subflow ID received in the Subflow-specific Sender Report.
- **Subflow-specific reports**: variable Subflow-specific report contains all the reports associated with the Subflow ID.  For a sender, it MUST include the Subflow-specific Sender Report (SSR).  For a receiver, it MUST include Subflow-specific Receiver Report (SRR).  Additionally, if the receiver supports subflow-specific extension reports then it MUST append them to the SRR.

##### 9.2.1.1.  MPRTCP for Subflow-specific SR, RR and XR

> [note-rtcp-reuse]

Example:

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P|reserved | PT=MPRTCP=211 |         encaps_length         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                     SSRC of packet sender                     |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |                 SSRC_1 (SSRC of first source)                 |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   | MPRTCP_Type=0 |  block length |         Subflow ID #1         |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |V=2|P|    RC   |   PT=SR=200   |             length            |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                         SSRC of sender                        |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |              NTP timestamp, most significant word             |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |             NTP timestamp, least significant word             |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                         RTP timestamp                         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    subflow's packet count                     |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                     subflow's octet count                     |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   | MPRTCP_Type=0 |  block length |         Subflow ID #2         |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |V=2|P|    RC   |   PT=RR=201   |             length            |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                     SSRC of packet sender                     |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   | fraction lost |       cumulative number of packets lost       |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           extended highest sequence number received           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                     inter-arrival jitter                      |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                         last SR (LSR)                         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                   delay since last SR (DLSR)                  |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |               Subflow specific extension reports              |
   |                             ....                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

   Figure 12: Example of reusing RTCP SR and RR inside an MPRTCP header
   (Bi-directional use-case, in case of uni-directional flow the subflow
                       will only send an SR or RR).

```

### 9.3.  MPRTCP Extension for Interface advertisement

This sub-section defines the RTCP header extension for in-band interface advertisement by the receiver.  The interface advertisement block describes a method to represent IPv4, IPv6 and generic DNS-type addresses in a block format.  It is based on the sub-reporting block in [RFC5760].  The interface advertisement SHOULD immediately follow the Receiver Report.  If the Receiver Report is not present, then it MUST be appended to the Sender Report.

The endpoint MUST advertise the interfaces it wants to use whenever an interface appears or disappears and also when it receives an Interface Advertisement.

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P|reserved | PT=MPRTCP=211 |         encaps_length         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                     SSRC of packet sender                     |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |                 SSRC_1 (SSRC of first source)                 |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  MPRTCP_Type  |  block length |            RTP Port           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                      Interface Address #1                     |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  MPRTCP_Type  |  block length |            RTP Port           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                      Interface Address #2                     |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  MPRTCP_Type  |  block length |            RTP Port           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                     Interface Address #..                     |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  MPRTCP_Type  |  block length |            RTP Port           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                      Interface Address #m                     |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

       Figure 13: MPRTP Interface Advertisement. (appended to SR/RR)
```

- **MPRTCP_Type**: 8 bits, The MPRTCP_Type corresponds to the type of interface address. Namely:
    - 1: IPv4 address
    - 2: IPv6 address
    - 3: DNS name
- **block length**: 8 bits, The length of the Interface Advertisement block in 32-bit word lengths.
    - For an IPv4 address, this should be 2 (8 octets including = 2 + 2 + 4).
    - For an IPv6 address, this should be 5 (20 octets = 2 +2 + 16).
    - For a DNS name, the length field indicates the number of word lengths making up the address string plus 1 (32-bit word for the 2 byte port number and 2 byte header).
- **RTP Port**: 2 octets, The port number to which the sender sends RTP data.  A port number of 0 is invalid and MUST NOT be used.
- **Interface Address**: 4 octets (IPv4), 16 octets (IPv6), or n octets (DNS name). The address to which receivers send feedback reports.  For IPv4 and IPv6, fixed-length address fields are used.  A DNS name is an arbitrary-length string.  The string MAY contain Internationalizing Domain Names in Applications (IDNA) domain names and MUST be UTF-8 [RFC3629] encoded.

## 10.  RTCP Timing reconsiderations for MPRTCP

MPRTP endpoints MUST conform to the timing rule imposed in [RFC4585], i.e., the total RTCP rate between the participants MUST NOT exceed 5% of the media rate.  For each endpoint, a subflow MUST send the aggregate and subflow-specific report.  The endpoint SHOULD schedule the RTCP reports for the active subflows based on the share of the transmitted or received bit rate to the average media bit rate, this method ensures fair sharing of the RTCP bandwidth.  Alternatively, the endpoint MAY schedule the reports in round-robin.

## 11.  SDP Considerations

### 11.1.  Signaling MPRTP Header Extension in SDP

To indicate the use of the MPRTP header extensions (see Section 9) in SDP, the sender MUST use the following URI in extmap: urn:ietf:params:rtp-hdrext:mprtp.  This is a media level parameter. Legacy RTP (non-MPRTP) clients will ignore this header extension, but can continue to parse and decode the packet (see Appendix A).

Example:

```
   v=0
   o=alice 2890844526 2890844527 IN IP4 192.0.2.1
   s=
   c=IN IP4 192.0.2.1
   t=0 0
   m=video 49170 RTP/AVP 98
   a=rtpmap:98 H264/90000
   a=fmtp:98 profile-level-id=42A01E;
   a=extmap:1 urn:ietf:params:rtp-hdrext:mprtp
```

### 11.2.  Signaling MPRTP capability in SDP

A participant of a media session MUST use SDP to indicate that it supports MPRTP.  Not providing this information will make the other endpoint ignore the RTCP extensions.

```
       mprtp-attrib = "a=" "mprtp" [
               SP mprtp-optional-parameter]
               CRLF   ; flag to enable MPRTP
```

The endpoint MUST use 'a=mprtp', if it is able to send and receive MPRTP packets.  Generally, senders and receivers MUST indicate this capability if they support MPRTP and would like to use it in the specific media session being signaled.  To exchange the additional interfaces, the endpoint SHOULD use the in-band signaling (See Section 9.3).  Alternatively, advertise in SDP (See [I-D.singh-mmusic-mprtp-sdp-extension]).

MPRTP endpoint multiplexes RTP and RTCP on a single port, sender MUST indicate support by adding "a=rtcp-mux" in SDP [RFC5761].  If an endpoint receives an SDP without "a=rtcp-mux" but contains "a=mprtp", then the endpoint MUST infer support for multiplexing.

MPRTP endpoint uses Reduced-Size RTCP [RFC5506] for reporting path characterisitcs per subflow (MPRTCP).  An MPRTP endpoint MUST indicate support by adding "a=rtcp-rsize" in SDP [RFC5761].  If an endpoint receives an "a=rtcp-rsize" but contains "a=mprtp", then the endpoint MUST infer support for Reduced-Size RTCP.

> [note-rtp-rtcp-mux]

### 11.3.  MPRTP with ICE

If the endpoints intend to use ICE [RFC5245] for discovering interfaces and running connectivity checks then the endpoint MUST advertise its ICE candidates in the initial OFFER, as defined in ICE [RFC5245].  Thereafter the endpoints run connectivity checks.

When an endpoint uses ICE's regular nomination [RFC5245] procedure, it chooses the best ICE candidate as the default path.  In the case of an MPRTP endpoint, if more than one ICE candidate succeeded the connectivity checks then an MPRTP endpoint MAY advertise (some of) these in-band in RTCP as MPRTP interfaces.

When an endpoint uses ICE's aggressive nomination [RFC5245] procedure, the selected candidate may change as more ICE checks complete.  Instead of sending updated offers as additional ICE candidates appear (transience), the endpoint it MAY use in-band signaling to advertise its interfaces, as defined in Section 9.3.

If the default interface disappears and the paths used for MPRTP are different from the one in the c= and m= lines then the an alternate interface for which the ICE checks were successful should be promoted to the c= and m= lines in the updated offer.

When a new interface appears then the application/endpoint should internally decide if it wishes to use it and sends an updated offer with ICE candidates of the all its interfaces including the new interface.  The receiving endpoint responds to the offer with all its ICE candidates in the answer and starts connectivity checks between all its candidates and the offerer's ICE candidates.  Similarly, the initiating endpoint starts connectivity checks between the its interfaces (incl. the new interface) and all the received ICE candidates in the answer.  If the connectivity checks succeed, the initiating endpoint MAY use in-band signaling (See Section 9.3) to advertise its new set of interfaces.

### 11.4.  Increased Throughput

The MPRTP layer MAY choose to augment paths to increase throughput. If the desired media rate exceeds the current media rate, the endpoints MUST renegotiate the application specific ("b=AS:xxx") [RFC4566] bandwidth.

### 11.5.  Offer/Answer

When SDP [RFC4566] is used to negotiate MPRTP sessions following the offer/answer model [RFC3264], the "a=mprtp" attribute (see Section 11.2) indicates the desire to use multiple interfaces to send or receive media data.  The initial SDP offer MUST include this attribute at the media level.  If the answerer wishes to also use MPRTP, it MUST include a media-level "a=mprtp" attribute in the answer.  If the answer does not contain an "a=mprtp" attribute, the offerer MUST NOT stream media over multiple paths and the offerer MUST NOT advertise additional MPRTP interfaces in-band or out-of- band.

When SDP is used in a declarative manner, the presence of an "a=mprtp" attribute signals that the sender can send or receive media data over multiple interfaces.  The receiver SHOULD be capable to stream media to the multiple interfaces and be prepared to receive media from multiple interfaces.

The following sections shows examples of SDP offer and answer for in-band and out-of-band signaling.

#### 11.5.1.  In-band Signaling Example

The following offer/answer shows that both the endpoints are MPRTP capable and SHOULD use in-band signaling for interfaces advertisements.

Offer:

```
     v=0
     o=alice 2890844526 2890844527 IN IP4 192.0.2.1
     s=
     c=IN IP4 192.0.2.1
     t=0 0
     m=video 49170 RTP/AVP 98
     a=rtpmap:98 H264/90000
     a=fmtp:98 profile-level-id=42A01E;
     a=rtcp-mux
     a=mprtp
```

Answer:

```
     v=0
     o=bob 2890844528 2890844529 IN IP4 192.0.2.2
     s=
     c=IN IP4 192.0.2.2
     t=0 0
     m=video 4000 RTP/AVP 98
     a=rtpmap:98 H264/90000
     a=fmtp:98 profile-level-id=42A01E;
     a=rtcp-mux
     a=mprtp
```

The endpoint MAY now use in-band RTCP signaling to advertise its multiple interfaces.  Alternatively, it MAY make another offer with the interfaces in SDP (out-of-band signaling) [I-D.singh-mmusic-mprtp-sdp-extension].

## 12.  IANA Considerations

The following contact information shall be used for all registrations in this document:

```
       Contact:    Varun Singh
                   mailto:varun.singh@iki.fi
                   tel:+358-9-470-24785
```

Note to the RFC-Editor: When publishing this document as an RFC, please replace "RFC XXXX" with the actual RFC number of this document and delete this sentence.

### 12.1.  MPRTP Header Extension

This document defines a new extension URI to the RTP Compact Header Extensions sub-registry of the Real-Time Transport Protocol (RTP) Parameters registry, according to the following data:

```
       Extension URI:  urn:ietf:params:rtp-hdrext:mprtp
       Description:  Multipath RTP
       Reference:  RFC XXXX
```

### 12.2.  MPRTCP Packet Type

A new RTCP packet format has been registered with the RTCP Control Packet type (PT) Registry:

```
        Name:           MPRTCP
        Long name:      Multipath RTCP
        Value:          211
        Reference:      RFC XXXX.
```

This document defines a substructure for MPRTCP packets.  A new sub-registry has been set up for the sub-report block type (MPRTCP_Type) values for the MPRTCP packet, with the following registrations created initially:

```
           Name:           Subflow Specific Report
           Long name:      Multipath RTP Subflow Specific Report
           Value:          0
           Reference:      RFC XXXX.

           Name:           IPv4 Address
           Long name:      IPv4 Interface Address
           Value:          1
           Reference:      RFC XXXX.

           Name:           IPv6 Address
           Long name:      IPv6 Interface Address
           Value:          2
           Reference:      RFC XXXX.

           Name:           DNS Name
           Long name:      DNS Name indicating Interface Address
           Value:          3
           Reference:      RFC XXXX.
```

Further values may be registered on a first come, first served basis. For each new registration, it is mandatory that a permanent, stable, and publicly accessible document exists that specifies the semantics of the registered parameter as well as the syntax and semantics of the associated sub-report block.  The general registration procedures of [RFC4566] apply.

### 12.3.  SDP Attributes

This document defines a new SDP attribute, "mprtp", within the existing IANA registry of SDP Parameters.

#### 12.3.1.  "mprtp" attribute

- Attribute Name: MPRTP
- Long Form: Multipath RTP
- Type of Attribute: media-level
- Charset Considerations: The attribute is not subject to the charset attribute.
- Purpose: This attribute is used to indicate support for Multipath RTP.  It can also provide one of many possible interfaces for communication.  These interface addresses may have been validated using ICE procedures.
- Appropriate Values: See Section 11.2 of RFC XXXX.

## 13.  Security Considerations

TBD

All drafts are required to have a security considerations section. See RFC 3552 [RFC3552] for a guide.

## 14.  Acknowledgements

Varun Singh, Saba Ahsan, and Teemu Karkkainen are supported by Trilogy (http://www.trilogy-project.org), a research project (ICT-216372) partially funded by the European Community under its Seventh Framework Program.  The views expressed here are those of the author(s) only.  The European Commission is not liable for any use that may be made of the information in this document.

The work of Varun Singh and Joerg Ott from Aalto University has been partially supported by the European Institute of Innovation and Technology (EIT) ICT Labs activity RCLD 11882.

Lars Eggert has received funding from the European Union's Horizon 2020 research and innovation program 2014-2018 under grant agreement No. 644866.  This document reflects only the authors' views and the European Commission is not responsible for any use that may be made of the information it contains.

Thanks to Roni Even , Miguel A.  Garcia , Ralf Globisch , Christer Holmberg , and Frederic Maze for providing valuable feedback on earlier versions of this draft.

## 15.  References

### 15.1.  Normative References

- [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, DOI 10.17487/ RFC2119, March 1997, <http://www.rfc-editor.org/info/rfc2119>.
- [RFC3629]  Yergeau, F., "UTF-8, a transformation format of ISO 10646", STD 63, RFC 3629, DOI 10.17487/RFC3629, November 2003, <http://www.rfc-editor.org/info/rfc3629>.
- [RFC5760]  Ott, J., Chesterfield, J., and E. Schooler, "RTP Control Protocol (RTCP) Extensions for Single-Source Multicast Sessions with Unicast Feedback", RFC 5760, DOI 10.17487/ RFC5760, February 2010, <http://www.rfc-editor.org/info/rfc5760>.
- [RFC5245]  Rosenberg, J., "Interactive Connectivity Establishment (ICE): A Protocol for Network Address Translator (NAT) Traversal for Offer/Answer Protocols", RFC 5245, DOI 10.17487/RFC5245, April 2010, <http://www.rfc-editor.org/info/rfc5245>.
- [RFC5285]  Singer, D. and H. Desineni, "A General Mechanism for RTP Header Extensions", RFC 5285, DOI 10.17487/RFC5285, July 2008, <http://www.rfc-editor.org/info/rfc5285>.
- [RFC3550]  Schulzrinne, H., Casner, S., Frederick, R., and V. Jacobson, "RTP: A Transport Protocol for Real-Time Applications", STD 64, RFC 3550, DOI 10.17487/RFC3550, July 2003, <http://www.rfc-editor.org/info/rfc3550>.
- [RFC5506]  Johansson, I. and M. Westerlund, "Support for Reduced-Size Real-Time Transport Control Protocol (RTCP): Opportunities and Consequences", RFC 5506, DOI 10.17487/RFC5506, April 2009, <http://www.rfc-editor.org/info/rfc5506>.
- [RFC4585]  Ott, J., Wenger, S., Sato, N., Burmeister, C., and J. Rey, "Extended RTP Profile for Real-time Transport Control Protocol (RTCP)-Based Feedback (RTP/AVPF)", RFC 4585, DOI 10.17487/RFC4585, July 2006, <http://www.rfc-editor.org/info/rfc4585>.
- [RFC5761]  Perkins, C. and M. Westerlund, "Multiplexing RTP Data and Control Packets on a Single Port", RFC 5761, DOI 10.17487/ RFC5761, April 2010, <http://www.rfc-editor.org/info/rfc5761>.
- [RFC7656]  Lennox, J., Gross, K., Nandakumar, S., Salgueiro, G., and B. Burman, Ed., "A Taxonomy of Semantics and Mechanisms for Real-Time Transport Protocol (RTP) Sources", RFC 7656, DOI 10.17487/RFC7656, November 2015, <http://www.rfc-editor.org/info/rfc7656>.

### 15.2.  Informative References

- [RFC3552]  Rescorla, E. and B. Korver, "Guidelines for Writing RFC Text on Security Considerations", BCP 72, RFC 3552, DOI 10.17487/RFC3552, July 2003, <http://www.rfc-editor.org/info/rfc3552>.
- [RFC6182]  Ford, A., Raiciu, C., Handley, M., Barre, S., and J. Iyengar, "Architectural Guidelines for Multipath TCP Development", RFC 6182, DOI 10.17487/RFC6182, March 2011, <http://www.rfc-editor.org/info/rfc6182>.
- [RFC4960]  Stewart, R., Ed., "Stream Control Transmission Protocol", RFC 4960, DOI 10.17487/RFC4960, September 2007, <http://www.rfc-editor.org/info/rfc4960>.
- [RFC5533]  Nordmark, E. and M. Bagnulo, "Shim6: Level 3 Multihoming Shim Protocol for IPv6", RFC 5533, DOI 10.17487/RFC5533, June 2009, <http://www.rfc-editor.org/info/rfc5533>.
- [RFC5117]  Westerlund, M. and S. Wenger, "RTP Topologies", RFC 5117, DOI 10.17487/RFC5117, January 2008, <http://www.rfc-editor.org/info/rfc5117>.
- [I-D.ietf-mmusic-rfc2326bis] Schulzrinne, H., Rao, A., Lanphier, R., Westerlund, M., and M. Stiemerling, "Real Time Streaming Protocol 2.0 (RTSP)", draft-ietf-mmusic-rfc2326bis-40 (work in progress), February 2014.
- [RFC3261]  Rosenberg, J., Schulzrinne, H., Camarillo, G., Johnston, A., Peterson, J., Sparks, R., Handley, M., and E. Schooler, "SIP: Session Initiation Protocol", RFC 3261, DOI 10.17487/RFC3261, June 2002, <http://www.rfc-editor.org/info/rfc3261>.
- [RFC3264]  Rosenberg, J. and H. Schulzrinne, "An Offer/Answer Model with Session Description Protocol (SDP)", RFC 3264, DOI 10.17487/RFC3264, June 2002, <http://www.rfc-editor.org/info/rfc3264>.
- [RFC4566]  Handley, M., Jacobson, V., and C. Perkins, "SDP: Session Description Protocol", RFC 4566, DOI 10.17487/RFC4566, July 2006, <http://www.rfc-editor.org/info/rfc4566>.
- [RFC6263]  Marjou, X. and A. Sollaud, "Application Mechanism for Keeping Alive the NAT Mappings Associated with RTP / RTP Control Protocol (RTCP) Flows", RFC 6263, DOI 10.17487/ RFC6263, June 2011, <http://www.rfc-editor.org/info/rfc6263>.
- [I-D.singh-mmusic-mprtp-sdp-extension] Singh, V., Ott, J., Karkkainen, T., Globisch, R., and T. Schierl, "Multipath RTP (MPRTP) attribute in Session Description Protocol", draft-singh-mmusic-mprtp-sdp- extension-04 (work in progress), September 2014.
- [I-D.reddy-mmusic-ice-best-interface-pcp] Reddy, T., Wing, D., Steeg, B., Penno, R., and V. Varun, "Improving ICE Interface Selection Using Port Control Protocol (PCP) Flow Extension", draft-reddy-mmusic-ice- best-interface-pcp-00 (work in progress), October 2013.
- [I-D.wing-mmusic-ice-mobility] Wing, D., Reddy, T., Patil, P., and P. Martinsen, "Mobility with ICE (MICE)", draft-wing-mmusic-ice- mobility-07 (work in progress), June 2014.
- [I-D.ietf-rmcat-eval-criteria] Varun, V., Ott, J., and S. Holmer, "Evaluating Congestion Control for Interactive Real-time Media", draft-ietf- rmcat-eval-criteria-05 (work in progress), March 2016.
- [ACM-MPRTP] Singh, V., Ahsan, S., and J. Ott, "MPRTP: multipath considerations for real-time media", in Proc. of ACM Multimedia Systems, MMSys, 2013.

## Appendix A.  Interoperating with Legacy Applications

Some legacy endpoints may abort processing incoming packets, if they are received from different source address.  This may occur due to the loop detection algorithm.  Additionally, it is also possible for the receiver to reset processing of the stream if the jump in packet sequence numbers received over multiple interface is large.  Both of these errors are based on implementation-specific details.

An MPRTP sender can use its multiple interfaces to send media to a legacy RTP client.  The legacy receiver will ignore the subflow RTP header extension and the receiver's de-jitter buffer will attempt to compensate for any mismatch in per-path latency.  However, the receiver will only send the overall or aggregate RTCP report which may be insufficient for an MPRTP sender to adequately schedule packets over multiple paths or detect if a path disappeared.

An MPRTP receiver can only use one of its interface when communicating with a legacy sender.

## Appendix B.  Change Log

Note to the RFC-Editor: please remove this section prior to publication as an RFC.

### B.1.  Changes in draft-ietf-avtcore-mprtp-03

- Incorporating review comments from Frederic Maze on incorporating new RTP Taxonomy.

### B.2.  Changes in draft-ietf-avtcore-mprtp-01, and -02

- Keep-alive versions, document needs review.
- Updated authors' affiliations.

### B.3.  Changes in draft-ietf-avtcore-mprtp-00

- Submitted as a WG item.

### B.4.  Changes in draft-singh-avtcore-mprtp-10

- Editorial updates based on review comments.
- Renamed length to encaps_length.

### B.5.  Changes in draft-singh-avtcore-mprtp-09

- Editorial updates based on review comments.
- Clarified use of a=rtcp-rsize.
- Fixed bug in block length of interface advertisements.

### B.6.  Changes in draft-singh-avtcore-mprtp-08

- Added reference to use of PCP for discovering new interfaces.

### B.7.  Changes in draft-singh-avtcore-mprtp-06 and -07

- Added reference to Mobility ICE.

### B.8.  Changes in draft-singh-avtcore-mprtp-05

- SDP extensions moved to draft-singh-mmusic-mprtp-sdp-extension-00. Kept only the basic 'a=mprtp' attribute in this document.
- Cleaned up ICE procedures for advertising only using in-band signaling.

### B.9.  Changes in draft-singh-avtcore-mprtp-04

- Fixed missing 0xBEDE header in MPRTP header format.
- Removed connectivity checks and keep-alives from in-band signaling.
- MPRTP and MPRTCP are multiplexed on a single port.
- MPRTCP packet headers optimized.
- Made ICE optional
- Updated Sections: 7.1.2, 8.1.x, 11.2, 11.4, 11.6.
- Added how to use MPRTP in RTSP (Section 12).
- Updated IANA Considerations section.

### B.10.  Changes in draft-singh-avtcore-mprtp-03

- Added this change log.
- Updated section 6, 7 and 8 based on comments from MMUSIC.
- Updated section 11 (SDP) based on comments of MMUSIC.
- Updated SDP examples with ICE and non-ICE in out-of-band signaling scenario.
- Added Appendix A on interop with legacy.
- Updated IANA Considerations section.

### B.11.  Changes in draft-singh-avtcore-mprtp-02

- MPRTCP protocol extensions use only one PT=210, instead of 210 and 211.
- RTP header uses 1-byte extension instead of 2-byte.
- Added section on RTCP Interval Calculations.
- Added "mprtp-interface" attribute in SDP considerations.

## B.12.  Changes in draft-singh-avtcore-mprtp-01

- Added MPRTP and MPRTCP protocol extensions and examples.
- WG changed from -avt to -avtcore.

## Editorial Comments

- [note-iceornot] Editor: Legacy applications do not require ICE for session establishment, therefore, MPRTP should not require it as well.
- [note-rtcp-reuse] Editor: inside the context of subflow specific reports can we reuse the payload type code for Sender Report (PT=200), Receiver Report (PT=201), Extension Report (PT=207).  Transport and Payload specific RTCP messages are session specific and SHOULD be used as before.
- [note-rtp-rtcp-mux] Editor: If a=mprtp is indicated, does the endpoint need to indicate a=rtcp-mux and a=rtcp-rsize? because MPRTP mandates the use of RTP and RTCP multiplexing, and Reduced-Size RTCP.

## Authors' Addresses

```
   Varun Singh
   Nemu Dialogue Systems Oy
   Runeberginkatu 4c A 4
   Helsinki  00100
   Finland

   Email: varun.singh@iki.fi
   URI:   http://www.callstats.io/


   Teemu Karkkainen
   Technical University of Munich
   Faculty of Informatics
   Boltzmannstrasse 3
   Garching bei Muenchen, DE  85748
   Germany

   Email: kaerkkae@in.tum.de


   Joerg Ott
   Technical University of Munich
   Faculty of Informatics
   Boltzmannstrasse 3
   Garching bei Muenchen, DE  85748
   Germany

   Email: ott@in.tum.de


   Saba Ahsan
   Aalto University
   School of Electrical Engineering
   Otakaari 5 A
   Espoo, FIN  02150
   Finland

   Email: saba.ahsan@aalto.fi


   Lars Eggert
   NetApp
   Sonnenallee 1
   Kirchheim  85551
   Germany

   Phone: +49 151 12055791
   Email: lars@netapp.com
   URI:   http://eggert.org/
```
