---
title: "Privacy Pass: Architectural Framework"
abbrev: PP architecture
docname: draft-davidson-pp-architecture-latest
date:
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: A. Davidson
    name: Alex Davidson
    org: Cloudflare Portugal
    street: Largo Rafael Bordalo Pinheiro 29
    city: Lisbon
    country: Portugal
    email: alex.davidson92@gmail.com

normative:
  RFC2119:
  RFC8446:
  draft-davidson-pp-protocol:
    title: "Privacy Pass: The Protocol"
    target: https://tools.ietf.org/html/draft-davidson-pp-protocol-00
    author:
      ins: A. Davidson
      org: Cloudflare Portugal
  draft-svaldez-pp-http-api:
    title: "Privacy Pass: HTTP API"
    target: https://github.com/alxdavids/privacy-pass-ietf/tree/master/drafts/draft-svaldez-pp-http-api
    author:
      ins: S. Valdez
      org: Google LLC
informative:
  TrustTokenAPI:
    title: Getting started with Trust Tokens
    target: https://web.dev/trust-tokens/
    author:
      name: Google
  PrivateStorage:
    title: The Path from S4 to PrivateStorage
    target: https://medium.com/least-authority/the-path-from-s4-to-privatestorage-ae9d4a10b2ae
    author:
      name: Liz Steininger
      ins: L. Steininger
      org: Least Authority
  OpenPrivacy:
    title: Token Based Services - Differences from PrivacyPass
    target: https://openprivacy.ca/assets/towards-anonymous-prepaid-services.pdf
    authors:
      -
        ins: E. Atwater
        org: OpenPrivacy, Canada
      -
        ins: S. J. Lewis
        org: OpenPrivacy, Canada
  Brave:
    title: Brave Rewards
    target: https://brave.com/brave-rewards/
  PPEXT:
    title: Privacy Pass Browser Extension
    target: https://github.com/privacypass/challenge-bypass-extension
  PPSRV:
    title: Cloudflare Supports Privacy Pass
    target: https://blog.cloudflare.com/cloudflare-supports-privacy-pass/
    author:
      ins: N. Sullivan
      org: Cloudflare

--- abstract

This document specifies the architectural framework for constructing
secure and anonymity-preserving instantiations of the Privacy Pass
protocol. It provides recommendations on how the protocol ecosystem
should be constructed to ensure the privacy of clients, and the security
of all participating entities.

--- middle

# Introduction

The Privacy Pass protocol provides an anonymity-preserving mechanism for
authorization of clients with servers. The protocol is detailed in
{{draft-davidson-pp-protocol}} and is intended for use in the
application-layer.

The way that the ecosystem around the protocol is set up can have
significant impacts on the stated privacy and security guarantees of the
protocol. For instance, the number of servers issuing Privacy Pass
tokens, along with the number of registered clients, determines the
anonymity set of each individual client. Moreover, this can be
influenced by other factors, such as: the key rotation policy used by
each server; and, the number of supported ciphersuites. There are also
client behavior patterns that can reduce the effective security of the
server.

In this document, we will provide a structural framework for building
the ecosystem around the Privacy Pass protocol. The core of the document
also includes policies for the following considerations.

- How server key material should be managed and accessed.
- Compatible server issuance and redemption running modes and associated
  expectations.
- How clients should evaluate Issuer trust relationships.
- Security and privacy properties of the protocol.
- A concrete assessment and parametrization of the privacy budget
  associated with different settings of the above policies.
- The incorporation of potential extensions into the wider ecosystem.

Finally, we will discuss existing applications that make use of the
Privacy Pass protocol, and highlight how these may fit with the proposed
framework.

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{RFC2119}}.

The following terms are used throughout this document.

- Server: An entity that issues anonymous tokens to clients. In
  symmetric verification cases, the Server must also verify tokens. Also
  referred to as the Issuer.
- Client: An entity that seeks authorization from a Server.

We assume that all protocol messages are encoded into raw byte format
before being sent. We use the TLS presentation language {{RFC8446}} to
describe the structure of the data that is communicated and stored.

