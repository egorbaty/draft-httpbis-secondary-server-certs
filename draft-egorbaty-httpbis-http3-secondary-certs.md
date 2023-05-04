---
title: "Secondary Certificate Authentication in HTTP/3"
abbrev: "Secondary Cert Auth in HTTP/3"
category: std

docname: draft-egorbaty-httpbis-http3-secondary-certs-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "HTTP"
venue:
  group: "HTTP"
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "egorbaty/draft-httpbis-http3-secondary-certs"
  latest: "https://egorbaty.github.io/draft-httpbis-http3-secondary-certs/draft-egorbaty-httpbis-http3-secondary-certs.html"

author:
 -
    fullname: Eric Gorbaty
    organization: Apple
    email: "e_gorbaty@apple.com"

normative:
  RFC5246:
  RFC7230:
  RFC7540:
  RFC8446:
  RFC9261:

informative:
  RFC7838:

--- abstract

A use of TLS Exported Authenticators is described which enables HTTP/3 servers to offer additional certificate-based
credentials after the connection is established. The means by which these credentials are used with requests is defined.

--- middle

# Introduction

HTTP clients need to know that the content they receive on a connection comes
from the origin that they intended to retrieve it from. The traditional form of
server authentication in HTTP has been in the form of a single X.509 certificate
provided during the TLS ([RFC5246], [RFC8446]) handshake.

Many existing HTTP [RFC7230] servers also have authentication requirements for
the resources they serve.  Of the bountiful authentication options available for
authenticating HTTP requests, client certificates present a unique challenge for
resource-specific authentication requirements because of the interaction with
the underlying TLS layer.

TLS 1.2 [RFC5246] supports one server and one client certificate on a
connection. These certificates may contain multiple identities, but only one
certificate may be provided.

Many HTTP servers host content from several origins. HTTP/3 permits clients to
reuse an existing HTTP connection to a server provided that the secondary origin
is also in the certificate provided during the TLS handshake.  In many cases,
servers choose to maintain separate certificates for different origins but
still desire the benefits of a shared HTTP connection.

TODO: Adapt properly for HTTP3 and QUIC, mentioning further benefits of shared HTTP connection over HTTP/3

TODO: Explain the use? Or somewhere else?

## Server Certificate Authentication

Section 9.1.1 of [RFC7540] describes how connections may be used to make
requests from multiple origins as long as the server is authoritative for both.
A server is considered authoritative for an origin if DNS resolves the origin to
the IP address of the server and (for TLS) if the certificate presented by the
server contains the origin in the Subject Alternative Names field.

[RFC7838] enables a step of abstraction from the DNS resolution. If both hosts
have provided an Alternative Service at hostnames which resolve to the IP
address of the server, they are considered authoritative just as if DNS resolved
the origin itself to that address. However, the server's one TLS certificate is
still required to contain the name of each origin in question.

{{?RFC8336}} relaxes the requirement to perform the DNS lookup if already
connected to a server with an appropriate certificate which claims support for a
particular origin.

Servers which host many origins often would prefer to have separate certificates
for some sets of origins. This may be for ease of certificate management (the
ability to separately revoke or renew them), due to different sources of
certificates (a CDN acting on behalf of multiple origins), or other factors
which might drive this administrative decision. Clients connecting to such
origins cannot currently reuse connections, even if both client and server would
prefer to do so.

Because the TLS SNI extension is exchanged in the clear, clients might also
prefer to retrieve certificates inside the encrypted context. When this
information is sensitive, it might be advantageous to request a general-purpose
certificate or anonymous ciphersuite at the TLS layer, while acquiring the
"real" certificate in HTTP after the connection is established.

## TLS Exported Authenticators

TLS Exported Authenticators [RFC9261] are structured messages that can be exported by
either party of a TLS connection and validated by the other party. Given an
established TLS connection, an authenticator message can be constructed proving
possession of a certificate and a corresponding private key. The mechanisms that this draft defines
are primarily focused on the servers ability to generate TLS Exported Authenticators.

