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
    street: Isafjordsgatan 22
    city: Kista
    code: SE-16440 Stockholm
    country: Sweden
    email: marco.tiloca@ri.se
 -
    ins: R. Hoeglund
    name: Rikard Hoeglund
    org: RISE AB
    street: Isafjordsgatan 22
    city: Kista
    code: SE-16440 Stockholm
    country: Sweden
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
  RFC8742:
  RFC8949:
  I-D.ietf-lake-edhoc:

informative:



--- abstract

This document defines possible optimization approaches for combining the lightweight authenticated key exchange protocol EDHOC run over CoAP with the first subsequent OSCORE transaction. This combination reduces the number of round trips required to set up an OSCORE Security Context and complete an OSCORE transaction using that context.

--- middle

# Introduction

This document presents possible optimization approaches to combine the lightweight authenticated key exchange protocol EDHOC {{I-D.ietf-lake-edhoc}}, when running over CoAP {{RFC7252}}, with the first subsequent OSCORE {{RFC8613}} transaction.

This allows for a minimum number of round trips necessary to setup the OSCORE Security Context and complete an OSCORE transaction, for example when an IoT device gets configured in a network for the first time.

The number of protocol round trips impacts the minimum number of flights, which can have a substantial impact on performance with certain radio technologies.

Without this optimization, it is not possible, not even in theory, to achieve the minimum number of flights. This optimization makes it possible also in practice, since the last message of the EDHOC protocol can be made relatively small (see Section 1 of {{I-D.ietf-lake-edhoc}}), thus allowing additional OSCORE protected CoAP data within target MTU sizes.

The goal of this document is to provide details on different alternatives for transporting and processing the necessary data, gather opinions on the different approaches, and select only one of those.

## Terminology

{::boilerplate bcp14}

The reader is expected to be familiar with terms and concepts defined in CoAP {{RFC7252}}, CBOR {{RFC8949}}, CBOR sequences {{RFC8742}}, OSCORE {{RFC8613}} and EDHOC {{I-D.ietf-lake-edhoc}}.

# Background

EDHOC is a 3-message key exchange protocol. Section 7.2 of {{I-D.ietf-lake-edhoc}} specifies how to transport EDHOC over CoAP: the EDHOC data (referred to as "EDHOC messages") are transported in the payload of CoAP requests and responses.

This draft deals with the case of the Initiator acting as CoAP Client and the Responder acting as CoAP Server; instead, the case of the Initiator acting as CoAP server cannot be optimized by using this approach.

That is, the CoAP Client sends a POST request containing EDHOC message_1 to a reserved resource at the CoAP Server. This triggers the EDHOC exchange on the CoAP Server, which replies with a 2.04 (Changed) Response containing EDHOC message_2. Finally, the CoAP client sends EDHOC message_3, as a CoAP POST request to the same resource used for EDHOC message_1. The Content-Format of these CoAP messages may be set to "application/edhoc".

After this exchange takes place, and after successful verifications specified in the EDHOC protocol, the Client and Server derive the OSCORE Security Context, as specified in Section 7.2.1 of {{I-D.ietf-lake-edhoc}}. Then, they are ready to use OSCORE.

This sequential way of running EDHOC and then OSCORE is specified in {{fig-non-combined}}. As shown in the figure, this mechanism takes 3 round trips to complete.

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
        |                                             +
OSCORE Sec Ctx                                OSCORE Sec Ctx
  Derivation                                     Derivation
        |                                             |
        | ------------- OSCORE Request -------------> |
        |                                             |
        | <------------ OSCORE Response ------------- |
        |                                             |