# Ecosystem participants {#ecosystem}

The Privacy Pass ecosystem refers to the global framework in which
multiple instances of the Privacy Pass protocol operate. This refers to
all servers that support the protocol, or any extension of it, along
with all of the Clients that may interact with these Servers.

The ecosystem itself, and the way it is constructed, is critical for
evaluating the privacy of each individual Client. We assume that a
Client's privacy refers to fraction of users that it represents in the
anonymity set that it belongs to. We discuss this more in {{privacy}}.

~~~
    +-----------------------------------------------------+
    |                                                     |
    | Ecosystem                                           |
    |                                                     |
    |                                                     |
    |                                                     |
    |      +-----+                                        |
    |      |     |                        +-----+         |
    |      | C1  | <--------------------> |     |         |
    |      |     |                        | S1  |         |
    |      +-----+        +-------------> |     |         |
    |                     |               +-----+         |
    |                     |                               |
    |      +-----+        |                               |
    |      |     | <------+                               |
    |      | C2  |                                        |
    |      |     | <------+               +-----+         |
    |      +-----+        +-------------> |     |         |
    |                                     | S2  |         |
    |                     +-------------> |     |         |
    |      +-----+        |               +-----+         |
    |      |     |        |                               |
    |      | C3  | <------+                               |
    |      |     |                                        |
    |      +-----+                                        |
    |                                                     |
    +-----------------------------------------------------+
~~~

In the above diagram, the arrows indicate the open channels between a
Client and a Server. An open channel indicates that a Client accepts
Privacy Pass tokens from this Server.

If no channel exists, this means that the Client
chooses not to accept tokens from (or redeem tokens with) that
particular Server. We discuss the roles of Servers and Clients further
in {{ecosystem-servers}} and {{ecosystem-clients}}, respectively.

## Servers {#ecosystem-servers}

Generally, Servers in the Privacy Pass ecosystem are entities whose
primary function is to undertake the role of the `Issuer` in
{{draft-davidson-pp-protocol}}. To facilitate this, the Issuer MUST hold
a Privacy Pass protocol keypair at any given time. The Issuer public key
MUST be made available to all Clients in such a way that key rotations
and other updates are publicly visible. The Issuer MAY also require
additional state for ensuring this. We provide a wider discussion in
{{key-mgmt}}.

Note that, in the core protocol instantiation from
{{draft-davidson-pp-protocol}}, the redemption phase is a symmetric
protocol. This means that the Issuer is the same Server that ultimately
processes token redemptions from Clients. However, plausible extensions
to the protocol specification may allow public verification of tokens by
entities which do not hold the secret Privacy Pass keying material. We
highlight possible Client and Server configurations in
{{running-modes}}.

The Server must be uniquely identifiable by all Clients with a
consistent identifier.

## Clients {#ecosystem-clients}

Clients in the Privacy Pass ecosystem are entities whose primary
function is to undertake the role of the `Client` in
{{draft-davidson-pp-protocol}}. Clients are assumed to only store data
related to the tokens that it has been issued by the Server. This
storage is used for constructing redemption requests.

Clients MAY choose not to accept tokens from Servers that they do not
trust. See {{client-issuer}} for a wider discussion.

### Client identifying information {#client-ip}

Privacy properties of this protocol do not take into account other
possibly identifying information available in an implementation, such as
a client's IP address. Servers which monitor IP addresses may use this
to track client redemption patterns over time. Clients cannot check
whether servers monitor such identifying information. Thus, clients
SHOULD minimize or remove identifying information where possible, e.g.,
by using anonymity-preserving tools such as Tor to interact with
Servers.

# Key management framework {#key-mgmt}

The key material and protocol configuration that a Server uses to issue
tokens corresponds to a number of different pieces of information.

- The ciphersuite that the Server is using.
- The public keys that are active for the Server.

For reasons that we address later in {{privacy}}, the way that the
Server publishes and maintains this information impacts the effective
privacy of the clients. In this section, we describe the main policies
that need to be satisfied for a key management system in a Privacy Pass
ecosystem.

