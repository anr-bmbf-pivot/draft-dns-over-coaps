---
v: 3

title: "DNS over CoAP (DoC)"
abbrev: DoC
docname: draft-ietf-core-dns-over-coap-latest
category: std
submissiontype: IETF

area: Applications
workgroup: CoRE
keyword: Internet-Draft

author:
 -  name: Martine Sophie Lenders
    org: Freie Universität Berlin
    abbrev: FU Berlin
    street: Takustrasse 9
    city: Berlin
    code: D-14195
    country: Germany
    email: m.lenders@fu-berlin.de
 -  name: Christian Amsüss
    email: christian@amsuess.com
 -  name: Cenk Gündoğan
    org: Huawei Technologies
    street: Riesstrasse 25
    city: Munich
    code: D-80992
    country: Germany
    email: cenk.gundogan@huawei.com
 -  name: Thomas C. Schmidt
    org: HAW Hamburg
    street: Berliner Tor 7
    city: Hamburg
    code: D-20099
    country: Germany
    email: t.schmidt@haw-hamburg.de
 -  name: Matthias Wählisch
    org: Freie Universität Berlin
    abbrev: FU Berlin
    street: Takustrasse 9
    city: Berlin
    code: D-14195
    country: Germany
    email: m.waehlisch@fu-berlin.de

normative:
  RFC1035: dns
  RFC7228: constr-nodes
  RFC7252: coap
  RFC7641: coap-observe
  RFC7959: coap-blockwise
  RFC8132: coap-fetch
  RFC8613: oscore
  RFC9147: dtls13

informative:
  RFC3986: uri
  RFC6690: core-link-format
  RFC8765: dns-push
  RFC8094: dodtls
  RFC8484: doh
  RFC9176: core-rd
  RFC9250: doq
  RFC8499: dns-terminology
  RFC7942: impl-status-section
  I-D.ietf-add-dnr: dnr
  I-D.ietf-core-href: cri


--- abstract

This document defines a protocol for sending DNS messages over the
Constrained Application Protocol (CoAP). These CoAP messages are protected
by DTLS-Secured CoAP (CoAPS) or Object Security for Constrained RESTful
Environments (OSCORE) to provide encrypted DNS message exchange for
constrained devices in the Internet of Things (IoT).

--- middle

Introduction
============

This document defines DNS over CoAP (DoC), a protocol to send DNS
{{-dns}} queries and get DNS responses over the Constrained Application
Protocol (CoAP) {{-coap}}. Each DNS query-response pair is mapped into a
CoAP message exchange. Each CoAP message is secured by DTLS {{-dtls13}} or
Object Security for Constrained RESTful Environments (OSCORE) {{-oscore}}
to ensure message integrity and confidentiality.

The application use case of DoC is inspired by DNS over HTTPS {{-doh}}
(DoH). DoC, however, aims for the deployment in the constrained Internet of
Things (IoT), which usually conflicts with the requirements introduced by
HTTPS.

To prevent TCP and HTTPS resource requirements, constrained IoT devices
could use DNS over DTLS {{-dodtls}}. In contrast to DNS over DTLS, DoC
utilizes CoAP features to mitigate drawbacks of datagram-based
communication. These features include: block-wise transfer, which solves
the Path MTU problem of DNS over DTLS (see {{-dodtls}}, section 5); CoAP
proxies, which provide an additional level of caching; re-use of data
structures for application traffic and DNS information, which saves memory
on constrained devices.

To prevent resource requirements of DTLS or TLS on top of UDP (e.g.,
introduced by DNS over QUIC {{-doq}}), DoC allows
for lightweight end-to-end payload encryption based on OSCORE.

~~~ aasvg

              . FETCH coaps://[2001:db8::1]/
             /
            /
           CoAP request
+------+   [DNS query]   +------+   DNS query     .---------------.
| DoC  |---------------->| DoC  |--- --- --- --->|      DNS        |
|Client|<----------------|Server|<--- --- --- ---| Infrastructure  |
+------+  CoAP response  +------+  DNS response   '---------------'
          [DNS response]
   \                        /\                                 /
    '-----DNS over CoAP----'  '--DNS over UDP/HTTPS/QUIC/...--'

