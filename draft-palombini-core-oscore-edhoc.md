---
title: "Combining EDHOC and OSCORE"
abbrev: "EDHOC + OSCORE"
docname: draft-palombini-core-oscore-edhoc-latest
cat: std

ipr: trust200902
area: Internet
workgroup: CoRE Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: F. Palombini
    name: Francesca Palombini
    organization: Ericsson
    email: francesca.palombini@ericsson.com
 -
    ins: M. Tiloca
    name: Marco Tiloca
    org: RISE AB
    email: marco.tiloca@ri.se
 -
    ins: R. Hoeglund
    name: Rikard Hoeglund
    org: RISE AB
    email: rikard.hoglund@ri.se
 -
    ins: S. Hristozov
    name: Stefan Hristozov
    organization: Fraunhofer AISEC
    email: stefan.hristozov@aisec.fraunhofer.de
 -
    ins: G. Selander
    name: Goeran Selander
    organization: Ericsson
    email: goran.selander@ericsson.com

normative:
  RFC2119:
  RFC7252:
  RFC8174:
  RFC8613:
  I-D.ietf-lake-edhoc:
  I-D.ietf-cbor-7049bis:
  I-D.ietf-lake-reqs:

informative:



--- abstract

This document defines possible optimization approaches for combining the lightweight authenticated key exchange protocol EDHOC run over CoAP with the first subsequent OSCORE transaction. This combination reduces the number of round trips required to set up an OSCORE Security Context and complete an OSCORE transaction using that context.

--- middle

# Introduction

This document presents possible optimization approaches to combine the lightweight authenticated key exchange protocol EDHOC {{I-D.ietf-lake-edhoc}}, when running over CoAP {{RFC7252}}, with the first subsequent OSCORE {{RFC8613}} transaction.

This allows for a minimum number of round trips necessary to setup the OSCORE Security Context and complete an OSCORE transaction, for example when an IoT device gets configured in a network for the first time.

The number of protocol round trips impacts the minimum number of flights, which can have a substantial impact on performance with certain radio technologies as discussed in Section 2.11 of {{I-D.ietf-lake-reqs}}.

Without this optimization, it is not possible, not even in theory, to achieve the minimum number of flights. This optimization makes it possible also in practice, since the last message of the EDHOC protocol can be made relatively small (see Section 1 of {{I-D.ietf-lake-edhoc}}), thus allowing additional OSCORE protected CoAP data within target MTU sizes {{I-D.ietf-lake-reqs}}.

The goal of this document is to provide details on different alternatives for transporting and processing the necessary data, gather opinions on the different approaches, and select only one of those.

## Terminology

{::boilerplate bcp14}

The reader is expected to be familiar with terms and concepts defined in CoAP {{RFC7252}}, CBOR {{I-D.ietf-cbor-7049bis}}, OSCORE {{RFC8613}} and EDHOC {{I-D.ietf-lake-edhoc}}.

# Background

EDHOC is a 3-message key exchange protocol. Section 7.1 of {{I-D.ietf-lake-edhoc}} specifies how to transport EDHOC over CoAP: the EDHOC data (referred to as "EDHOC messages") are transported in the payload of CoAP requests and responses.

This draft deals with the case of the Initiator acting as CoAP Client and the Responder acting as CoAP Server. (The case of the Initiator acting as CoAP server cannot be optimized in this way.) That is, the CoAP Client sends a POST request containing the EDHOC message 1 to a reserved resource at the CoAP Server. This triggers the EDHOC exchange on the CoAP Server, which replies with a 2.04 (Changed) Response containing the EDHOC message 2. Finally, the EDHOC message 3 is sent by the CoAP Client in a CoAP POST request to the same resource used for the EDHOC message 1. The Content-Format of these CoAP messages is set to "application/edhoc".

After this exchange takes place, and after successful verifications specified in the EDHOC protocol, the Client and Server derive the OSCORE Security Context, as specified in Section 7.1.1 of {{I-D.ietf-lake-edhoc}}. Then, they are ready to use OSCORE.

This sequential way of running EDHOC and then OSCORE is specified in {{fig-non-combined}}. As shown in the figure, this mechanism is executed in 3 round trips.

~~~~~~~~~~~~~~~~~
   CoAP Client                                  CoAP Server
        | ------------- EDHOC message_1 ------------> |
        |                                             |
        | <------------ EDHOC message_2 ------------- |
        |                                             |