Note that we only specify a set of guidelines and recommendations for
operating a public key registry in this document. Actual specification
of such a registry and how it operates will be covered elsewhere.

## Public key registries

Issuer's must provide their public keys to clients along with details
about the cryptographic ciphersuite that they are using. In {{privacy}},
we address the importance of providing Clients with sources of truth for
learning the Server's key configuration.

In particular, Server key material MUST be publicly available in a
tamper-proof data structure, which we refer to as a key registry. A
registry must be globally consistent. Clients using the same registry
should coordinate in some way to ensure they have a consistent view of
said registry. This can be done via gossiping or some other mechanism.
The exact mechanism for this coordination will be described elsewhere.
It is assumed there will be at least one such mechanism.

It is RECOMMENDED that each key registry is an append-only data
structure, such as a Merkle Tree. The key registry should be operated
independently of any Issuer that publishes key material to the registry.
This ensures that any Client can make better judgements on whether to
trust the registry and, transitively, each Server.

~~~
+--------------------------------------------------------+
|                                                        |
| Ecosystem                            +---+             |
|                                      | C |             |
|  +--------------+ <------ pkS1 ----> +---+             |
|  |  Registry 1  |                                      |
|  ++-------------+ <-------------- pkS1 --------> +---+ |
|   |                                              | C | |
|   |   +--------------+ <--------- pkS3 --------> +---+ |
|   |   |  Registry 2  |                                 |
|  pkS1 +----^-------^-+ <--------- pkS2 --------> +---+ |
|   |        |       |                             | C | |
|   |       pkS2    pkS3                           +---+ |
|   |        |       |                                   |
|  ++---+  +-+--+  +-+--+                                |
|  | S1 |  | S2 |  | S3 |                                |
|  +----+  +----+  +----+                                |
|                                                        |
+--------------------------------------------------------+
~~~

While there may be multiple key registries for a given ecosystem, a
Server MUST only publish its key material to a single registry. This
ensures that the Server is keeping a consistent view of its key
material.

## Key rotation

Token issuance associates all issued tokens with a particular choice of
key. If a Server issues tokens with many keys, then this may harm the
anonymity of the Client. For example, they would be able to map the
Client's access patterns by inspecting which key each token they possess
has been issued under.

To prevent against this, Servers MUST only use one private key for
issuing tokens at any given time. Servers may use two or more keys for
redemption to allow Servers for seamless key rotation.

Key rotations must be limited in frequency for similar reasons. See
{{parametrization}} for guidelines on what frequency of key rotations
are permitted.

## Ciphersuites

Since a Server is only permitted to have a single active issuing key,
this implies that only a single ciphersuite is allowed per issuance
period. If a Server wishes to change their ciphersuite, they MUST do so
during a key rotation.

# Server running modes {#running-modes}

We provide an overview of some of the possible frameworks for
configuring the way that Servers run in the Privacy Pass ecosystem. In
short, Servers may be configured to provide symmetric issuance and
redemption with clients. While some Servers may be configured as proxies
that accept Privacy Pass data and send it to another Server that
actually processes issuance or redemption data. Finally, we also
consider instances of the protocol that may permit public verification.

The intention with providing each of these running modes is to cover the
different applications that utilize variants of the Privacy Pass
protocol. We RECOMMEND that any Privacy Pass Server implementation
adheres to one of these frameworks.

## Single-Issuer Single-Verifier {#sisv}

The simplest way of considering the Privacy Pass protocol is in a
setting where the same server plays the role of issuer and verifier, we
call this "Single-Issuer Single-Verifier" (SISV).

When S wants to issue tokens to C, they invoke the issuance protocol
where C generates their own inputs, and S uses their secret key skS. In
this setting, C can only perform token redemption with S. When a token
redemption is required, C and S invoke the redemption phase of the
protocol, where C uses an issued token from a previous exchange, and S
uses skS to validate the redemption.

## Single-Issuer Delegated-Verifier {#sidv}

In this setting, each Client C obtains issued tokens from a Server S via
the issuance phase of the protocol. The difference is that C can prove
that they hold a valid authorization with any verifier V. We still only
consider S to hold their own secret key. We name this mode
"Single-Issuer Delegated-Verifier" (SIDV).