~~~
{: #fig-overview-arch title="Basic DoC architecture"}


The most important components of DoC can be seen in {{fig-overview-arch}}: A DoC
client tries to resolve DNS information by sending DNS queries carried within
CoAP requests to a DoC server.
That DoC server is a DNS client (i.e., a stub or recursive resolver) that resolves DNS information by using other DNS transports such
as DNS over UDP {{-dns}}, DNS over HTTPS {{-doh}}, or DNS over QUIC {{-doq}} when communicating with the upstream
DNS infrastructure.
Using that information, the DoC server then replies to the queries of the DoC client with DNS
responses carried within CoAP responses.

Note that this is an orthogonal specification to DoH, as it the CoRE-specific FETCH method is used.
This was done to take benefit from having the DNS query in the payload as with POST, but still
having the caching advantages we would gain with GET.
Having the DNS query in the payload means we do not need extra base64 encoding, which would increase
code complexity and message sizes and are able to transfer a query block-wise.


Terminology
===========

A server that provides the service specified in this document is called a "DoC
server" to differentiate it from a classic "DNS server".
A DoC server acts either as a DNS stub resolver {{-dns-terminology}} or a DNS recursive resolver {{-dns-terminology}}.

A client using the service specified in this document to retrieve the
DNS information is called a "DoC client".

The term "constrained nodes" is used as defined in {{-constr-nodes}}.

The terms "CoAP payload" and "CoAP body" are used as defined in {{-coap-blockwise}}, Section 2.

{::boilerplate bcp14-tagged}

Selection of a DoC Server
=========================

In this document, it is assumed that the DoC client knows the DoC server and the DNS resource at the
DoC server.
Possible options could be manual configuration of a URI {{-uri}} or CRI {{-cri}},
or automatic configuration, e.g., using a CoRE resource directory
{{-core-rd}}, DHCP or Router Advertisement options {{-dnr}}.
Automatic configuration SHOULD only be done from a trusted source.

TBD DNR Service Parameters + SVCB Resource Records (also see [#22](https://github.com/core-wg/draft-dns-over-coap/issues/22)):

- `coap` Exists already in TLS Application-Layer Protocol Negotiation (ALPN) Protocol IDs
  registry {{?RFC8323}}
- Never mandated for DTLS, should be kept for CoAP over TLS; DTLS needs its own ALPN ID
- SVCB: Also needs to considering OSCORE/EDHOC

When discovering the DNS resource through a link mechanism that allows describing a resource type
(e.g., the Resource Type Attribute in {{-core-link-format}}), the resource type "core.dns" can be
used to identify a generic DNS resolver that is available to the client.

While there is no path specified it is RECOMMENDED to use the root path "/" for the DNS resource to
keep the CoAP requests small.

Basic Message Exchange
======================

The "application/dns-message" Content-Format    {#sec:content-format}
--------------------------------------------
This document defines a CoAP Content-Format number for the Internet media type "application/dns-message" to be the mnemonic 553--based on the port assignment of DNS.
This media type is defined as in {{-doh}} Section 6, i.e., a single DNS message encoded in the DNS on-the-wire format {{-dns}}.
Both DoC client and DoC server MUST be able to parse contents in the "application/dns-message" format.

DNS Queries in CoAP Requests    {#sec:queries}
----------------------------

A DoC client encodes a single DNS query in one or more CoAP request
messages that use the CoAP FETCH {{-coap-fetch}} method.
Requests SHOULD include an Accept option to indicate the type of content that can be parsed in the response.

Since CoAP provides reliability of the message layer (e.g. CON) the retransmission mechanism of the
DNS protocol as defined in {{-dns}} is not needed.

### Request Format

When sending a CoAP request, a DoC client MUST include the DNS query in the body of the CoAP request.
As specified in {{-coap-fetch}} Section 2.3.1, the type of content of the body MUST be indicated using the Content-Format option.
This document specifies the usage of Content-Format "application/dns-message" (details see {{sec:content-format}}).
A DoC server MUST be able to parse requests of Content-Format "application/dns-message".

### Support of CoAP Caching {#sec:req-caching}

The DoC client SHOULD set the ID field of the DNS header always to 0 to enable a CoAP cache (e.g., a CoAP proxy en-route) to respond to the same DNS queries with a cache entry.
This ensures that the CoAP Cache-Key (see {{-coap-fetch}} Section 2) does not change when multiple DNS queries for the same DNS data, carried in CoAP requests, are issued.

### Examples

The following example illustrates the usage of a CoAP message to
resolve "example.org. IN AAAA" based on the URI "coaps://\[2001:db8::1\]/". The
CoAP body is encoded in "application/dns-message" Content Format.

    FETCH coaps://[2001:db8::1]/
    Content-Format: application/dns-message
    Accept: application/dns-message
    Payload: 00 00 01 20 00 02 00 00 00 00 00 00 07 65 78 61 [binary]
             6d 70 6c 65 03 6f 72 67 00 00 1c 00 01 c0 0c 00 [binary]
             01 00 01                                        [binary]


DNS Responses in CoAP Responses
-------------------------------

Each DNS query-response pair is mapped to a CoAP REST request-response
operation. DNS responses are provided in the body of the CoAP response.
A DoC server MUST be able to produce responses in the "application/dns-message"
Content-Format (details see {{sec:content-format}}) when requested.
A DoC client MUST understand responses in "application/dns-message" format
when it does not send an Accept option.
Any other response format than "application/dns-message" MUST be indicated with
the Content-Format option by the DoC server.

### Response Codes and Handling DNS and CoAP errors

A DNS response indicates either success or failure in the Response code of
the DNS header (see {{-dns}} Section 4.1.1). It is RECOMMENDED that
CoAP responses that carry any valid DNS response use a "2.05 Content"
response code.

CoAP responses use non-successful response codes MUST NOT contain a DNS response
and MUST only be used on errors in the CoAP layer or when a request does not
fulfill the requirements of the DoC protocol.

Communication errors with a DNS server (e.g., timeouts) SHOULD be indicated
by including a SERVFAIL DNS response in a successful CoAP response.

A DoC client might try to repeat a non-successful exchange unless otherwise prohibited.
The DoC client might also decide to repeat a non-successful exchange with a different URI, for instance, when the response indicates an unsupported Content-Format.

### Support of CoAP Caching {#sec:resp-caching}

For reliability and energy saving measures content decoupling and thus en-route caching on proxies takes a far greater role than it does, e.g., in HTTP.
Likewise, CoAP utilizes cache validation to refresh stale cache entries without large messages which often uses hashing over the message content for ETag generation.
As such, the approach to guarantee the same cache key for DNS responses as proposed in DoH ({{-doh}}, section 5.1) is not sufficient and needs to be updated so that the TTLs in the response are more often the same regardless of query time.

The DoC server MUST ensure that any sum of the Max-Age value of a CoAP response and any TTL in the
DNS response is less or equal to the corresponding TTL received from an upstream DNS server.
This also includes the default Max-Age value of 60 seconds (see {{-coap}}, section 5.10.5) when no Max-Age option is provided.
The DoC client MUST then add the Max-Age value of the carrying CoAP response to all TTLs in a DNS response on reception and use these calculated TTLs for the associated records.

The RECOMMENDED algorithm to assure the requirement for the DoC is to set the Max-Age option of a response to the minimum TTL of a DNS response and to subtract this value from all TTLs of that DNS response.
This prevents expired records unintentionally being served from an intermediate CoAP cache.
Additionally, it allows for the ETag value for cache validation, if it is based on the content of the response, not to change even if the TTL values are updated by an upstream DNS cache.
If only one record set per DNS response is assumed, a simplification of this algorithm is to just set all TTLs in the response to 0 and set the TTLs at the DoC client to the value of the Max-Age option.


<!--
With short responses, a usable ETag might be almost as long as the response.
With long-lived responses, the client does not need to revalidate often.
With responses large enough to be fragmented,
it's best practice for servers to set an ETag anyway.
As specified in {{-coap}} and {{-coap-fetch}}, if the response associated with
the ETag is still valid, the response uses the "2.03 Valid" code and consequently
carries no payload.
-->

### Examples

The following examples illustrate the replies to the query "example.org. IN
AAAA record", recursion turned on. Successful responses carry one answer
record including address 2001:db8:1::1:2:3:4 and TTL 58719.

A successful response:

    2.05 Content
    Content-Format: application/dns-message
    Max-Age: 58719
    Payload: 00 00 81 a0 00 01 00 01 00 00 00 00 07 65 78 61 [binary]
             6d 70 6c 65 03 6f 72 67 00 00 1c 00 01 c0 0c 00 [binary]
             1c 00 01 00 01 37 49 00 10 20 01 0d b8 00 01 00 [binary]
             00 00 01 00 02 00 03 00 04                      [binary]

When a DNS error (SERVFAIL in this case) is noted in the DNS response, the CoAP
response still indicates success:

    2.05 Content
    Content-Format: application/dns-message
    Payload: 00 00 81 a2 00 01 00 00 00 00 00 00 07 65 78 61 [binary]
             6d 70 6c 65 03 6f 72 67 00 00 1c 00 01          [binary]

When an error occurs on the CoAP layer, the DoC server SHOULD respond with
an appropriate CoAP error, for instance "4.15 Unsupported Content-Format"
if the Content-Format option in the request was not set to
"application/dns-message" and the Content-Format is not otherwise supported by
the server.

CoAP/CoRE Integration
=====================

DNS Push
--------
DNS Push requires additional overhead, which conflicts with constrained resources,
This is the reason why it is RECOMMENDED to use CoAP Observe {{-coap-observe}} instead of DNS Push
in the DoC domain.

If the CoAP request indicates that the DoC client wants to observe a resource record, a DoC server
MAY use a DNS Subscribe message {{-dns-push}} instead of a classic DNS query to fetch the
information on behalf of a DoC client.
If this is not supported by the DoC server, it MUST act as if the resource were not observable.

Whenever the DoC server receives a DNS Push message {{-dns-push}} from the DNS
infrastructure for an observed resource record, the DoC server sends an appropriate Observe response
to the DoC client.

If no more DoC clients observe a resource record for which the DoC server has an open subscription,
the DoC server MUST use a DNS Unsubscribe message {{-dns-push}} to close its subscription to the
resource record as well.

OSCORE
------
It is RECOMMENDED to carry DNS messages end-to-end encrypted using OSCORE {{-oscore}}.
The exchange of the security context is out of scope of this document.

Mapping DoC to DoH
------------------
This document provides no specification how to map between DoC and DoH, e.g., at a CoAP-HTTP-proxy,
and it is NOT RECOMMENDED.
Rewriting the FETCH method ({{sec:queries}}) and the TTL rewriting ({{sec:resp-caching}}) specified
in this draft for performance benefits would be non-trivial.
It is RECOMMENDED to use a DNS forwarder to map between DoC and DoH, as would be done to map between
any other DNS transport.

Considerations for Unencrypted Use {#sec:unencrypted-coap}
==================================
While not recommended,
DoC can be used without any encryption
e.g., in very constrained environments where encryption is not possible or necessary.
It can also be used when lower layers provide secure communication between client and server.
In both cases,
potential benefits of
unencrypted DoC usage over classic DNS are e.g. block-wise transfer or alternative CoAP
Content-Formats to overcome link-layer constraints.
For unencrypted DoC usage the ID of the DNS message MAY still be set to 0 to leverage the cache
benefits described in {{sec:req-caching}}.
In this case an unpredictable token MUST be picked for each CoAP request that carries such a DNS
message.


Implementation Status
=====================

This section records the status of known implementations of the protocol
defined by this specification at the time of posting of this
Internet-Draft, and is based on a proposal described in
{{-impl-status-section}}.  The description of implementations in this
section is intended to assist the IETF in its decision processes in
progressing drafts to RFCs.  Please note that the listing of any individual
implementation here does not imply endorsement by the IETF.  Furthermore,
no effort has been spent to verify the information presented here that was
supplied by IETF contributors.  This is not intended as, and must not be
construed to be, a catalog of available implementations or their features.
Readers are advised to note that other implementations may exist.

According to {{-impl-status-section}}, "this will allow reviewers and
working groups to assign due consideration to documents that have the
benefit of running code, which may serve as evidence of valuable
experimentation and feedback that have made the implemented protocols more
mature.  It is up to the individual working groups to use this information
as they see fit".

DoC Client
----------
The authors of this document provide a [DoC client implementation available
in the IoT operating system RIOT][gcoap_dns].

Level of maturity:
: production

Version compability:
: draft-ietf-core-dns-over-coap-02

License:
: LGPL-2.1

Contact information:
: `Martine Lenders <m.lenders@fu-berlin.de>`

Last update of this information:
: March 2023

DoC Server
----------
The authors of this document provide a [DoC server implementation in
Python][aiodnsprox].

Level of maturity:
: production

Version compability:
: draft-ietf-core-dns-over-coap-02

License:
: MIT

Contact information:
: `Martine Lenders <m.lenders@fu-berlin.de>`

Last update of this information:
: March 2023

Security Considerations
=======================

When using unencrypted CoAP (see {{sec:unencrypted-coap}}), setting the ID of a DNS message to 0 as
specified in {{sec:req-caching}} could open the DNS cache of a DoC client to cache poisoning attacks
via response spoofing.
This documents requires an unpredictable CoAP token in each DoC query from the client when CoAP is
not secured to mitigate such an attack over DoC (see {{sec:unencrypted-coap}}).

For encrypted usage with DTLS or OSCORE the impact of a fixed ID on security is limited, as both
harden against injecting spoofed responses.
Consequently, it is of little concern to leverage the benefits of CoAP caching by setting the ID to
0.

TODO more security


IANA Considerations
===================

New "application/dns-message" Content-Format
--------------------------------------------

IANA is requested to assign CoAP Content-Format ID for the DNS message media
type in the "CoAP Content-Formats" sub-registry, within the "CoRE Parameters"
registry {{-coap}}, corresponding to the "application/dns-message" media
type from the "Media Types" registry:

Media-Type: application/dns-message

Encoding: -

Id: 553

Reference: \[TBD-this-spec\]

New "core.dns" Resource Type
----------------------------

IANA is requested to assign a new Resource Type (rt=) Link Target Attribute, "core.dns" in the
"Resource Type (rt=) Link Target Attribute Values" sub-registry, within the "CoRE Parameters"
register {{-core-link-format}}.

Attribute Value: core.dns

Description: DNS over CoAP resource.

Reference: \[TBD-this-spec\] {{selection-of-a-doc-server}}


--- back

Change Log
==========

Since [draft-ietf-core-dns-over-coap-01]
----------------------------------------

- Specify DoC server role in terms of DNS terminology
- Clarify communication of DoC to DNS infrastructure is agnostic of the transport
- Add subsection on how to implement DNS Push in DoC
- Add appendix on reference implementation

Since [draft-ietf-core-dns-over-coap-00]
----------------------------------------

- SVGify ASCII art
- Move section on "DoC Server Considerations" (was Section 5.1) to its own draft
  ([draft-lenders-dns-cns])
- Replace layer violating statement for CON with statement of fact
- Add security considerations on ID=0

Since [draft-lenders-dns-over-coap-04]
--------------------------------------

- Removed change log of draft-lenders-dns-over-coap

# Acknowledgments
{:unnumbered}

The authors of this document want to thank Ben Schwartz and Tim Wicinski for their feedback and
comments.

[draft-ietf-core-dns-over-coap-01]: https://datatracker.ietf.org/doc/html/draft-ietf-core-dns-over-coap-01
[draft-ietf-core-dns-over-coap-00]: https://datatracker.ietf.org/doc/html/draft-ietf-core-dns-over-coap-00
[draft-lenders-dns-cns]: https://datatracker.ietf.org/doc/draft-lenders-dns-cns/
[draft-lenders-dns-over-coap-04]: https://datatracker.ietf.org/doc/html/draft-lenders-dns-over-coap-04
[gcoap_dns]: https://doc.riot-os.org/group__net__gcoap__dns.html
[aiodnsprox]: https://github.com/anr-bmbf-pivot/aiodnsprox