EDHOC verification                                    |
        |                                             |
        | ------------- EDHOC message_3 ------------> |
        |                                             |
        |                                    EDHOC verification
        |                                             |
OSCORE Sec Ctx                                OSCORE Sec Ctx
  Derivation                                    Derivation
        |                                             |
        | -------------- OSCORE Request ------------> |
        |                                             |
        | <------------ OSCORE Response ------------- |
        |                                             |
~~~~~~~~~~~~~~~~~
{: #fig-non-combined title="EDHOC and OSCORE run sequentially" artwork-align="center"}

The number of roundtrips can be minimized: after receiving the EDHOC message 2, the CoAP Client has all the information needed to derive the OSCORE Security Context before sending the EDHOC message 3.

This means that the Client can potentially send at the same time both the EDHOC message 3 and the subsequent OSCORE Request. On a semantic level, this approach practically requires to send two separate REST requests at the same time.

The high level message flow of running EDHOC and OSCORE combined is shown in {{fig-combined}}.

Defining the specific details of how to transport the data and of their processing order is the goal of this specification.

~~~~~~~~~~~~~~~~~
   CoAP Client                                  CoAP Server
        | ------------- EDHOC message_1 ------------> |
        |                                             |
        | <------------ EDHOC message_2 ------------- |
        |                                             |
EDHOC verification +                                  |
  OSCORE Sec Ctx                                      |
    Derivation                                        |
        |                                             |
        | ------------- EDHOC message_3 ------------> |
        |              + OSCORE Request               |
        |                                             |
        |                                  EDHOC verification +
        |                                     OSCORE Sec Ctx
        |                                        Derivation
        |                                             |
        | <------------ OSCORE Response ------------- |
        |                                             |
~~~~~~~~~~~~~~~~~
{: #fig-combined title="EDHOC and OSCORE combined" artwork-align="center"}

# EDHOC in OSCORE {#edhoc-in-oscore}

This approach consists in sending the EDHOC message 3 inside an OSCORE message (i.e., an OSCORE protected CoAP message).

The request is in practice the OSCORE Request from {{fig-non-combined}}, sent to a protected resource and with the correct CoAP method and options, with the addition that it also transports the EDHOC message 3.

As the EDHOC message 3 may be too large to be included in a CoAP Option, e.g. if containing a large public key certificate chain, it would have to be transported in the CoAP payload.

The payload of the request is formatted as a CBOR sequence {{I-D.ietf-lake-reqs}} of two CBOR-wrapped byte string: the EDHOC message 3 and the OSCORE ciphertext, in this order.

Note that the OSCORE ciphertext is not computed over the EDHOC message 3, which is not protected by OSCORE. That is, the client first prepares the OSCORE Request as in {{fig-non-combined}}. Then, it reformats the payload to include also the EDHOC message 3, as defined above.

The usage of this approach is indicated by a signalling information, which can be either a new EDHOC option (see {{sign-1}}) or the OSCORE option with a particular Flag Bit set (see {{sign-2}}).

When receiving such a request, the Server needs to perform the following processing, in addition to the EDHOC, OSCORE and CoAP processing:

1. Check the signalling information to identify that this is an OSCORE + EDHOC request.

2. Extract the EDHOC message 3 from the payload.

3. Execute the EDHOC processing, including verifications and OSCORE Security Context derivation.

4. Decrypt and verify the remaining OSCORE protected CoAP request as defined by OSCORE.

5. Process the CoAP request.

The following sections expand on the 2 ways of signalling that the EDHOC message is transported in the OSCORE message.

## Signalling in a New EDHOC Option {#sign-1}

<!-- Malisa preferred option -->

One way to signal that the Server is to extract and process the EDHOC message 3 before the OSCORE message is processed is to define a new CoAP Option, called the EDHOC Option.

This Option being present means that the message contains EDHOC data in the payload, that must be extracted and processed before the rest of the message can be processed.

In particular, the EDHOC message is to be extracted from the CoAP payload, as the CBOR-wrapped byte string first element of a CBOR sequence.

The Option is critical, Safe-to-Forward, and part of the Cache-Key.

The Option value is always empty. If any value is sent, the value is simply discarded.

The Option must occur at most once.

The Option is of Class U for OSCORE.

~~~~~~~~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Ver| T |  TKL  |      Code     |          Message ID           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Token (if any, TKL bytes) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  OSCORE option  |   EDHOC option  | other options (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1 1 1 1 1 1 1 1|    Payload (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-opt title="CoAP message for EDHOC and OSCORE combined - signalled with EDHOC option" artwork-align="center"}

An example based on the OSCORE test vector from Appendix C.4 of {{RFC8613}} and the EDHOC test vector from Appendix B.2 of {{I-D.ietf-lake-edhoc}} is given in {{fig-edhoc-opt-2}}. The example assumes that the EDHOC option is registered with CoAP option number 13.

~~~~~~~~~~~~~~~~~
   o  OSCORE option value: 0x0914 (2 bytes)

   o  ciphertext: 0x612f1092f1776f1c1668b3825e (13 bytes)

   o  EDHOC option value: - (0 bytes)

   o  EDHOC message 3: 085253c3991999a5ffb86921e99b607c067770e0
      (20 bytes)

   From there:

   o  Protected CoAP request (OSCORE message): 0x44025d1f0000397439
      6c6f63616c686f737462 0914 04 ff 54085253C3991999A5FFB86921E99
      B607C067770E0 4d612f1092f1776f1c1668b3825e (58 bytes)
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-opt-2 title="CoAP message for EDHOC and OSCORE combined - signalled with EDHOC option" artwork-align="center"}


## Signalling in the OSCORE Option {#sign-2}

<!-- Klaus preferred option -->

Another way to signal that the EDHOC message is to be extracted from the CoAP payload as the CBOR-wrapped byte string first element of a CBOR sequence, and that the processing defined in {{edhoc-in-oscore}} is to be executed, is to use one of the OSCORE Flag Bits.

Bit Position: 1

Name: EDHOC

Description: Set to 1 if the payload is a sequence of EDHOC data and OSCORE payload.

Reference: this document

The OSCORE Option value with the EDHOC bit set is given in {{fig-edhoc-bit}}.

~~~~~~~~~~~~~~~~~
 0 1 2 3 4 5 6 7 <------------- n bytes -------------->
+-+-+-+-+-+-+-+-+--------------------------------------
|0 1 0|h|k|  n  |       Partial IV (if any) ...
+-+-+-+-+-+-+-+-+--------------------------------------

<- 1 byte -> <----- s bytes ------>
+------------+----------------------+------------------+
| s (if any) | kid context (if any) | kid (if any) ... |
+------------+----------------------+------------------+
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-bit title="The OSCORE Option Value with EDHOC bit set" artwork-align="center"}


~~~~~~~~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Ver| T |  TKL  |      Code     |          Message ID           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Token (if any, TKL bytes) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  OSCORE opt(including EDHOC bit)  |  other options (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1 1 1 1 1 1 1 1|    Payload (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-bit-2 title="CoAP message for EDHOC and OSCORE combined - signalled within the OSCORE option" artwork-align="center"}

An example based on the OSCORE test vector from Appendix C.4 of {{RFC8613}} and the EDHOC test vector from Appendix B.2 of {{I-D.ietf-lake-edhoc}} is given in {{fig-edhoc-bit-3}}. The example assumes that the EDHOC option is registered with CoAP option number 13.

~~~~~~~~~~~~~~~~~
   o  OSCORE option value without EDHOC bit set: 0x0914 (2 bytes)

   o  OSCORE option value with EDHOC bit set: 0x4914 (2 bytes)

   o  ciphertext: 0x612f1092f1776f1c1668b3825e (13 bytes)

   o  EDHOC message 3: 085253c3991999a5ffb86921e99b607c067770e0
      (20 bytes)

   From there:

   o  Protected CoAP request (OSCORE message): 0x44025d1f000039743
      96c6f63616c686f737462 4914 ff 54085253C3991999A5FFB86921E99B
      607C067770E0 4d612f1092f1776f1c1668b3825e (58 bytes)
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-bit-3 title="CoAP message for EDHOC and OSCORE combined - signalled with EDHOC option" artwork-align="center"}

# Security Considerations

The same security considerations from OSCORE {{RFC8613}} and EDHOC {{I-D.ietf-lake-edhoc}} hold for this document.

TODO (more considerations)

# IANA Considerations

Depending on the option chosen, this document will either register a new CoAP option number to the CoAp Option Number registry, or a new bit to the OSCORE Flag Bits registry.

--- back

# Acknowledgments
{:numbered="false"}

The authors sincerely thank Christian Amsuess, Klaus Hartke, Jim Schaad and Malisa Vucinic for their feedback and comments in the discussion leading up to this draft.

The work on this document has been partly supported by VINNOVA and the Celtic-Next project CRITISEC.