When C interacts with V, V can ask C to provide proof of authorization
to the separate issuer S. The first stage of the redemption phase of the
protocol is invoked between C and V, which sees C send an unused
redemption token to V. This message is then used in a redemption
exchange between V and S, where V plays the role of the Client. Then S
sends the result of the redemption verification to V, and V uses this
result to determine whether C has a valid token.

## Single-Issuer Asynchronous-Verifier {#siav}

This setting is inspired by recently proposed APIs such as
{{TrustTokenAPI}}. It is similar to the SIDV configuration, except that
the verifiers V no longer interact with the issuer S. Only C interacts
with S, and this is done asynchronously to the authorization request
from V. Hence "Asynchronous-Verifier" (SIAV).

When V invokes a redemption for C, C then invokes a redemption exchange
with S in a separate session. If verification is carried out
successfully by S, S instead returns a Signed Redemption Record (SRR)
that contains the following information:

~~~ json
"result": {
  "timestamp":"2019-10-09-11:06:11",
  "verifier": "V",
},
"signature":sig,
~~~

The `signature` field carries a signature evaluated over the contents of
`result` using a long-term signing key for the issuer S, of which the
corresponding public key is well-known to C and V. This would need to be
published alongside other public key data for S. Then C can prove that
they hold a valid authorization from S to V by sending the SRR to V. The
SRR can be verified by V by verifying the signature, using the
well-known public key for S.

Such records can be cached to display again in the future. The issuer
can also add an expiry date to the record to determine when the client
must refresh the record.

## Single-Issuer Public-Verifier {#sipv}

We consider the case where Client redemptions can be verified publicly
using the Issuer public key. This allows for defining extensions of
Privacy Pass that use public-key cryptography to allow public
verification.

In this case, the Client C obtains a redemption token from S. The
redemption token is publicly verifiable in the sense that any entity
that knows the public key for S can verify the token. This running mode
is known as "Single-Issuer Public-Verifier" (SIPV).

## Bounded-Issuers {#bi-mode}

Each of the configurations above can be generalized to settings where a
bounded number of issuers are allowed, and verifiers can invoke
authorization verification for any of the available issuers.
Subsequently, this leads to three new configurations known as BISV,
BIFV, BIAV and BIPV.

As we will discuss later in {{privacy}}, configuring a large number of
issuers can lead to privacy concerns for the clients in the ecosystem.
Therefore, we are careful to ensure that the number of issuers is kept
strictly bounded. The actual issuers can be replaced with different
issuers as long as the total never exceeds this bound. Moreover, issuer
replacements also have an effect on client anonymity that is similar to
when a key rotation occurs. Issuer so replacement should only be
permitted at similar intervals.

See {{privacy}} for more details about safe choices of this bound.

# Client-Issuer trust relationship {#client-issuer}

It is important, based on the architecture above, that any Client can
determine whether it would like to interact with a given Issuer in the
ecosystem. Note that this decision must be taken before a Client issues
a valid redemption to the Server, since redemptions reveal the anonymity
set that the Client belongs to.

This judgement can be based on a multitude of factors, associated with
the way that a Server presents itself in the ecosystem. A non-exhaustive
list of Server characteristics that a Client MAY want to check are the
following.

- Which key registry a Server posts their key updates to.
- How frequent key updates are issued, and which ciphersuite they use.
- The reason given to initiate the redemption.

To aid Client trust decisions, a Server can publish a "Privacy Pass
policy" that documents the procedures that the Server uses to ensure
that Client privacy is respected. If a Server does not publish such a
document then the Client may choose to use its own judgement, or to
reject the Server altogether.

It should be noted that the Client trust decision can be made apriori by
specifying an allow-list of all Issuers that it accepts tokens from.
This means that these checks do not have to be performed online.

# Privacy considerations {#privacy}

In the Privacy Pass protocol {{draft-davidson-pp-protocol}}, redemption
tokens intentionally encode very little information beyond which key was
used to sign them. The protocol intentionally uses components that
provide cryptographic guarantees of this fact. However, even with these
guarantees, the way that the ecosystem is constructed can be used to
identify clients based on this limited information.

