---
stand_alone: true
ipr: trust200902
docname: draft-barthel-icmpv6-schc-00
cat: info
pi:
  symrefs: 'yes'
  sortrefs: 'yes'
  strict: 'yes'
  compact: 'yes'
  toc: 'yes'

title: LPWAN Static Context Header Compression (SCHC) for ICMPv6
abbrev: LPWAN ICMPv6 compression
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
  org: Institut MINES TELECOM ; IMT Atlantique
  street:
  - 2 rue de la Chataigneraie
  - CS 17607
  city: 35576 Cesson-Sevigne Cedex
  country: France
  email: Laurent.Toutain@imt-atlantique.fr
- ins: A. Kandasamy
  name: Arunprabhu Kandasamy
  org: Acklio
normative:
  rfc4443:
  rfc4861:
  rfc4884:
  I-D.ietf-lpwan-ipv6-static-context-hc:

--- abstract

ICMPv6 is a companion protocol to IPv6 that is used to configure hosts in the network, inform of errors
in packet
delivery and provide a basic device management with the ping command. 

<!--This document describes
> how ICMPv6 headers can be compressed using SCHC.  This document does not discuss the
> Neighbor Discovery compression.
-->

This document decribes
how to adapt ICMPv6 as defined in {{rfc4443}} to LPWAN using SCHC header compression mechanism
and how to protect the LPWAN network and the End-Device from undesirable ICMPv6 traffic. 





--- middle



# Introduction 

{{rfc4443}} specifies ICMPv6 and defines a generic message format to be used either by nodes to inform 
the source about errors during packet delivery or by applications for simple host 
configuration or management.

{{rfc4443}} defines 4 error messages: Destination Unreachable, Packet Too Big, 
Time Exceeded and Parameter Problem. Echo Request and Echo Reply are also defined in 
this RFC to support the ping program. Other message types, such as for Neighbor Discovery
Protocol {{rfc4861}} are defined in other RFCs. 

This document focuses on the compression of {{rfc4443}} messages over a LPWAN network.

In the LPWAN architecture, different scenarii can be exhibited:

* the End-Device is the source of an ICMP message, mainly in response to an incorrect incoming IPv6
message, or in response to a ping request. In that case, the End-Device should be protected from this 
traffic and the core SCHC C/D should act as a proxy to avoid unwanted traffic on the LPWAN.

* the End-Device is the destination of the ICMP message, mainly in response to a
packet sent by the End-Device to the network that generates an error message. This document describes in
section XXXX the compression to be applied.


# End-Device as source of ICMPv6 message

As stated in {{rfc4443}}, a node should generate an ICMPv6 message in response to an
IPv6 packet that is malformed or which cannot be processed due to some incorrect field value.


The intent of this document is to spare both the End-Device and the LPWAN network this un-necessary traffic.
The incorrect packets should be caught at the core SCHC C/D and the ICMPv6 notification should be sent back from there.


~~~~

  End-Device      NGW     core SCHC C/D                 Internet Host
     
       |           |            |    Destination Port=XXX    |
       |           |            |<---------------------------|
       |           |            |                            |
       |           |            |--------------------------->|
       |           |            | ICMPv6 Port Unreachable    |
       |           |            |                            |
       |           |            |                            |


