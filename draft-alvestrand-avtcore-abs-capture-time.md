---
title: "Absolute Capture Timestamp RTP header extension"
abbrev: "Abs capture timestamp"
category: info

docname: draft-alvestrand-avtcore-abs-capture-timestamp-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Audio/Video Transport Core Maintenance"
keyword:
 - webrtc
 - capture timestamp
 - rtcweb
venue:
  group: "Audio/Video Transport Core Maintenance"
  type: "Working Group"
  mail: "avt@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/avt/"
  github: "alvestrand/id-abs-capture-timestamp"
  latest: "https://alvestrand.github.io/id-abs-capture-timestamp/draft-alvestrand-avtcore-abs-capture-timestamp.html"

author:
 -
    fullname: "Harald Alvestrand"
    organization: Google
    email: "hta@google.com"

normative:

informative:


--- abstract

This document describes an RTP header extension that can be use to carry information
about the capture time of a video frame / audio sample, and include information that
allows the capture time to be estimated by the receiver when the frame may have passed
over multiple hops before reaching the receiver.


--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Absolute Capture Time

The Absolute Capture Time extension is used to stamp RTP packets with a NTP
timestamp showing when the first audio or video frame in a packet was originally
captured. The intent of this extension is to provide a way to accomplish
audio-to-video synchronization when RTCP-terminating intermediate systems (e.g.
mixers) are involved.

**Name:**
"Absolute Capture Time"; "RTP Header Extension for Absolute Capture Time"

**Formal name:**
<http://www.webrtc.org/experiments/rtp-hdrext/abs-capture-time>

**Status:**
This extension is defined here to allow for experimentation. Experience with the experiment has shown that it is useful; this draft therefore presents it to the
IETF for consideration of whether to standardize it or leave it as a proprietary
extension.

## RTP header extension format

### Data layout overview
Data layout of the shortened version of `abs-capture-time` with a 1-byte header
\+ 8 bytes of data:

      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |  ID   | len=7 |     absolute capture timestamp (bit 0-23)     |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |             absolute capture timestamp (bit 24-55)            |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |  ... (56-63)  |
     +-+-+-+-+-+-+-+-+

Data layout of the extended version of `abs-capture-time` with a 1-byte header +
16 bytes of data:

      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |  ID   | len=15|     absolute capture timestamp (bit 0-23)     |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |             absolute capture timestamp (bit 24-55)            |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |  ... (56-63)  |   estimated capture clock offset (bit 0-23)   |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |           estimated capture clock offset (bit 24-55)          |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |  ... (56-63)  |
     +-+-+-+-+-+-+-+-+

### Data layout details
#### Absolute capture timestamp

Absolute capture timestamp is the NTP timestamp of when the first frame in a
packet was originally captured. This timestamp MUST be based on the same clock
as the clock used to generate NTP timestamps for RTCP sender reports on the
capture system.

It's not always possible to do an NTP clock readout at the exact moment of when
a media frame is captured. A capture system MAY postpone the readout until a
more convenient time. A capture system SHOULD have known delays (e.g. from
hardware buffers) subtracted from the readout to make the final timestamp as
close to the actual capture time as possible.

This field is encoded as a 64-bit unsigned fixed-point number with the high 32
bits for the timestamp in seconds and low 32 bits for the fractional part. This
is also known as the UQ32.32 format and is what the RTP specification defines as
the canonical format to represent NTP timestamps.

#### Estimated capture clock offset

Estimated capture clock offset is the sender's estimate of the offset between
its own NTP clock and the capture system's NTP clock. The sender is here defined
as the system that owns the NTP clock used to generate the NTP timestamps for
the RTCP sender reports on this stream. The sender system is typically either
the capture system or a mixer.

This field is encoded as a 64-bit two’s complement **signed** fixed-point number
with the high 32 bits for the seconds and low 32 bits for the fractional part.
It’s intended to make it easy for a receiver, that knows how to estimate the
sender system’s NTP clock, to also estimate the capture system’s NTP clock:

     Capture NTP Clock = Sender NTP Clock + Capture Clock Offset

### Further details

#### Capture system

A receiver MUST treat the first CSRC in the CSRC list of a received packet as if
it belongs to the capture system. If the CSRC list is empty, then the receiver
MUST treat the SSRC as if it belongs to the capture system. Mixers SHOULD put
the most prominent CSRC as the first CSRC in a packet’s CSRC list.

#### Intermediate systems

An intermediate system (e.g. mixer) MAY adjust these timestamps as needed. It
MAY also choose to rewrite the timestamps completely, using its own NTP clock as
reference clock, if it wants to present itself as a capture system for A/V-sync
purposes.

#### Timestamp interpolation

A sender SHOULD save bandwidth by not sending `abs-capture-time` with every
RTP packet. It SHOULD still send them at regular intervals (e.g. every second)
to help mitigate the impact of clock drift and packet loss. Mixers SHOULD always
send `abs-capture-time` with the first RTP packet after changing capture system.

A receiver SHOULD memorize the capture system (i.e. CSRC/SSRC), capture
timestamp, and RTP timestamp of the most recently received `abs-capture-time`
packet on each received stream. It can then use that information, in combination
with RTP timestamps of packets without `abs-capture-time`, to extrapolate
missing capture timestamps.

Timestamp interpolation works fine as long as there’s reasonably low NTP/RTP
clock drift. This is not always true. Senders that detect "jumps" between its
NTP and RTP clock mappings SHOULD send `abs-capture-time` with the first RTP
packet after such a thing happening.

# Security Considerations

This extension carries information that may allow an attacker to identify different
media streams on a connection. However, this information is already carried in the
RTP SSRC, which is not encrypted, so it is unlikely that much additional information
is exposed.

# IANA Considerations

If the WG decides that this extension should be registered as a standardized
extension, IANA is requested to perform the appropriate registration.

If the WG decides that this is a private extension, the URL xxxxx is used to
identify the extension.


--- back

# Acknowledgments
{:numbered="false"}

Chen Xing, for writing the original version of this specification.