The goal of the Privacy Pass ecosystem is to construct an environment
that can easily measure (and maximize) the relative anonymity of any
client that is part of it. An inherent feature of being part of this
ecosystem is that any client can only remain private relative to the
entire space of users using the protocol. Moreover, by owning tokens for
a given set of keys, the client's anonymity set shrinks to the total
number of clients controlling tokens for the same keys.

In the following, we consider the possible ways that Servers and Issuers
can leverage their position to try and reduce the anonymity sets that
Clients belong to (or, user segregation). For each case, we provide
mitigations that the Privacy Pass ecosystem must implement to prevent
these actions.

## Server key rotation

Techniques to introduce Client "segregation" can be used to reduce
Client anonymity. Such techniques are closely linked to the type of key
schedule that is used by the server. When a server rotates their key,
any client that invokes the issuance protocol in this key cycle will be
part of a group of possible Clients owning valid tokens for this key. To
mechanize this attack strategy, a Server could introduce a key rotation
policy that forces Clients into small key cycles. Thus, reducing the
size of the anonymity set for these Clients.

We RECOMMEND that Servers should only invoke key rotation for fairly
large periods of time such as between 1 and 12 weeks. Key rotations
represent a trade-off between Client privacy and continued server
security. Therefore, it is still important that key rotations occur on a
fairly regular cycle to reduce the harmfulness of a Server key
compromise.

## Large numbers of Issuers {#issuers}

Similarly to the Issuer rotation dynamic that is raised above, if there
are a large number of Issuers then segregation can occur. In the BIFV,
BIAV and BIPV running modes ({{running-modes}}), a verifier OV can
trigger redemptions for any of the available Issuers. Each redemption
token that a client holds essentially corresponds to a bit of
information about the client that OV can learn. Therefore, there is an
exponential loss in anonymity relative to the number of Issuers that
there are.

For example, if there are 32 Issuers, then OV learns 32 bits of
information about the client. If the distribution of issuer trust is
anything close to a uniform distribution, then this is likely to
uniquely identify any client amongst all other Internet users. Assuming
a uniform distribution is clearly the worst-case scenario, and unlikely
to be accurate, but it provides a stark warning against allowing too
many Issuers at any one time.

In cases where Clients can hold tokens for all Issuers at any given
time, a strict bound SHOULD be applied to the active number of Issuers
in the ecosystem. We propose that allowing no more than 4 Issuers at any
one time is highly preferable (leading to a maximum of 64 possible user
segregations). However, as highlighted in {{parametrization}}, having a
very large user base (> 5 million users), could potentially allow for
larger values. Issuer replacements should only occur with the same
frequency as config rotations as they can lead to similar losses in
anonymity if clients still hold redemption tokens for previously active
Issuers.

In addition, we RECOMMEND that trusted registries indicate at all times
which Issuers are deemed to be active. If a client is asked to invoke
any Privacy Pass exchange for an issuer that is not declared active,
then the client SHOULD refuse to retrieve the server configuration
during the protocol.

### Allowing larger number of Issuers {#more-issuers}

The bounds on the numbers of Issuers that we proposed above are very
restrictive. This is due to the fact that we considered a situation
where a Client could be issued (and forced to redeem) tokens for any
issuing key.

An alternative system is to ensure a robust strategy for ensuring that
Clients only possess redemption tokens for a similarly small number of
Issuers at any one time. This prevents a malicious verifier from being
able to invoke redemptions for many Issuers since the Client would only
be holding redemption tokens for a small set of Issuers. When a Client
is issued tokens from a new issuer and already has tokens from the
maximum number of Issuers, it simply deletes the oldest set of
redemption tokens in storage and then stores the newly acquired tokens.

For example, if Clients ensure that they only hold redemption tokens for
4 Issuers, then this increases the potential size of the anonymity sets
that the Client belongs to. However, this doesn't protect Clients
completely as it would if only 4 Issuers were permitted across the whole
system. For example, these 4 Issuers could be different for each Client.
Therefore, the selection of Issuers they possess tokens for is still
revealing. Understanding this trade-off is important in deciding the
effective anonymity of each Client in the system.