~~~~
{: #Fig-ICMPv6-up title='Example of ICMPv6 error message sent back to the Internet'}

{{Fig-ICMPv6-up}} shows an example of an IPv6 packet trying to reach an End-Device. The port used
as destination is not known from the core SCHC C/D. Instead of sending the packet over
the LPWAN and having this packet rejected by the End-Device, the core SCHC C/D issues
an ICMPv6 error message "Destination Unreachable" on behalf of the End-Device.

# End-Device as destination of the ICMPv6 message

In this situation, an End-Device is configured to send information to a server. If this
server is not accessible on the Internet, an ICMPv6 message will be generated back towards the End-Device by an intermediate router.
This information can be useful to the End-Device, for example for reducing the reporting rate in case of periodic reporting of data.

~~~~

  End-Device      NGW     core SCHC C/D                Internet Server
     
       |           |            |                            |
       | SCHC compressed IPv6   |                            |
       |~~~~~~~~~~~|----------->|----------------------X     |
       |           |            | <---------------------     |
       |<~~~~~~~~~~|------------| ICMPv6 Host unreachable    |
       |SCHC compressed ICMPv6  |                            |
       |           |            |                            |
       |           |            |                            |


~~~~
{: #Fig-ICMPv6-down title='Example of ICMPv6 error message sent back to the End-Device'}

{{Fig-ICMPv6-down}} illustrates this behavior. {{rfc4443}} states that the "ICMPv6 error message MUST include as much 
of the IPv6 offending (invoking) packet ... as possible".
In order to comply with this requirement, if there is enough information in the incoming ICMPv6 message for the core SCHC C/D to
identify the rule that has been used to uncompress the erroneous IPv6 packet, this Rule Id MUST be
sent in the compressed ICMPv6 message to the End-Device.

A SCHC rule is defined to compress the ICMPv6 header itself.

# Ping management

If a ping request is generated by an End-Device, then SCHC compression rules apply. If the
End-Device is the destination of the ping request, the default behavior is to avoid
sending the ping request over the LPWAN. Three behaviors can be defined, with the code value
being used to distinguish them:

* code = 0, the SCHC C/D answers on the behalf of the End-Device if a rule regarding the 
IPv6 address of the End-Device is found.

* code = 1, the SCHC C/D queries the NGW (or maintains a local database) and answers with
the number of seconds since the End-Device last transmission.

* code =  2, the SCHC C/D compresses the ICMPv6 message and forwards it to the End-Device.

~~~~

  End-Device      NGW     core SCHC C/D                 Internet Host
     
       |           |            |    Echo request code=0     |
       |           |            |<---------------------------|
       |           |            |                            |
       |           |            |--------------------------->|
       |           |            |    Echo reply code = 0     |
       |           |            |                            |
       |           |            |    Echo request code=1     |
       |           |<==========>|<---------------------------|
       |           |            |                            |
       |           |            |--------------------------->|
       |           |            |    Echo reply code = 1     |
       |           |            |    last seen               |
       |           |            |                            |
       |           |            |    Echo request code=2     |
       |<~~~~~~~~~~|------------|<---------------------------|
       |           |            |                            |
       |~~~~~~~~~~~|----------->|--------------------------->|
       |           |            |    Echo reply code = 1     |
       |           |            |    last seen               |
       |           |            |                            |

~~~~
{: #Fig-ICMPv6-ping title='Example of ICMPv6 error message'}

# ICMPv6 Error Message compression.

ICMPv6 Error messages defined in {{rfc4443}} contains several fields has shown in 
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
      |                exceeding the minimum IPv6 MTU [IPv6]          |

~~~~ 
{: #Fig-ICMP-error title='ICMPv6 Error Message'}

Type can take the values between 1 and 4, code can be set between 0 and 6. Value is
unused for Destination Unreachable and Time exceed message. It contains the MTU for
Packet Too Big and a pointer on the byte causing the error for Parameter error message. 
Therefore this value should not be higher than 1280 bytes in LPWAN networks.

The following generic rule can be used to compress ICMPv6 messages. Some more specific
rules can also be defined. 

Type field can be associated to a matching list [1, 2, 3, 4] which be compressed into 2 
bits. Code can be reduced to 3 bits using LSB CDF. Value can be set to 11 bits using LSB 
CDF, but if the End-Device is known to send smaller packets, then the size of this field can
be reduced.

The rest of the ICMPv6 message contains the beginning of the message that have generated
this ICMPv6 error message. This information can be used to identify the SCHC rule that
was used to decompress the erroneous packet. If the rule has been found then the rule id 
can be added at the end of the compressed ICMPv6 message. Otherwise the compressed 
packet ends after the compressed value field.



## Traceroute case


# ICMPv6 Proxying Error Message.

# ICMPv6 Echo Request/Reply Management.

--- back
