---
title: "TCP RST Diagnostic Payload"
abbrev: "AC Glue for VPN Models"
category: std

docname: draft-boucadair-tcpm-rst-diagnostic-payload-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "WIT"
workgroup: "tcpm"
keyword:
 - Service diagostc


author:
 -
    fullname: Mohamed Boucadair
    organization: Orange
    email: mohamed.boucadair@orange.com

 -
    fullname: Tirumaleswar Reddy
    organization: Nokia
    country: India
    email: kondtir@gmail.com

 -
    fullname: Jason Xing
    organization: Tencent
    email: kerneljasonxing@gmail.com


contributor:

normative:

informative:
  IANA-TCP:
    title: Transmission Control Protocol (TCP) Parameters
    date: false
    target: https://www.iana.org/assignments/tcp-parameters/tcp-parameters.xhtml#

  Private-Enterprise-Numbers:
    title: Private Enterprise Numbers
    date: 4 May 2020
    target: https://www.iana.org/assignments/enterprise-numbers

--- abstract

   This document specifies a diagnostic payload format returned in TCP
   RST segments.  Such payloads are used to share with an endpoint the
   reasons for which a TCP connection has been reset.  Sharing this
   information is meant to ease diagnostic and troubleshooting.

--- middle

# Introduction

   A TCP connection {{!RFC9293}} can be reset by a peer for various
   reasons, e.g., received data does not correspond to an active
   connection.  Also, a TCP connection can be reset by an on-path
   service function (e.g., Carrier Grade NAT (CGN) {{?RFC6888}}, NAT64
   {{?RFC6146}}, or firewall) for several reasons.  Typically, a Network
   Address Translator (NAT) function can generate an RST segment to
   notify an endpoint upon the expiry of the lifetime of the
   corresponding mapping entry or because an RST segment was received
   from a peer ({{Section 2.2 of ?RFC7857}}).  A TCP connection can also be
   closed by a user or an application at any time.  However, the peer
   that receives an RST segment does not have any hint about the reason
   that led to terminating the connection.  Likewise, the application
   that relies upon such a TCP connection may not easily identify the
   reason for the connection closure.  Troubleshooting such events at
   the remote side of the connection that receives the RST segment may
   not be trivial.

   This document fills this void by specifying a format of the
   diagnostic payload that is returned in an RST segment.  Returning
   such data is consistent with the provision in {{Section 3.5.3 of  !RFC9293}} for RST segments, especially:

   {: quote}
   >"TCP implementations SHOULD allow a received RST segment to
   >  include data (SHLD-2)."

   This document does not change the conditions under which an RST
   segment is generated ({{Section 3.5.2 of !RFC9293}}).

   The generic procedure for processing an RST segment is specified in
   {{Section 3.5.3 of !RFC9293}}.  Only the deviations from that procedure
   to insert and validate a diagnostic payload is provided in {{payload}}.
   {{examples}} provides a set of examples to illustrate the use of TCP RST
   diagnostic payloads.

   This document specifies the format and the overall approach to ease
   maintaining the list of codes while allowing for adding new codes as
   needed in the future and accommodating any existing vendor-specific
   codes.  An initial version of error codes is available in Table 2.
   However, the authoritative source to retrieve the full list of error
   codes is the IANA-maintained registry ({{causes}}).

   Preliminary investigation based on some major CGN vendors revealed
   that RSTs with data are not discarded and are translated according to
   any matching mapping entry.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

   This document makes use of the terms defined in {{Section 4 of !RFC9293}}.

#  RST Diagnostic Payload {#payload}

   The RST diagnostic payload MUST be encoded using Concise Binary
   Object Representation (CBOR) Sequence {{!RFC8742}}.  The Concise Data
   Definition Language (CDDL) {{!RFC8610}} for the diagnostic payload is
   shown in {{cddl}}.

~~~~ CDDL
    ; This defines an array, the elements of which are to be used
    ; in a CBOR Sequence. There is exactly one occurrence.
    diagnostic-payload = [magic-cookie, reason]
    ; Magic cookie to identify a payload that follows this specification
    magic-cookie = 12345
    ; Reset reason details:
    reason= {
      ? reason-code: uint,
      ? pen:uint,
      ? reason-description: tstr,
    }
