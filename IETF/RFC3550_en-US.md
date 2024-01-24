# RTP: A Transport Protocol for Real-Time Applications

> Origin [https://datatracker.ietf.org/doc/html/rfc3550](https://datatracker.ietf.org/doc/html/rfc3550)

## Status of this Memo

This document specifies an Internet standards track protocol for the Internet community, and requests discussion and suggestions for improvements.  Please refer to the current edition of the "Internet Official Protocol Standards" (STD 1) for the standardization state and status of this protocol.  Distribution of this memo is unlimited.

## Copyright Notice

Copyright (C) The Internet Society (2003).  All Rights Reserved.

## Abstract

This memorandum describes RTP, the real-time transport protocol.  RTP provides end-to-end network transport functions suitable for applications transmitting real-time data, such as audio, video or simulation data, over multicast or unicast network services.  RTP does not address resource reservation and does not guarantee quality-of-service for real-time services.  The data transport is augmented by a control protocol (RTCP) to allow monitoring of the data delivery in a manner scalable to large multicast networks, and to provide minimal control and identification functionality.  RTP and RTCP are designed to be independent of the underlying transport and network layers.  The protocol supports the use of RTP-level translators and mixers.

Most of the text in this memorandum is identical to RFC 1889 which it obsoletes.  There are no changes in the packet formats on the wire, only changes to the rules and algorithms governing how the protocol is used.  The biggest change is an enhancement to the scalable timer algorithm for calculating when to send RTCP packets in order to minimize transmission in excess of the intended rate when many participants join a session simultaneously.

## Table of Contents

```
   1.  Introduction ................................................   4
       1.1  Terminology ............................................   5
   2.  RTP Use Scenarios ...........................................   5
       2.1  Simple Multicast Audio Conference ......................   6
       2.2  Audio and Video Conference .............................   7
       2.3  Mixers and Translators .................................   7
       2.4  Layered Encodings ......................................   8
   3.  Definitions .................................................   8
   4.  Byte Order, Alignment, and Time Format ......................  12
   5.  RTP Data Transfer Protocol ..................................  13
       5.1  RTP Fixed Header Fields ................................  13
       5.2  Multiplexing RTP Sessions ..............................  16
       5.3  Profile-Specific Modifications to the RTP Header .......  18
            5.3.1  RTP Header Extension ............................  18
   6.  RTP Control Protocol -- RTCP ................................  19
       6.1  RTCP Packet Format .....................................  21
       6.2  RTCP Transmission Interval .............................  24
            6.2.1  Maintaining the Number of Session Members .......  28
       6.3  RTCP Packet Send and Receive Rules .....................  28
            6.3.1  Computing the RTCP Transmission Interval ........  29
            6.3.2  Initialization ..................................  30
            6.3.3  Receiving an RTP or Non-BYE RTCP Packet .........  31
            6.3.4  Receiving an RTCP BYE Packet ....................  31
            6.3.5  Timing Out an SSRC ..............................  32
            6.3.6  Expiration of Transmission Timer ................  32
            6.3.7  Transmitting a BYE Packet .......................  33
            6.3.8  Updating we_sent ................................  34
            6.3.9  Allocation of Source Description Bandwidth ......  34
       6.4  Sender and Receiver Reports ............................  35
            6.4.1  SR: Sender Report RTCP Packet ...................  36
            6.4.2  RR: Receiver Report RTCP Packet .................  42
            6.4.3  Extending the Sender and Receiver Reports .......  42
            6.4.4  Analyzing Sender and Receiver Reports ...........  43
       6.5  SDES: Source Description RTCP Packet ...................  45
            6.5.1  CNAME: Canonical End-Point Identifier SDES Item .  46
            6.5.2  NAME: User Name SDES Item .......................  48
            6.5.3  EMAIL: Electronic Mail Address SDES Item ........  48
            6.5.4  PHONE: Phone Number SDES Item ...................  49
            6.5.5  LOC: Geographic User Location SDES Item .........  49
            6.5.6  TOOL: Application or Tool Name SDES Item ........  49
            6.5.7  NOTE: Notice/Status SDES Item ...................  50
            6.5.8  PRIV: Private Extensions SDES Item ..............  50
       6.6  BYE: Goodbye RTCP Packet ...............................  51
       6.7  APP: Application-Defined RTCP Packet ...................  52
   7.  RTP Translators and Mixers ..................................  53
       7.1  General Description ....................................  53
       7.2  RTCP Processing in Translators .........................  55
       7.3  RTCP Processing in Mixers ..............................  57
       7.4  Cascaded Mixers ........................................  58
   8.  SSRC Identifier Allocation and Use ..........................  59
       8.1  Probability of Collision ...............................  59
       8.2  Collision Resolution and Loop Detection ................  60
       8.3  Use with Layered Encodings .............................  64
   9.  Security ....................................................  65
       9.1  Confidentiality ........................................  65
       9.2  Authentication and Message Integrity ...................  67
   10. Congestion Control ..........................................  67
   11. RTP over Network and Transport Protocols ....................  68
   12. Summary of Protocol Constants ...............................  69
       12.1 RTCP Packet Types ......................................  70
       12.2 SDES Types .............................................  70
   13. RTP Profiles and Payload Format Specifications ..............  71
   14. Security Considerations .....................................  73
   15. IANA Considerations .........................................  73
   16. Intellectual Property Rights Statement ......................  74
   17. Acknowledgments .............................................  74
   Appendix A.   Algorithms ........................................  75
   Appendix A.1  RTP Data Header Validity Checks ...................  78
   Appendix A.2  RTCP Header Validity Checks .......................  82
   Appendix A.3  Determining Number of Packets Expected and Lost ...  83
   Appendix A.4  Generating RTCP SDES Packets ......................  84
   Appendix A.5  Parsing RTCP SDES Packets .........................  85
   Appendix A.6  Generating a Random 32-bit Identifier .............  85
   Appendix A.7  Computing the RTCP Transmission Interval ..........  87
   Appendix A.8  Estimating the Interarrival Jitter ................  94
   Appendix B.   Changes from RFC 1889 .............................  95
   References ...................................................... 100
   Normative References ............................................ 100
   Informative References .......................................... 100
   Authors' Addresses .............................................. 103
   Full Copyright Statement ........................................ 104
```

## 1. Introduction

This memorandum specifies the real-time transport protocol (RTP), which provides end-to-end delivery services for data with real-time characteristics, such as interactive audio and video.  Those services include payload type identification, sequence numbering, timestamping and delivery monitoring.  Applications typically run RTP on top of UDP to make use of its multiplexing and checksum services; both protocols contribute parts of the transport protocol functionality. However, RTP may be used with other suitable underlying network or transport protocols (see Section 11).  RTP supports data transfer to multiple destinations using multicast distribution if provided by the underlying network.

Note that RTP itself does not provide any mechanism to ensure timely delivery or provide other quality-of-service guarantees, but relies on lower-layer services to do so.  It does not guarantee delivery or prevent out-of-order delivery, nor does it assume that the underlying network is reliable and delivers packets in sequence.  The sequence numbers included in RTP allow the receiver to reconstruct the sender's packet sequence, but sequence numbers might also be used to determine the proper location of a packet, for example in video decoding, without necessarily decoding packets in sequence.

While RTP is primarily designed to satisfy the needs of multi-participant multimedia conferences, it is not limited to that particular application.  Storage of continuous data, interactive distributed simulation, active badge, and control and measurement applications may also find RTP applicable.

This document defines RTP, consisting of two closely-linked parts:

- the real-time transport protocol (RTP), to carry data that has real-time properties.
- the RTP control protocol (RTCP), to monitor the quality of service and to convey information about the participants in an on-going session.  The latter aspect of RTCP may be sufficient for "loosely controlled" sessions, i.e., where there is no explicit membership control and set-up, but it is not necessarily intended to support all of an application's control communication requirements.  This functionality may be fully or partially subsumed by a separate session control protocol, which is beyond the scope of this document.

RTP represents a new style of protocol following the principles of application level framing and integrated layer processing proposed by Clark and Tennenhouse [10].  That is, RTP is intended to be malleable to provide the information required by a particular application and will often be integrated into the application processing rather than being implemented as a separate layer.  RTP is a protocol framework that is deliberately not complete.  This document specifies those functions expected to be common across all the applications for which RTP would be appropriate.  Unlike conventional protocols in which additional functions might be accommodated by making the protocol more general or by adding an option mechanism that would require parsing, RTP is intended to be tailored through modifications and/or additions to the headers as needed.  Examples are given in Sections 5.3 and 6.4.3.

Therefore, in addition to this document, a complete specification of RTP for a particular application will require one or more companion documents (see Section 13):

- a profile specification document, which defines a set of payload type codes and their mapping to payload formats (e.g., media encodings).  A profile may also define extensions or modifications to RTP that are specific to a particular class of applications. Typically an application will operate under only one profile.  A profile for audio and video data may be found in the companion RFC 3551 [1].
- payload format specification documents, which define how a particular payload, such as an audio or video encoding, is to be carried in RTP.

A discussion of real-time services and algorithms for their implementation as well as background discussion on some of the RTP design decisions can be found in [11].

### 1.1 Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14, RFC 2119 [2] and indicate requirement levels for compliant RTP implementations.

## 2. RTP Use Scenarios

The following sections describe some aspects of the use of RTP.  The examples were chosen to illustrate the basic operation of applications using RTP, not to limit what RTP may be used for.  In these examples, RTP is carried on top of IP and UDP, and follows the conventions established by the profile for audio and video specified in the companion RFC 3551.

### 2.1 Simple Multicast Audio Conference

A working group of the IETF meets to discuss the latest protocol document, using the IP multicast services of the Internet for voice communications.  Through some allocation mechanism the working group chair obtains a multicast group address and pair of ports.  One port is used for audio data, and the other is used for control (RTCP) packets.  This address and port information is distributed to the intended participants.  If privacy is desired, the data and control packets may be encrypted as specified in Section 9.1, in which case an encryption key must also be generated and distributed.  The exact details of these allocation and distribution mechanisms are beyond the scope of RTP.

The audio conferencing application used by each conference participant sends audio data in small chunks of, say, 20 ms duration. Each chunk of audio data is preceded by an RTP header; RTP header and data are in turn contained in a UDP packet.  The RTP header indicates what type of audio encoding (such as PCM, ADPCM or LPC) is contained in each packet so that senders can change the encoding during a conference, for example, to accommodate a new participant that is connected through a low-bandwidth link or react to indications of network congestion.

The Internet, like other packet networks, occasionally loses and reorders packets and delays them by variable amounts of time.  To cope with these impairments, the RTP header contains timing information and a sequence number that allow the receivers to reconstruct the timing produced by the source, so that in this example, chunks of audio are contiguously played out the speaker every 20 ms.  This timing reconstruction is performed separately for each source of RTP packets in the conference.  The sequence number can also be used by the receiver to estimate how many packets are being lost.

Since members of the working group join and leave during the conference, it is useful to know who is participating at any moment and how well they are receiving the audio data.  For that purpose, each instance of the audio application in the conference periodically multicasts a reception report plus the name of its user on the RTCP (control) port.  The reception report indicates how well the current speaker is being received and may be used to control adaptive encodings.  In addition to the user name, other identifying information may also be included subject to control bandwidth limits. A site sends the RTCP BYE packet (Section 6.6) when it leaves the conference.

### 2.2 Audio and Video Conference

If both audio and video media are used in a conference, they are transmitted as separate RTP sessions.  That is, separate RTP and RTCP packets are transmitted for each medium using two different UDP port pairs and/or multicast addresses.  There is no direct coupling at the RTP level between the audio and video sessions, except that a user participating in both sessions should use the same distinguished (canonical) name in the RTCP packets for both so that the sessions can be associated.

One motivation for this separation is to allow some participants in the conference to receive only one medium if they choose.  Further explanation is given in Section 5.2.  Despite the separation, synchronized playback of a source's audio and video can be achieved using timing information carried in the RTCP packets for both sessions.

### 2.3 Mixers and Translators

So far, we have assumed that all sites want to receive media data in the same format.  However, this may not always be appropriate. Consider the case where participants in one area are connected through a low-speed link to the majority of the conference participants who enjoy high-speed network access.  Instead of forcing everyone to use a lower-bandwidth, reduced-quality audio encoding, an RTP-level relay called a mixer may be placed near the low-bandwidth area.  This mixer resynchronizes incoming audio packets to reconstruct the constant 20 ms spacing generated by the sender, mixes these reconstructed audio streams into a single stream, translates the audio encoding to a lower-bandwidth one and forwards the lower- bandwidth packet stream across the low-speed link.  These packets might be unicast to a single recipient or multicast on a different address to multiple recipients.  The RTP header includes a means for mixers to identify the sources that contributed to a mixed packet so that correct talker indication can be provided at the receivers.

Some of the intended participants in the audio conference may be connected with high bandwidth links but might not be directly reachable via IP multicast.  For example, they might be behind an application-level firewall that will not let any IP packets pass. For these sites, mixing may not be necessary, in which case another type of RTP-level relay called a translator may be used.  Two translators are installed, one on either side of the firewall, with the outside one funneling all multicast packets received through a secure connection to the translator inside the firewall.  The translator inside the firewall sends them again as multicast packets to a multicast group restricted to the site's internal network.

Mixers and translators may be designed for a variety of purposes.  An example is a video mixer that scales the images of individual people in separate video streams and composites them into one video stream to simulate a group scene.  Other examples of translation include the connection of a group of hosts speaking only IP/UDP to a group of hosts that understand only ST-II, or the packet-by-packet encoding translation of video streams from individual sources without resynchronization or mixing.  Details of the operation of mixers and translators are given in Section 7.

### 2.4 Layered Encodings

Multimedia applications should be able to adjust the transmission rate to match the capacity of the receiver or to adapt to network congestion.  Many implementations place the responsibility of rate-adaptivity at the source.  This does not work well with multicast transmission because of the conflicting bandwidth requirements of heterogeneous receivers.  The result is often a least-common denominator scenario, where the smallest pipe in the network mesh dictates the quality and fidelity of the overall live multimedia "broadcast".

Instead, responsibility for rate-adaptation can be placed at the receivers by combining a layered encoding with a layered transmission system.  In the context of RTP over IP multicast, the source can stripe the progressive layers of a hierarchically represented signal across multiple RTP sessions each carried on its own multicast group. Receivers can then adapt to network heterogeneity and control their reception bandwidth by joining only the appropriate subset of the multicast groups.

Details of the use of RTP with layered encodings are given in Sections 6.3.9, 8.3 and 11.

## 3. Definitions

- **RTP payload**: The data transported by RTP in a packet, for example audio samples or compressed video data.  The payload format and interpretation are beyond the scope of this document.
- **RTP packet**: A data packet consisting of the fixed RTP header, a possibly empty list of contributing sources (see below), and the payload data.  Some underlying protocols may require an encapsulation of the RTP packet to be defined.  Typically one packet of the underlying protocol contains a single RTP packet, but several RTP packets MAY be contained if permitted by the encapsulation method (see Section 11).
- **RTCP packet**: A control packet consisting of a fixed header part similar to that of RTP data packets, followed by structured elements that vary depending upon the RTCP packet type.  The formats are defined in Section 6.  Typically, multiple RTCP packets are sent together as a compound RTCP packet in a single packet of the underlying protocol; this is enabled by the length field in the fixed header of each RTCP packet.
- **Port**: The "abstraction that transport protocols use to distinguish among multiple destinations within a given host computer.  TCP/IP protocols identify ports using small positive integers." [12] The transport selectors (TSEL) used by the OSI transport layer are equivalent to ports.  RTP depends upon the lower-layer protocol to provide some mechanism such as ports to multiplex the RTP and RTCP packets of a session.
- **Transport address**: The combination of a network address and port that identifies a transport-level endpoint, for example an IP address and a UDP port.  Packets are transmitted from a source transport address to a destination transport address.
- **RTP media type**: An RTP media type is the collection of payload types which can be carried within a single RTP session.  The RTP Profile assigns RTP media types to RTP payload types.
- **Multimedia session**: A set of concurrent RTP sessions among a common group of participants.  For example, a videoconference (which is a multimedia session) may contain an audio RTP session and a video RTP session.
- **RTP session**: An association among a set of participants communicating with RTP.  A participant may be involved in multiple RTP sessions at the same time.  In a multimedia session, each medium is typically carried in a separate RTP session with its own RTCP packets unless the the encoding itself multiplexes multiple media into a single data stream.  A participant distinguishes multiple RTP sessions by reception of different sessions using different pairs of destination transport addresses, where a pair of transport addresses comprises one network address plus a pair of ports for RTP and RTCP.  All participants in an RTP session may share a common destination transport address pair, as in the case of IP multicast, or the pairs may be different for each participant, as in the case of individual unicast network addresses and port pairs.  In the unicast case, a participant may receive from all other participants in the session using the same pair of ports, or may use a distinct pair of ports for each.

    The distinguishing feature of an RTP session is that each maintains a full, separate space of SSRC identifiers (defined next).  The set of participants included in one RTP session consists of those that can receive an SSRC identifier transmitted by any one of the participants either in RTP as the SSRC or a CSRC (also defined below) or in RTCP.  For example, consider a three- party conference implemented using unicast UDP with each participant receiving from the other two on separate port pairs. If each participant sends RTCP feedback about data received from one other participant only back to that participant, then the conference is composed of three separate point-to-point RTP sessions.  If each participant provides RTCP feedback about its reception of one other participant to both of the other participants, then the conference is composed of one multi-party RTP session.  The latter case simulates the behavior that would occur with IP multicast communication among the three participants.

    The RTP framework allows the variations defined here, but a particular control protocol or application design will usually impose constraints on these variations.
- **Synchronization source (SSRC)**: The source of a stream of RTP packets, identified by a 32-bit numeric SSRC identifier carried in the RTP header so as not to be dependent upon the network address. All packets from a synchronization source form part of the same timing and sequence number space, so a receiver groups packets by synchronization source for playback.  Examples of synchronization sources include the sender of a stream of packets derived from a signal source such as a microphone or a camera, or an RTP mixer (see below).  A synchronization source may change its data format, e.g., audio encoding, over time.  The SSRC identifier is a randomly chosen value meant to be globally unique within a particular RTP session (see Section 8).  A participant need not use the same SSRC identifier for all the RTP sessions in a multimedia session; the binding of the SSRC identifiers is provided through RTCP (see Section 6.5.1).  If a participant generates multiple streams in one RTP session, for example from separate video cameras, each MUST be identified as a different SSRC.
- **Contributing source (CSRC)**: A source of a stream of RTP packets that has contributed to the combined stream produced by an RTP mixer (see below).  The mixer inserts a list of the SSRC identifiers of the sources that contributed to the generation of a particular packet into the RTP header of that packet.  This list is called the CSRC list.  An example application is audio conferencing where a mixer indicates all the talkers whose speech was combined to produce the outgoing packet, allowing the receiver to indicate the current talker, even though all the audio packets contain the same SSRC identifier (that of the mixer).
- **End system**: An application that generates the content to be sent in RTP packets and/or consumes the content of received RTP packets.  An end system can act as one or more synchronization sources in a particular RTP session, but typically only one.
- **Mixer**: An intermediate system that receives RTP packets from one or more sources, possibly changes the data format, combines the packets in some manner and then forwards a new RTP packet.  Since the timing among multiple input sources will not generally be synchronized, the mixer will make timing adjustments among the streams and generate its own timing for the combined stream. Thus, all data packets originating from a mixer will be identified as having the mixer as their synchronization source.
- **Translator**: An intermediate system that forwards RTP packets with their synchronization source identifier intact.  Examples of translators include devices that convert encodings without mixing, replicators from multicast to unicast, and application-level filters in firewalls.
- **Monitor**: An application that receives RTCP packets sent by participants in an RTP session, in particular the reception reports, and estimates the current quality of service for distribution monitoring, fault diagnosis and long-term statistics. The monitor function is likely to be built into the application(s) participating in the session, but may also be a separate application that does not otherwise participate and does not send or receive the RTP data packets (since they are on a separate port).  These are called third-party monitors.  It is also acceptable for a third-party monitor to receive the RTP data packets but not send RTCP packets or otherwise be counted in the session.
- **Non-RTP means**: Protocols and mechanisms that may be needed in addition to RTP to provide a usable service.  In particular, for multimedia conferences, a control protocol may distribute multicast addresses and keys for encryption, negotiate the encryption algorithm to be used, and define dynamic mappings between RTP payload type values and the payload formats they represent for formats that do not have a predefined payload type value.  Examples of such protocols include the Session Initiation Protocol (SIP) (RFC 3261 [13]), ITU Recommendation H.323 [14] and applications using SDP (RFC 2327 [15]), such as RTSP (RFC 2326 [16]).  For simple applications, electronic mail or a conference database may also be used.  The specification of such protocols and mechanisms is outside the scope of this document.

## 4. Byte Order, Alignment, and Time Format

All integer fields are carried in network byte order, that is, most significant byte (octet) first.  This byte order is commonly known as big-endian.  The transmission order is described in detail in [3]. Unless otherwise noted, numeric constants are in decimal (base 10).

All header data is aligned to its natural length, i.e., 16-bit fields are aligned on even offsets, 32-bit fields are aligned at offsets divisible by four, etc.  Octets designated as padding have the value zero.

Wallclock time (absolute date and time) is represented using the timestamp format of the Network Time Protocol (NTP), which is in seconds relative to 0h UTC on 1 January 1900 [4].  The full resolution NTP timestamp is a 64-bit unsigned fixed-point number with the integer part in the first 32 bits and the fractional part in the last 32 bits.  In some fields where a more compact representation is appropriate, only the middle 32 bits are used; that is, the low 16 bits of the integer part and the high 16 bits of the fractional part. The high 16 bits of the integer part must be determined independently.

An implementation is not required to run the Network Time Protocol in order to use RTP.  Other time sources, or none at all, may be used (see the description of the NTP timestamp field in Section 6.4.1). However, running NTP may be useful for synchronizing streams transmitted from separate hosts.

The NTP timestamp will wrap around to zero some time in the year 2036, but for RTP purposes, only differences between pairs of NTP timestamps are used.  So long as the pairs of timestamps can be assumed to be within 68 years of each other, using modular arithmetic for subtractions and comparisons makes the wraparound irrelevant.

## 5. RTP Data Transfer Protocol

### 5.1 RTP Fixed Header Fields

The RTP header has the following format:

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P|X|  CC   |M|     PT      |       sequence number         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                           timestamp                           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           synchronization source (SSRC) identifier            |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |            contributing source (CSRC) identifiers             |
   |                             ....                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The first twelve octets are present in every RTP packet, while the list of CSRC identifiers is present only when inserted by a mixer. The fields have the following meaning:

- **version (V)**: 2 bits

    This field identifies the version of RTP.  The version defined by this specification is two (2).  (The value 1 is used by the first draft version of RTP and the value 0 is used by the protocol initially implemented in the "vat" audio tool.)
- **padding (P)**: 1 bit

    If the padding bit is set, the packet contains one or more additional padding octets at the end which are not part of the payload.  The last octet of the padding contains a count of how many padding octets should be ignored, including itself.  Padding may be needed by some encryption algorithms with fixed block sizes or for carrying several RTP packets in a lower-layer protocol data unit.
- **extension (X)**: 1 bit

    If the extension bit is set, the fixed header MUST be followed by exactly one header extension, with a format defined in Section 5.3.1.
- **CSRC count (CC)**: 4 bits

    The CSRC count contains the number of CSRC identifiers that follow the fixed header.
- **marker (M)**: 1 bit

    The interpretation of the marker is defined by a profile.  It is intended to allow significant events such as frame boundaries to be marked in the packet stream.  A profile MAY define additional marker bits or specify that there is no marker bit by changing the number of bits in the payload type field (see Section 5.3).
- **payload type (PT)**: 7 bits

    This field identifies the format of the RTP payload and determines its interpretation by the application.  A profile MAY specify a default static mapping of payload type codes to payload formats. Additional payload type codes MAY be defined dynamically through non-RTP means (see Section 3).  A set of default mappings for audio and video is specified in the companion RFC 3551 [1].  An RTP source MAY change the payload type during a session, but this field SHOULD NOT be used for multiplexing separate media streams (see Section 5.2).
    A receiver MUST ignore packets with payload types that it does not understand.
- **sequence number**: 16 bits

    The sequence number increments by one for each RTP data packet sent, and may be used by the receiver to detect packet loss and to restore packet sequence.  The initial value of the sequence number SHOULD be random (unpredictable) to make known-plaintext attacks on encryption more difficult, even if the source itself does not encrypt according to the method in Section 9.1, because the packets may flow through a translator that does.  Techniques for choosing unpredictable numbers are discussed in [17].
- **timestamp**: 32 bits

    The timestamp reflects the sampling instant of the first octet in the RTP data packet.  The sampling instant MUST be derived from a clock that increments monotonically and linearly in time to allow synchronization and jitter calculations (see Section 6.4.1).  The resolution of the clock MUST be sufficient for the desired synchronization accuracy and for measuring packet arrival jitter (one tick per video frame is typically not sufficient).  The clock frequency is dependent on the format of data carried as payload and is specified statically in the profile or payload format specification that defines the format, or MAY be specified dynamically for payload formats defined through non-RTP means.  If RTP packets are generated periodically, the nominal sampling instant as determined from the sampling clock is to be used, not a reading of the system clock.  As an example, for fixed-rate audio the timestamp clock would likely increment by one for each sampling period.  If an audio application reads blocks covering 160 sampling periods from the input device, the timestamp would be increased by 160 for each such block, regardless of whether the block is transmitted in a packet or dropped as silent.

    The initial value of the timestamp SHOULD be random, as for the sequence number.  Several consecutive RTP packets will have equal timestamps if they are (logically) generated at once, e.g., belong to the same video frame.  Consecutive RTP packets MAY contain timestamps that are not monotonic if the data is not transmitted in the order it was sampled, as in the case of MPEG interpolated video frames.  (The sequence numbers of the packets as transmitted will still be monotonic.)

    RTP timestamps from different media streams may advance at different rates and usually have independent, random offsets. Therefore, although these timestamps are sufficient to reconstruct the timing of a single stream, directly comparing RTP timestamps from different media is not effective for synchronization. Instead, for each medium the RTP timestamp is related to the sampling instant by pairing it with a timestamp from a reference clock (wallclock) that represents the time when the data corresponding to the RTP timestamp was sampled.  The reference clock is shared by all media to be synchronized.  The timestamp pairs are not transmitted in every data packet, but at a lower rate in RTCP SR packets as described in Section 6.4.

    The sampling instant is chosen as the point of reference for the RTP timestamp because it is known to the transmitting endpoint and has a common definition for all media, independent of encoding delays or other processing.  The purpose is to allow synchronized presentation of all media sampled at the same time.

    Applications transmitting stored data rather than data sampled in real time typically use a virtual presentation timeline derived from wallclock time to determine when the next frame or other unit of each medium in the stored data should be presented.  In this case, the RTP timestamp would reflect the presentation time for each unit.  That is, the RTP timestamp for each unit would be related to the wallclock time at which the unit becomes current on the virtual presentation timeline.  Actual presentation occurs some time later as determined by the receiver.

    An example describing live audio narration of prerecorded video illustrates the significance of choosing the sampling instant as the reference point.  In this scenario, the video would be presented locally for the narrator to view and would be simultaneously transmitted using RTP.  The "sampling instant" of a video frame transmitted in RTP would be established by referencing its timestamp to the wallclock time when that video frame was presented to the narrator.  The sampling instant for the audio RTP packets containing the narrator's speech would be established by referencing the same wallclock time when the audio was sampled. The audio and video may even be transmitted by different hosts if the reference clocks on the two hosts are synchronized by some means such as NTP.  A receiver can then synchronize presentation of the audio and video packets by relating their RTP timestamps using the timestamp pairs in RTCP SR packets.
- **SSRC**: 32 bits

    The SSRC field identifies the synchronization source.  This identifier SHOULD be chosen randomly, with the intent that no two synchronization sources within the same RTP session will have the same SSRC identifier.  An example algorithm for generating a random identifier is presented in Appendix A.6.  Although the probability of multiple sources choosing the same identifier is low, all RTP implementations must be prepared to detect and resolve collisions.  Section 8 describes the probability of collision along with a mechanism for resolving collisions and detecting RTP-level forwarding loops based on the uniqueness of the SSRC identifier.  If a source changes its source transport address, it must also choose a new SSRC identifier to avoid being interpreted as a looped source (see Section 8.2).
- **CSRC list**: 0 to 15 items, 32 bits each

    The CSRC list identifies the contributing sources for the payload contained in this packet.  The number of identifiers is given by the CC field.  If there are more than 15 contributing sources, only 15 can be identified.  CSRC identifiers are inserted by mixers (see Section 7.1), using the SSRC identifiers of contributing sources.  For example, for audio packets the SSRC identifiers of all sources that were mixed together to create a packet are listed, allowing correct talker indication at the receiver.

### 5.2 Multiplexing RTP Sessions

For efficient protocol processing, the number of multiplexing points should be minimized, as described in the integrated layer processing design principle [10].  In RTP, multiplexing is provided by the destination transport address (network address and port number) which is different for each RTP session.  For example, in a teleconference composed of audio and video media encoded separately, each medium SHOULD be carried in a separate RTP session with its own destination transport address.

Separate audio and video streams SHOULD NOT be carried in a single RTP session and demultiplexed based on the payload type or SSRC fields.  Interleaving packets with different RTP media types but using the same SSRC would introduce several problems:

1. If, say, two audio streams shared the same RTP session and the same SSRC value, and one were to change encodings and thus acquire a different RTP payload type, there would be no general way of identifying which stream had changed encodings.
2. An SSRC is defined to identify a single timing and sequence number space.  Interleaving multiple payload types would require different timing spaces if the media clock rates differ and would require different sequence number spaces to tell which payload type suffered packet loss.
3. The RTCP sender and receiver reports (see Section 6.4) can only describe one timing and sequence number space per SSRC and do not carry a payload type field.
4. An RTP mixer would not be able to combine interleaved streams of incompatible media into one stream.
5. Carrying multiple media in one RTP session precludes: the use of different network paths or network resource allocations if appropriate; reception of a subset of the media if desired, for example just audio if video would exceed the available bandwidth; and receiver implementations that use separate processes for the different media, whereas using separate RTP sessions permits either single- or multiple-process implementations.

Using a different SSRC for each medium but sending them in the same RTP session would avoid the first three problems but not the last two.

On the other hand, multiplexing multiple related sources of the same medium in one RTP session using different SSRC values is the norm for multicast sessions.  The problems listed above don't apply: an RTP mixer can combine multiple audio sources, for example, and the same treatment is applicable for all of them.  It may also be appropriate to multiplex streams of the same medium using different SSRC values in other scenarios where the last two problems do not apply.

### 5.3 Profile-Specific Modifications to the RTP Header

The existing RTP data packet header is believed to be complete for the set of functions required in common across all the application classes that RTP might support.  However, in keeping with the ALF design principle, the header MAY be tailored through modifications or additions defined in a profile specification while still allowing profile-independent monitoring and recording tools to function.

- The marker bit and payload type field carry profile-specific information, but they are allocated in the fixed header since many applications are expected to need them and might otherwise have to add another 32-bit word just to hold them.  The octet containing these fields MAY be redefined by a profile to suit different requirements, for example with more or fewer marker bits.  If there are any marker bits, one SHOULD be located in the most significant bit of the octet since profile-independent monitors may be able to observe a correlation between packet loss patterns and the marker bit.
- Additional information that is required for a particular payload format, such as a video encoding, SHOULD be carried in the payload section of the packet.  This might be in a header that is always present at the start of the payload section, or might be indicated by a reserved value in the data pattern.
- If a particular class of applications needs additional functionality independent of payload format, the profile under which those applications operate SHOULD define additional fixed fields to follow immediately after the SSRC field of the existing fixed header.  Those applications will be able to quickly and directly access the additional fields while profile-independent monitors or recorders can still process the RTP packets by interpreting only the first twelve octets.
- If it turns out that additional functionality is needed in common across all profiles, then a new version of RTP should be defined to make a permanent change to the fixed header.

#### 5.3.1 RTP Header Extension

An extension mechanism is provided to allow individual implementations to experiment with new payload-format-independent functions that require additional information to be carried in the RTP data packet header.  This mechanism is designed so that the header extension may be ignored by other interoperating implementations that have not been extended.

Note that this header extension is intended only for limited use. Most potential uses of this mechanism would be better done another way, using the methods described in the previous section.  For example, a profile-specific extension to the fixed header is less expensive to process because it is not conditional nor in a variable location.  Additional information required for a particular payload format SHOULD NOT use this header extension, but SHOULD be carried in the payload section of the packet.

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |      defined by profile       |           length              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                        header extension                       |
   |                             ....                              |
```

If the X bit in the RTP header is one, a variable-length header extension MUST be appended to the RTP header, following the CSRC list if present.  The header extension contains a 16-bit length field that counts the number of 32-bit words in the extension, excluding the four-octet extension header (therefore zero is a valid length).  Only a single extension can be appended to the RTP data header.  To allow multiple interoperating implementations to each experiment independently with different header extensions, or to allow a particular implementation to experiment with more than one type of header extension, the first 16 bits of the header extension are left open for distinguishing identifiers or parameters.  The format of these 16 bits is to be defined by the profile specification under which the implementations are operating.  This RTP specification does not define any header extensions itself.

## 6. RTP Control Protocol -- RTCP

The RTP control protocol (RTCP) is based on the periodic transmission of control packets to all participants in the session, using the same distribution mechanism as the data packets.  The underlying protocol MUST provide multiplexing of the data and control packets, for example using separate port numbers with UDP.  RTCP performs four functions:

1. The primary function is to provide feedback on the quality of the data distribution.  This is an integral part of the RTP's role as a transport protocol and is related to the flow and congestion control functions of other transport protocols (see Section 10 on the requirement for congestion control).  The feedback may be directly useful for control of adaptive encodings [18,19], but experiments with IP multicasting have shown that it is also critical to get feedback from the receivers to diagnose faults in the distribution.  Sending reception feedback reports to all participants allows one who is observing problems to evaluate whether those problems are local or global.  With a distribution mechanism like IP multicast, it is also possible for an entity such as a network service provider who is not otherwise involved in the session to receive the feedback information and act as a third-party monitor to diagnose network problems.  This feedback function is performed by the RTCP sender and receiver reports, described below in Section 6.4.
2. RTCP carries a persistent transport-level identifier for an RTP source called the canonical name or CNAME, Section 6.5.1.  Since the SSRC identifier may change if a conflict is discovered or a program is restarted, receivers require the CNAME to keep track of each participant.  Receivers may also require the CNAME to associate multiple data streams from a given participant in a set of related RTP sessions, for example to synchronize audio and video.  Inter-media synchronization also requires the NTP and RTP timestamps included in RTCP packets by data senders.
3. The first two functions require that all participants send RTCP packets, therefore the rate must be controlled in order for RTP to scale up to a large number of participants.  By having each participant send its control packets to all the others, each can independently observe the number of participants.  This number is used to calculate the rate at which the packets are sent, as explained in Section 6.2.
4. A fourth, OPTIONAL function is to convey minimal session control information, for example participant identification to be displayed in the user interface.  This is most likely to be useful in "loosely controlled" sessions where participants enter and leave without membership control or parameter negotiation.  RTCP serves as a convenient channel to reach all the participants, but it is not necessarily expected to support all the control communication requirements of an application.  A higher-level session control protocol, which is beyond the scope of this document, may be needed.

Functions 1-3 SHOULD be used in all environments, but particularly in the IP multicast environment.  RTP application designers SHOULD avoid mechanisms that can only work in unicast mode and will not scale to larger numbers.  Transmission of RTCP MAY be controlled separately for senders and receivers, as described in Section 6.2, for cases such as unidirectional links where feedback from receivers is not possible.

Non-normative note:  In the multicast routing approach called Source-Specific Multicast (SSM), there is only one sender per "channel" (a source address, group address pair), and receivers (except for the channel source) cannot use multicast to communicate directly with other channel members.  The recommendations here accommodate SSM only through Section 6.2's option of turning off receivers' RTCP entirely.  Future work will specify adaptation of RTCP for SSM so that feedback from receivers can be maintained.

### 6.1 RTCP Packet Format

This specification defines several RTCP packet types to carry a variety of control information:

- **SR**:   Sender report, for transmission and reception statistics from participants that are active senders
- **RR**:   Receiver report, for reception statistics from participants that are not active senders and in combination with SR for active senders reporting on more than 31 sources
- **SDES**: Source description items, including CNAME
- **BYE**:  Indicates end of participation
- **APP**:  Application-specific functions

Each RTCP packet begins with a fixed part similar to that of RTP data packets, followed by structured elements that MAY be of variable length according to the packet type but MUST end on a 32-bit boundary.  The alignment requirement and a length field in the fixed part of each packet are included to make RTCP packets "stackable". Multiple RTCP packets can be concatenated without any intervening separators to form a compound RTCP packet that is sent in a single packet of the lower layer protocol, for example UDP.  There is no explicit count of individual RTCP packets in the compound packet since the lower layer protocols are expected to provide an overall length to determine the end of the compound packet.

Each individual RTCP packet in the compound packet may be processed independently with no requirements upon the order or combination of packets.  However, in order to perform the functions of the protocol, the following constraints are imposed:

- Reception statistics (in SR or RR) should be sent as often as bandwidth constraints will allow to maximize the resolution of the statistics, therefore each periodically transmitted compound RTCP packet MUST include a report packet.
- New receivers need to receive the CNAME for a source as soon as possible to identify the source and to begin associating media for purposes such as lip-sync, so each compound RTCP packet MUST also include the SDES CNAME except when the compound RTCP packet is split for partial encryption as described in Section 9.1.
- The number of packet types that may appear first in the compound packet needs to be limited to increase the number of constant bits in the first word and the probability of successfully validating RTCP packets against misaddressed RTP data packets or other unrelated packets.

Thus, all RTCP packets MUST be sent in a compound packet of at least two individual packets, with the following format:

- **Encryption prefix**:  If and only if the compound packet is to be encrypted according to the method in Section 9.1, it MUST be prefixed by a random 32-bit quantity redrawn for every compound packet transmitted.  If padding is required for the encryption, it MUST be added to the last packet of the compound packet.
- **SR or RR**:  The first RTCP packet in the compound packet MUST always be a report packet to facilitate header validation as described in Appendix A.2.  This is true even if no data has been sent or received, in which case an empty RR MUST be sent, and even if the only other RTCP packet in the compound packet is a BYE.
- **Additional RRs**:  If the number of sources for which reception statistics are being reported exceeds 31, the number that will fit into one SR or RR packet, then additional RR packets SHOULD follow the initial report packet.
- **SDES**:  An SDES packet containing a CNAME item MUST be included in each compound RTCP packet, except as noted in Section 9.1. Other source description items MAY optionally be included if required by a particular application, subject to bandwidth constraints (see Section 6.3.9).
- **BYE or APP**:  Other RTCP packet types, including those yet to be defined, MAY follow in any order, except that BYE SHOULD be the last packet sent with a given SSRC/CSRC.  Packet types MAY appear more than once.

An individual RTP participant SHOULD send only one compound RTCP packet per report interval in order for the RTCP bandwidth per participant to be estimated correctly (see Section 6.2), except when the compound RTCP packet is split for partial encryption as described in Section 9.1.  If there are too many sources to fit all the necessary RR packets into one compound RTCP packet without exceeding the maximum transmission unit (MTU) of the network path, then only the subset that will fit into one MTU SHOULD be included in each interval.  The subsets SHOULD be selected round-robin across multiple intervals so that all sources are reported.

It is RECOMMENDED that translators and mixers combine individual RTCP packets from the multiple sources they are forwarding into one compound packet whenever feasible in order to amortize the packet overhead (see Section 7).  An example RTCP compound packet as might be produced by a mixer is shown in Fig. 1.  If the overall length of a compound packet would exceed the MTU of the network path, it SHOULD be segmented into multiple shorter compound packets to be transmitted in separate packets of the underlying protocol.  This does not impair the RTCP bandwidth estimation because each compound packet represents at least one distinct participant.  Note that each of the compound packets MUST begin with an SR or RR packet.

An implementation SHOULD ignore incoming RTCP packets with types unknown to it.  Additional RTCP packet types may be registered with the Internet Assigned Numbers Authority (IANA) as described in Section 15.

```
   if encrypted: random 32-bit integer
   |
   |[--------- packet --------][---------- packet ----------][-packet-]
   |
   |                receiver            chunk        chunk
   V                reports           item  item   item  item
   --------------------------------------------------------------------
   R[SR #sendinfo #site1#site2][SDES #CNAME PHONE #CNAME LOC][BYE##why]
   --------------------------------------------------------------------
   |                                                                  |
   |<-----------------------  compound packet ----------------------->|
   |<--------------------------  UDP packet ------------------------->|

   #: SSRC/CSRC identifier

              Figure 1: Example of an RTCP compound packet
```

### 6.2 RTCP Transmission Interval

RTP is designed to allow an application to scale automatically over session sizes ranging from a few participants to thousands.  For example, in an audio conference the data traffic is inherently self- limiting because only one or two people will speak at a time, so with multicast distribution the data rate on any given link remains relatively constant independent of the number of participants. However, the control traffic is not self-limiting.  If the reception reports from each participant were sent at a constant rate, the control traffic would grow linearly with the number of participants. Therefore, the rate must be scaled down by dynamically calculating the interval between RTCP packet transmissions.

For each session, it is assumed that the data traffic is subject to an aggregate limit called the "session bandwidth" to be divided among the participants.  This bandwidth might be reserved and the limit enforced by the network.  If there is no reservation, there may be other constraints, depending on the environment, that establish the "reasonable" maximum for the session to use, and that would be the session bandwidth.  The session bandwidth may be chosen based on some cost or a priori knowledge of the available network bandwidth for the session.  It is somewhat independent of the media encoding, but the encoding choice may be limited by the session bandwidth.  Often, the session bandwidth is the sum of the nominal bandwidths of the senders expected to be concurrently active.  For teleconference audio, this number would typically be one sender's bandwidth.  For layered encodings, each layer is a separate RTP session with its own session bandwidth parameter.

The session bandwidth parameter is expected to be supplied by a session management application when it invokes a media application, but media applications MAY set a default based on the single-sender data bandwidth for the encoding selected for the session.  The application MAY also enforce bandwidth limits based on multicast scope rules or other criteria.  All participants MUST use the same value for the session bandwidth so that the same RTCP interval will be calculated.

Bandwidth calculations for control and data traffic include lower-layer transport and network protocols (e.g., UDP and IP) since that is what the resource reservation system would need to know.  The application can also be expected to know which of these protocols are in use.  Link level headers are not included in the calculation since the packet will be encapsulated with different link level headers as it travels.

The control traffic should be limited to a small and known fraction of the session bandwidth: small so that the primary function of the transport protocol to carry data is not impaired; known so that the control traffic can be included in the bandwidth specification given to a resource reservation protocol, and so that each participant can independently calculate its share.  The control traffic bandwidth is in addition to the session bandwidth for the data traffic.  It is RECOMMENDED that the fraction of the session bandwidth added for RTCP be fixed at 5%.  It is also RECOMMENDED that 1/4 of the RTCP bandwidth be dedicated to participants that are sending data so that in sessions with a large number of receivers but a small number of senders, newly joining participants will more quickly receive the CNAME for the sending sites.  When the proportion of senders is greater than 1/4 of the participants, the senders get their proportion of the full RTCP bandwidth.  While the values of these and other constants in the interval calculation are not critical, all participants in the session MUST use the same values so the same interval will be calculated.  Therefore, these constants SHOULD be fixed for a particular profile.

A profile MAY specify that the control traffic bandwidth may be a separate parameter of the session rather than a strict percentage of the session bandwidth.  Using a separate parameter allows rate- adaptive applications to set an RTCP bandwidth consistent with a "typical" data bandwidth that is lower than the maximum bandwidth specified by the session bandwidth parameter.

The profile MAY further specify that the control traffic bandwidth may be divided into two separate session parameters for those participants which are active data senders and those which are not; let us call the parameters S and R.  Following the recommendation that 1/4 of the RTCP bandwidth be dedicated to data senders, the RECOMMENDED default values for these two parameters would be 1.25% and 3.75%, respectively.  When the proportion of senders is greater than S/(S+R) of the participants, the senders get their proportion of the sum of these parameters.  Using two parameters allows RTCP reception reports to be turned off entirely for a particular session by setting the RTCP bandwidth for non-data-senders to zero while keeping the RTCP bandwidth for data senders non-zero so that sender reports can still be sent for inter-media synchronization.  Turning off RTCP reception reports is NOT RECOMMENDED because they are needed for the functions listed at the beginning of Section 6, particularly reception quality feedback and congestion control.  However, doing so may be appropriate for systems operating on unidirectional links or for sessions that don't require feedback on the quality of reception or liveness of receivers and that have other means to avoid congestion.

The calculated interval between transmissions of compound RTCP packets SHOULD also have a lower bound to avoid having bursts of packets exceed the allowed bandwidth when the number of participants is small and the traffic isn't smoothed according to the law of large numbers.  It also keeps the report interval from becoming too small during transient outages like a network partition such that adaptation is delayed when the partition heals.  At application startup, a delay SHOULD be imposed before the first compound RTCP packet is sent to allow time for RTCP packets to be received from other participants so the report interval will converge to the correct value more quickly.  This delay MAY be set to half the minimum interval to allow quicker notification that the new participant is present.  The RECOMMENDED value for a fixed minimum interval is 5 seconds.

An implementation MAY scale the minimum RTCP interval to a smaller value inversely proportional to the session bandwidth parameter with the following limitations:

- For multicast sessions, only active data senders MAY use the reduced minimum value to calculate the interval for transmission of compound RTCP packets.
- For unicast sessions, the reduced value MAY be used by participants that are not active data senders as well, and the delay before sending the initial compound RTCP packet MAY be zero.
- For all sessions, the fixed minimum SHOULD be used when calculating the participant timeout interval (see Section 6.3.5) so that implementations which do not use the reduced value for transmitting RTCP packets are not timed out by other participants prematurely.
- The RECOMMENDED value for the reduced minimum in seconds is 360 divided by the session bandwidth in kilobits/second.  This minimum is smaller than 5 seconds for bandwidths greater than 72 kb/s.

The algorithm described in Section 6.3 and Appendix A.7 was designed to meet the goals outlined in this section.  It calculates the interval between sending compound RTCP packets to divide the allowed control traffic bandwidth among the participants.  This allows an application to provide fast response for small sessions where, for example, identification of all participants is important, yet automatically adapt to large sessions.  The algorithm incorporates the following characteristics:

- The calculated interval between RTCP packets scales linearly with the number of members in the group.  It is this linear factor which allows for a constant amount of control traffic when summed across all members.
- The interval between RTCP packets is varied randomly over the range [0.5,1.5] times the calculated interval to avoid unintended synchronization of all participants [20].  The first RTCP packet sent after joining a session is also delayed by a random variation of half the minimum RTCP interval.
- A dynamic estimate of the average compound RTCP packet size is calculated, including all those packets received and sent, to automatically adapt to changes in the amount of control information carried.
- Since the calculated interval is dependent on the number of observed group members, there may be undesirable startup effects when a new user joins an existing session, or many users simultaneously join a new session.  These new users will initially have incorrect estimates of the group membership, and thus their RTCP transmission interval will be too short.  This problem can be significant if many users join the session simultaneously.  To deal with this, an algorithm called "timer reconsideration" is employed.  This algorithm implements a simple back-off mechanism which causes users to hold back RTCP packet transmission if the group sizes are increasing.
- When users leave a session, either with a BYE or by timeout, the group membership decreases, and thus the calculated interval should decrease.  A "reverse reconsideration" algorithm is used to allow members to more quickly reduce their intervals in response to group membership decreases.
- BYE packets are given different treatment than other RTCP packets. When a user leaves a group, and wishes to send a BYE packet, it may do so before its next scheduled RTCP packet.  However, transmission of BYEs follows a back-off algorithm which avoids floods of BYE packets should a large number of members simultaneously leave the session.

This algorithm may be used for sessions in which all participants are allowed to send.  In that case, the session bandwidth parameter is the product of the individual sender's bandwidth times the number of participants, and the RTCP bandwidth is 5% of that.

Details of the algorithm's operation are given in the sections that follow.  Appendix A.7 gives an example implementation.

#### 6.2.1 Maintaining the Number of Session Members

Calculation of the RTCP packet interval depends upon an estimate of the number of sites participating in the session.  New sites are added to the count when they are heard, and an entry for each SHOULD be created in a table indexed by the SSRC or CSRC identifier (see Section 8.2) to keep track of them.  New entries MAY be considered not valid until multiple packets carrying the new SSRC have been received (see Appendix A.1), or until an SDES RTCP packet containing a CNAME for that SSRC has been received.  Entries MAY be deleted from the table when an RTCP BYE packet with the corresponding SSRC identifier is received, except that some straggler data packets might arrive after the BYE and cause the entry to be recreated.  Instead, the entry SHOULD be marked as having received a BYE and then deleted after an appropriate delay.

A participant MAY mark another site inactive, or delete it if not yet valid, if no RTP or RTCP packet has been received for a small number of RTCP report intervals (5 is RECOMMENDED).  This provides some robustness against packet loss.  All sites must have the same value for this multiplier and must calculate roughly the same value for the RTCP report interval in order for this timeout to work properly. Therefore, this multiplier SHOULD be fixed for a particular profile.

For sessions with a very large number of participants, it may be impractical to maintain a table to store the SSRC identifier and state information for all of them.  An implementation MAY use SSRC sampling, as described in [21], to reduce the storage requirements. An implementation MAY use any other algorithm with similar performance.  A key requirement is that any algorithm considered SHOULD NOT substantially underestimate the group size, although it MAY overestimate.

### 6.3 RTCP Packet Send and Receive Rules

The rules for how to send, and what to do when receiving an RTCP packet are outlined here.  An implementation that allows operation in a multicast environment or a multipoint unicast environment MUST meet the requirements in Section 6.2.  Such an implementation MAY use the algorithm defined in this section to meet those requirements, or MAY use some other algorithm so long as it provides equivalent or better performance.  An implementation which is constrained to two-party unicast operation SHOULD still use randomization of the RTCP transmission interval to avoid unintended synchronization of multiple instances operating in the same environment, but MAY omit the "timer reconsideration" and "reverse reconsideration" algorithms in Sections 6.3.3, 6.3.6 and 6.3.7.

To execute these rules, a session participant must maintain several pieces of state:

- **tp**: the last time an RTCP packet was transmitted;
- **tc**: the current time;
- **tn**: the next scheduled transmission time of an RTCP packet;
- **pmembers**: the estimated number of session members at the time tn was last recomputed;
- **members**: the most current estimate for the number of session members;
- **senders**: the most current estimate for the number of senders in the session;
- **rtcp_bw**: The target RTCP bandwidth, i.e., the total bandwidth that will be used for RTCP packets by all members of this session, in octets per second.  This will be a specified fraction of the "session bandwidth" parameter supplied to the application at startup.
- **we_sent**: Flag that is true if the application has sent data since the 2nd previous RTCP report was transmitted.
- **avg_rtcp_size**: The average compound RTCP packet size, in octets, over all RTCP packets sent and received by this participant.  The size includes lower-layer transport and network protocol headers (e.g., UDP and IP) as explained in Section 6.2.
- **initial**: Flag that is true if the application has not yet sent an RTCP packet.

Many of these rules make use of the "calculated interval" between packet transmissions.  This interval is described in the following section.

#### 6.3.1 Computing the RTCP Transmission Interval

To maintain scalability, the average interval between packets from a session participant should scale with the group size.  This interval is called the calculated interval.  It is obtained by combining a number of the pieces of state described above.  The calculated interval T is then determined as follows:

1. If the number of senders is less than or equal to 25% of the membership (members), the interval depends on whether the participant is a sender or not (based on the value of we_sent). If the participant is a sender (we_sent true), the constant C is set to the average RTCP packet size (avg_rtcp_size) divided by 25% of the RTCP bandwidth (rtcp_bw), and the constant n is set to the number of senders.  If we_sent is not true, the constant C is set to the average RTCP packet size divided by 75% of the RTCP bandwidth.  The constant n is set to the number of receivers (members - senders).  If the number of senders is greater than 25%, senders and receivers are treated together.  The constant C is set to the average RTCP packet size divided by the total RTCP bandwidth and n is set to the total number of members.  As stated in Section 6.2, an RTP profile MAY specify that the RTCP bandwidth may be explicitly defined by two separate parameters (call them S and R) for those participants which are senders and those which are not.  In that case, the 25% fraction becomes S/(S+R) and the 75% fraction becomes R/(S+R).  Note that if R is zero, the percentage of senders is never greater than S/(S+R), and the implementation must avoid division by zero.
2. If the participant has not yet sent an RTCP packet (the variable initial is true), the constant Tmin is set to 2.5 seconds, else it is set to 5 seconds.
3. The deterministic calculated interval Td is set to max(Tmin, n*C).
4. The calculated interval T is set to a number uniformly distributed between 0.5 and 1.5 times the deterministic calculated interval.
5. The resulting value of T is divided by e-3/2=1.21828 to compensate for the fact that the timer reconsideration algorithm converges to a value of the RTCP bandwidth below the intended average.

This procedure results in an interval which is random, but which, on average, gives at least 25% of the RTCP bandwidth to senders and the rest to receivers.  If the senders constitute more than one quarter of the membership, this procedure splits the bandwidth equally among all participants, on average.

#### 6.3.2 Initialization

Upon joining the session, the participant initializes tp to 0, tc to 0, senders to 0, pmembers to 1, members to 1, we_sent to false, rtcp_bw to the specified fraction of the session bandwidth, initial to true, and avg_rtcp_size to the probable size of the first RTCP packet that the application will later construct.  The calculated interval T is then computed, and the first packet is scheduled for time tn = T.  This means that a transmission timer is set which expires at time T.  Note that an application MAY use any desired approach for implementing this timer.

The participant adds its own SSRC to the member table.

#### 6.3.3 Receiving an RTP or Non-BYE RTCP Packet

When an RTP or RTCP packet is received from a participant whose SSRC is not in the member table, the SSRC is added to the table, and the value for members is updated once the participant has been validated as described in Section 6.2.1.  The same processing occurs for each CSRC in a validated RTP packet.

When an RTP packet is received from a participant whose SSRC is not in the sender table, the SSRC is added to the table, and the value for senders is updated.

For each compound RTCP packet received, the value of avg_rtcp_size is updated:

```
      avg_rtcp_size = (1/16) * packet_size + (15/16) * avg_rtcp_size
```
where packet_size is the size of the RTCP packet just received.


#### 6.3.4 Receiving an RTCP BYE Packet

Except as described in Section 6.3.7 for the case when an RTCP BYE is to be transmitted, if the received packet is an RTCP BYE packet, the SSRC is checked against the member table.  If present, the entry is removed from the table, and the value for members is updated.  The SSRC is then checked against the sender table.  If present, the entry is removed from the table, and the value for senders is updated.

Furthermore, to make the transmission rate of RTCP packets more adaptive to changes in group membership, the following "reverse reconsideration" algorithm SHOULD be executed when a BYE packet is received that reduces members to a value less than pmembers:

- The value for tn is updated according to the following formula:
    ```
         tn = tc + (members/pmembers) * (tn - tc)
    ```
- The value for tp is updated according the following formula:
    ```
         tp = tc - (members/pmembers) * (tc - tp).
    ```
- The next RTCP packet is rescheduled for transmission at time tn, which is now earlier.
- The value of pmembers is set equal to members.

This algorithm does not prevent the group size estimate from incorrectly dropping to zero for a short time due to premature timeouts when most participants of a large session leave at once but some remain.  The algorithm does make the estimate return to the correct value more rapidly.  This situation is unusual enough and the consequences are sufficiently harmless that this problem is deemed only a secondary concern.

#### 6.3.5 Timing Out an SSRC

At occasional intervals, the participant MUST check to see if any of the other participants time out.  To do this, the participant computes the deterministic (without the randomization factor) calculated interval Td for a receiver, that is, with we_sent false. Any other session member who has not sent an RTP or RTCP packet since time tc - MTd (M is the timeout multiplier, and defaults to 5) is timed out.  This means that its SSRC is removed from the member list, and members is updated.  A similar check is performed on the sender list.  Any member on the sender list who has not sent an RTP packet since time tc - 2T (within the last two RTCP report intervals) is removed from the sender list, and senders is updated.

If any members time out, the reverse reconsideration algorithm described in Section 6.3.4 SHOULD be performed.

The participant MUST perform this check at least once per RTCP transmission interval.

#### 6.3.6 Expiration of Transmission Timer

When the packet transmission timer expires, the participant performs the following operations:

- The transmission interval T is computed as described in Section 6.3.1, including the randomization factor.
- If tp + T is less than or equal to tc, an RTCP packet is transmitted.  tp is set to tc, then another value for T is calculated as in the previous step and tn is set to tc + T.  The transmission timer is set to expire again at time tn.  If tp + T is greater than tc, tn is set to tp + T.  No RTCP packet is transmitted.  The transmission timer is set to expire at time tn.
- pmembers is set to members.

If an RTCP packet is transmitted, the value of initial is set to FALSE.  Furthermore, the value of avg_rtcp_size is updated:

```
      avg_rtcp_size = (1/16) * packet_size + (15/16) * avg_rtcp_size
```

where packet_size is the size of the RTCP packet just transmitted.

#### 6.3.7 Transmitting a BYE Packet

When a participant wishes to leave a session, a BYE packet is transmitted to inform the other participants of the event.  In order to avoid a flood of BYE packets when many participants leave the system, a participant MUST execute the following algorithm if the number of members is more than 50 when the participant chooses to leave.  This algorithm usurps the normal role of the members variable to count BYE packets instead:

- When the participant decides to leave the system, tp is reset to tc, the current time, members and pmembers are initialized to 1, initial is set to 1, we_sent is set to false, senders is set to 0, and avg_rtcp_size is set to the size of the compound BYE packet. The calculated interval T is computed.  The BYE packet is then scheduled for time tn = tc + T.
- Every time a BYE packet from another participant is received, members is incremented by 1 regardless of whether that participant exists in the member table or not, and when SSRC sampling is in use, regardless of whether or not the BYE SSRC would be included in the sample.  members is NOT incremented when other RTCP packets or RTP packets are received, but only for BYE packets.  Similarly, avg_rtcp_size is updated only for received BYE packets.  senders is NOT updated when RTP packets arrive; it remains 0.
- Transmission of the BYE packet then follows the rules for transmitting a regular RTCP packet, as above.

This allows BYE packets to be sent right away, yet controls their total bandwidth usage.  In the worst case, this could cause RTCP control packets to use twice the bandwidth as normal (10%) -- 5% for non-BYE RTCP packets and 5% for BYE.

A participant that does not want to wait for the above mechanism to allow transmission of a BYE packet MAY leave the group without sending a BYE at all.  That participant will eventually be timed out by the other group members.

If the group size estimate members is less than 50 when the participant decides to leave, the participant MAY send a BYE packet immediately.  Alternatively, the participant MAY choose to execute the above BYE backoff algorithm.

In either case, a participant which never sent an RTP or RTCP packet MUST NOT send a BYE packet when they leave the group.

#### 6.3.8 Updating we_sent

The variable we_sent contains true if the participant has sent an RTP packet recently, false otherwise.  This determination is made by using the same mechanisms as for managing the set of other participants listed in the senders table.  If the participant sends an RTP packet when we_sent is false, it adds itself to the sender table and sets we_sent to true.  The reverse reconsideration algorithm described in Section 6.3.4 SHOULD be performed to possibly reduce the delay before sending an SR packet.  Every time another RTP packet is sent, the time of transmission of that packet is maintained in the table.  The normal sender timeout algorithm is then applied to the participant -- if an RTP packet has not been transmitted since time tc - 2T, the participant removes itself from the sender table, decrements the sender count, and sets we_sent to false.

#### 6.3.9 Allocation of Source Description Bandwidth

This specification defines several source description (SDES) items in addition to the mandatory CNAME item, such as NAME (personal name) and EMAIL (email address).  It also provides a means to define new application-specific RTCP packet types.  Applications should exercise caution in allocating control bandwidth to this additional information because it will slow down the rate at which reception reports and CNAME are sent, thus impairing the performance of the protocol.  It is RECOMMENDED that no more than 20% of the RTCP bandwidth allocated to a single participant be used to carry the additional information.  Furthermore, it is not intended that all SDES items will be included in every application.  Those that are included SHOULD be assigned a fraction of the bandwidth according to their utility.  Rather than estimate these fractions dynamically, it is recommended that the percentages be translated statically into report interval counts based on the typical length of an item.

For example, an application may be designed to send only CNAME, NAME and EMAIL and not any others.  NAME might be given much higher priority than EMAIL because the NAME would be displayed continuously in the application's user interface, whereas EMAIL would be displayed only when requested.  At every RTCP interval, an RR packet and an SDES packet with the CNAME item would be sent.  For a small session operating at the minimum interval, that would be every 5 seconds on the average.  Every third interval (15 seconds), one extra item would be included in the SDES packet.  Seven out of eight times this would be the NAME item, and every eighth time (2 minutes) it would be the EMAIL item.

When multiple applications operate in concert using cross-application binding through a common CNAME for each participant, for example in a multimedia conference composed of an RTP session for each medium, the additional SDES information MAY be sent in only one RTP session.  The other sessions would carry only the CNAME item.  In particular, this approach should be applied to the multiple sessions of a layered encoding scheme (see Section 2.4).

### 6.4 Sender and Receiver Reports

RTP receivers provide reception quality feedback using RTCP report packets which may take one of two forms depending upon whether or not the receiver is also a sender.  The only difference between the sender report (SR) and receiver report (RR) forms, besides the packet type code, is that the sender report includes a 20-byte sender information section for use by active senders.  The SR is issued if a site has sent any data packets during the interval since issuing the last report or the previous one, otherwise the RR is issued.

Both the SR and RR forms include zero or more reception report blocks, one for each of the synchronization sources from which this receiver has received RTP data packets since the last report. Reports are not issued for contributing sources listed in the CSRC list.  Each reception report block provides statistics about the data received from the particular source indicated in that block.  Since a maximum of 31 reception report blocks will fit in an SR or RR packet, additional RR packets SHOULD be stacked after the initial SR or RR packet as needed to contain the reception reports for all sources heard during the interval since the last report.  If there are too many sources to fit all the necessary RR packets into one compound RTCP packet without exceeding the MTU of the network path, then only the subset that will fit into one MTU SHOULD be included in each interval.  The subsets SHOULD be selected round-robin across multiple intervals so that all sources are reported.

The next sections define the formats of the two reports, how they may be extended in a profile-specific manner if an application requires additional feedback information, and how the reports may be used. Details of reception reporting by translators and mixers is given in Section 7.

#### 6.4.1 SR: Sender Report RTCP Packet

```
        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
header |V=2|P|    RC   |   PT=SR=200   |             length            |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                         SSRC of sender                        |
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
sender |              NTP timestamp, most significant word             |
info   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |             NTP timestamp, least significant word             |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                         RTP timestamp                         |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                     sender's packet count                     |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                      sender's octet count                     |
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
report |                 SSRC_1 (SSRC of first source)                 |
block  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  1    | fraction lost |       cumulative number of packets lost       |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |           extended highest sequence number received           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                      interarrival jitter                      |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                         last SR (LSR)                         |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                   delay since last SR (DLSR)                  |
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
report |                 SSRC_2 (SSRC of second source)                |
block  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  2    :                               ...                             :
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
       |                  profile-specific extensions                  |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The sender report packet consists of three sections, possibly followed by a fourth profile-specific extension section if defined. The first section, the header, is 8 octets long.  The fields have the following meaning:

- **version (V)**: 2 bits

    Identifies the version of RTP, which is the same in RTCP packets as in RTP data packets.  The version defined by this specification is two (2).
- **padding (P)**: 1 bit

    If the padding bit is set, this individual RTCP packet contains some additional padding octets at the end which are not part of the control information but are included in the length field.  The last octet of the padding is a count of how many padding octets should be ignored, including itself (it will be a multiple of four).  Padding may be needed by some encryption algorithms with fixed block sizes.  In a compound RTCP packet, padding is only required on one individual packet because the compound packet is encrypted as a whole for the method in Section 9.1.  Thus, padding MUST only be added to the last individual packet, and if padding is added to that packet, the padding bit MUST be set only on that packet.  This convention aids the header validity checks described in Appendix A.2 and allows detection of packets from some early implementations that incorrectly set the padding bit on the first individual packet and add padding to the last individual packet.
- **reception report count (RC)**: 5 bits

    The number of reception report blocks contained in this packet.  A value of zero is valid.
- **packet type (PT)**: 8 bits

    Contains the constant 200 to identify this as an RTCP SR packet.
- **length**: 16 bits

    The length of this RTCP packet in 32-bit words minus one, including the header and any padding.  (The offset of one makes zero a valid length and avoids a possible infinite loop in scanning a compound RTCP packet, while counting 32-bit words avoids a validity check for a multiple of 4.)
- **SSRC**: 32 bits
    The synchronization source identifier for the originator of this SR packet.

The second section, the sender information, is 20 octets long and is present in every sender report packet.  It summarizes the data transmissions from this sender.  The fields have the following meaning:

- **NTP timestamp**: 64 bits

    Indicates the wallclock time (see Section 4) when this report was sent so that it may be used in combination with timestamps returned in reception reports from other receivers to measure round-trip propagation to those receivers.  Receivers should expect that the measurement accuracy of the timestamp may be limited to far less than the resolution of the NTP timestamp.  The measurement uncertainty of the timestamp is not indicated as it may not be known.  On a system that has no notion of wallclock time but does have some system-specific clock such as "system uptime", a sender MAY use that clock as a reference to calculate relative NTP timestamps.  It is important to choose a commonly used clock so that if separate implementations are used to produce the individual streams of a multimedia session, all implementations will use the same clock.  Until the year 2036, relative and absolute timestamps will differ in the high bit so (invalid) comparisons will show a large difference; by then one hopes relative timestamps will no longer be needed.  A sender that has no notion of wallclock or elapsed time MAY set the NTP timestamp to zero.
- **RTP timestamp**: 32 bits

    Corresponds to the same time as the NTP timestamp (above), but in the same units and with the same random offset as the RTP timestamps in data packets.  This correspondence may be used for intra- and inter-media synchronization for sources whose NTP timestamps are synchronized, and may be used by media-independent receivers to estimate the nominal RTP clock frequency.  Note that in most cases this timestamp will not be equal to the RTP timestamp in any adjacent data packet.  Rather, it MUST be calculated from the corresponding NTP timestamp using the relationship between the RTP timestamp counter and real time as maintained by periodically checking the wallclock time at a sampling instant.
- **sender's packet count**: 32 bits

    The total number of RTP data packets transmitted by the sender since starting transmission up until the time this SR packet was generated.  The count SHOULD be reset if the sender changes its SSRC identifier.
- **sender's octet count**: 32 bits

    The total number of payload octets (i.e., not including header or padding) transmitted in RTP data packets by the sender since starting transmission up until the time this SR packet was generated.  The count SHOULD be reset if the sender changes its SSRC identifier.  This field can be used to estimate the average payload data rate.

    The third section contains zero or more reception report blocks depending on the number of other sources heard by this sender since the last report.  Each reception report block conveys statistics on the reception of RTP packets from a single synchronization source. Receivers SHOULD NOT carry over statistics when a source changes its SSRC identifier due to a collision.  These statistics are:
- **SSRC_n (source identifier)**: 32 bits

      The SSRC identifier of the source to which the information in this reception report block pertains.
- **fraction lost**: 8 bits

    The fraction of RTP data packets from source SSRC_n lost since the previous SR or RR packet was sent, expressed as a fixed point number with the binary point at the left edge of the field.  (That is equivalent to taking the integer part after multiplying the loss fraction by 256.)  This fraction is defined to be the number of packets lost divided by the number of packets expected, as defined in the next paragraph.  An implementation is shown in Appendix A.3.  If the loss is negative due to duplicates, the fraction lost is set to zero.  Note that a receiver cannot tell whether any packets were lost after the last one received, and that there will be no reception report block issued for a source if all packets from that source sent during the last reporting interval have been lost.
- **cumulative number of packets lost**: 24 bits

    The total number of RTP data packets from source SSRC_n that have been lost since the beginning of reception.  This number is defined to be the number of packets expected less the number of packets actually received, where the number of packets received includes any which are late or duplicates.  Thus, packets that arrive late are not counted as lost, and the loss may be negative if there are duplicates.  The number of packets expected is defined to be the extended last sequence number received, as defined next, less the initial sequence number received.  This may be calculated as shown in Appendix A.3.
- **extended highest sequence number received**: 32 bits

    The low 16 bits contain the highest sequence number received in an RTP data packet from source SSRC_n, and the most significant 16 bits extend that sequence number with the corresponding count of sequence number cycles, which may be maintained according to the algorithm in Appendix A.1.  Note that different receivers within the same session will generate different extensions to the sequence number if their start times differ significantly.
- **interarrival jitter**: 32 bits

    An estimate of the statistical variance of the RTP data packet interarrival time, measured in timestamp units and expressed as an unsigned integer.  The interarrival jitter J is defined to be the mean deviation (smoothed absolute value) of the difference D in packet spacing at the receiver compared to the sender for a pair of packets.  As shown in the equation below, this is equivalent to the difference in the "relative transit time" for the two packets; the relative transit time is the difference between a packet's RTP timestamp and the receiver's clock at the time of arrival, measured in the same units.

    If Si is the RTP timestamp from packet i, and Ri is the time of arrival in RTP timestamp units for packet i, then for two packets i and j, D may be expressed as

    ```
         D(i,j) = (Rj - Ri) - (Sj - Si) = (Rj - Sj) - (Ri - Si)
    ```

    The interarrival jitter SHOULD be calculated continuously as each data packet i is received from source SSRC_n, using this difference D for that packet and the previous packet i-1 in order of arrival (not necessarily in sequence), according to the formula
    ```
         J(i) = J(i-1) + (|D(i-1,i)| - J(i-1))/16
    ```

    Whenever a reception report is issued, the current value of J is sampled.

    The jitter calculation MUST conform to the formula specified here in order to allow profile-independent monitors to make valid interpretations of reports coming from different implementations. This algorithm is the optimal first-order estimator and the gain parameter 1/16 gives a good noise reduction ratio while maintaining a reasonable rate of convergence [22].  A sample implementation is shown in Appendix A.8.  See Section 6.4.4 for a discussion of the effects of varying packet duration and delay before transmission.
- **last SR timestamp (LSR)**: 32 bits

    The middle 32 bits out of 64 in the NTP timestamp (as explained in Section 4) received as part of the most recent RTCP sender report (SR) packet from source SSRC_n.  If no SR has been received yet, the field is set to zero.
- **delay since last SR (DLSR)**: 32 bits

    The delay, expressed in units of 1/65536 seconds, between receiving the last SR packet from source SSRC_n and sending this reception report block.  If no SR packet has been received yet from SSRC_n, the DLSR field is set to zero.

    Let SSRC_r denote the receiver issuing this receiver report. Source SSRC_n can compute the round-trip propagation delay to SSRC_r by recording the time A when this reception report block is received.  It calculates the total round-trip time A-LSR using the last SR timestamp (LSR) field, and then subtracting this field to leave the round-trip propagation delay as (A - LSR - DLSR).  This is illustrated in Fig. 2.  Times are shown in both a hexadecimal representation of the 32-bit fields and the equivalent floating- point decimal representation.  Colons indicate a 32-bit field divided into a 16-bit integer part and 16-bit fraction part.

    This may be used as an approximate measure of distance to cluster receivers, although some links have very asymmetric delays.

    ```
    [10 Nov 1995 11:33:25.125 UTC]       [10 Nov 1995 11:33:36.5 UTC]
    n                 SR(n)              A=b710:8000 (46864.500 s)
    ---------------------------------------------------------------->
                        v                 ^
    ntp_sec =0xb44db705 v               ^ dlsr=0x0005:4000 (    5.250s)
    ntp_frac=0x20000000  v             ^  lsr =0xb705:2000 (46853.125s)
        (3024992005.125 s)  v           ^
    r                      v         ^ RR(n)
    ---------------------------------------------------------------->
                            |<-DLSR->|
                            (5.250 s)

    A     0xb710:8000 (46864.500 s)
    DLSR -0x0005:4000 (    5.250 s)
    LSR  -0xb705:2000 (46853.125 s)
    -------------------------------
    delay 0x0006:2000 (    6.125 s)

            Figure 2: Example for round-trip time computation
    ```

#### 6.4.2 RR: Receiver Report RTCP Packet

```
        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
header |V=2|P|    RC   |   PT=RR=201   |             length            |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                     SSRC of packet sender                     |
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
report |                 SSRC_1 (SSRC of first source)                 |
block  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  1    | fraction lost |       cumulative number of packets lost       |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |           extended highest sequence number received           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                      interarrival jitter                      |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                         last SR (LSR)                         |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                   delay since last SR (DLSR)                  |
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
report |                 SSRC_2 (SSRC of second source)                |
block  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  2    :                               ...                             :
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
       |                  profile-specific extensions                  |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The format of the receiver report (RR) packet is the same as that of the SR packet except that the packet type field contains the constant 201 and the five words of sender information are omitted (these are the NTP and RTP timestamps and sender's packet and octet counts). The remaining fields have the same meaning as for the SR packet.

An empty RR packet (RC = 0) MUST be put at the head of a compound RTCP packet when there is no data transmission or reception to report.

#### 6.4.3 Extending the Sender and Receiver Reports

A profile SHOULD define profile-specific extensions to the sender report and receiver report if there is additional information that needs to be reported regularly about the sender or receivers.  This method SHOULD be used in preference to defining another RTCP packet type because it requires less overhead:

- fewer octets in the packet (no RTCP header or SSRC field);