~~~~~~~~~~~~~~~~~
{: #fig-non-combined title="EDHOC and OSCORE run sequentially" artwork-align="center"}

The number of roundtrips can be minimized as follows. Already after receiving EDHOC message_2 and before sending EDHOC message_3, the CoAP Client has all the information needed to derive the OSCORE Security Context.

This means that the Client can potentially send at the same time both EDHOC message_3 and the subsequent OSCORE Request. On a semantic level, this approach practically requires to send two separate REST requests at the same time.

The high level message flow of running EDHOC and OSCORE combined is shown in {{fig-combined}}.

Defining the specific details of how to transport the data and of their processing order is the goal of this specification, as defined in {{edhoc-in-oscore}}.

~~~~~~~~~~~~~~~~~
   CoAP Client                                  CoAP Server
        | ------------- EDHOC message_1 ------------> |
        |                                             |
        | <------------ EDHOC message_2 ------------- |
        |                                             |
EDHOC verification                                    |
        +                                             |
  OSCORE Sec Ctx                                      |
    Derivation                                        |
        |                                             |
        | ---- EDHOC message_3 + OSCORE Request ----> |
        |                                             |
        |                                    EDHOC verification
        |                                             +
        |                                     OSCORE Sec Ctx
        |                                        Derivation
        |                                             |
        | <------------ OSCORE Response ------------- |
        |                                             |
~~~~~~~~~~~~~~~~~
{: #fig-combined title="EDHOC and OSCORE combined" artwork-align="center"}

# EDHOC in OSCORE {#edhoc-in-oscore}

This approach consists in sending EDHOC message_3 inside an OSCORE protected CoAP message.

The resulting EDHOC + OSCORE request is in practice the OSCORE Request from {{fig-non-combined}}, sent to a protected resource and with the correct CoAP method and options, with the addition that it also transports EDHOC message_3.

Since EDHOC message_3 may be too large to be included in a CoAP Option, e.g. if containing a large public key certificate chain, it has to be transported in the CoAP payload.

In particular, the payload of the EDHOC + OSCORE request is formatted as a CBOR sequence {{RFC8742}} composed of two CBOR byte strings in the following order. The first CBOR byte string has as value EDHOC message_3. The second CBOR byte string has as value the OSCORE ciphertext of the original OSCORE Request.

Note that the OSCORE ciphertext is not computed over EDHOC message_3, which is not protected by OSCORE. That is, the client first prepares the OSCORE Request as in {{fig-non-combined}}. Then, it reformats the payload to include also EDHOC message_3, as defined above. The result is the EDHOC + OSCORE request to send.

The usage of this approach is indicated by a signalling information in the EDHOC + OSCORE request. This can be either a new EDHOC Option (see {{sign-1}}) or a particular Flag Bit set in the OSCORE Option (see {{sign-2}}).

When receiving such a request, the Server needs to perform the following processing, in addition to the EDHOC, OSCORE and CoAP processing:

1. Check the signalling information to identify that this is an EDHOC + OSCORE request.

2. Extract EDHOC message_3 from the payload of the EDHOC + OSCORE request, as the value of the first CBOR byte string in the CBOR sequence.

3. Execute the EDHOC processing on EDHOC message_3, including verifications and the OSCORE Security Context derivation, as per Section 5.4 and Section 7.2.1 of {{I-D.ietf-lake-edhoc}}, respectively.

4. Extract the OSCORE ciphertext from the payload of the EDHOC + OSCORE request, as the value of the second CBOR byte string in the CBOR sequence. Then, set the CoAP payload of the request to the extracted ciphertext.

5. Decrypt and verify the OSCORE protected CoAP request resulting from step 4, as per Section 8.2 of {{RFC8613}}.

6. Process the CoAP request resulting from step 5.

The following sections expand on the two ways of signalling that EDHOC message_3 is transported in the EDHOC + OSCORE request.

## Signalling in a New EDHOC Option {#sign-1}

<!-- Malisa preferred option -->

One way to signal that the Server has to extract and process EDHOC message_3 before processing the OSCORE protected CoAP request is to use a new CoAP Option, called the EDHOC Option and defined in this section.

The EDHOC Option has the properties summarized in {{fig-edhoc-option}}, which extends Table 4 of {{RFC7252}}. The option is Critical, Safe-to-Forward, and part of the Cache-Key. The option MUST occur at most once and is always empty. If any value is sent, the value is simply ignored. The option is intended only for CoAP requests and is of Class U for OSCORE {{RFC8613}}.

~~~~~~~~~~~
+-----+---+---+---+---+-------+--------+--------+---------+
| No. | C | U | N | R | Name  | Format | Length | Default |
+-----+---+---+---+---+-------+--------+--------+---------+
| TBD | x |   |   |   | EDHOC | Empty  |   0    | (none)  |
+-----+---+---+---+---+-------+--------+--------+---------+
     C=Critical, U=Unsafe, N=NoCacheKey, R=Repeatable
~~~~~~~~~~~
{: #fig-edhoc-option title="The EDHOC Option." artwork-align="center"}

The presence of this option means that the message payload contains also EDHOC data, that must be extracted and processed as defined in {{edhoc-in-oscore}}, before the rest of the message can be processed.

{{fig-edhoc-opt}} shows the format of a CoAP message containing both the EDHOC data and the OSCORE ciphertext, using the newly defined EDHOC option for signaling.

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
|1 1 1 1 1 1 1 1|    Payload
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-opt title="CoAP message for EDHOC and OSCORE combined - signaled with the EDHOC Option" artwork-align="center"}

An example based on the OSCORE test vector from Appendix C.4 of {{RFC8613}} and the EDHOC test vector from Appendix B.2 of {{I-D.ietf-lake-edhoc}} is given in {{fig-edhoc-opt-2}}. In particular, the example assumes that:

* The used Partial IV is 0, consistently with the first request protected with the new OSCORE Security Context.

* The Sender ID of the client is 0x20. This corresponds to the EDHOC Connection Identifier C_R, which is encoded as the bstr_identifier 0x08 in EDHOC message_3.

* The EDHOC option is registered with CoAP option number 13.

~~~~~~~~~~~~~~~~~
   o  OSCORE option value: 0x090020 (3 bytes)

   o  EDHOC option value: - (0 bytes)

   o  EDHOC message_3: 0x085253c3991999a5ffb86921e99b607c067770e0
      (20 bytes)

   o  ciphertext: 0x612f1092f1776f1c1668b3825e (13 bytes)
      
   From there:

   o  Protected CoAP request (OSCORE message):
      0x44025d1f 00003974
        39 6c6f63616c686f7374
        63 090020
        40
        ff 54085253C3991999A5FFB86921E99B607C067770E0
           4d612f1092f1776f1c1668b3825e
      (59 bytes)
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-opt-2 title="CoAP message for EDHOC and OSCORE combined - signaled with the EDHOC Option" artwork-align="center"}


## Signalling in the OSCORE Option {#sign-2}

<!-- Klaus preferred option -->

Another way to signal that the Server has to extract and process EDHOC message_3 before processing the OSCORE protected CoAP request is to use one of the OSCORE Flag Bits of the OSCORE Option, as defined below.

Bit Position: 1

Name: EDHOC

Description: Set to 1 if the message payload contains also EDHOC data, that must be extracted and processed as defined in {{edhoc-in-oscore}}, before the rest of the message can be processed.

Reference: this document

The OSCORE Option value with the EDHOC bit set is given in {{fig-edhoc-bit}}.

~~~~~~~~~~~~~~~~~
 0 1 2 3 4 5 6 7 <------------- n bytes -------------->
+-+-+-+-+-+-+-+-+--------------------------------------
|0|1|0|h|k|  n  |       Partial IV (if any) ...
+-+-+-+-+-+-+-+-+--------------------------------------

 <- 1 byte -> <----- s bytes ------>
+------------+----------------------+------------------+
| s (if any) | kid context (if any) | kid (if any) ... |
+------------+----------------------+------------------+
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-bit title="The OSCORE Option Value with the EDHOC bit set" artwork-align="center"}

