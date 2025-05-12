---
title: "Workload Identifier Hint for TLS ClientHello"
abbrev: "Workload Identifier Hint for TLS ClientHello"
category: std

docname: draft-rosomakho-tls-wimse-cert-hint-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: ""
workgroup: "Transport Layer Security"
keyword:
 - tls
 - wimse
 - identifier
 - workload
 - hint
venue:
  group: "Transport Layer Security"
  type: ""
  mail: "tls@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/tls/"
  github: "yaroslavros/tls-wimse-cert-hint"
  latest: "https://yaroslavros.github.io/tls-wimse-cert-hint/draft-rosomakho-tls-wimse-cert-hint.html"

author:
 -
      ins: "Y. Rosomakho"
      fullname: Yaroslav Rosomakho
      organization: Zscaler
      email: yrosomakho@zscaler.com

normative:

informative:


--- abstract

This document defines a TLS extension that allows clients to indicate one or more workload identifiers in the ClientHello message. These identifiers serve as identity hints, enabling the server to determine whether client authentication is required and to select appropriate policy or certificate validation logic before completing the handshake. This mechanism supports more efficient policy enforcement in mutual TLS deployments and can be used to optimise authentication decisions based on the declared identity context of the client. To protect the confidentiality of workload identifiers, this extension can be used in combination with Encrypted Client Hello (ECH).


--- middle

# Introduction

Mutual TLS (mTLS) is commonly used to authenticate both endpoints of a {{!TLS=RFC8446}} connection, especially in service-to-service communication within distributed systems. In many deployments, client authentication is conditional: only certain clients are required to present a certificate, and the decision is based on the nature of the client, its identity or role.

This document defines a TLS extension that allows clients to indicate one or more Workload Identifiers [Reference TBD] in the `ClientHello` message ({{Section 4.1.2 of TLS}}). These identifiers act as identity hints that inform the server of the client intended identity before the TLS handshake is completed. Based on this information, the server can determine whether client certificate authentication is desirable and, if so, what policy or certificate validation rules should apply.

This approach enables more flexible and efficient authentication strategies in environments where different clients may be subject to different requirements. For example:

- A server may enforce mTLS only for specific workloads and allow others to connect without client certificate authentication.
- A server may use the provided identifiers to generate an appropriate list of Certificate Authorities extension ({{Section 4.2.4 of TLS}}) in `CertificateRequest` message ({{Section 4.3.2 of TLS}}).
- The server may reject the connection early if none of the advertised identifiers are authorised.

Workload Identifiers are conveyed as a list of URIs, allowing clients to indicate multiple possible identities. The interpretation and validation of these identifiers is implementation-specific and may be governed by local policy or external trust frameworks.

In cases where the Workload Identifiers are sensitive or may reveal internal deployment details, clients are encouraged to include this extension only in `ClientHelloInner` of Encrypted Client Hello ({{!ECH=I-D.ietf-tls-esni}}) to ensure confidentiality of the identity hints.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# TLS Extension Format

This document defines a new TLS extension named `workload_identifier_hint`, which is carried in the `ClientHello` message. The extension provides the server with one or more workload identifiers that the client associates with itself. This allows the server to evaluate authentication requirements prior to sending a `CertificateRequest` message.

The `workload_identifier_hint` extension is structured as follows:

~~~
   opaque WorkloadIdentifier<1..2^16-1>;

   struct {
       WorkloadIdentifier identifiers<3..2^16-1>;
   } WorkloadIdentifierHintExtension;
~~~

`identifiers`:

: A list of Workload Identifiers as defined in TBD. Each `WorkloadIdentifier` is an absolute URI encoded in UTF-8, as defined in {{!URI=RFC3986}}.

Clients MAY include multiple identifiers if they represent multiple possible roles or fallback identities.

The extension MUST appear only in the ClientHello. Servers MUST abort TLS handshake with an `unexpected_message` alert if this extension appears in any other handshake message. Similarly, clients MUST abort TLS handshake if this extension appears in any message from the server.

## Server Processing Rules

Upon receiving the extension, the server:

- MAY use the identifiers to determine whether to send a `CertificateRequest` message.
- MAY use the identifiers to construct Certificate Authorities extension in the `CertificateRequest` message.
- MAY use the identifiers to select a trust anchor or policy.
- MAY reject the handshake early with `handshake_failure` alert if none of the identifiers are acceptable.
- MUST NOT treat inclusion of the extension as proof of identity. The identifiers are advisory and unauthenticated until verified during client authentication.

If the extension is absent, the server proceeds with the default client authentication behavior.

# Security Considerations

This extension is intended to improve the flexibility of client authentication policies in TLS. However, because it introduces unauthenticated identity hints early in the handshake, several security considerations apply.

## Confidentiality of Workload Identifiers

Workload Identifiers may contain sensitive information, such as internal service names, deployment structure, or tenant-specific data. Since this extension is sent in the clear as part of the ClientHello, exposure of these identifiers may allow passive observers to infer client roles, access patterns, or security posture.

To mitigate this risk, clients SHOULD include this extension only in ClinetHelloInner if {{ECH}} is available. ECH encrypts the ClinetHelloInner and its extensions under the server's public key, preventing visibility of the identifiers to on-path observers.

If ECH is not in use, clients SHOULD avoid including sensitive or detailed identifiers in this extension unless required by policy.

## Unauthenticated Hints

The workload identifiers conveyed in this extension are not authenticated. They are advisory in nature and MUST NOT be treated by the server as a proof of identity. Servers MUST perform full cryptographic verification of the client certificate before relying on any identity claim.

Servers MAY enforce policies based on the presence or absence of expected identifiers in the `ClientHello`. However, this enforcement must be restricted to access control decisions prior to authentication, such astriggering client authentication or rejecting the handshake.

## Identifier Enumeration

If ECH is not deployed, an attacker with network visibility may collect workload identifiers by observing repeated TLS handshakes. This could aid in reconnaissance or allow inference of infrastructure details. To reduce this risk, clients may:

- Use generic or opaque identifiers when full disclosure is not required.
- Limit use of the extension to trusted networks or peers.
- Use ECH to encrypt the extension contents.

## Server Response Behaviour

Servers receiving unknown or malformed identifiers SHOULD ignore them and proceed with the default authentication policy. Servers SHOULD NOT terminate connections solely due to unrecognised identifiers unless explicitly configured to do so.

# IANA Considerations

IANA is requested to assign a new value from the TLS ExtensionType Values registry:

- The Extension Name should be workload_identifier_hint
- The TLS 1.3 value should be CH
- The DTLS-Only value should be N
- The Recommended value should be Y
- The Reference should be this document


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