~~~~
{: #cddl title='Structure of the RST Diagnostic Payload'}

   The RST diagnostic payload comprises a magic cookie that is used to
   unambiguously identify an RST payload that follows this
   specification.  It MUST be set to the RFC number to be assigned to
   this document.

   > Note to the RFC Editor: Please replace "12345" with the RFC number
   > assigned to this document.

   All parameters in the reason component of an RST diagnostic payload
   are mapped to their CBOR key values as specified in {{cbor}}.  The
   description of these parameters is as follows:

   reason-code:
   :  This parameter takes a value from an available registry
      such as the "TCP Failure Causes" registry ({{causes}}).

   pen:
   :  Includes a Private Enterprise Number
      [Private-Enterprise-Numbers].  This parameter MAY be included when
      the reason code is not taken from the IANA-maintained registry
      ({{causes}}), but from a vendor-specific registry.

   reason-description:
   :  Includes a brief description of the reset reason
      encoded as UTF-8 {{!RFC3629}}.  This parameter SHOULD NOT be included
      if a reason code is supplied.  This parameter is useful only for
      reset reasons that are not yet registered or for application-
      specific reset reasons.

   At least one of "reason-code" and "reason-description" parameters
   MUST be included in an RST diagnostic payload.  The "pen" parameter
   MUST be omitted if a reason code from the IANA-maintained registry
   ({{causes}}) fits the reset case.

   Malformed RST diagnostic payload messages that include the magic
   cookie MUST be silently ignored by the receiver.

   A peer that receives a valid diagnostic payload may pass the reset
   reason information to the local application in addition to the
   information (MUST-12) described in {{Section 3.6 of !RFC9293}}.  That
   information may also be logged locally, unless a local policy
   specifies otherwise.  How the information is passed to an application
   and how it is stored locally is implementation-specific.

   As per {{Section 3.6 of !RFC9293}}, one or more RST segments can be sent
   to reset a connection.  Whether a TCP endpoint elects to send more
   than one RST with only a subset of them that include the diagnostic
   payload is implementation-specific.

#  Some Examples {#examples}

   To ease readability, the CBOR diagnostic notation ({{Section 8 of !RFC8949}})
   with the parameter names rather than their CBOR key values
   in {{cbor}} is used in Figures 3, 4, 5, and 6.

   {{fig-2}} depicts an example of an RST diagnostic payload that is
   generated to inform the peer that the TCP connection is reset because
   an ACK was received from that peer while the connection is still in
   the LISTEN state ({{Section 3.10.7.2 of ?RFC9293}}).

~~~~ CDDL
   19 3039 # unsigned(12345)
   A1    # map(1)
      01 # unsigned(1)
      02 # unsigned(2)
~~~~
{: #fig-2 title='Example of an RST Diagnostic Payload with Reason Code (CBOR Encoding)'}

   {{fig-3}} depicts the same RST diagnostic payload as the one shown in
   {{fig-2}} but following the CBOR diagnostic notation.

~~~~ CDDL
   [
     12345,
     {
       "reason-code": 2
     }
   ]
~~~~
{: #fig-3 title='Example of an RST Diagnostic Payload with Reason Code (Diagnostic Notation)'}

   {{fig-4}} shows an example of an RST diagnostic payload that includes
   a free description to report a case that is not covered by an
   appropriate code from the IANA-maintained registry ({{causes}}).

~~~~ CDDL
   [
     12345,
     {
       "reason-description": "brief human-readable description"
     }
   ]
~~~~
{: #fig-4 title='Example of an RST Diagnostic Payload with Reason Description (Diagnostic Notation)'}

   An RST diagnostic payload may also be sent by an on-path service
   function.  For example, the following diagnostic payload is returned
   by a NAT function upon expiry of the mapping entry to which the TCP
   connection is bound ({{fig-5}}).

~~~~ CDDL
   [
     12345,
     {
       "reason-code": 8
     }
   ]
~~~~
{: #fig-5 title='Example of an RST Diagnostic Payload to Report Connection Timeout (Diagnostic Notation)'}

   {{fig-6}} illustrates an RST diagnostic payload that is returned by a
   peer that resets a TCP connection for a reason code 1234 defined by a
   vendor with the private enterprise number 32473.

~~~~ CDDL
   [
     12345,
     {
       "reason-code": 1234,
       "pen": 32473
     }
   ]
~~~~
{: #fig-6 title='Example of an RST Diagnostic Payload to Report Vendor-Specific Reason Code (Diagnostic Notation)'}

   {{fig-6}} uses the Enterprise Number 32473 defined for documentation
   use {{?RFC5612}}.

# IANA Considerations

##  RST Diagnostic Payload CBOR Key Values {#cbor}

   IANA is requested to create a new registry titled "RST Diagnostic
   Payload CBOR Key Values" under the "Transmission Control Protocol
   (TCP) Parameters" registry group [IANA-TCP].

   The key value MUST be an integer in the 1-255 range.

   The assignment policy for this registry is "IETF Review" ({{Section 4.8 of !RFC8126}}).

   The structure of this subregistry and the initial values are provided
   below:


| Parameter Name     | CBOR  Key| CBOR Major Type & Information| Reference    |
|:------------------:|:--------:|:----------------------------:|:------------:|
| reason-code        |   1      | 0 unsigned                   |[ThisDocument]|
| pen                |   2      | 0 unsigned                   |[ThisDocument]|
| reason-description |   3      | 3 text string                |[ThisDocument]|
{: #sub title='Initial CBOR Keys'}

##  New Registry for TCP Failure Causes {#causes}

   This document requests IANA to create a new registry entitled "TCP
   Failure Causes" under the "Transmission Control Protocol (TCP)
   Parameters" registry group [IANA-TCP].

   Values are taken from the 1-65535 range.

   The assignment policy for this registry is "Expert Review"
   ({{Section 4.5 of !RFC8126}}).

   The designated experts may approve registration once they checked
   that the new requested code is not covered by an existing code and if
   the provided reasoning to register the new code is acceptable.  A
   registration request may supply a pointer to a specification where
   that code is defined.  However, a registration may be accepted even
   if no permanent and readily available public specification is
   available.

   The registry is initially populated with the values listed in
   {{initial}}.

 | Value | Description                                                    | Specification (if available)               |
 |:-----:|:---------------------------------------------------------------|:-------------------------------------------|
 | 1     | Illegal Option                                                 | {{Section 3.1 of !RFC9293}}                |
 | 2     | New data is received after CLOSE is called                     | Sections 3.6.1 and  3.10.7.1 of [RFC9293]  |
 | 3     | ABORT Process                                                  | {{Section 3.10.5 of !RFC9293}}                |
 | 4     | Received ACK while the connection is still in the LISTEN state | {{Section 3.10.7.2 of !RFC9293}}           |
 | 5     | Malformed Message                                              | [ThisDocument]                             |
 | 6     | Not Authorized                                                 | [ThisDocument]                             |
 | 7     | Resource Exceeded                                              | [ThisDocument]                             |
 | 8     | Network Failure                                                | [ThisDocument]                             |
 | 9     | Reset received from he peer                                    | [ThisDocument]                             |
 | 10    | Destination Unreachable                                        | [ThisDocument]                             |
 | 11    | Connection Timeout                                             | [ThisDocument]                             |
 | 12    | Too much outstanding data                                      | {{Section 3.6 of !RFC8684}}                |
 | 13    | Unacceptable performance                                       | {{Section 3.6 of !RFC8684}}                |
 | 14    | Middlebox interference                                         | {{Section 3.6 of !RFC8684}}                |
 {: #initial title='Initial TCP Failure Causes'}

   Note that codes in the 6-10 range can be used by service functions
   such as translators.


#  Security Considerations

   {{!RFC9293}} discusses TCP-related security considerations.  In
   particular, RST-specific attacks and their mitigations are discussed
   in {{Section 3.10.7.3 of !RFC9293}}.

   In addition to these considerations, it is RECOMMENDED to control the
   size of acceptable diagnostic payload and keep it as brief as
   possible.  The RECOMMENDED acceptable maximum size of the RST
   diagnostic payload is 255 octets.

   Also, it is RECOMMENDED to avoid leaking privacy-related information
   as part of the diagnostic payload (e.g., including a description such
   as "user X resets explicitly the connection" is not recommended).
   The "reason-description" string, when present, should not include any
   private information that an observer would not otherwise have access
   to.

   The presence of vendor-specific reason codes (Section 3) may be used
   to fingerprint hosts.  Such a concern does not apply if the reason
   codes are taken from the IANA-maintained registry.  Implementers are,
   thus, encouraged to register new codes within IANA instead of
   maintaining specific registries.

   The reason description, when present, is not intended to be displayed
   to end users, but to be consumed by applications.  Such a description
   may carry a malicious message to mislead the end-user.


--- back

# Acknowledgments
{:numbered="false"}

   The "diagnostic payload" name is inspired by {{Section 5.5.2 of  ?RFC7252}}
  that was cited by Carsten Bormann in the tcpm mailing list.

   Thanks to Jon Shallow for the comments.  Thanks also to Li Jinghui
   for the discussion.