{{fig-edhoc-bit-2}} shows the format of a CoAP message containing both the OSCORE ciphertext and EDHOC message_3, using the Flag Bit 1 in the OSCORE Option for signaling.

~~~~~~~~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Ver| T |  TKL  |      Code     |          Message ID           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Token (if any, TKL bytes) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  OSCORE opt (with EDHOC bit set)  |  other options (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1 1 1 1 1 1 1 1|    Payload
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-bit-2 title="CoAP message for EDHOC and OSCORE combined - signaled within the OSCORE option" artwork-align="center"}

An example based on the OSCORE test vector from Appendix C.4 of {{RFC8613}} and the EDHOC test vector from Appendix B.2 of {{I-D.ietf-lake-edhoc}} is given in {{fig-edhoc-bit-3}}. The same assumptions as in {{sign-1}} apply for this example, with respect to the Partial IV and the Sender ID of the client.

~~~~~~~~~~~~~~~~~
   o  OSCORE option value without EDHOC bit set: 0x090020 (3 bytes)

   o  OSCORE option value with EDHOC bit set: 0x490020 (3 bytes)

   o  EDHOC message_3: 0x085253c3991999a5ffb86921e99b607c067770e0
      (20 bytes)
      
   o  ciphertext: 0x612f1092f1776f1c1668b3825e (13 bytes)

   From there:

   o  Protected CoAP request (OSCORE message):
      0x44025d1f 00003974
        39 6c6f63616c686f7374
        63 490020
        ff 54085253C3991999A5FFB86921E99B607C067770E0
           4d612f1092f1776f1c1668b3825e
      (58 bytes)
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-bit-3 title="CoAP message for EDHOC and OSCORE combined - signaled within the OSCORE Option" artwork-align="center"}

# Security Considerations

The same security considerations from OSCORE {{RFC8613}} and EDHOC {{I-D.ietf-lake-edhoc}} hold for this document.

TODO (more considerations)

# IANA Considerations

Depending on the chosen signaling method, this document will register either a new CoAP Option number to the "CoAP Option Numbers" registry, or a new bit to the "OSCORE Flag Bits" registry.

--- back

# Acknowledgments
{:numbered="false"}

The authors sincerely thank Christian Amsuess, Klaus Hartke, Jim Schaad and Malisa Vucinic for their feedback and comments in the discussion leading up to this draft.

The work on this document has been partly supported by VINNOVA and the Celtic-Next project CRITISEC; and by the H2020 project SIFIS-Home (Grant agreement 952652).
