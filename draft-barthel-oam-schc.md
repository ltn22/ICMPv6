---
stand_alone: true
ipr: trust200902
docname: draft-barthel-oam-schc-00
cat: info
pi:
  symrefs: 'yes'
  sortrefs: 'yes'
  strict: 'yes'
  compact: 'yes'
  toc: 'yes'

title: OAM for LPWAN using Static Context Header Compression (SCHC)
abbrev: SCHC OAM for LPWAN
wg: lpwan Working Group
author:
- ins: D. Barthel
  name: Dominique Barthel
  org: Orange SA
  street:
  - 28 chemin du Vieux Chene
  - BP 98
  city: 38243 Meylan Cedex
  country: France
  email: dominique.barthel@orange.com
- ins: L. Toutain
  name: Laurent Toutain
  org: Institut MINES TELECOM; IMT Atlantique
  street:
  - 2 rue de la Chataigneraie
  - CS 17607
  city: 35576 Cesson-Sevigne Cedex
  country: France
  email: laurent.toutain@imt-atlantique.fr
- ins: A. Kandasamy
  name: Arunprabhu Kandasamy
  org: Acklio
  street:
  - 2bis rue de la Chataigneraie
  city: 35510 Cesson-Sevigne Cedex
  country: France
  email: arun@ackl.io  
- ins: D. Dujovne
  name: Diego Dujovne
  org: EIT UDP
  street:
  - xxxx
  city: Santiago
  country: Chile
  email: diego.dujovne@mail.udp.cl
- ins: J.C. Zuniga
  name: Juan Carlos Zuniga
  org: Sigfox
  street:
  - xxx
  city: Labege
  country: France
  email: juancarlos.zuniga@sigfox.com
normative:
  RFC2119:
  RFC4443:
  RFC4861:
  RFC4884:
  RFC6291:
  RFC8174:
  RFC8200:
  I-D.ietf-lpwan-ipv6-static-context-hc:
informative:
  I-D.ietf-lpwan-overview:

--- abstract

OAM {{RFC6291}} bla-bla-bla.

With IP protocols now generalizing to constrained networks, users expect to be able to Operate, Administer and Maintain them with the familiar tools and protocols they already use on less constrained networks.

The primitive functionalities of OAM are achieved with the ICMPv6 protocol.

ICMPv6 defines messages that inform the source of IPv6 packets of errors
during packet delivery.
It also defines the Echo Request/Reply messages that are used for basic network troubleshooting (ping command).
ICMPv6 messages are transported on IPv6.


This document describes
how basic OAM is performed on Low Power Wide Area Networks (LPWANs) by compressing ICMPv6/IPv6 headers
and by protecting the LPWAN network and the Device from undesirable ICMPv6 traffic.


--- middle


# Introduction

ICMPv6 {{RFC4443}} is a companion protocol to IPv6 {{RFC8200}}.

{{RFC4443}} defines a generic message format.
This format is used for messages to be sent back to the source of an IPv6 packet to inform it about errors during packet delivery. 

More specifically, {{RFC4443}} defines 4 error messages: Destination Unreachable, Packet Too Big,
Time Exceeded and Parameter Problem.

{{RFC4443}} also defines the Echo Request and Echo Reply messages, which provide support for the ping application. 

Other ICMPv6 messages are defined in other RFCs, such as an extended format of the same messages {{RFC4884}}
and other messages used by the Neighbor Discovery Protocol {{RFC4861}}.

This document focuses on using Static Context Header Compression (SCHC) to compress {{RFC4443}} messages that need to be transmitted over the LPWAN network, and on having the LPWAN gateway proxying the Device to save it the unwanted traffic.

LPWANs' salient characteristics are described in {{I-D.ietf-lpwan-overview}}

# Terminology

This draft re-uses the Terminology defined in {{I-D.ietf-lpwan-ipv6-static-context-hc}}.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", 
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", 
and "OPTIONAL" in this document are to be interpreted as 
described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they 
appear in all capitals, as shown here. 

# Use cases

In the LPWAN architecture, we can distinguish the following cases:

* the Device is the (purported) source of an ICMP error message, mainly in response to an incorrect incoming IPv6
message, or in response to a ping request. In this case, as much as possible, the core SCHC C/D should act as a proxy and originate the ICMP message, so that the Device and the LPWAN network are protected from this unwanted traffic.

