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

2.3 Mixers and Translators

So far, we have assumed that all sites want to receive media data in the same format.  However, this may not always be appropriate. Consider the case where participants in one area are connected through a low-speed link to the majority of the conference participants who enjoy high-speed network access.  Instead of forcing everyone to use a lower-bandwidth, reduced-quality audio encoding, an RTP-level relay called a mixer may be placed near the low-bandwidth area.  This mixer resynchronizes incoming audio packets to reconstruct the constant 20 ms spacing generated by the sender, mixes these reconstructed audio streams into a single stream, translates the audio encoding to a lower-bandwidth one and forwards the lower- bandwidth packet stream across the low-speed link.  These packets might be unicast to a single recipient or multicast on a different address to multiple recipients.  The RTP header includes a means for mixers to identify the sources that contributed to a mixed packet so that correct talker indication can be provided at the receivers.

Some of the intended participants in the audio conference may be connected with high bandwidth links but might not be directly reachable via IP multicast.  For example, they might be behind an application-level firewall that will not let any IP packets pass. For these sites, mixing may not be necessary, in which case another type of RTP-level relay called a translator may be used.  Two translators are installed, one on either side of the firewall, with the outside one funneling all multicast packets received through a secure connection to the translator inside the firewall.  The translator inside the firewall sends them again as multicast packets to a multicast group restricted to the site's internal network.

Mixers and translators may be designed for a variety of purposes.  An example is a video mixer that scales the images of individual people in separate video streams and composites them into one video stream to simulate a group scene.  Other examples of translation include the connection of a group of hosts speaking only IP/UDP to a group of hosts that understand only ST-II, or the packet-by-packet encoding translation of video streams from individual sources without resynchronization or mixing.  Details of the operation of mixers and translators are given in Section 7.

2.4 Layered Encodings

Multimedia applications should be able to adjust the transmission rate to match the capacity of the receiver or to adapt to network congestion.  Many implementations place the responsibility of rate-adaptivity at the source.  This does not work well with multicast transmission because of the conflicting bandwidth requirements of heterogeneous receivers.  The result is often a least-common denominator scenario, where the smallest pipe in the network mesh dictates the quality and fidelity of the overall live multimedia "broadcast".

Instead, responsibility for rate-adaptation can be placed at the receivers by combining a layered encoding with a layered transmission system.  In the context of RTP over IP multicast, the source can stripe the progressive layers of a hierarchically represented signal across multiple RTP sessions each carried on its own multicast group. Receivers can then adapt to network heterogeneity and control their reception bandwidth by joining only the appropriate subset of the multicast groups.

Details of the use of RTP with layered encodings are given in Sections 6.3.9, 8.3 and 11.

3. Definitions

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