## Partitioning of Issuer key material {#split-view}

If there are multiple key registries, or if a key registry colludes with
an Issuer, then it is possible to provide a split-view of an Issuer's
key material to different Clients. This would involve posting different
key material in different locations, or actively modifying the key
material at a given location.

Key registries should operate independently of Issuer's in the
ecosystem, and within the guidelines stated in {{key-mgmt}}. Any Client
should follow the recommendations in {{client-issuer}} for determining
whether an Issuer and its key material should be trusted.

## Additional token metadata

In {{draft-davidson-pp-protocol}}, it is permissible to add public and
private metadata bits to redemption tokens. The core protocol
instantiation that is described does not include additional metadata.
However, future instantiations may use this functionality to provide
redemption verifiers with additional information about the user.

Note that any arbitrary bits of information can be used to further
segment the size of the user's anonymity set. Any Issuer that wanted to
track a single user could add a single metadata bit to user tokens. For
the tracked user it would set the bit to `1`, and `0` otherwise. Adding
additional bits provides an exponential increase in tracking granularity
similarly to introducing more Issuers (though with more potential
targeting).

For this reason, the amount of metadata used by an Issuer in creating
redemption tokens must be taken into account -- together with the bits
of information that Issuer's may learn about Clients otherwise. We
discuss this more in {{parametrization}}.

## Tracking and identity leakage

Privacy losses may be encountered if too many redemptions are allowed in
a short burst. For instance, in the Internet setting, this may allow
delegated or asynchronous verifiers to learn more information from the
metadata that the Client may hold (such as first-party cookies for other
domains). Mitigations for this issue are similar to those proposed in
{{issuers}} for tackling the problem of having large number of issuers.

In SIAV, cached SRRs and their associated issuer public keys have a
similar tracking potential to first party cookies in the browser
setting. These considerations will be covered in a separate document,
detailing Privacy Pass protocol integration into the wider web
architecture {{draft-svaldez-pp-http-api}}.

## Client incentives for anonymity reduction

Clients may see an incentive in accepting all tokens that are issued by
a server, even if the tokens fail later verification checks. This is
because tokens effectively represent a form of currency that they can
later redeem for some sort of benefit. The verification checks that are
put in place are there to ensure that the Client does not sacrifice
their anonymity. However, a Client may judge the "monetary" benefit of
owning tokens to be greater than their own privacy.

Firstly, a Client behaving in this way would not be compliant with the
protocol, as laid out in {{draft-davidson-pp-protocol}}.

Secondly, acting in this way only affects the privacy of the immediate
Client. There is an exception if a large number of Clients colluded to
accept bad data, then any Client that didn't accept would be part of a
smaller anonymity set. However, such a situation would be identical to
the situation where the total number of Clients in the ecosystem is
small. Therefore, the reduction in the size of the anonymity set would
be equivalent; see {{issuers}} for more details.

# Security considerations {#security}

We present a number of security considerations that prevent malicious
Clients from abusing the protocol.

## Double-spend protection

All issuing server should implement a robust storage-query mechanism for
checking that tokens sent by Clients have not been spent before. Such
tokens only need to be checked for each issuer individually. But all
Issuers must perform global double-spend checks to avoid Clients from
exploiting the possibility of spending tokens more than once against
distributed token checking systems. For the same reason, the global data
storage must have quick update times. While an update is occurring it
may be possible for a malicious Client to spend a token more than once.

## Issuer key rotation

We highlighted previously that short key-cycles can be used to reduce
Client privacy. However, regular key rotations of the issuing key are
still recommended to maintain good server key hygiene.

We recommend that Privacy Pass issuing keys are rotated from anywhere
between 1 and 12 weeks. With an active user-base, a week gives a fairly
large window for Clients to participate in the Privacy Pass protocol and
thus enjoy the anonymity guarantees of being part of a larger group. The
low ceiling of 12 weeks prevents a key compromise from being too
destructive. If a server realizes that a key compromise has occurred
then the server should sample a new key and upload the public key to the
key registry; invoking any revocation procedures that may apply for the
old key.