Each Authenticator is computed using a Handshake Context and Finished MAC Key
derived from the TLS session.  The Handshake Context is identical for both
parties of the TLS connection, while the Finished MAC Key is dependent on
whether the Authenticator is created by the client or the server.

Successfully verified Authenticators result in certificate chains, with verified
possession of the corresponding private key, which can be supplied into a
collection of available certificates. Likewise, descriptions of desired
certificates can be supplied into these collections.


## HTTP-Layer Certificate Authentication

This draft defines HTTP/3 frames to carry the relevant certificate messages,
enabling certificate-based authentication of servers independent of TLS version. This mechanism can be implemented at
the HTTP layer without breaking the existing interface between HTTP and applications above it.

TLS Exported Authenticators [RFC9261] allow the opportunity for an HTTP/3 server to send a certificate frame
on the control stream which can be used to prove the servers authenticity for multiple origins, and not require
a secondary TLS negotiation. TODO: For a proxy??

To indicate support for HTTP-Layer certificate authentication, both the client
and server have to send the `SETTINGS_HTTP_SERVER_CERT_AUTH`setting in order to
indicate that they both support handling of CERTIFICATE frames,
as well as making requests/responses for authenticated origins in those frames.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Discovering Additional Certificates at the HTTP/3 Layer {#discovery}

A certificate chain with proof of possession of the private key corresponding to
the end-entity certificate is sent as a sequence of `CERTIFICATE` frames (see
{{http-cert}}) on the control stream to the client. Once the holder of a certificate has sent the
chain and proof, this certificate chain is cached by the recipient and available
for future use.

TODO: Possibility of using headers to allow clients to indicate interest in using a particular certificate ID?

## Indicating Support for HTTP-Layer Certificate Authentication {#setting}

In order to indicate support for HTTP-Layer certificate authentication, both the client and the server MUST send
a SETTINGS_HTTP_SERVER_CERT_AUTH value set to "1" in their SETTINGS frame. Endpoints MUST
NOT use any of the authentication functionality described in this draft unless the
parameter has been negotiated.

`SETTINGS_HTTP_SERVER_CERT_AUTH` indicates that servers are able to offer additional
certificates to demonstrate control over other origin hostnames, and that clients
are able to make requests for hostnames received in a TLS Exported Authenticator
that the server sends.

## Making Certificates or Requests Available {#cert-available}

When both peers have advertised support for HTTP-layer certificates in a given
direction as in {{setting}}, the indicated endpoint can supply additional
certificates into the connection at any time.  That is, if both endpoints have
sent `SETTINGS_HTTP_SERVER_CERT_AUTH` and validated the value received from the
peer, the server may send certificates unprompted, at any time.

TODO: For 0-RTT, might it be the case that servers can send certs unprompted as
long as the client has advertised support? OK to send unknown frame type and discard?

Certificates supplied by servers can be considered by clients without further
action by the server. A server SHOULD NOT send certificates which do not cover
origins which it is prepared to service on the current connection, and should NOT send
them if the client has not indicated support.

TODO - Involve CONNECT?

~~~ drawing
Client                                      Server
   <-------------- (control stream) CERTIFICATE --
   ...
   -- (stream N) GET /from-new-origin --------->
   <----------------------- (stream N) 200 OK --

~~~
{: #ex-http3-server-proactive title="Proactive server authentication"}

### Requiring Additional Server Certificates

TODO - Should we find an answer for this, or all certs are unprompted?? CONNECT?

# Certificates Frames for HTTP/3 {#certs-http3}

## CERTIFICATE {#http-cert}

The CERTIFICATE frame (type=0xTBD) provides an exported authenticator
message from the TLS layer that provides a chain of certificates, associated
extensions and proves possession of the private key corresponding to the
end-entity certificate.

A server sends a CERTIFICATE frame on the control stream. The client is permitted
to make subsequent requests for resources upon receipt of a CERTIFICATE frame without
further action from the server.

Upon receiving a complete series of CERTIFICATE frames, the receiver may
validate the Exported Authenticator value by using the exported authenticator
API. This returns either an error indicating that the message was invalid, or
the certificate chain and extensions used to create the message.

The CERTIFICATE frame MUST be sent on stream zero.  A CERTIFICATE frame
received on any other stream MUST not be used for server authentication.

~~~~~~~~~~ ascii-art
CERTIFICATE Frame {
  Type (i) = 0xTBD,
  Length (i),
  Cert-ID (i), ???
  Authenticator (...),
}
~~~~~~~~~~
{: title="CERTIFICATE Frame"}

The authenticator field is a portion of the opaque data returned from the TLS connection exported
authenticator authenticate API. See {{exp-auth}} for more details on the
input to this API.

TODO: Do anything with this?
The Cert-ID field identifies the certificate...
The server MAY include a certificate ID to identify the certificate that they have sent.
The client MAY include the certificate ID that they are using in subsequent requests by....?

TODO: Authenticator fragments? Or all authenticators come in one frame?

### Exported Authenticator Characteristics {#exp-auth}

The Exported Authenticator API defined in [RFC9261]
takes as input a request, a set of certificates, and supporting information
about the certificate (OCSP, SCT, etc.).  The result is an opaque token which is
used when generating the `CERTIFICATE` frame.

Upon receipt of a `CERTIFICATE` frame, an endpoint MUST perform the following
steps to validate the token it contains:

- TODO: Verify that certificates should be received
- TODO: Validate that the API is the same. Request-ID probably not needed.
- Using the `get context` API, retrieve the `certificate_request_context` used
  to generate the authenticator, if any.  Verify that the `certificate_request_context`
  begins with the supplied Request-ID (?), if any.
- Use the `validate` API to confirm the validity of the authenticator with
  regard to the generated request (if any).

If the authenticator cannot be validated, this SHOULD be treated as a connection
error of type `CERTIFICATE_UNREADABLE`.

Once the authenticator is accepted, the endpoint can perform any other checks
for the acceptability of the certificate itself.  Clients MUST NOT accept any
end-entity certificate from an exported authenticator which does not contain
the Required Domain extension(TODO??); see {{extension}} and {{impersonation}}.

# TODO: Indicating Failures During HTTP-Layer Certificate Authentication {#errors}

Because this draft permits certificates to be exchanged at the HTTP framing
layer instead of the TLS layer, several certificate-related errors which are
defined at the TLS layer might now occur at the HTTP framing layer.

There are two classes of errors which might be encountered, and they are handled
differently.

## TODO: Define any errors? Misbehavior

This category of errors could indicate a peer failing to follow restrictions in
this document, or might indicate that the connection is not fully secure.  These
errors are fatal to stream or connection, as appropriate.

CERTIFICATE_UNREADABLE (0xERROR-TBD3):
: An exported authenticator could not be validated.

## Invalid Certificates
TODO: Validates the existence of a cert-ID slightly.

Unacceptable certificates (expired, revoked, or insufficient to satisfy the
request) are not treated as stream or connection errors.  This is typically not
an indication of a protocol failure.  Servers SHOULD process requests with the
indicated certificate, likely resulting in a "4XX"-series status code in the
response. Clients SHOULD establish a new connection in an attempt to reach an
authoritative server.

# Required Domain Certificate Extension TODO: Do we care about this? {#extension}

The Required Domain extension allows certificates to limit their use with
Secondary Certificate Authentication.  A client MUST verify that the server has
proven ownership of the indicated identity before accepting the limited
certificate over Secondary Certificate Authentication.

The identity in this extension is a restriction asserted by the requester of the
certificate and is not verified by the CA.  Conforming CAs SHOULD mark the
requiredDomain extension as non-critical.  Conforming CAs MUST require the
presence of a CAA record {{!RFC6844}} prior to issuing a certificate with this
extension.  Because a Required Domain value of "*" has a much higher risk of
reuse if compromised, conforming Certificate Authorities are encouraged to
require more extensive verification prior to issuing such a certificate.

The required domain is represented as a GeneralName, as specified in Section
4.2.1.6 of {{!RFC5280}}. Unlike the subject field, conforming CAs MUST NOT issue
certificates with a requiredDomain extension containing empty GeneralName
fields.  Clients that encounter such a certificate when processing a
certification path MUST consider the certificate invalid.

The wildcard character "*" MAY be used to represent that any previously
authenticated identity is acceptable.  This character MUST be the entirety of
the name if used and MUST have a type of "dNSName".  (That is, "*" is
acceptable, but "*.com" and "w*.example.com" are not).

    id-ce-requiredDomain OBJECT IDENTIFIER ::=  { id-ce TBD1 }

    RequiredDomain ::= GeneralName

# Security Considerations {#security}

This mechanism defines an alternate way to obtain server and client certificates
other than in the initial TLS handshake. While the signature of exported
authenticator values is expected to be equally secure, it is important to
recognize that a vulnerability in this code path is at least equal to a
vulnerability in the TLS handshake.

## Impersonation

This mechanism could increase the impact of a key compromise. Rather than
needing to subvert DNS or IP routing in order to use a compromised certificate,
a malicious server now only needs a client to connect to *some* HTTPS site under
its control in order to present the compromised certificate. Clients SHOULD
consult DNS for hostnames presented in secondary certificates if they would have
done so for the same hostname if it were present in the primary certificate.

As recommended in {{?RFC8336}}, clients opting not to consult DNS ought to
employ some alternative means to increase confidence that the certificate is
legitimate.

One such means is the Required Domain certificate extension defined in
{extension}. Clients MUST require that server certificates presented via this
mechanism contain the Required Domain extension and require that a certificate
previously accepted on the connection (including the certificate presented in
TLS) lists the Required Domain in the Subject field or the Subject Alternative
Name extension.

As noted in the Security Considerations of
[RFC9261], it is difficult to formally prove that an
endpoint is jointly authoritative over multiple certificates, rather than
individually authoritative on each certificate.  As a result, clients MUST NOT
assume that because one origin was previously colocated with another, those
origins will be reachable via the same endpoints in the future.  Clients MUST
NOT consider previous secondary certificates to be validated after TLS session
resumption.  However, clients MAY proactively query for previously-presented
secondary certificates.

## Fingerprinting

This draft defines a mechanism which could be used to probe servers for origins
they support, but opens no new attack versus making repeat TLS connections with
different SNI values.

Servers can also learn information about clients using this mechanism. The
hostnames a user agent finds interesting and retrieves certificates for might
indicate origins the user has previously accessed.

## Persistence of Service

CNAME records in the DNS are frequently used to delegate authority for an origin
to a third-party provider.  This delegation can be changed without notice, even
to the third-party provider, simply by modifying the CNAME record in question.

After the owner of the domain has redirected traffic elsewhere by changing the
CNAME, new connections will not arrive for that origin, but connections which
are properly directed to this provider for other origins would continue to claim
control of this origin (via Secondary Certificates).  This is
proper behavior based on the third-party provider's configuration, but would
likely not be what is intended by the owner of the origin.

This is not an issue which can be mitigated by the protocol, but something about
which third-party providers SHOULD educate their customers before using the
features described in this document.

## Confusion About State

TODO: Valid? There may surely be confusions about state here. Not sure if the old
wording is a good way to put this.

Implementations need to be aware of the potential for confusion about the state
of a connection. The presence or absence of a validated certificate can change
during the processing of a request, potentially multiple times, as
`CERTIFICATE` frames are received. A client that uses certificate
authentication needs to be prepared to reevaluate the authorization state of a
request as the set of certificates changes.


# IANA Considerations

TODO IANA Considerations


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
