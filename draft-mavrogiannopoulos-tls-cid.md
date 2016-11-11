---
title: Datagram Transport Transport Layer Security (DTLS) Transport-Agnostic Security Association Extension for TLS
abbrev: DTLS ta_sa Extension
docname: draft-mavrogiannopoulos-tls-cid-latest
category: std

ipr: trust200902
area: Security
workgroup: TLS Working Group
keyword: Internet-Draft

stand_alone: yes
pi:
  rfcedstyle: yes
  toc: yes
  tocindent: yes
  sortrefs: yes
  symrefs: yes
  strict: yes
  comments: yes
  inline: yes
  text-list-symbols: o-*+
  compact: yes
  subcompact: yes

author:
 -
    ins: N. Mavrogiannopoulos
    name: Nikos Mavrogiannopoulos
    organization: RedHat
    email: nmav@redhat.com
 -
    ins: H. Tschofenig
    name: Hannes Tschofenig
    organization: ARM
    email: hannes.tschofenig@arm.com
 -
    ins: T. Fossati
    name: Thomas Fossati
    organization: Nokia
    email: thomas.fossati@nokia.com

normative:
  RFC2119:
  RFC4226:
  RFC5246:
  RFC5705:
  RFC6066:
  RFC6347:
  I-D.ietf-tls-tls13:
  I-D.rescorla-tls-dtls13:

informative:
  I-D.barrett-mobile-dtls:
  RFC4301:
  DTLSMOB:
    title: DTLS Mobility
    author:
      -
        ins: R. Seggelmann
      -
        ins: M. Tuexen
      -
        ins: E.P. Rathgeb
    date: 2012


--- abstract

This memo proposes a new Datagram Transport Transport Layer Security (DTLS) extension that provides the ability to negotiate, during handshake, a transport independent identifier that is unique per security association. This identifier effectively decouples the DTLS session from the underlying transport protocol, allowing the same security association to be migrated across different instances of the same transport, or to a completely different transport.

--- middle

# Introduction

DTLS security context demultiplexing is done via the 5-tuple. Therefore, it needs to be re-negotiated from scratch whenever the transport identifiers change. For example, when moving the network attachment from WLAN to a cellular connection, or when the IP address of the IoT devices changes during a sleep cycle. A NAT device may also modify the source UDP port after an short idle period.  In such situations, there is not enough information in the DTLS record header for a server that is handling multiple concurrent sessions to associate the new address to an existing client.

This memo proposes a new TLS extension {{RFC6066}} that provides the ability to negotiate, at handshake time, a transport independent identifier that is unique per security association. We call this identifier 'Connection ID (CID)'.  Its function is to effectively decouple the DTLS session from the underlying transport protocol, allowing the same DTLS security association to be migrated across different instances of the same transport, or even to a completely different transport as showed in {{fig:transp-handover}}.

~~~~~~~~~~
                                     00
                                     /\
                                     :
 IP                    UDP           : DTLS Record Header
 +-----+-----+-------+ +-----+-----+ : +---------+-------+------
 | src | dst | proto | | src | dst | : | Seq#i   |  CID  | ...
 +-----+-----+-------+ +-----+-----+ : +---------+-------+------
 `----------------+----------------' :              ^
                   `................ : .............'
  <Handover event>                   :
                   GSM-SMS           : DTLS Record Header
                   +-------+-------+ : +---------+-------+-----
                   | tp-oa | tp-da | : | Seq#i+1 |  CID  | ...
                   +-------+-------+ : +---------+-------+-----
                                     :
                                     \/
                                     00