## Token exhaustion

When a Client holds tokens for an issuer, it is possible for any
verifier to invoke that Client to redeem tokens for that issuer. This
can lead to an attack where a malicious verifier can force a Client to
spend all of their tokens for a given issuer. To prevent this from
happening, methods should be put into place to prevent many tokens from
being redeemed at once.

For example, it may be possible to cache a redemption for the entity
that is invoking a token redemption. If the verifier requests more
tokens then the Client simply returns the cached token that it returned
previously. This could also be handled by simply not redeeming any
tokens for verification if a redemption had already occurred in a given
time window.

In SIAV, the Client instead caches the SRR that it received in the
asynchronous redemption exchange with the issuer. If the same verifier
attempts another redemption request, then the Client simply returns the
cached SRR. The SRRs can be revoked by the issuer, if need be, by
providing an expiry date or by signaling that records from a particular
window need to be refreshed.

## Avoiding Issuer centralization

[[OPEN ISSUE: explain potential and mitigations for Issuer
centralization]]

# Protocol parametrization {#parametrization}

We provide a summary of the parameters that we use in the Privacy Pass
protocol ecosystem. These parameters are informed by both privacy and
security considerations that are highlighted in {{privacy}} and
{{security}}, respectively. These parameters are intended as a single
reference point for those implementing the protocol.

Firstly, let U be the total number of users, I be the total number of
issuers. We let M be the total number of metadata bits that are allowed
to be added by any given issuer. Assuming that each user accept tokens
from a uniform sampling of all the possible issuers, as a worst-case
analysis, this segregates users into a total of 2^I buckets. As such, we
see an exponential reduction in the size of the anonymity set for any
given user. This allows us to specify the privacy constraints of the
protocol below, relative to the setting of A.

| parameter | value |
|---|---|
| Minimum anonymity set size (A) | 5000 |
| Recommended key lifetime (L) | 2 - 24 weeks |
| Recommended key rotation frequency (F) | L/2 |
| Maximum additional metadata bits (M) | 1 |
| Maximum allowed issuers (I) | (log_2(U/A)-1)/2 |
| Maximum active issuance keys | 1 |
| Maximum active redemption keys | 2 |
| Minimum cryptographic security parameter | 128 bits |

## Justification

We make the following assumptions in these parameter choices.

- Inferring the identity of a user in a 5000-strong anonymity set is
  difficult.
- After 2 weeks, all clients in a system will have rotated to the new
  key.

In terms of additional metadata, the only concrete applications of
Privacy Pass that use additional metadata require just a single bit.
Therefore, we set the ceiling of permitted metadata to 1 bit for now,
this may be revisited in future revisions.

The maximum choice of I is based on the equation 1/2 * U/2^(2I) = A.
This is derived from the fact that permitting I issuers lead to 2^I
segregations of the total user-base U. Moreover, if we permit M = 1,
then this effectively halves the anonymity set for each issuer, and thus
we incur a factor of 2I in the exponent. By reducing I, we limit the
possibility of performing the attacks mentioned in {{privacy}}.

We must also account for each user holding issued data for more then one
possible active keys. While this may also be a vector for monitoring the
access patterns of clients, it is likely to unavoidable that clients
hold valid issuance data for the previous key epoch. This also means
that the server can continue to verify redemption data for a previously
used key. This makes the rotation period much smoother for clients.

For privacy reasons, it is recommended that key epochs are chosen that
limit clients to holding issuance data for a maximum of two keys. By
choosing F = L/2 then the minimum value of F is a week, since the
minimum recommended value of L is 2 weeks. Therefore, by the initial
assumption, then all users should only have access to only two keys at
any given time. This reduces the anonymity set by another half at most.

Finally, the minimum security parameter size is related to the
cryptographic security offered by the protocol that is run. This
parameter corresponds to the number of operations that any adversary has
in breaking one of the security guarantees in the Privacy Pass protocol
{{draft-davidson-pp-protocol}}.

