# RTP Payload Format for Reed Solomon FEC

> Origin [https://datatracker.ietf.org/doc/html/draft-galanos-fecframe-rtp-reedsolomon-03](https://datatracker.ietf.org/doc/html/draft-galanos-fecframe-rtp-reedsolomon-03)

## Abstract

This document defines an RTP payload format for the Reed-Solomon Forward Error Correction (FEC) codes for the erasure channel. The format defined by this document enables the protection of source media encapsulated in RTP with one or more repair flows and is based on the FEC framework and the SDP Elements for FEC Framework. The Reed-Solomon codes used in this document belong to the class of Maximum Distance Separable (MDS) codes which means they offer optimal protection, and they are effective in front of random or bursty packet losses.

## Status of this Memo

This Internet-Draft is submitted in full conformance with the provisions of BCP 78 and BCP 79.

Internet-Drafts are working documents of the Internet Engineering Task Force (IETF). Note that other groups may also distribute working documents as Internet-Drafts. The list of current Internet-Drafts is at http://datatracker.ietf.org/drafts/current/.

Internet-Drafts are draft documents valid for a maximum of six months and may be updated, replaced, or obsoleted by other documents at any time. It is inappropriate to use Internet-Drafts as reference material or to cite them other than as "work in progress."

This Internet-Draft will expire on September 15, 2011.

## Copyright Notice

Copyright (c) 2011 IETF Trust and the persons identified as the document authors. All rights reserved.

This document is subject to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF Documents (http://trustee.ietf.org/license-info) in effect on the date of publication of this document. Please review these documents carefully, as they describe your rights and restrictions with respect to this document. Code Components extracted from this document must include Simplified BSD License text as described in Section 4.e of the Trust Legal Provisions and are provided without warranty as described in the Simplified BSD License.

## Table of Contents

```
   1.  Introduction . . . . . . . . . . . . . . . . . . . . . . . . .  4
   2.  Requirements Notation  . . . . . . . . . . . . . . . . . . . .  5
   3.  Definitions, Notations and Abbreviations . . . . . . . . . . .  5
     3.1.  Definitions  . . . . . . . . . . . . . . . . . . . . . . .  5
     3.2.  Notations  . . . . . . . . . . . . . . . . . . . . . . . .  6
     3.3.  Abbreviations  . . . . . . . . . . . . . . . . . . . . . .  7
   4.  Reed Solomon Codes . . . . . . . . . . . . . . . . . . . . . .  7
   5.  Source Block Creation  . . . . . . . . . . . . . . . . . . . .  8
   6.  Packet Formats . . . . . . . . . . . . . . . . . . . . . . . .  9
     6.1.  FEC Source Packets . . . . . . . . . . . . . . . . . . . . 10
     6.2.  FEC Repair Packets . . . . . . . . . . . . . . . . . . . . 10
       6.2.1.  RTP header format  . . . . . . . . . . . . . . . . . . 10
       6.2.2.  Repair FEC Payload ID encoding format  . . . . . . . . 11
       6.2.3.  Repair Symbol Format . . . . . . . . . . . . . . . . . 12
   7.  Payload Format Parameters  . . . . . . . . . . . . . . . . . . 12
     7.1.  Media Type Registration  . . . . . . . . . . . . . . . . . 13
       7.1.1.  Registration of audio/reed-solomon-fec . . . . . . . . 13
       7.1.2.  Registration of video/reed-solomon-fec . . . . . . . . 14
       7.1.3.  Registration of text/reed-solomon-fec  . . . . . . . . 15
       7.1.4.  Registration of application/reed-solomon-fec . . . . . 17
     7.2.  Mapping of SDP Parameters  . . . . . . . . . . . . . . . . 18
   8.  Protection and Recovery Procedures . . . . . . . . . . . . . . 18
     8.1.  Overview . . . . . . . . . . . . . . . . . . . . . . . . . 19
     8.2.  FEC Repair Packet Construction . . . . . . . . . . . . . . 19
     8.3.  Source Packet Reconstruction . . . . . . . . . . . . . . . 19
       8.3.1.  Associating the Source and Repair Packets  . . . . . . 19
       8.3.2.  Recovering the source packet . . . . . . . . . . . . . 20
   9.  SDP Examples . . . . . . . . . . . . . . . . . . . . . . . . . 20
   10. Implementation Considerations  . . . . . . . . . . . . . . . . 21
   11. Offer/Answer considerations  . . . . . . . . . . . . . . . . . 22
   12. Security Considerations  . . . . . . . . . . . . . . . . . . . 22
     12.1. Problem Statement  . . . . . . . . . . . . . . . . . . . . 22
     12.2. Attacks Against the Data Flow  . . . . . . . . . . . . . . 22
       12.2.1. Access to Confidential Contents  . . . . . . . . . . . 22
       12.2.2. Content Corruption . . . . . . . . . . . . . . . . . . 23
     12.3. Attacks Against the FEC Parameters . . . . . . . . . . . . 24
   13. IANA Considerations  . . . . . . . . . . . . . . . . . . . . . 24
   14. Acknowledgments  . . . . . . . . . . . . . . . . . . . . . . . 24
   15. References . . . . . . . . . . . . . . . . . . . . . . . . . . 25
     15.1. Normative References . . . . . . . . . . . . . . . . . . . 25
     15.2. Informative References . . . . . . . . . . . . . . . . . . 25
   Authors' Addresses . . . . . . . . . . . . . . . . . . . . . . . . 26
```

1. Introduction

This document defines new RTP payload formats for the Forward Error Correction (FEC) that is generated by the Reed-Solomon code.

By nature, interactive Real-time applications are extremely sensitive to delay and require very low latency. As a result, retransmission of lost packets and using other closed-loop schemes are not valid options while the use of Forward Error Correction (FEC) is an effective approach.

A primary requirement from FEC for real time applications is the ability to correctly recover from both random and bursty packet losses. The Reed-Solomon FEC codes used in this document belong to the class of Maximum Distance Separable (MDS) codes that are optimal in terms of erasure recovery capability for both situations.

The format defined by this document enables the protection of media source flow with one or more repair flows without adding additional information to the source packets. Such behavior reduces the delay presented by any FEC scheme and maintains backwards compatibility with non FEC-enabled receivers.

Number of previous drafts were composed to draw different FEC schemes suitable for different applications. The scheme defined in this draft is designed to compensate a burst of packet loss over RTP networks with minimum delay, which is needed in interactive IP-based applications such as video conferencing.

The method described in this document is generic to all media types and provides the sender with the flexibility of deciding if FEC protection is required and if so, how many protecting packets and how many source packets to use in a block according to network conditions. Furthermore it allows applying unequal error protection that provides different level of protection to different packets. For example, it can be combined with Scalable Video Coding to protect only the base layer packets of the video flow. At the receiver, both the FEC and original media are received. If no media packets are lost, the FEC packets can be ignored. In the event of a loss, the FEC packets can be combined with other received media to recover all or part of the missing media packets.

The Reed-Solomon codes used in this document have been specified in [RFC5510] and are compatible with Luigi Rizzo codec (see [Rizzo97]). This document is compliant with the Forward Error Correction (FEC) Framework (described in [I-D.ietf-fecframe-framework]) and SDP Elements for FEC Framework (described in [I-D.ietf-fecframe-sdp-elements] [RFC4566]). This draft completes [I-D.roca-fecframe-rs] by defining Reed-Solomon usage for RTP transport ([RFC3550]) and specifies the appropriate media types (see [RFC4288] [RFC4855] [RFC4856]).

## 2. Requirements Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119].


## 3. Definitions, Notations and Abbreviations

This document uses the following definitions and notations. For further definitions that apply to FEC Framework in general, see [I-D.ietf-fecframe-framework].

### 3.1. Definitions

This document uses the following terms and definitions. Some of them are FEC scheme specific and are in line with [RFC5052]:

- **Source symbol**: unit of data used during the encoding process.
- **Encoding symbol**: unit of data generated by the encoding process. With systematic codes, source symbols are part of the encoding symbols.
- **Repair symbol**: encoding symbol that is not a source symbol.
- **Code rate**: the k/n ratio, i.e., the ratio between the number of source symbols and the number of encoding symbols. By definition, the code rate is such that: 0 < code rate <= 1. A code rate close to 1 indicates that a small number of repair symbols have been produced during the encoding process.
- **Systematic code**: FEC code in which the source symbols are part of the encoding symbols. The Reed-Solomon codes introduced in this document are systematic.
- **Source block**: a block of k source symbols that are considered together for the encoding.
- **Packet Erasure Channel**: a communication path where packets are either dropped (e.g., by a congested router, or because the number of transmission errors exceeds the correction capabilities of the physical layer codes) or received. When a packet is received, it is assumed that this packet is not corrupted.

Some of them are FECFRAME framework specific and are in line with [I-D.ietf-fecframe-framework]:

- **Application Data Unit (ADU)**: a unit of data coming from (sender) or given to (receiver) the media delivery application. In this document, an ADU MUST use an RTP encapsulation.
- **(Source) ADU Flow**: a flow of ADUs from a media delivery application and to which FEC protection is applied. In this document, there MUST be a single ADU flow per FECFRAME framework instance.
- **ADU Block**: a set of ADUs that are considered together by the FECFRAME instance for the purpose of the FEC scheme.
- **FEC Framework Configuration Information**: the FEC scheme specific information that enables the synchronization of the FECFRAME sender and receiver instances.
- **FEC Source Packet**: an RTP data packet submitted to (sender) or received from (receiver) the transport protocol. In this document, FEC Source Packets and ADU MUST be the same (e.g., for backward compatibility purposes).
- **FEC Repair Packet**: an RTP repair packet submitted to (sender) or received from (receiver) the transport protocol. It contains a repair symbol along with its Explicit Repair FEC Payload ID.

3.2. Notations

This document uses the following notations: Some of them are FEC scheme specific:

- `k`      denotes the number of source symbols in a source block.
- `max_k`  denotes the maximum number of source symbols for any source block.
- `n_r`    denotes the number of repair symbols generated for a source block.
- `n`      denotes the number of encoding symbols generated for a source block. Therefore: `n = k + n_r`.
- `max_n`  denotes the maximum number of encoding symbols generated for any source block.
- `E`      denotes the encoding symbol length in bytes.
- `S`      denotes the encoding symbol length in units of m-bit elements. When m = 8, then S and E are equal.
- `GF(q)`  denotes a finite field (also known as Galois Field) with q elements. We assume that q = 2^^m in this document.
- `m`      defines the length of the elements in the finite field, in bits. In this document, m belongs to {2..16}.
- `q`      defines the number of elements in the finite field. We have: q = 2^^m in this specification.
- `CR`     denotes the "code rate", i.e., the k/n ratio.
- `a^^b`   denotes a raised to the power b.

3.3.  Abbreviations

This document uses the following abbreviations:

- `ADU`    stands for Application Data Unit.
- `ESI`    stands for Encoding Symbol ID.
- `FEC`    stands for Forward Error Correction code.
- `FFCI`   stands for FEC Framework Configuration Information.
- `RS`     stands for Reed-Solomon.
- `MDS`    stands for Maximum Distance Separable code.

## 4. Reed Solomon Codes

Reed Solomon codes take a group of k source symbols and generates n - k repair symbols. A receiver needs to receive any k of the n source or repair symbols in order to recover the remaining n-k symbols. As explained in [RFC5510], the Reed-Solomon algorithm operates over m-bit elements, where m is the finite field exponent, GF(2^m), taken from the various encoding symbols. Symbols are composed of S "m-bit elements" each. In the usual case of GF(2^8), elements are bytes and the symbol size S is equal to the symbol size in bytes.

The detailed operation and theory behind Reed Solomon codes is out of the scope of this document. For more information on Reed Solomon codes, the reader is referred to [Rizzo97] and [RFC5510].


## 5. Source Block Creation

This draft specifies how to protect an RTP source flow using one or more FEC repair flows. In order to authorize backward compatibility, the source flow is not modified at all by the FEC Scheme (i.e., FEC source packets are identical to the RTP packets received from the application).

A source block for the Reed-Solomon code contains k source symbols and encoding generates n_r = n - k repair symbols. In this scheme, each Application Data Unit (ADU) is composed of an RTP packet received from the application and corresponds to a single source symbol. Therefore a source block (also called ADU block) contains exactly k RTP packets. In this scheme, each FEC repair packet contains a single repair symbol generated during Reed-Solomon encoding. Therefore, for this block, the n_r FEC repair packets are transmitted along with the k unmodified FEC source packets. Note that the k and n_r values may vary from one block to another since they depend on the source ADU flow and source block creation algorithm (see below).

The following steps MUST be followed in order to create a source block:

1. Determine the largest ADU size in bytes of the source block (i.e., the largest RTP packet size, considering both the RTP header and the payload). The encoding symbol length in bytes for this block, E, is given by: E = 2 + largest ADU size.

2. For each ADU of this source block, of index i, create a byte array of size E as follows:

    - A. The first two bytes, L[i] (Length), contain the length of this ADU, as an unsigned 16-bit integer stored in network byte order (i.e., big endian). This length is for the ADU itself and does not include the L[i] or Pad[i] fields.
    - B. Append the entire RTP packet (including its RTP header).
    - C. Then zero padding is added after ADU i (if needed) in field Pad[i] for alignment purposes, up to a size of exactly E bytes.  Of course, the largest packet of this block does not contain any padding.
3. Append all the byte arrays one after the other so that the RTP packets are in increasing RTP sequence number, taking wraparound of the RTP sequence number into account.

Figure Figure 1 illustrates how a source block is created from 4 RTP packets, ADU 0 to ADU 3, with different sizes, and FEC repair packets are created.

```
                        Encoding Symbol Length (E)
   < --------------------------------------------------------- >
   +----+-----------------------+------------------------------+
   |L[0]|        ADU 0          |            Pad[0]            |
   +----+----------+------------+------------------------------+
   |L[1]| ADU 1    |                         Pad[1]            |
   +----+----------+-------------------------------------------+
   |L[2]|                    ADU 2                             |
   +----+------+-----------------------------------------------+
   |L[3]|ADU 3 |                             Pad[3]            |
   +----+------+-----------------------------------------------+
   \__________________________  _______________________________/
                              \/
                     simple FEC encoding

   +-----------------------------------------------------------+
   |                       Repair 4                            |
   +-----------------------------------------------------------+
   .                                                           .
   .                                                           .
   +-----------------------------------------------------------+
   |                       Repair 7                            |
   +-----------------------------------------------------------+

    Figure 1: Source block creation, for code rate 1/2 (equal number of
              source and repair symbols, 4 in this example).
```

Note that neither the initial 2 bytes nor the optional padding are sent over the network (remember that the source packets are sent unmodified, without any extra padding). However, they are considered during FEC encoding. It means that a receiver who lost a certain FEC source packet (e.g., the UDP datagram containing this FEC source packet) will be able to recover the ADUI if FEC decoding succeeds. Thanks to the initial 2 bytes, this receiver will get rid of the padding (if any).


## 6. Packet Formats

This section defines the formats of the source and repair packets.

### 6.1. FEC Source Packets

The FEC Framework requires that FEC source packets contain information identifying the source block and the position within the source block occupied by the packet. However, in order to maintain backwards compatibility, the scheme defined by this document enables the receiver to get this information without appending additional information to the source packet. Specifically this information is obtained using the combination of sequence number found in the RTP header and information provided in the repair FEC Payload ID of each FEC repair packet. This solution enables both non-FEC-capable and FEC-capable receivers to receive and interpret the same source packets sent in a multicast session.

### 6.2. FEC Repair Packets

The FEC repair packets contain information that enables the receiver to reconstruct the source block. This is done by using the RTP header of the FEC repair packets as well as the repair FEC Payload ID dedicated header (see [I-D.ietf-fecframe-framework], section 6.4.1) that is placed within the RTP payload. There MUST be a single repair symbol per FEC repair packet. Figure Figure 2 shows such a FEC repair packet.

```
                +------------------------------+
                |          IP Header           |
                +------------------------------+
                |       Transport Header       |
                +------------------------------+
                |          RTP Header          |
                +------------------------------+  \
                |     Repair FEC Payload ID    |   |
                +------------------------------+    > RTP Payload
                |        Repair Symbol         |   |
                +------------------------------+  /

                  Figure 2: Format of FEC repair packets
```

#### 6.2.1. RTP header format

The RTP header is formatted according to [RFC3550] with some further clarifications listed below:
- Marker (M) Bit: This bit is not used for this payload type, and is set to 0.
- Payload Type: The (dynamic) payload type for the repair packets is determined through out-of-band means. Note that this document registers new payload formats for the repair packets (Refer to Section 5 for details). According to [RFC3550], an RTP receiver that cannot recognize a payload type must discard it. This provides for backward compatibility. The FEC mechanisms can then be used in a multicast group with mixed FEC-capable and non-FEC- capable receivers. If a non-FEC-capable receiver receives a repair packet, it will not recognize the payload type, and hence, will discard the repair packet. In case more than one repair flow is used, different Payload Types will be used to distinguish between the different flows.
- Sequence Number (SN): The sequence number maintains the standard definition. It is one higher than the sequence number in the previously transmitted repair packet. The initial value of the sequence number is random (unpredictable) [RFC3550].
- Timestamp (TS): The timestamp is set to a time corresponding to the repair packet's transmission time. Note that the timestamp value has no use in the actual FEC protection process and is usually useful for jitter calculations. FEC packets that are the result of the same FEC encoding operation will use the same value as their Timestamp.
- Synchronization Source (SSRC): The SSRC value is randomly assigned as suggested by [RFC3550].

#### 6.2.2. Repair FEC Payload ID encoding format

A FEC repair packet MUST contain a Repair FEC Payload ID that is prepended to the repair symbol(s) as illustrated in Figure 2.  More precisely the Repair FEC Payload ID encoding format includes information that enables the receiver to reconstruct the source block and to identify the FEC repair packets associated with each source block, in their correct order.

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |      Enc. Symbol ID (ESI)     |            SN_base            |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |    Source Block Length (k)    |              n_r              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

                        Figure 3: FEC Header Format
```

The format of the Repair FEC Payload ID is shown in figure Figure 3. It includes the following fields:

- **Encoding Symbol ID, ESI (8-bit field)**: this field identifies the repair symbol contained in this FEC repair packet. This value is such that k <= ESI <= n - 1 for repair symbols.
- **SN_base (16-bit field)**: the lowest RTP sequence number (taking wraparound into account) of the FEC source packets in the associated source block. This SN_base also identifies the source block. In order to avoid any risk of confusion, two consecutive source blocks MUST use different SN_base values, which is easily verified by the sender (this situation might happen with Reed- Solomon over GF(2^16)).
- **Source block length, k (16-bit field)**: the number of consecutive FEC Source Packets considered.
- **n_r (8-bit field)**: the number of FEC repair packets used to protect this source block.

Though the n_r field is not absolutely required (the maximum number of repair symbols can be deduced from the finite field parameter, m), it MAY be useful for the receiver to determine if all the repair packets have been sent (e.g., if n_r FEC repair packets have already been received for this source block). However, since the transmission of FEC repair packets is not necessarily reliable nor guaranteed to preserved packet transmission order, so great care must be taken in figuring out if more FEC repair packets are expected or not. The n_r field MAY also be useful to receivers that need to anticipate the memory block that is required to store all the FEC repair packets.

#### 6.2.3. Repair Symbol Format

The repair symbol follows directly the repair FEC Payload ID in the FEC repair packet (see figure Figure 2). Note that the first two bytes of a repair symbol contain the result of the Reed-Solomon encoding over the packet sizes in the source block. Therefore, the size of an FEC repair packet (FEC Payload ID and FEC repair symbol) is larger than the longest source packet (2 bytes longer). This should be taken under consideration when deciding on the Maximum Transmission Unit (MTU) size used for the source packets.

## 7. Payload Format Parameters

According to the FEC framework, when RTP is used as a transport for repair packet flows, the scheme must define an RTP Payload Format for the repair symbol. This section provides the media subtype registration for the Reed-Solomon FEC. The parameters that are required to configure the FEC encoding and decoding operations are also defined in this section.

### 7.1. Media Type Registration

This registration is done using the template defined in [RFC4288] and following the guidance provided in [RFC4855][RFC4856].

#### 7.1.1. Registration of audio/reed-solomon-fec

Type name: audio

Subtype name: reed-solomon-fec

Required parameters:

- max_n: The upper limit for the sum of source and repair packets that belong to the same FEC block. max_n is a positive integer. The application can change both k and n-k. max_n is the upper limit for n. The value of max_n must be equal to or lower than the codec limitation (2^m).
- repair-window: The time that spans the source packets and the corresponding repair packets. The size of the repair window is specified in microseconds.
- The repair-window impacts the maximum number of source packets in a FEC block at the sender side, and defines the time which the receiver should wait for the repair packets. The repair-window value may be negotiated between the sender and receiver. The details of such negotiation are out-of-scope for this document.
- element-size: a non-negative integer indicating the length of each encoding elements in bits. This value equals to the "m" parameter in the GF (represented by 2^m).

Optional parameters: None.

Encoding considerations: This media type is framed and binary, see section 4.8 in [RFC4288]

Security considerations: Please see security consideration in [I-D.ietf-fecframe-framework]

Interoperability considerations: None.

Published specification: TBD

Applications that use this media type: Multimedia applications that want to improve resiliency against packet loss by sending redundant data in addition to the source media.

Additional information: None.

Magic number(s): none defined

File extension(s): none defined

Macintosh file type code(s): none defined

Person & email address to contact for further information: Sarit Galanos, sarit@radvision.com

Intended usage: COMMON

Restrictions on usage: This media type depends on RTP framing, and hence is only defined for transfer via RTP [RFC3550]. Transport within other framing protocols is not defined at this time.

#### 7.1.2. Registration of video/reed-solomon-fec

Type name: video

Subtype name: reed-solomon-fec

Required parameters:
- max_n: The upper limit for the sum of source and repair packets that belong to the same FEC block. max_n is a positive integer. The application can change both k and n-k. max_n is the upper limit for n. The value of max_n must be equal to or lower than the codec limitation (2^m).
- repair-window: The time that spans the source packets and the corresponding repair packets. The size of the repair window is specified in microseconds.
- The repair-window impacts the maximum number of source packets in a FEC block at the sender side and defines the time which the receiver should wait for the repair packets. The repair-window value may be negotiated between the sender and receiver. the details of such negotiation are out-of-scope for this document.
- element-size: a non-negative integer indicating the length of each encoding elements in bits. This value equals to the "m" parameter in the GF (represented by 2^m).

Optional parameters: None.

Encoding considerations: This media type is framed and binary, see section 4.8 in [RFC4288]

Security considerations: Please see security consideration in [I-D.ietf-fecframe-framework]

Interoperability considerations: None.

Published specification: TBD

Applications that use this media type: Multimedia applications that want to improve resiliency against packet loss by sending redundant data in addition to the source media.

Additional information: None.

Magic number(s): none defined

File extension(s): none defined

Macintosh file type code(s): none defined

Person & email address to contact for further information: Sarit Galanos, sarit@radvision.com

Intended usage: COMMON

Restrictions on usage: This media type depends on RTP framing, and hence is only defined for transfer via RTP [RFC3550]. Transport within other framing protocols is not defined at this time.

#### 7.1.3. Registration of text/reed-solomon-fec

Type name: text

Subtype name: reed-solomon-fec

Required parameters:

- max_n: The upper limit for the sum of source and repair packets that belong to the same FEC block. max_n is a positive integer. The application can change both k and n-k. max_n is the upper limit for n. The value of max_n must be equal to or lower than the codec limitation (2^m).
- repair-window: The time that spans the source packets and the corresponding repair packets. The size of the repair window is specified in microseconds.
- The repair-window impacts the maximum number of source packets in a FEC block at the sender side, and defines the time which the receiver should wait for the repair packets. The repair-window value may be negotiated between the sender and receiver. the details of such negotiation are out-of-scope for this document.
- element-size: a non-negative integer indicating the length of each encoding elements in bits. This value equals to the "m" parameter in the GF (represented by 2^m).

Optional parameters: None.

Encoding considerations: This media type is framed and binary, see section 4.8 in [RFC4288]

Security considerations: Please see security consideration in [I-D.ietf-fecframe-framework]

Interoperability considerations: None.

Published specification: TBD

Applications that use this media type: Multimedia applications that want to improve resiliency against packet loss by sending redundant data in addition to the source media.

Additional information: None.

Magic number(s): none defined

File extension(s): none defined

Macintosh file type code(s): none defined

Person & email address to contact for further information: Sarit Galanos, sarit@radvision.com

Intended usage: COMMON

Restrictions on usage: This media type depends on RTP framing, and hence is only defined for transfer via RTP [RFC3550]. Transport within other framing protocols is not defined at this time.


#### 7.1.4. Registration of application/reed-solomon-fec

Type name: application

Subtype name: reed-solomon-fec

Required parameters:

- max_n: The upper limit for the sum of source and repair packets that belong to the same FEC block. max_n is a positive integer. The application can change both k and n-k. max_n is the upper limit for n. The value of max_n must be equal to or lower than the codec limitation (2^m).
- repair-window: The time that spans the source packets and the corresponding repair packets. The size of the repair window is specified in microseconds.
- The repair-window impacts the maximum number of source packets in a FEC block at the sender side, and defines the time which the receiver should wait for the repair packets. The repair-window value may be negotiated between the sender and receiver. the details of such negotiation are out-of-scope for this document.
- element-size: a non-negative integer indicating the length of each encoding elements in bits. This value equals to the "m" parameter in the GF (represented by 2^m).

Optional parameters: None.

Encoding considerations: This media type is framed and binary, see section 4.8 in [RFC4288]

Security considerations: Please see security consideration in [I-D.ietf-fecframe-framework]

Interoperability considerations: None.

Published specification: TBD

Applications that use this media type: Multimedia applications that want to improve resiliency against packet loss by sending redundant data in addition to the source media.

Additional information: None.

Magic number(s): none defined

File extension(s): none defined

Macintosh file type code(s): none defined

Person & email address to contact for further information: Sarit Galanos, sarit@radvision.com

Intended usage: COMMON

Restrictions on usage: This media type depends on RTP framing, and hence is only defined for transfer via RTP [RFC3550]. Transport within other framing protocols is not defined at this time.

### 7.2. Mapping of SDP Parameters

For a proper operation details of the FEC scheme have to be communicated between the sender and the receiver. Specifically, the receiver has to know the relationship between the source and the repair flows, how the sender applied protection to the source flow and how the repair flows can be used to recover the lost data. One way to provide this information is to use the Session Description Protocol (SDP) [RFC4566].

The mapping of the media type specification for "reed-solomon-fec" and their parameters in SDP is as follows:

- The media type (e.g., "application") goes into the "m=" line as the media name.
- The media subtype ("reed-solomon-fec") goes into the "a=rtpmap" line as the encoding name.
- The remaining required payload-format-specific parameters ("max_n", "repair-window") go into the "a=fmtp" line by copying them directly from the media type string as a semicolon-separated list of parameter=value pairs.

See section 9 for SDP examples.

## 8. Protection and Recovery Procedures

This section provides a complete specification of the protection and recovery procedures.

### 8.1. Overview

The FEC repair packets allow end-systems to recover from media packet losses (also called erasures). The following sections specify the steps involved in generating the FEC repair packets and reconstructing the missing source packets from the FEC repair packets.

### 8.2. FEC Repair Packet Construction

The source block is created and encoding is performed based on the guidelines given in Section 5. The RTP header of a FEC repair packet is formed based on the guidelines given in Section 6.2.1. The repair FEC Payload ID is formed based on the guidelines given in Section 6.2.2. The n_r FEC repair packets can then be sent along with the k original unmodified source packets.

### 8.3. Source Packet Reconstruction

Recovery requires two distinct operations. The first operation determines which packets (source and repair) must be considered in order to recover the missing packets of a given block. Once this is done, the second step is the reconstruction of the missing data.

#### 8.3.1. Associating the Source and Repair Packets

Association of the FEC source packets and FEC repair packets is done using a combination of the source packet sequence number and the information found in the RTP header and the repair FEC Payload ID of the FEC repair packets. The first step is to accumulate some of the n_r = n - k repair packets that were generated. For that the application follows the steps listed below:

- For each received packet, retrieve the payload type parameter from the RTP header to identify the packet as a FEC repair packet of the Reed-Solomon scheme or a source packet. In case multiple repair flows are used, different payload types will be used to distinguish between the different repair flows.
- If a FEC repair packet is received, retrieve the sequence number (SN) from the RTP header and the ESI, SN_base, k and n_r parameters from the repair FEC Payload ID.
  * With these parameters, identify the collection of source packets that are included in this source block. For example, if SN_base = 991 and k = 10, the receiver deduces that this source block is composed of the 10 source packets with RTP sequence numbers 991 to 1000 (inclusive).
  * With these parameters, identify the collection of FEC repair packets generated for this source block. For example, if SN_base = 991, ESI = 12, k = 10, n_r = 4, the receiver deduces that 4 FEC repair packets with RTP sequence numbers 1001, 1002, 1003 and 1004 have been generated for this source block.

#### 8.3.2. Recovering the source packet

In order to recover the lost source packets, the application has to rebuild the source block according to the guidelines given in Section 5 and append the repair symbol to it in the correct order. Zero padding will replace the lost packets in the constructed source block. The size of each source block data packet in bytes will be equal to the size of the repair symbol found in the repair packets. The repair symbol size is the size of the RTP payload in the repair packet without the repair FEC Payload ID (see Figure 2). The application will then append the repair symbol taken from each repair packet. This new block is provided to the Reed-Solomon code.

Reconstruction of lost packets (source or repair packets) is possible only if at least any k packets were received (source or repair).

The Reed-Solomon code will reconstruct the lost data into the provided source block overriding the zero padded blocks. The application can then recover the lost packets as follows:

- The first two bytes specify the RTP packet size.
- According to the RTP packet size the application can retrieve the RTP packet (RTP header and payload).
- Any extra padding bytes (if any) are ignored.

## 9. SDP Examples

The following example demonstrates source flow with flow ID of 0 that is protected by a single repair flow R1.

```
   v=0
   o=sarit 1122334455 1122334466 IN IP4 fec.example.com
   s= Reed Solomon FEC Example
   t=0 0
   a=group:FEC S1 R1
   m=video 30000 RTP/AVP 100
   c=IN IP4 233.252.0.1/127
   a=rtpmap:100 MP2T/90000
   a=fec-source-flow: id=0
   a=mid:S1
   m=application 30000 RTP/AVP 110
   c=IN IP4 233.252.0.2/127
   a=rtpmap:110 reed-solomon-fec /90000
   a=fmtp:110 max_n:16; repair-window:200000; symbol-size:8
   a=mid:R1

                                 Figure 4
```

## 10. Implementation Considerations

Using Reed-Solomon FEC protection over RTP may be useful for efficiently overcoming network packet losses in interactive communications where latency constraints apply. Protection may be applied for small encoding blocks, and therefore latency caused by waiting for the FEC repair packets is minimized.

This document allows the application to set the FEC recovery capabilities dynamically according to the experienced and measured loss rate, for optimizing bandwidth utilization while recovering from network errors.

When FEC protection is used due to network congestion conditions, it is important that the application will reduce the bandwidth used for FEC protection from the bandwidth used by the source flow, in order not to overload the already congested network with the additional FEC repair packets.

In order to minimize bandwidth overhead for repair packets, algorithm for applying FEC on source packets should be designed carefully. Using source packets with similar lengths (when possible) can minimize the bandwidth overhead of the FEC repair packets.

In order to maximize the FEC recovery capabilities, when a ratio of k/n is chosen, the larger the encoding blocks size (n) is, the stronger the FEC protection is. Of course, on the other hand the larger the source block size is, the larger the latency is (caused by waiting for the FEC repair packet). The application should choose carefully the FEC block size in order to maximize the FEC recovery capabilities while keeping an acceptable latency at the receiver waiting for the FEC repair packets.

## 11. Offer/Answer considerations

None.


## 12. Security Considerations

### 12.1. Problem Statement

A content delivery system is potentially subject to many attacks. Some of them target the network (e.g., to compromise the routing infrastructure, by compromising the congestion control component), others target the Content Delivery Protocol (CDP) (e.g., to compromise its normal behavior), and finally some attacks target the content itself. Since this document focuses on various FEC schemes, this section only discusses the additional threats that their use within the FECFRAME framework can create to an arbitrary CDP.

More specifically, these attacks may have several goals:
- those that are meant to give access to a confidential content (e.g., in case of a non-free content),
- those that try to corrupt the ADU Flows being transmitted (e.g., to prevent a receiver from using it),
- and those that try to compromise the receiver's behavior (e.g., by making the decoding of an object computationally expensive).

These attacks can be launched either against the data flow itself (e.g., by sending forged FEC Source/Repair Packets) or against the FEC parameters that are sent either in-band (e.g., in the Repair FEC Payload ID) or out-of-band (e.g., in a session description).

### 12.2. Attacks Against the Data Flow

First of all, let us consider the attacks against the data flow.

#### 12.2.1. Access to Confidential Contents

Access control to the ADU Flow being transmitted is typically provided by means of encryption. This encryption can be done within the content provider itself, by the application (for instance by using the Secure Real-time Transport Protocol (SRTP) [RFC3711]), or at the Network Layer, on a packet per packet basis when IPsec/ESP is used [RFC4303]. If confidentiality is a concern, it is RECOMMENDED that one of these solutions be used. Even if we mention these attacks here, they are not related nor facilitated by the use of FEC.

#### 12.2.2. Content Corruption

Protection against corruptions (e.g., after sending forged FEC Source/Repair Packets) is achieved by means of a content integrity verification/sender authentication scheme. This service is usually provided at the packet level. In this case, after removing all forged packets, the ADU Flow may be sometimes recovered. Several techniques can provide this source authentication/content integrity service:

- at the application level, the Secure Real-time Transport Protocol (SRTP) [RFC3711] provides several solutions to authenticate the source and check the integrity of RTP and RTCP messages, among other services. For instance, associated to the Timed Efficient Stream Loss-Tolerant Authentication (TESLA) [RFC4383], SRTP is an attractive solution that is robust to losses, provides a true authentication/integrity service, and does not create any prohibitive processing load or transmission overhead. Yet, checking a packet requires a small delay (a second or more) after its reception with TESLA. Other building blocks can be used within SRTP to provide authentication/content integrity services.
- at the Network Layer, IPsec/ESP offers (among other services) an integrity verification mechanism that can be used to provide authentication/content integrity services.

Techniques relying on public key cryptography (e.g., digital signatures) require that public keys be securely associated to the entities. This can be achieved by a Public Key Infrastructure (PKI), or by a PGP Web of Trust, or by pre-distributing the public keys of each group member.

Techniques relying on symmetric key cryptography require that a secret key be shared by all group members. This can be achieved by means of a group key management protocol, or simply by pre- distributing the secret key (but this manual solution has many limitations).

It is up to the developer and deployer, who know the security requirements and features of the target application area, to define which solution is the most appropriate. Nonetheless it is RECOMMENDED that at least one of these techniques be used.

### 12.3. Attacks Against the FEC Parameters

Let us now consider attacks against the FEC parameters included in the FFCI that are usually sent out-of-band (e.g., in a session description). Attacks on these FEC parameters can prevent the decoding of the associated object. For instance modifying the m field (when applicable) will lead a receiver to consider a different code. Modifying the E parameter will lead a receiver to consider bad Repair Symbols for a received FEC Repair Packet.

It is therefore RECOMMENDED that security measures be taken to guarantee the FFCI integrity. When the FFCI is sent out-of-band in a session description, this latter SHOULD be protected, for instance by digitally signing it.

Attacks are also possible against some FEC parameters included in the Explicit Repair FEC Payload ID. For instance modifying the SN_base of a FEC Repair Packet will lead a receiver to assign this packet to a wrong block.

It is therefore RECOMMENDED that security measures be taken to guarantee the Explicit Repair FEC Payload ID integrity. To that purpose, one of the packet-level source authentication/content integrity techniques of Section 12.2.2 can be used.


## 13. IANA Considerations

New media subtypes are subject to IANA registration. For the registration of the payload formats and their parameters introduced in this document, refer to Section 7.1.


## 14. Acknowledgments

Some parts of this document are borrowed from the following documents: [RFC5109], [I-D.ietf-fecframe-1d2d-parity-scheme], [I-D.roca-fecframe-rs], [I-D.ietf-avt-reedsolomon]. The author would like to thank the editors of these documents. The authors would also like to thank the members of the FP7 3DPresence project consortium for their contribution to this draft writing.


## 15. References

### 15.1. Normative References

- [RFC2119] Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997.
- [RFC3550] Schulzrinne, H., Casner, S., Frederick, R., and V. Jacobson, "RTP: A Transport Protocol for Real-Time Applications", STD 64, RFC 3550, July 2003.
- [RFC4566] Handley, M., Jacobson, V., and C. Perkins, "SDP: Session Description Protocol", RFC 4566, July 2006.
- [RFC4288] Freed, N. and J. Klensin, "Media Type Specifications and Registration Procedures", BCP 13, RFC 4288, December 2005.
- [RFC4855] Casner, S., "Media Type Registration of RTP Payload Formats", RFC 4855, February 2007.
- [RFC4856] Casner, S., "Media Type Registration of Payload Formats in the RTP Profile for Audio and Video Conferences", RFC 4856, February 2007.
- [RFC5510] Lacan, J., Roca, V., Peltotalo, J., and S. Peltotalo, "Reed-Solomon Forward Error Correction (FEC) Schemes", RFC 5510, April 2009.
- [I-D.ietf-fecframe-framework] Watson, M., Begen, A., and V. Roca, "Forward Error Correction (FEC) Framework", draft-ietf-fecframe-framework-14 (work in progress), March 2011.

### 15.2. Informative References

- [I-D.ietf-fecframe-1d2d-parity-scheme] Begen, A., "RTP Payload Format for Non-Interleaved and Interleaved Parity FEC", draft-ietf-fecframe-1d2d-parity-scheme-01 (work in progress), May 2009.
- [RFC5109] Li, A., "RTP Payload Format for Generic Forward Error Correction", RFC 5109, December 2007.
- [I-D.ietf-avt-reedsolomon] Rosenberg, J. and H. Schulzrinne, "An RTP Payload Format for Reed Solomon Codes", May 1999.
- [Rizzo97] Rizzo, L., "Effective Erasure Codes for Reliable Computer Communication Protocols", ACM SIGCOMM Computer Communication Review Vol.27, No.2, pp.24-36, April 1997.
- [RFC4303] Kent, S., "IP Encapsulating Security Payload (ESP)", RFC 4303, December 2005.
- [RFC3711] Baugher, M., McGrew, D., Naslund, M., Carrara, E., and K. Norrman, "The Secure Real-time Transport Protocol (SRTP)", RFC 3711, March 2004.
- [RFC4383] Baugher, M. and E. Carrara, "The Use of Timed Efficient Stream Loss-Tolerant Authentication (TESLA) in the Secure Real- time Transport Protocol (SRTP)", RFC 4383, February 2006.
- [I-D.ietf-fecframe-sdp-elements] Begen, A., "Session Description Protocol Elements for FEC Framework", draft-ietf-fecframe-sdp-elements-11 (work in progress), October 2010.
- [I-D.roca-fecframe-rs] Roca, V., Cunche, M., Lacan, J., Bouabdallah, A., and K. Matsuzono, "Reed-Solomon Forward Error Correction (FEC) Schemes for FECFRAME", draft-roca-fecframe-rs-03 (work in progress), July 2010.
- [RFC5052] Watson, M., Luby, M., and L. Vicisano, "Forward Error Correction (FEC) Building Block", RFC 5052, August 2007.