* the Device is the destination of the ICMP message, mainly in response to a
packet sent by the Device to the network that generates an error. In this case, we want the ICMP message to reach the Device, and this document describes in
section {{ErrMsgCompr}} what SCHC compression should be applied.

* the Device is the originator of an Echo Request message, and therefore the destination of the Echo Reply message.

* the Device is the destination of an Echo Request message, and therefore the purported source of an Echo Reply message.

These cases are further described in {{DetailedBehavior}}.

# Detailed behavior {#DetailedBehavior}

## Device is the source of an ICMPv6 error message {#ProxyErrMsg}

As stated in {{RFC4443}}, a node should generate an ICMPv6 message in response to an
IPv6 packet that is malformed or which cannot be processed due to some incorrect field value.


The general intent of this document is to spare both the Device and the LPWAN network this un-necessary traffic.
The incorrect packets should be caught at the core SCHC C/D and the ICMPv6 notification should be sent back from there.


~~~~

     Device       NGW     core SCHC C/D                 Internet Host

       |           |            |    Destination Port=XXX    |
       |           |            |<---------------------------|
       |           |            |                            |
       |           |            |--------------------------->|
       |           |            | ICMPv6 Port Unreachable    |
       |           |            |                            |
       |           |            |                            |


~~~~
{: #Fig-ICMPv6-up title='Example of ICMPv6 error message sent back to the Internet'}

{{Fig-ICMPv6-up}} shows an example of an IPv6 packet trying to reach a Device. Let's assume that the port number used
as destination port is not "known" (needs better definition) from the core SCHC C/D. Instead of sending the packet over
the LPWAN and having this packet rejected by the Device, the core SCHC C/D issues
an ICMPv6 error message "Destination Unreachable" (Type 1) with Code 1 ("Port Unreachable") on behalf of the Device.


TODO: This assumes that all ports that the Device listens to will be matched by a SCHC rule. Is this the basic assumption of SCHC that all packets that do not match a rule are rejected? If yes, why do have fragmentation also for uncompressed packets?

TODO: discuss the various Type/Code that are expected to be generated in response to various errors.  

## Device is the destination of an ICMPv6 error message

In this situation, we assume that a Device has been configured to send information to a server on the Internet. If this
server becomes no longer accessible, an ICMPv6 message will be generated back towards the Device by an intermediate router.
This information can be useful to the Device, for example for reducing the reporting rate in case of periodic reporting of data.
Therefore, we compress the ICMPv6 message using SCHC and forward it to the Device over the LPWAN.

~~~~

     Device       NGW     core SCHC C/D                Internet Server

       |           |            |                            |
       | SCHC compressed IPv6   |                            |
       |~~~~~~~~~~~|----------->|----------------------X     |
       |           |            | <---------------------     |
       |<~~~~~~~~~~|------------| ICMPv6 Host unreachable    |
       |SCHC compressed ICMPv6  |                            |
       |           |            |                            |
       |           |            |                            |


~~~~
{: #Fig-ICMPv6-down title='Example of ICMPv6 error message sent back to the Device'}

{{Fig-ICMPv6-down}} illustrates this behavior. The ICMPv6 error message is compressed
as described in {{ErrMsgCompr}} and forwarded over the LPWAN to the Device.

### ICMPv6 error message compression. {#ErrMsgCompr}

The ICMPv6 error messages defined in {{RFC4443}} contain the fields shown in
{{Fig-ICMP-error}}.

~~~~

       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |     Type      |     Code      |          Checksum             |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                            Value                              |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                    As much of invoking packet                 |
      +                as possible without the ICMPv6 packet          +
      |                exceeding the minimum IPv6 MTU                 |

~~~~
{: #Fig-ICMP-error title='ICMPv6 Error Message format'}

{{RFC4443}} states that Type can take the values 1 to 4, and Code can be set to values between 0 and 6.
Value is unused for the Destination Unreachable and Time Exceeded messages. It contains the MTU for
the Packet Too Big message and a pointer to the byte causing the error for the Parameter Error message.
Therefore, Value is never expected to be greater than 1280 in LPWAN networks.

The following generic rule can therefore be used to compress all ICMPv6 error messages as defined today.
More specific rules can also be defined to achieve better compression of some error messages.

The Type field can be associated to a matching list \[1, 2, 3, 4\] and is therefore compressed down to 2
bits. Code can be reduced to 3 bits using the LSB CDA. Value can be sent on 11 bits using the LSB
CDA, but if the Device is known to send smaller packets, then the size of this field can
be further reduced.

By {{RFC4443}}, the rest of the ICMPv6 message must contain as much as possible of the IPv6 offending (invoking) packet that triggered
this ICMPv6 error message. This information is used to try and identify the SCHC rule that
was used to decompress the offending IPv6 packet. If the rule can be found then the Rule Id
is added at the end of the compressed ICMPv6 message. Otherwise the compressed
packet ends with the compressed Value field.


 {{RFC4443}} states that the "ICMPv6 error message MUST include as much
of the IPv6 offending (invoking) packet ... as possible".
In order to comply with this requirement, if there is enough information in the incoming ICMPv6 message for the core SCHC C/D to
identify the rule that has been used to decompress the erroneous IPv6 packet, this Rule Id must be
sent in the compressed ICMPv6 message to the Device.
TODO: the erroneous IPv6 packet header (not just the Rule Id) should be sent back. This includes the Rule Id and the compression residue. This means the SCHC C/D uses the context backwards (in the reverse direction). How does the Device know it must also use the context backwards?

TODO: how does one know that the "payload" of a compressed-header packet is in fact another compressed header?

## Device does a ping {#DevicePings}

If a ping request is generated by a Device, then SCHC compression applies.

The format of an ICMPv6 Echo Request message is described in {{Fig-ICMPv6-Echo-Request}}, with Type=128 and Code=0.

~~~~

       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |     Type      |     Code      |          Checksum             |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |           Identifier          |        Sequence Number        |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |     Data ...
      +-+-+-+-+-
  
~~~~
{: #Fig-ICMPv6-Echo-Request title='ICMPv6 Echo Request message format'}

If we assume that one rule will be devoted to compressing Echo Request messages, then Type and Code are known in the rule to be 128 and 0 and can therefore be elided with the not-sent CDA.

Checksum can be reconstructed with the compute-checksum CDA and therefore is not transmitted.

{{RFC4443}} states that Identifier and Sequence Number are meant to
"aid in matching Echo Replies to this Echo Request" and that they "may be zero".
Data is "zero or more bytes of arbitrary data".

We recommend that Identifier be zero, Sequence Number be a counter on 3 bits, and Data be zero bytes (absent). Therefore, Identifier is elided with the not-sent CDA, Sequence Number is transmitted on 3 bits with the LSB CDA and no Data is transmitted.

The transmission cost of the Echo Request message is therefore the size of the Rule Id + 3 bits.

When the destination receives the Echo Request message, it will respond back with a Echo Reply message.
This message bears the same format as the Echo Request message but with Type = 129 
(see {{Fig-ICMPv6-Echo-Request}}).

{{RFC4443}} states that the Identifier, Sequence Number and Data fields of the Echo Reply message shall contain the same values as the invoking Echo Request message. Therefore, a rule shall be used similar to that used for compressing the Echo Request message.

TODO: how about a shared rule for Echo Request and Echo Reply with an LSB(1) CDA on the Type field? Or exploiting the Up/Down direction field in the rule?

## Device is ping'ed

If the Device is ping'ed (i.e., is the destination of an Echo Request message), the default behavior is to avoid
propagating the Echo Request message over the LPWAN.

This is the recommended behavior with the Code 0 (default value) of the Echo Request message.
In addition, this document defines two other Code values to achieve two other behaviors.

The resulting three behaviors are shown on {{Fig-ICMPv6-ping}} and described below:


~~~~

     Device       NGW     core SCHC C/D                 Internet Host

       |           |            |    Echo Request, Code=0    |
       |           |            |<---------------------------|
       |           |            |                            |
       |           |            |--------------------------->|
       |           |            |    Echo Reply,   Code=0    |
       |           |            |                            |
       |           |            |    Echo Request, Code=1    |
       |           |<==========>|<---------------------------|
       |           |            |                            |
       |           |            |--------------------------->|
       |           |            |    Echo Reply,   Code=1    |
       |           |            |    last seen               |
       |           |            |                            |
       |           |            |    Echo Request, Code=2    |
       |<~~~~~~~~~~|------------|<---------------------------|
       |           |            |                            |
       |~~~~~~~~~~~|----------->|--------------------------->|
       |           |            |    Echo Reply,   Code=2    |
       |           |            |    last seen               |
       |           |            |                            |

~~~~
{: #Fig-ICMPv6-ping title='Examples of ICMPv6 Echo Request/Reply'}

* Code = 0: The Echo Request message is not propagated on the LPWAN to the Device. If the SCHC C/D finds a rule in the context with the IPv6 address of the Device, it responds with an Echo Reply on behalf of the Device. If no rule is found with that IPv6 address, the SCHC C/D does not respond. 

TODO: again, we are assuming that no compression rule is equivalent to the device not providing the service.

* Code = 1: the SCHC C/D queries the NGW (or maintains a local database) and answers with
the number of seconds since the Device last transmission.

TODO: what does it mean to answer "with the number of seconds ..."? There is no such field in an Echo Reply message. Do we overwrite one of the fields (Identifier, Sequencer Number, Data)? They are all supposed to be copied from the Echo Request. Do we change their definition with Code==1 or Code==2?

* Code =  2: the SCHC C/D compresses the ICMPv6 message and forwards it to the Device. The Echo Reply message sent by the Device is also compressed.
Since the Echo Request message comes from the Internet, the values of the Identifier, Sequence Number and Data fields cannot be known in advance, and therefore must be transmitted.
However, it is likely that the Echo Request with Code 2 will be firewalled from the Internet and restricted to authorized users.
Therefore, the Echo Request message can be assumed to have the same content as recommended in {{DevicePings}}, and the same compression rules apply.


# Traceroute

The traceroute6 program sends successive probe packets destined to a chosen target
but with the Hop Limit value successively incremented from the initial value 1.

It expects to receive a "Time Exceeded" (Type = 3) "Hop Limit" (Code = 0) ICMPv6 error message back from the successive routers along the path to the destination.

The probe packet is usually a UDP datagram, but can also be a TCP datagram or even an ICMPv6 message.
The destination port is chosen in the unassigned range in hope that the destination, when eventually reached,
will respond with a "Destination Unreachable" (Type = 1) "Port Unreachable" (Code = 4) ICMPv6 error message.

It is not anticipated that a Device will want to traceroute a destination on the Internet.

By contrast, a host on the Internet may attempt to traceroute an IPv6 address that is assigned to an LPWAN device. This is described in {{Fig-traceroute}}.

~~~~

     Device       NGW     core SCHC C/D                 Internet 

       |           |            | Hop Limit=1, Dest Port=XXX |
       |           |            |<---------------------------|
       |           |            |                            |
       |           |            |--------------------------->|
       |           |            |   ICMPv6 Hop Limit error   |
       |           |            |                            |
       |           |            |                            |
       |           |            | Hop Limit=2, Dest Port=XXX |
       |           |            |<---------------------------|
       |           |            |                            |
       |           |            |--------------------------->|
       |           |            |  ICMPv6 Port Unreachable   |


~~~~
{: #Fig-traceroute title='Example of traceroute to the LPWAN Device'}


When the probe packet first reaches the core SCHC C/D, its remaining Hop Limit is 1. The core SCHC C/D will
respond back with a "Time Exceeded" (Type = 3) "Hop Limit" (Code = 0) ICMPv6 error message.
Later on, when the probe packet reaches the code SCHC C/D with a Hop Limit value of 2, the core SCHC C/D will,
as explained in {{ProxyErrMsg}}, answer back with a "Destination Unreachable" (Type = 1) "Port Unreachable" (Code = 4) ICMPv6 error message.
This is what the traceroute6 command expects.
Therefore, the traceroute6 command will work with LPWAN IPv6 destinations, except for the time displayed for the destination, which is actually the time to its proxy.

However, if the probe packet happens to hit a port that matches a SCHC rule for that Device, the packet will be compressed with this rule and sent over the LPWAN, which is unfortunate.
Forwarding of packets to the Device over the LPWAN should only be done from authenticated/trusted sources anyway.
Rate-limitation on top of authentication will mitigate this nuisance.


# Security considerations

TODO

# IANA Considerations

TODO


--- back