~~~~~~~~~~
{: #fig:transp-handover title="Transparent Handover of DTLS Session"}

We present two methods for producing the CID: the first uses a single value generated unilaterally by the server which is fixed throughout the session, whereas the second provides a sequence of identifiers that are created using a HMAC-based OTP algorithm {{RFC4226}} keyed with the session shared secret. The latter allows a client to shift to a new identifier, for example when switching networks, and is intended as a measure to counteract tracking.  However, it must be noted that this is not a generally viable way to avoid being tracked: in fact, it becomes totally ineffective when the client is oblivious of changes in the underlying transport identifiers (e.g., on NAT rebind after timeout), and also does not guarantee unique identifiers (see {{sec:clash}}).  For both methods the generated CID is 32-bits in size, like the Security Parameter Index (SPI) in IPsec {{RFC4301}}.

Similar approaches to support transparent handover of a DTLS session have been described in {{I-D.barrett-mobile-dtls}} and {{DTLSMOB}}.

# Conventions used in this document

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in {{RFC2119}}.

# Transport Agnostic Security Associatiation Extension

In order to negotiate a Transport Agnostic Security Association, clients include an extension of type "ta_sa" in the (extended) client hello ({{sec:ext-cli}}).  Servers that receive an extended hello containing a "ta_sa" extension MAY agree to use a Transport Agnostic Security Association by including an extension of type "ta_sa" in the extended server hello ({{sec:srv-ext}}).

In both server and client agree, the DTLSCiphertext format does change after the DTLS connection state is updated; i.e., for the sending side, after the ChangeCipherSpec message is sent.  For the receiving sides, after the ChangeCipherSpec is received.

The DTLSCiphertext format is changed for both the client and the server.  However, only a client can initiate a switch to an unused 'cid' value; a server MUST utilize the same value seen on the last valid message received by the client.  A server which receives a 'cid' value which is not expected (e.g., a value outside its advertised window) MAY ignore the packet.


~~~~~~~
         struct {
              ContentType type;
              ProtocolVersion version;
              uint16 epoch;
              uint48 sequence_number;
              uint32 cid;                          // New field
              uint16 length;
              select (CipherSpec.cipher_type) {
                case block:  GenericBlockCipher;
                case aead:   GenericAEADCipher;
              } fragment;
         } DTLSCiphertext;
~~~~~~~
{: #fig:record title="Modified DTLS Record Format"}


## Extended Client Hello
{: #sec:ext-cli }

In order to negotiate a Transport Agnostic Security Association, clients include an extension of type "ta_sa" in the (extended) client hello.  The "extension_data" field of this extension SHALL contain the ClientSecAssocData structure in {{fig:cli-ext}}.

 In the case of the fixed(0) type, the cid of the packets after ChangeCipherSpec is sent explicitly by the server.

In the case of hotp(1) option the initial 'cid' is calculated using the HOTP algorithm ({{RFC4226}}) as follows:

- A 20-byte string is generated {{RFC5705}} exporter.  The key material exporter uses the label "EXPORTER-ta-security- association-hotp" without the quotes, and without any context value.
- The initial 'cid' equals to the first HOTP value (i.e., the 31-bit value of Sbits in {{RFC4226}} notation), as generated by using the previously exported value as K.

Subsequent values of the HOTP algorithm can be used in place of the initial, as long as they fall into the negotiated window_size (see {{fig:srv-ext}}).

~~~~~~~
         enum {
             fixed(0), hotp(1), (255)
         } SecAssocType;

         struct {
              SecAssocType types<1..2^8-1>;
         } ClientSecAssocData;
~~~~~~~
{: #fig:cli-ext title="ta_sa extension, client"}

## Extended Server Hello
{: #sec:srv-ext }

Servers that receive an extended hello containing a "ta_sa" extension MAY agree to use a Transport Agnostic Security Association by including an extension of type "ta_sa", with "extension_data" being ServerSecAssocData, in the extended server hello ({{fig:srv-ext}}).

~~~~~~~
         struct {
              SecAssocType type;
              select (type) {
                  case fixed:
                      struct {
                          uint32 cid_value;
                      };
                  case hotp:
                      struct {
                          uint16 window_size;
                      };
              };
         } ServerSecAssocData;
~~~~~~~
{: #fig:srv-ext title="ta_sa extension, server"}

In the case of the fixed(0) type, the cid_value contains the value to be used as 'cid'.  In the case of hotp(1), the window_size must be greater or equal to 1, and indicates the number of HOTP values that the server can recognize for this particular client.

# Clashing HOTP CIDs
{: #sec:clash }
Under the assumption that HOTP behaves like a PRF, the following table encodes the probability of a (32-bit) HOTP CID clash event in relation to the number of sessions a server might be concurrently handling.

| Sessions | Collision probability                        |
|:---------|:---------------------------------------------|
| 10       |  1.16415320717e-08, or about 1 in 85,899,347 |
| 100      |  1.16415254059e-06, or about 1 in 858,994    |
| 1000     |  0.000116408545826, or about 1 in 8,590      |
| 10000    |  0.011574031737, or about 1 in 86            |
| 100000   |  0.687813095694, or about 1 in 1             |
| 1000000  |  1.0, or about 1 in 1                        |

TODO

# Security Considerations

TODO

# IANA Considerations

This document defines a TLS extension: ts_sa(TODO).  This extension is assigned from the TLS ExtensionType registry defined in {{RFC5246}}.

# Acknowledgments

Thanks to
Achim Krauss,
Carsten Bormann,
Kai Hudalla,
Simon Bernard,
Stephen Farrell,
for helpful comments and discussions that have shaped the document.

This work is partially supported by the European Commission under Horizon 2020 grant agreement no. 688421 Measurement and Architecture for a Middleboxed Internet (MAMI). This support does not imply endorsement.

--- back