## Example parameterization

Using the specification above, we can give some example
parameterizations. For example, the current Privacy Pass browser
extension {{PPEXT}} has nearly 300000 active users (from Chrome and
Firefox). As a result, log_2(U/A) is approximately 6 and so the maximum
value of I should be 3.

If the value of U is much bigger (e.g. 5 million) then this would permit
I = (log_2(5000000/5000)-1)/2 ~= 4 issuers.

## Allowing more Issuers

Using the recommendations in {{more-issuers}}, it is possible to
tolerate larger number of Issuers if Clients in the ecosystem ensure
that they only store tokens for a small number of them. In particular,
if Clients limit their storage of redemption tokens to the bound implied
by I, then prevents a malicious verifier from triggering redemptions for
all Issuers in the ecosystem.

# Extension integration policy {#extensions}

The Privacy Pass protocol and ecosystem are both intended to be
receptive to extensions that expand the current set of functionality. As
specified in {{draft-davidson-pp-protocol}}, all extensions to the
Privacy Pass protocol SHOULD be specified as separate documents that
modify the content of this document in some way. We provide guidance on
the type of modifications that are possible in the following.

Any such extension should also come with a detailed analysis of the
privacy impacts of the extension, why these impacts are justified, and
guidelines on changes to the parametrization in {{parametrization}}.
Similarly, extensions MAY also add new Server running modes, if
applicable, to those that are documented in {{running-modes}}.

Any extension to the Privacy Pass protocol must adhere to the guidelines
specified in {{key-mgmt}} for managing Issuer public key data.

# Existing applications {#applications}

The following is a non-exhaustive list of applications that currently
make use of the Privacy Pass protocol, or some variant of the underlying
functionality.

## Cloudflare challenge pages

Cloudflare uses an implementation of the Privacy Pass protocol for
allowing clients that have previously interacted with their Internet
challenge protection system to bypass future challenges {{PPSRV}}. These
challenges can be expensive for clients, and there have been cases where
bugs in the implementations can severely degrade client accessibility.

Clients must install a browser extension {{PPEXT}} that acts as the
Privacy Pass client in an exchange with Cloudflare's Privacy Pass
server, when an initial challenge solution is provided. The client
extension stores the issued tokens and presents a valid redemption token
when it sees future Cloudflare challenges. If the redemption token is
verified by the server, the client passes through the security mechanism
without completing a challenge.

## Trust Token API

The Trust Token API {{TrustTokenAPI}} has been devised as a generic API
for providing Privacy Pass functionality in the browser setting. The API
is intended to be implemented directly into browsers so that server's
can directly trigger the Privacy Pass workflow.

## Zero-knowledge Access Passes

The PrivateStorage API developed by Least Authority is a solution for
uploading and storing end-to-end encrypted data in the cloud. A recent
addition to the API {{PrivateStorage}} allows clients to generate
Zero-knowledge Access Passes (ZKAPs) that the client can use to show
that it has paid for the storage space that it is using. The ZKAP
protocol is based heavily on the Privacy Pass redemption mechanism. The
client receives ZKAPs when it pays for storage space, and redeems the
passes when it interacts with the PrivateStorage API.

## Basic Attention Tokens

The browser Brave uses Basic Attention Tokens (BATs) to provide the
basis for an anonymity-preserving rewards scheme {{Brave}}. The BATs are
essentially Privacy Pass redemption tokens that are provided by a
central Brave server when a client performs some action that triggers a
reward event (such as watching an advertisement). When the client
amasses BATs, it can redeem them with the Brave central server for
rewards.

## Token Based Services

Similarly to BATs, a more generic approach for providing anonymous peers
to purchase resources from anonymous servers has been proposed
{{OpenPrivacy}}. The protocol is based on a variant of Privacy Pass and
is intended to allow clients purchase (or pre-purchase) services such as
message hosting, by using Privacy Pass redemption tokens as a form of
currency. This is also similar to how ZKAPs are used.

--- back

# Contributors

- Alex Davidson (alex.davidson92@gmail.com)
- Christopher Wood (caw@heapingbits.net)
