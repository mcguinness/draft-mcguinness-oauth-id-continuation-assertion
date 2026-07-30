---
title: "Identity Continuation Assertion for OAuth 2.0 Token Exchange"
abbrev: "Identity Continuation Assertion"
category: std

docname: draft-mcguinness-oauth-id-continuation-assertion-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - oauth
 - token exchange
 - identity chaining
 - delegation
 - id-jag
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-oauth-id-continuation-assertion"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-id-continuation-assertion/draft-mcguinness-oauth-id-continuation-assertion.html"

author:
 -
    fullname: "Karl McGuinness"
    organization: "Independent"
    email: "public@karlmcguinness.com"

normative:
  RFC6749:
  RFC7519:
  RFC7523:
  RFC7638:
  RFC7662:
  RFC7800:
  RFC8414:
  RFC8693:
  RFC8707:
  RFC8725:
  RFC9396:
  RFC9449:
  I-D.ietf-oauth-identity-assertion-authz-grant:
  OIDC.FrontChannelLogout:
    title: "OpenID Connect Front-Channel Logout 1.0"
    target: "https://openid.net/specs/openid-connect-frontchannel-1_0.html"
    date: false
    author:
      - org: "OpenID Foundation"
  SAML2.Core:
    title: "Assertions and Protocols for the OASIS Security Assertion Markup Language (SAML) V2.0"
    target: "https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf"
    date: 2005-03
    author:
      - org: "OASIS"

informative:
  RFC6755:
  RFC6838:
  RFC7009:
  RFC8417:
  RFC8705:
  RFC9700:
  I-D.fletcher-transaction-token-chaining-profile:
  I-D.ietf-oauth-identity-chaining:
  I-D.ietf-oauth-transaction-tokens:
  I-D.ietf-wimse-arch:
  I-D.li-oauth-delegated-authorization:
  I-D.mcguinness-oauth-actor-receipts:
  I-D.mcguinness-oauth-actor-proofs:
  OIDC.Core:
    title: "OpenID Connect Core 1.0"
    target: "https://openid.net/specs/openid-connect-core-1_0.html"
    date: false
    author:
      - org: "OpenID Foundation"
  GRANT-MGMT:
    title: "Grant Management for OAuth 2.0"
    target: "https://openid.net/specs/oauth-v2-grant-management.html"
    date: false
    author:
      - org: "OpenID Foundation"

...

--- abstract

This document profiles OAuth 2.0 Token Exchange to define the Identity
Continuation Assertion: a short-lived, sender-constrained JSON Web Token (JWT)
that carries verifiable evidence of a delegation chain into a Token Exchange
request. A Continuation Authorization Server can then issue an onward
authorization grant without subjecting the user to a new interactive
authentication. The primary use case is same-Identity-Provider (same-IdP)
chaining between software-as-a-service (SaaS) applications, in which several
Resource Authorization Servers trust a single enterprise IdP and each names
the user under its own audience-local (pairwise) subject identifier.
Because only the IdP holds the map from a root delegation to each audience's
subject, and crossing a boundary that renames the user is a re-issuance
rather than an attenuation, the IdP remains the sole issuer of the onward
Identity Assertion JWT Authorization Grant (ID-JAG) at every cross-boundary
hop. The assertion is the evidence carried into the exchange; the resulting
grant is the authority. This
profile complements, and does not replace, offline attenuated delegation,
which is appropriate for intra-domain fan-out that does not change the user's
subject identifier.

--- middle

# Introduction

Modern enterprise software-as-a-service (SaaS) deployments increasingly chain
calls across application boundaries on behalf of a single human user. A user
authenticates once at an enterprise Identity Provider (IdP) and invokes an
application; completing the user's request requires that application to call a
second application, which in turn must call a third. Increasingly the calling
party is an automated agent acting for the user, fanning out onward calls while
the user is no longer present, which makes this pattern both more common and
more acute. Each application exposes an API protected by its own Resource
Authorization Server (RAS), and all of these Resource Authorization Servers
trust the same enterprise IdP for single sign-on (SSO), for delegation
continuity, and for resolving the user's identity into an audience-local
subject identifier.

The key use case addressed by this document is therefore:

~~~
ExpenseApp -> ExpenseSaaS/ExpenseRAS -> TravelSaaS/TravelRAS
           -> BookingSaaS/BookingRAS
~~~

The user authenticates once at the enterprise IdP and is never redirected
through a new interactive flow for an onward hop. The IdP remains the trust
anchor, and each onward grant is audience-scoped and carries the subject
identifier appropriate for its target RAS.

## Why a New Input Is Needed {#motivation}

The first hop is an ordinary identity assertion exchange: ExpenseApp holds a
credential for the authenticated user (for example, an ID Token or refresh
token) and exchanges it at the IdP for an Identity Assertion JWT Authorization
Grant (ID-JAG), exactly as in
{{I-D.ietf-oauth-identity-assertion-authz-grant}}. Every subsequent hop is
different: by the time TravelSaaS must call BookingSaaS, the user is no longer
present and TravelSaaS holds no end-user credential to present. Only chain
context crossed the boundary, so TravelSaaS cannot perform a normal exchange
even though the same IdP could mint the next grant.

This document defines the missing input. An Identity Continuation Assertion
is a short-lived, sender-constrained, verifiable statement about the
in-flight delegation chain, presented to the IdP in place of an end-user
credential so the IdP can resolve the next audience's subject and issue the
next onward grant ({{core-principle}}).

## Core Principle {#core-principle}

Each RAS trusts only the IdP to name the user and to scope authority over its
resources; it does not accept another SaaS domain's word for either. The IdP
therefore remains the issuer at every cross-boundary hop as a matter of trust
direction, whatever form subject identifiers take. Pairwise subjects
strengthen that trust rule into a structural impossibility: where each RAS
resolves the user under its own audience-local subject identifier, only the
IdP holds the map from the root delegation to those per-audience subjects, and
no downstream party or offline authority can mint a correct onward `sub`.
Deployments whose subject identifiers are globally consistent (for example,
email-based) still require the IdP at every hop for trust direction and
revocation; adopting audience-local subjects additionally yields the pairwise
privacy property.

An onward hop that re-subjects the user is therefore a re-issuance, not an
attenuation. A holder can only narrow authority it already possesses; it
cannot synthesize a subject it was never issued. The Identity Continuation
Assertion supplies the fresh, sender-constrained evidence the IdP needs to
mint the next grant, used as a Token Exchange subject token to obtain an
onward ID-JAG, the output of this profile.

The profile therefore serves two inseparable requirements of this deployment
model: identity continuity, resolving the same root user into each new
audience-local subject, and authorization continuity, proving that the
requested onward authority remains within the root delegation.

This document defines the evidence format, Token Exchange parameters,
validation rules, and IANA registrations needed for that exchange. It does not
define a new access token format, does not allow a Resource Server to consume
the Identity Continuation Assertion directly, and does not allow a Chain
Authority to name the user for the target audience.

## Relationship to ID-JAG and Identity Chaining

This profile builds directly on OAuth 2.0 Token Exchange {{RFC8693}} and JSON
Web Tokens (JWTs) {{RFC7519}}, and it is a continuation extension of the ID-JAG
{{I-D.ietf-oauth-identity-assertion-authz-grant}} within the cross-domain model
of OAuth Identity and Authorization Chaining Across Domains
{{I-D.ietf-oauth-identity-chaining}}. ID-JAG covers the exchange at the first
hop, where the requester still holds an end-user credential. It does not by
itself cover later hops, where a downstream service holds no end-user
credential to exchange and cannot mint the next audience's subject
({{core-principle}}). This profile supplies the missing input for those hops and
reuses ID-JAG unchanged as the onward grant type, rather than defining a
competing grant.

It composes with, and is positioned relative to, the Workload Identity in
Multi-System Environments (WIMSE) architecture {{I-D.ietf-wimse-arch}} and
offline attenuated delegation tokens used for intra-domain fan-out.

Concretely, this profile adds to the ID-JAG exchange: a new `subject_token`
type ({{names}}) for the continuation evidence, the `identity_continuation_handle`
claim the IdP embeds in each continuation-capable ID-JAG
({{root-establishment}}, {{chain-id}}), continuation-aware Resource
Authorization Server processing that binds that claim to authorization state
({{ras-processing}}), the IdP validation rules for a continuation request
({{validation}}), the intra-domain Transaction Token carrier for chain context
({{transaction-token-context}}), and authorization server metadata signals
({{metadata}}). This profile therefore profiles both ID-JAG issuance and its
acceptance at the Resource Authorization Server.

Open design questions on which the author seeks working group feedback are
collected in {{open-items}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the following terms:

Identity Provider (IdP):
: The enterprise authority that authenticates the user, anchors trust for a
  set of Resource Authorization Servers, holds the map from a root delegation
  to each audience's subject identifier, and issues onward authorization
  grants. In this profile, the IdP is the Continuation Authorization Server.

Continuation Authorization Server:
: The Authorization Server to which an Identity Continuation Assertion is
  presented to obtain an onward grant. In the same-IdP use case of this
  document, it is the enterprise IdP at every hop.

Resource Authorization Server (RAS):
: An Authorization Server that protects a particular SaaS API and trusts the
  enterprise IdP for SSO and subject resolution. It consumes an ID-JAG as an
  authorization grant and issues access tokens for its API.

Resource Server (RS):
: The service that hosts a protected API and consumes the access tokens issued
  by its Resource Authorization Server. A Resource Server never consumes an
  Identity Continuation Assertion, no token it consumes carries an `identity_continuation_handle`,
  and its authorization decisions never depend on one. A workload co-located
  with a Resource Server MAY receive an `identity_continuation_handle` as control-plane context
  accompanying a request ({{transaction-token-context}}); it MUST NOT place a received
  handle into any token it issues ({{chain-id}}, rule 4) and is otherwise
  subject to the disclosure limits of {{privacy}}.

ID-JAG:
: An Identity Assertion JWT Authorization Grant
  {{I-D.ietf-oauth-identity-assertion-authz-grant}}: the onward authorization
  grant the IdP issues for a target RAS.

Identity Continuation Assertion:
: A short-lived, sender-constrained JWT, defined by this document, issued by a
  Chain Authority and presented to the IdP as the `subject_token` of a Token
  Exchange request in order to obtain an onward authorization grant. Its token
  type is `urn:ietf:params:oauth:token-type:identity-continuation`.

Chain:
: The server-side delegation record the IdP establishes for a continuable
  delegation ({{root-establishment}}): a tree of hops under one governing
  authorization, with lineage read along each hop's parent path
  ({{chain-id}}).

Chain Authority:
: The party trusted by the Continuation Authorization Server to assert chain
  evidence for a given tenant and to issue Identity Continuation Assertions.
  It is a role, not a new component: commonly an existing party the IdP
  already trusts (a Resource Authorization Server, a transaction-token
  service, or a domain gateway), though it may be a dedicated service. It is
  never the authority for resolving the target audience's user subject
  identifier.

Current actor (presenting actor):
: The workload that presents the Identity Continuation Assertion and the Token
  Exchange request to the IdP. It is identified by the `act` claim
  and authenticated by the `actor_token`.

Tenant:
: The administrative boundary (for example, an enterprise customer of the IdP)
  within which a chain is established and trust in a Chain Authority is
  configured. How a tenant is determined is deployment-defined and otherwise
  out of scope for this document.

Trust domain:
: The administrative and authentication boundary within which a party, such
  as a Chain Authority or a workload identity issuer, can directly
  authenticate workloads and vouch for their identities (for example, one
  SaaS provider's production environment). It is comparable to the
  trust-domain concept of the WIMSE architecture {{I-D.ietf-wimse-arch}};
  how a trust domain is identified is deployment-defined.

Continuation Handle (`identity_continuation_handle`):
: An opaque, unguessable, IdP-generated reference to one hop of a delegation
  chain; see {{chain-id}}.

Hop:
: One continuation point in a chain: the root delegation context, or a
  successful continuation recorded with an immutable reference to its parent
  hop. A chain's hops form a tree, and a hop's lineage is the path from that
  hop to the root.

Root-chain envelope:
: The state the IdP records when it establishes a chain and evaluates every
  continuation against. It is derived from the user's authentication and
  consent and from tenant policy, and comprises:

  * the authenticated user;
  * the authentication context (`auth_time`, `acr`, `amr`);
  * the authorization basis for onward targets;
  * the continuation authorization: the actors or trust domains permitted to
    continue the chain, and the basis on which that permission was
    established ({{root-establishment}});
  * any maximum actor-chain depth set by policy;
  * the chain's governing authorization ({{lifecycle}}); and
  * the chain's expiry.

  Every envelope dimension is a ceiling captured at establishment: later
  policy MAY narrow or revoke any dimension, but MUST NOT broaden one, and
  broader authority requires establishing a new chain
  ({{root-establishment}}). The authorization basis is the ceiling on
  onward targets, captured from the user's consent and tenant policy and
  recorded (for example, a policy version or an evaluated maximal grant);
  the IdP evaluates every continuation against it at request time. A deployment whose onward targets are known at
  establishment may record the basis as explicit target entries, each
  binding one audience and resource to the scopes, and optionally the
  authorization details {{RFC9396}}, that may be requested for that target;
  dynamic fan-out, in which onward targets are not enumerable at
  establishment, needs no enumeration. See {{validation}} and {{lifecycle}}.

Audience-local (pairwise) subject:
: The subject identifier under which a particular RAS names the user. Distinct
  Resource Authorization Servers may name the same user with different
  identifiers; only the IdP holds the map between them.

# When to Use This Profile Versus Offline Attenuation {#decision-rule}

The deciding question is subject resolution, not cost.

* If the next audience requires a *different* subject identifier (pairwise
  subjects, where only the IdP holds the map), only the IdP can mint that
  hop: an Identity Continuation Assertion to the IdP is used, because
  offline tokens cannot produce the next audience's `sub`.

* If the *same* subject identifier, or a key-based workload identity,
  suffices down a local fan-out, there is no reason to round-trip the IdP:
  an offline
  attenuated delegation token (for example,
  {{I-D.li-oauth-delegated-authorization}}) is used instead, and the IdP is
  reserved for the boundary hops.

In the canonical flow both appear: an offline attenuated stack inside
TravelSaaS as it fans out to sub-agents, and an Identity Continuation Assertion
when TravelSaaS crosses to BookingRAS under a new subject.

# The Identity Continuation Assertion {#assertion}

## Token Type and Media Type {#names}

The Identity Continuation Assertion is identified as follows:

~~~
Name:        Identity Continuation Assertion
Token type:  urn:ietf:params:oauth:token-type:identity-continuation
JOSE typ:    oauth-identity-continuation+jwt
~~~

The assertion is a JWT {{RFC7519}}. Its JSON Object Signing and Encryption
(JOSE) header `typ` value `oauth-identity-continuation+jwt` corresponds to the
media type `application/oauth-identity-continuation+jwt` ({{iana}}). The IdP
MUST verify the `typ` header to ensure that the JWT is processed as an Identity
Continuation Assertion and is not confused with any other JWT, as recommended
by {{RFC8725}}.

## Claims {#assertion-claims}

The following is a non-normative example of the Identity Continuation Assertion
claim set:

~~~ json
{
  "iss": "https://ca.travel.example/",
  "aud": "https://idp.example/",
  "identity_continuation_handle": "kW4uJ8pTe2NxA6rQvD1zYs",

  "act": {
    "iss": "https://travel.example/",
    "sub": "travel-service"
  },

  "cnf": {
    "jkt": "base64url-current-actor-key-thumbprint"
  },

  "iat": 1710000500,
  "exp": 1710000800,
  "jti": "continuation-assertion-01"
}
~~~

The claims have the following meanings and requirements:

`iss`:
: REQUIRED. The Chain Authority that issued the assertion. The IdP MUST verify
  that this issuer is a trusted Chain Authority for this tenant and that the
  assertion was signed with a key authorized for that issuer.

`aud`:
: REQUIRED. It MUST be a single string containing the issuer identifier of the
  Continuation Authorization Server being asked to issue the onward grant. The
  IdP MUST validate `aud` using exact string matching against its issuer
  identifier. A token endpoint URL that differs from the issuer identifier MUST
  NOT be used as this claim's value.

`identity_continuation_handle`:
: REQUIRED. The continuation handle being continued. It binds the request to a
  specific hop of a chain: the IdP resolves the chain, the parent hop, the
  user, and the audience-local subject from it. See {{chain-id}}.

`act`:
: REQUIRED. The current actor presenting the Token Exchange request, encoded
  as a single-level `act` claim per {{RFC8693}}. The `act` object contains a
  REQUIRED `iss` and a REQUIRED `sub`, both non-empty strings. Additional
  members MAY carry further identity attributes of the actor; a recipient
  MUST ignore members it does not understand, and the members `exp`, `nbf`,
  `aud`, `scope`, `cnf`, and nested `act` MUST NOT be present. Additional
  members are non-authoritative: the IdP MUST NOT use them for identity
  comparison, continuation authorization, lineage construction, or grant
  issuance. The IdP MUST
  reject an assertion whose `act` does not conform to this schema. The
  assertion names no other actor: all lineage is constructed by the IdP
  ({{onward-id-jag}}), and nested `act` values are reserved for a future
  extension ({{open-items}}).

  Where an offline attenuated stack delegated to the current actor, the
  Chain Authority MAY retain audit evidence for that offline segment
  separately from the assertion, for example per-hop issuer receipts
  {{I-D.mcguinness-oauth-actor-receipts}} or actor-signed hop proofs
  {{I-D.mcguinness-oauth-actor-proofs}}; such evidence SHOULD remain in the
  control plane ({{privacy}}).

`cnf`:
: REQUIRED. A confirmation claim {{RFC7800}} that binds the assertion to the
  key of the actor that will present it to the IdP. `cnf` MUST contain exactly
  one confirmation method: `jkt`, the JWK SHA-256 thumbprint {{RFC7638}} of
  the key the actor proves possession of using Demonstrating Proof of
  Possession (DPoP) {{RFC9449}}. Issuance-time binding and live proof of
  possession are specified in {{sender-constrained-presentation}}.

`iat`, `exp`:
: REQUIRED. The assertion lifetime MUST be short. `exp` MUST be after `iat`,
  and `exp - iat` MUST be no more than 300 seconds; see {{validation}}. The
  bound accommodates issuance-to-presentation latency while keeping the
  replay-state retention window small.

`jti`:
: REQUIRED. A unique identifier used by the IdP for replay detection. The value
  MUST be unique per issuer (`iss`) within the assertion validity window.

The assertion MUST NOT contain a top-level `sub`, `auth_time`, `acr`, `amr`,
or `sid` claim ({{validation}}, rule 8). The IdP obtains the user and root
authentication context from the
root-chain envelope indexed by `identity_continuation_handle`, and identifies the current actor
from the `act` claim and the `actor_token`; the assertion is evidence, not
a second source of identity.

The assertion MAY carry other top-level claims. A recipient MUST ignore
claims it does not understand, and additional claims are non-authoritative:
the IdP MUST NOT use them in validation, continuation authorization, or
grant issuance.

## Claims That Are Deliberately Excluded {#excluded-claims}

The following Token Exchange request values MUST NOT be conveyed by the
Identity Continuation Assertion:

~~~
audience (target)
resource
scope
authorization_details
requested_token_type
~~~

These stay in the Token Exchange request so that direct and chained calls
have an identical shape ({{token-exchange}}). The assertion's `aud` claim is
different: it identifies the Continuation Authorization Server that consumes
the assertion, not the requested target audience.

## Assertion Issuance and Key Binding {#assertion-issuance}

A presenting actor obtains an Identity Continuation Assertion from the Chain
Authority for a given `identity_continuation_handle` ({{flow}}). The Chain
Authority:

* MUST bind the assertion to the presenting actor's key via `cnf`
  ({{assertion-claims}}), and MUST do so only after establishing that the
  presenting actor controls that key and is the actor recorded in the
  `act` claim; and

* SHOULD be in the presenting actor's trust domain. It can satisfy these
  requirements only for actors it can directly authenticate, so each trust
  domain normally operates or designates its own Chain Authority. Issuing assertions for
  actors in another trust domain is NOT RECOMMENDED unless the Chain
  Authority has an authenticated relationship with those actors and their
  keys.

How the actor authenticates to the Chain Authority and proves control of its
key is deployment-specific; it is typically the workload's existing mutually
authenticated credential. The issuance interaction itself (endpoint, request
parameters, and responses) is out of scope for this document; a standard
issuance profile is an open item ({{open-items}}). An assertion whose `cnf` is not bound to the
presenting actor's key is not presentable, because the IdP requires live
proof of possession of the confirmed key
({{sender-constrained-presentation}}).

## Chain-Context Provenance {#context-provenance}

The mechanism used to transport chain context between workloads is
deployment-specific, but its security properties are not. A Chain Authority
MUST issue an Identity Continuation Assertion only after establishing all of
the following:

1. the `identity_continuation_handle` was received through an authenticated,
   confidential, and integrity-protected channel from a participant in the
   chain (its root actor, a continuer, or a workload conveying context on
   their behalf), or was obtained from equivalent authenticated state held
   by the Chain Authority;

2. the presenting actor is authorized under Chain Authority policy to continue
   the chain;

3. the presenting actor controls the key placed in `cnf`; and

4. the actor placed in `act` is the authenticated presenting actor
   ({{assertion-claims}}), and, where an offline attenuated stack delegated
   to that actor, the Chain Authority has validated the delegation
   artifact's integrity and delegation rules (for example, an actor-signed
   hop-proof chain {{I-D.mcguinness-oauth-actor-proofs}}) before issuing.

Possession of `identity_continuation_handle` alone is insufficient to satisfy these
requirements, and values received as propagated context, including actor
history or requested authority, MUST NOT override the root-chain envelope.
The transport MAY carry deployment-specific hints, but the assertion's
authoritative content is only the claims defined in {{assertion-claims}},
and the IdP applies its own root-chain and current-actor policy at the
exchange.

Because a Transaction Token ({{transaction-token-context}}) is not a workload
credential and may be replayed within its lifetime, a Chain Authority MUST NOT
treat receipt of one as sufficient to issue. It MUST additionally authenticate
the requesting workload independently, verify proof of possession of the key
it will place in `cnf`, bind that actor to the transaction, reject any
caller-supplied substitution of the handle, and recheck that the underlying
Resource Authorization Server authorization is still active. A single
transaction legitimately fans out to several targets, so universal single-use
is too strict; instead the Chain Authority MUST enforce policy-bounded
issuance per (transaction, actor), with rate and fan-out limits and audit
records, so that one replayed Transaction Token cannot mint unbounded
assertions. A caller MAY supply a target or purpose hint to bound issuance,
but such a hint MUST NOT become authoritative over the IdP's target decision
({{validation}}, rule 14).

## Transaction Token Chain Context {#transaction-token-context}

Within a trust domain, chain context travels inside the domain's existing
context propagation, typically a Transaction Token
{{I-D.ietf-oauth-transaction-tokens}}. A Transaction Token Service derives the
current hop's `identity_continuation_handle` from the authorization state the Resource
Authorization Server bound at acceptance ({{ras-processing}}) and places it in
a namespaced member of the token's request context:

~~~ json
"tctx": {
  "identity_continuation": {
    "iss": "https://idp.example/",
    "tenant": "tenant-123",
    "handle": "kW4uJ8pTe2NxA6rQvD1zYs"
  }
}
~~~

The requester MUST NOT supply or override `tctx.identity_continuation` through
`request_details` or any client-controlled input; the Transaction Token
Service populates it solely from bound authorization state. The Transaction
Token is an intra-domain object ({{rationale-txn}}), MUST NOT be accepted
outside its trust domain, and is propagated unchanged by workloads within the
domain.

Authorized transaction-chain workloads, including API implementations, MAY
read `identity_continuation_handle` from this context. It conveys no authority (rule 7),
MUST NOT be placed in any access token or external authorization claim
(rule 4), and SHOULD be excluded from logs and traces. A Transaction Token is
not a workload credential and may be replayed within its lifetime, so a Chain
Authority MUST NOT treat possession of one as sufficient to issue an
assertion; the independent checks of {{context-provenance}} apply.

# Continuation Handles (`identity_continuation_handle`) {#chain-id}

An `identity_continuation_handle` is an opaque, unguessable, IdP-generated reference to
one hop of a delegation chain. The IdP creates the root hop when it
establishes a chain ({{root-establishment}}) and a child hop for each
successful continuation; each child holds an immutable reference to the hop
whose handle was presented.

A chain is therefore a tree. Sibling continuations that present the same
handle, including concurrently, create sibling hops, and a hop's lineage is
the path from that hop to the root, independent of every other branch. From
the presented handle the IdP resolves the chain, the parent hop, and the
correct per-audience subject.

`identity_continuation_handle` is a non-bearer reference, closer in spirit to a grant
identifier than to a token, following the pattern of the OpenID Connect
`sid` claim {{OIDC.FrontChannelLogout}} and the `txn` claim {{RFC8417}}:
opaque, issuer-generated identifiers that index server-side state (rule 7).
The IdP embeds it as a claim in each continuation-capable ID-JAG (rule 1);
the accepting Resource Authorization Server binds it to authorization state
({{ras-processing}}) and never lets it reach an access token or the protected
API (rule 4; see also {{privacy}}).

The following rules apply:

1. The IdP MUST establish a chain for every direct ID-JAG issuance under a
   continuation-capable governing authorization ({{root-establishment}}), and
   MUST embed a fresh hop reference as the `identity_continuation_handle`
   claim of the issued ID-JAG, for the root hop and for each continuation hop.
   Handle values MUST NOT be reused across hops.

2. `identity_continuation_handle` MUST contain at least 128 bits of entropy, MUST NOT
   contain user-identifying information, and MUST consist of 22 to 256
   characters drawn from the base64url alphabet (`A`-`Z`, `a`-`z`, `0`-`9`,
   `-`, `_`).

3. `identity_continuation_handle` appears as a claim in a continuation-capable ID-JAG
   and in intra-domain chain context (for example, a Transaction Token,
   {{transaction-token-context}}). It crosses a trust boundary only inside an
   ID-JAG (to the accepting Resource Authorization Server) or an Identity
   Continuation Assertion (to the IdP), never as a standalone value.

4. `identity_continuation_handle` MUST NOT appear in an access token or in a Resource
   Server's external authorization claims. Authorized transaction-chain
   workloads MAY observe it in intra-domain chain context
   ({{transaction-token-context}}), subject to the non-authority,
   confidentiality, and log-suppression rules stated there. Because each hop's
   handle is distinct and each Resource Authorization Server sees only its own
   hop's value in the ID-JAG it accepts, no handle correlates the user across
   SaaS boundaries (actor lineage and timing remain correlation channels;
   {{privacy}}).

5. End-to-end audit correlation is performed at the IdP, which holds the hop
   graph for every chain. Per-Resource-Server logs use that server's own
   pairwise subject.

6. A Resource Authorization Server binds `identity_continuation_handle` to the
   authorization state it establishes ({{ras-processing}}); Resource
   Authorization Servers, Resource Servers, and Chain Authorities MUST NOT
   modify the value.

7. A hop is continuable only after its parent ID-JAG has been accepted and
   bound by the target Resource Authorization Server ({{ras-processing}}). The
   IdP records a hop as pending on issuance and treats a valid assertion from
   the hop's mapped Chain Authority ({{root-establishment}}) as evidence that
   the hop reached the accepted state; a handle copied from an
   issued-but-rejected ID-JAG is therefore not continuable. `identity_continuation_handle`
   alone is not proof of authorization and MUST NOT be treated as a bearer
   credential; the IdP MUST use it only as a lookup handle for the hop, its
   chain state, subject resolution, and policy evaluation.

8. A hop's parent reference is immutable. The IdP MUST derive lineage solely
   by walking parent references from the presented hop to the root, and MUST
   NOT maintain or extend a single chain-wide actor history: concurrent
   sibling continuations are independent branches.

In deployments that already maintain a delegation or mission identifier, the
IdP MAY derive hop handles from that identifier internally (for example, by
a keyed one-way derivation), provided every value remains distinct per hop,
unguessable, and unlinkable, per rules 1, 2, and 8.

## Handle Freshness and Unlinkability {#chain-id-privacy}

Handles are hop-specific by construction: no two hops share a value (rule 1),
so audiences and branches do not acquire a shared identifier for the
delegation. A given handle is still observed by every participant on its
conveyance path (the party that received it, workloads that propagate it, the
Chain Authority, and the IdP), so unlinkability holds across hops, not within
a path, and actor lineage and timing remain correlation channels
({{privacy}}). Handles of a revoked chain, or of a hop subtree the IdP has
revoked ({{lifecycle}}), fail validation ({{validation}}, rule 7) at the
next exchange.

# Chain Lifetime and Revocation {#lifecycle}

A chain is continuable only while the IdP considers it active. Because every
cross-boundary hop is an exchange at the IdP ({{flow}}), the IdP is in the loop
at each hop and can stop a chain at any point.

Three distinct lifetimes apply and MUST NOT be conflated. An ID-JAG's `exp`
governs only how long that grant JWT may be redeemed, and is short. The
authorization state a Resource Authorization Server binds at acceptance
({{ras-processing}}) has its own local lifetime and revocation. The chain's
continuation lifetime is authoritative at the IdP and may be much longer than
any single ID-JAG's `exp`; it is the lifetime bounded below.

Every chain has a continuation-capable governing authorization
({{root-establishment}}): the server-side consent and policy record that
permits continuation. That record has a lifecycle anchor, resolved at
establishment from the root subject token: the OAuth authorization grant
behind a refresh token, or the IdP session resolved from a session
reference ({{root-establishment}}). The binding is to the server-side
records, not to a token value: refresh-token rotation does not affect a
grant-anchored chain, while revocation or expiry of the anchoring grant
ends every chain rooted in it, and termination of an anchoring session
ends every chain rooted in that session. Withdrawal of the consent or
policy that made the authorization continuation-capable likewise ends its
chains, even while the anchor itself remains active. A session-anchored
chain's lifetime MUST NOT exceed the session's, so only grant-anchored
chains outlive logout.

The IdP MUST bound the continuation lifetime of a chain by the lifetime of
its governing authorization, and MUST reject a continuation of an expired
chain ({{validation}}).

The governing authorization is distinct from the
root authentication context: continuing a chain MUST NOT extend that
context, and `auth_time`, `acr`, and `amr` are fixed at root issuance and
inherited unchanged by onward grants ({{security-assurance}}).

The IdP MUST be able to revoke a chain, and MUST stop honoring continuation
for a revoked chain. Ending the governing authorization or its anchor revokes every chain rooted in it:
revocation of the anchoring grant does so (for example, through a grant
management interface, or through token revocation {{RFC7009}} where the
server's revocation policy revokes the underlying grant, which that
specification leaves to server policy), as does
termination of the user's session where the session anchors, or withdrawal
of the underlying consent or policy. An IdP MAY additionally revoke an individual hop's subtree, stopping
continuation along one branch while the rest of the chain remains
continuable.

Revocation therefore takes effect at the next hop, with no need to reach
chain evidence already held by intermediate services, unlike an offline
attenuated delegation token whose minted child remains usable for its
lifetime without contacting an authority. Access tokens already issued by a
Resource Authorization Server remain valid for their own lifetimes under
that server's revocation mechanisms.

Revocation is a control-plane capability, but the delegation belongs to the
user. IdPs are encouraged to make active chains visible and revocable through user-
and administrator-facing management interfaces, presenting the root context,
the chain's hop graph and actor lineage, the targets granted so far, and the
chain's expiry. Grant Management for OAuth 2.0 {{GRANT-MGMT}} provides a
comparable management model for standing grants.

# Token Exchange Profile {#token-exchange}

An Identity Continuation Assertion is used as the `subject_token` of an OAuth
2.0 Token Exchange request {{RFC8693}}. A direct request and a chained request
have the same shape: the client changes only `subject_token` and
`subject_token_type`. Chain establishment is the IdP's act, not a request
parameter ({{root-establishment}}).

## Direct ID-JAG Request

A direct request, in which the subject token is a normal subject token such as
an ID Token or refresh token:

~~~
grant_type=urn:ietf:params:oauth:grant-type:token-exchange
requested_token_type=urn:ietf:params:oauth:token-type:id-jag
audience=https://ras.travel.example/
resource=https://api.travel.example/
scope=trips.read
subject_token=<id_token | refresh_token>
subject_token_type=<normal-subject-token-type>
actor_token=<sender-constrained-current-actor-credential> (OPTIONAL)
actor_token_type=<actor-token-type>                       (OPTIONAL)
~~~

On a direct request, `actor_token` is OPTIONAL ({{root-establishment}}).

## Establishing a Chain {#root-establishment}

A chain is established by the IdP, not requested by the client: the party
performing the root exchange cannot generally know whether downstream
services will need to continue the delegation, and requiring it to predict
would foreclose continuation for the entire downstream graph. The IdP MUST
establish a chain for each direct exchange in which it issues an ID-JAG
under a continuation-capable governing authorization (below), and MUST
return the root hop's handle ({{response-param}}); chain state MAY be
materialized lazily, on first continuation, and the root handle remains
resolvable to its chain regardless ({{validation}}, rule 7). Where the governing authorization does not permit continuation, no chain
is established and no handle is returned; its absence tells the client
that the delegation is not continuable. Tenant enablement ({{metadata}})
is a capability signal and authorizes nothing.

The chain's properties are not negotiated. The root subject token resolves
the chain's lifecycle anchor, and the IdP then locates the governing
authorization attached to that anchor; the IdP MUST NOT establish a chain
from a subject token it cannot resolve to an anchor. Three rooting paths
are defined:

* a refresh token resolves to the OAuth authorization grant it belongs to,
  and the chain is anchored to that grant; only grant-anchored chains may
  outlive the establishing session ({{lifecycle}});

* an ID Token carrying a `sid` claim {{OIDC.FrontChannelLogout}} that the
  IdP resolves to an active session it established for this user and
  client anchors the chain to that session; and

* a SAML 2.0 assertion whose authentication statement carries a
  `SessionIndex` {{SAML2.Core}} that the IdP resolves to an active session
  it established for this user and presenter likewise anchors the chain to
  that session.

These correspond to the two functions that subject tokens perform in the
ID-JAG profile, identity assertion (an ID Token or SAML assertion
conveying the authenticated subject) and identity continuity (a refresh
token continuing an existing authorization grant); the distinction is
implicit in that profile's choice of subject tokens rather than stated by
it. Each rooting path anchors the chain to the record its function
references. An access token is a resource-authorization credential,
constructed for presentation to a protected resource rather than for
identity brokering; it is not an ID-JAG subject token and roots nothing
here.

A chain is established only where that governing authorization is
continuation-capable: its consent and policy explicitly permit
continuation and establish the authorization basis, the continuation
authorization (the permitted actors or trust domains), the maximum
actor-chain depth where bounded, and the chain's lifetime and revocation
behavior, all of which are recorded in the root-chain envelope. These
always come from server-side consent and policy state, never from token
claims. The IdP consumes `sid` and `SessionIndex` solely to resolve the
session; these values MUST NOT appear in assertions or conveyed chain
context ({{privacy}}). An unused
chain is inert state: its establishment exercises no authority beyond
what the continuation-capable governing authorization already granted.

When establishing a chain, the IdP MUST determine the root actor: the
authenticated OAuth client of the direct request, identified through the
authoritative client-to-actor mapping of {{client-identity}}. An
`actor_token` is OPTIONAL on a direct request. If present, it MUST be of a
type the IdP accepts for continuation and currently valid, MUST designate
the IdP where its type defines an audience or applicability restriction,
MUST be sender-constrained to a key whose possession the request proves
(for example, the request's DPoP proof key), and MUST identify the same
entity as the authenticated client. The IdP MUST record the root actor in
the root hop only after this validation; the root actor begins every
branch's lineage ({{onward-id-jag}}).

The IdP embeds the root hop's handle as the `identity_continuation_handle`
claim of the issued ID-JAG ({{response-param}}); the target Resource
Authorization Server binds it on acceptance ({{ras-processing}}).

When it issues the root hop, the IdP records, for that hop, the target
Resource Authorization Server and the Chain Authorities authorized to attest
continuation from it. A continuation is accepted only when its Identity
Continuation Assertion comes from a Chain Authority mapped to the accepting
Resource Authorization Server's domain ({{validation}}). The mapped Chain
Authority issues only from accepted, currently-active authorization state
({{ras-processing}}), which is how the IdP relies on RAS acceptance without a
callback.

Like the continuation exchange ({{validation}}), root establishment is
at-least-once: a retry after a lost response MAY establish a second chain.
Both chains are bounded by the same consent and policy, and an unused chain
expires.

## Chained ID-JAG Request

A chained request, in which the subject token is an Identity Continuation
Assertion:

~~~
grant_type=urn:ietf:params:oauth:grant-type:token-exchange
requested_token_type=urn:ietf:params:oauth:token-type:id-jag
audience=https://ras.travel.example/
resource=https://api.travel.example/
scope=trips.read
subject_token=<identity-continuation-assertion>
subject_token_type=<identity-continuation-token-type>
actor_token=<sender-constrained-current-actor-credential>
actor_token_type=<actor-token-type>
~~~

The `subject_token_type` value above is
`urn:ietf:params:oauth:token-type:identity-continuation`.

The requested `audience`, `resource`, `scope`, `requested_token_type`, and
any `authorization_details` are supplied by the Token Exchange request and
never by the assertion. Following
{{I-D.ietf-oauth-identity-assertion-authz-grant}}, `audience` identifies the
target Resource Authorization Server and `resource` ({{RFC8693}}, originally
defined in {{RFC8707}}) identifies the protected resource.

The request MAY also include `authorization_details` {{RFC9396}}, which the
ID-JAG profile supports in both the exchange and the issued grant; the
authorization-basis check ({{validation}}, rule 14) applies to it as to
scope. Client authentication is required on every exchange
({{client-identity}}) and is omitted from the example bodies for brevity.

## Sender-Constrained Presentation {#sender-constrained-presentation}

The Identity Continuation Assertion is sender-constrained: it is bound by `cnf`
to the key of the actor that presents it ({{assertion-claims}}). This profile
uses proof of possession aligned with the direct ID-JAG request
{{I-D.ietf-oauth-identity-assertion-authz-grant}}:

* The presenting actor MUST demonstrate live possession of the confirmed key
  by including a DPoP proof JWT {{RFC9449}} computed with that key, carried in
  the `DPoP` HTTP header (and therefore not in the request body shown above).
  The IdP MUST verify that the DPoP proof key thumbprint equals the
  assertion's `cnf.jkt`, and MUST reject a request that does not prove
  possession of the confirmed key.

* The Token Exchange request MUST include an `actor_token`, which authenticates
  the workload identity of the current actor, the actor named in the
  assertion's `act` claim. The `actor_token` MUST itself be sender-constrained
  to the same key confirmed by the assertion's `cnf`. For a JWT actor token,
  the IdP MUST verify a `cnf.jkt` value that matches both the assertion and
  the live proof. For an opaque actor token, the IdP MUST obtain the
  equivalent confirmation value from authoritative token metadata, such as a
  token introspection response {{RFC7662}}. A bearer actor token MUST NOT be
  accepted. The `actor_token` MUST be of a type the IdP accepts for
  continuation and MUST be currently valid under that type's rules. Where
  its type defines an audience or equivalent applicability restriction,
  that restriction MUST designate the IdP: a token that a trusted issuer
  minted for another service is not a valid `actor_token`. These checks bind the
  authenticated workload identity, the assertion, and the demonstrated key
  to one actor.

Actor identity comparison is exact: two actor identities denote the same
entity only if their issuer and subject identifiers are octet-for-octet equal. The
IdP MUST apply this comparison when matching the `actor_token` to the
`act` claim and to the authenticated client ({{client-identity}}).
Comparison occurs within the tenant context already established for the
exchange ({{validation}}, rules 6 and 10); identities from different tenants
are never compared equal.

The onward ID-JAG the IdP issues is itself sender-constrained to the same key
through its own `cnf`, using the DPoP-based binding the ID-JAG profile defines
({{onward-id-jag}}). This version deliberately standardizes a single mandatory-to-implement
confirmation method for interoperability. A mutual-TLS-bound variant
{{RFC8705}} is not specified: the ID-JAG profile currently defines only
DPoP-based binding,
and a different confirmation method in the onward grant would make the target
Resource Authorization Server validate the onward grant's confirmation
differently from a directly issued ID-JAG. The presenting actor proves
possession of the key again when it
exchanges the ID-JAG at the target Resource Authorization Server, exactly as
for a directly issued ID-JAG
{{I-D.ietf-oauth-identity-assertion-authz-grant}}.

The key binding is established per assertion at issuance. A workload that
rotates its key obtains its next assertion, and an actor token, bound to the
new key; rotation takes effect at the next issuance and requires no
coordination with the IdP.

## Client Identity and Authentication {#client-identity}

The party presenting a continuation exchange is an OAuth client of the IdP,
and the current actor is that client:

* The current actor MUST authenticate as an OAuth client. As with the direct
  ID-JAG exchange, this profile is intended for confidential clients.
* The IdP MUST verify that the authenticated client and the current actor
  are the same entity, comparing the client's actor identity (its `iss` and
  `sub` pair) with the `actor_token` and the `act` claim using the exact
  comparison of {{sender-constrained-presentation}}. The IdP learns the
  client's actor identity from an authoritative mapping, typically recorded
  in the client registration or resolvable from a shared client identifier
  namespace such as Client ID Metadata Documents. The mapping is a trust
  anchor ({{security}}): the IdP MUST use only mapping sources it trusts
  as authoritative for the tenant, and MUST NOT accept a self-asserted
  actor identity whose binding to the actor's trust domain it has not
  verified.
* A deployment MAY use a single sender-constrained JWT as both the client
  assertion and the `actor_token` where the requirements of both roles
  coincide: {{RFC7523}} requires such a JWT's `sub` to be the workload's
  `client_id`, and the IdP MUST be configured to accept the workload
  identity issuer as an authorized issuer of client assertions for that
  client. Otherwise the client authenticates with its registered method and
  additionally presents the `actor_token`.

The `client_id` of the onward ID-JAG is the identifier under which the
current actor is registered for the target Resource Authorization Server.
The ID-JAG profile {{I-D.ietf-oauth-identity-assertion-authz-grant}} defines
how the IdP knows that identifier (an out-of-band per-RAS mapping, or a
shared namespace such as Client ID Metadata Documents) and requires the
grant's `client_id` to match the client that redeems it. A continuing
workload therefore needs a client registration, or a resolvable client
identity, at each target it will reach; in this document's examples the
workload identifier doubles as that client identifier.

## Continuation Handle Delivery {#response-param}

The IdP delivers the hop reference as the `identity_continuation_handle` claim of the
issued ID-JAG ({{chain-id}}, rule 1; {{onward-id-jag}}), not as a separate
Token Exchange response parameter. Each hop's handle is distinct and its
parent reference is immutable ({{chain-id}}, rules 1 and 8). The accepting
Resource Authorization Server binds it to authorization state
({{ras-processing}}); the current domain then surfaces it to continuers
through intra-domain chain context ({{transaction-token-context}}).

There is no advisory chain-expiry response parameter. Chain lifetime is
authoritative at the IdP ({{lifecycle}}); a deployment that needs advance
warning of expiry conveys it through authenticated task or authorization
state, an optional ID-JAG claim, or a management API, not through the
Token Exchange response.

## Authorization Server Metadata {#metadata}

An IdP that supports this profile SHOULD signal it in its authorization server
metadata {{RFC8414}} with the following parameter:

`identity_continuation_supported`:
: OPTIONAL. Boolean value indicating that the IdP accepts Identity Continuation
  Assertions of the
  `urn:ietf:params:oauth:token-type:identity-continuation` subject token type
  and issues continuation-capable ID-JAGs carrying the
  `identity_continuation_handle` claim. Default `false`.

A Resource Authorization Server advertises separately, in its
`authorization_grant_profiles_supported`, that it recognizes the
continuation-capable ID-JAG grant profile and binds the
`identity_continuation_handle` claim to authorization state ({{ras-processing}}).
These are distinct capabilities: the IdP signal is about issuing and accepting
continuation, the Resource Authorization Server signal is about binding it.

Absent these signals, a party learns of support out of band or by attempting an
exchange.

# Same-IdP Core Flow {#flow}

The canonical same-IdP SaaS-to-SaaS flow proceeds as follows. Steps 3 through
7 repeat for each additional cross-boundary hop (for example, TravelRAS to
BookingRAS), each continuation performed by the current-domain workload after
its domain's Resource Authorization Server has bound the parent hop.

1. ExpenseApp authenticates the user and requests an ID-JAG for ExpenseRAS
   via Token Exchange.

2. The IdP establishes a chain ({{root-establishment}}), records the
   root-chain envelope (including the Chain Authorities mapped to ExpenseRAS),
   and issues the ID-JAG carrying the root hop's `identity_continuation_handle`
   claim (H0).

3. ExpenseApp redeems the ID-JAG at ExpenseRAS for an access token; ExpenseRAS
   validates the ID-JAG and binds H0 to that authorization state
   ({{ras-processing}}), and ExpenseApp calls ExpenseAPI. The Expense domain's
   Transaction Token Service derives H0 into intra-domain chain context
   ({{transaction-token-context}}); no audience-local subject crosses the SaaS
   boundary.

4. To reach TravelRAS, ExpenseService, the current Expense-domain workload,
   obtains an Identity Continuation Assertion for H0 from the Expense-domain
   Chain Authority ({{assertion-issuance}}), bound to its own key.

5. ExpenseService presents the assertion to the IdP as the `subject_token`,
   requesting an ID-JAG for TravelRAS.

6. The IdP validates the request ({{validation}}), resolves the user's
   TravelRAS subject, records a new hop, and issues the TravelRAS ID-JAG
   carrying a fresh handle (H1) with an IdP-constructed `act` chain
   (`expense-service` acting for `expense-app`).

7. ExpenseService redeems the ID-JAG at TravelRAS; TravelRAS binds H1 and
   issues the access token. TravelService then continues from H1 to BookingRAS
   by repeating from step 3 in the Travel domain.

# IdP Validation for the Continuation Exchange {#validation}

Before issuing an onward ID-JAG, the IdP MUST reject the Token Exchange request
unless all of the following hold. The rules are a conjunction with no
implied evaluation order, and one rule's input can come from another's
resolution: for a Chain Authority trusted through a federation mechanism
({{security}}), the tenant of rule 6 is resolved from the handle of rule 7.

1. the request contains exactly one each of `grant_type`, `subject_token`,
   `subject_token_type`, `requested_token_type`, `actor_token`, and
   `actor_token_type`; the `grant_type` is
   `urn:ietf:params:oauth:grant-type:token-exchange`, and the
   `subject_token_type` is
   `urn:ietf:params:oauth:token-type:identity-continuation`;

2. the request contains exactly one `audience` parameter and exactly one
   `resource` parameter;

3. the assertion is a JWT containing exactly one value for each required claim
   defined in {{assertion-claims}}; `iss`, `aud`, `identity_continuation_handle`, and `jti` are
   non-empty strings; `act` and `cnf` are JSON objects, `cnf` containing exactly one
   confirmation method ({{assertion-claims}}); `iat` and `exp` are
   JSON numbers representing NumericDate values; and the JOSE `typ` header
   value is `oauth-identity-continuation+jwt`;

4. the assertion signature validates using a key authorized for the assertion
   issuer, and the JOSE `alg` is an asymmetric signature algorithm on the IdP's
   configured allowlist (the `none` algorithm MUST be rejected; see
   {{security-alg}});

5. the assertion `aud` exactly matches the IdP's issuer identifier;

6. the assertion `iss` is a trusted Chain Authority for this tenant;

7. `identity_continuation_handle` identifies a known, active hop of an unexpired,
   unrevoked chain, and the actor lineage the continuation would produce,
   after same-actor collapse ({{onward-id-jag}}), is within any
   actor-chain depth bound set by policy; the bound measures distinct
   lineage entries, not hops, so a workload continuing from its own hop
   does not consume depth;

8. the assertion does not contain a top-level `sub`, `auth_time`, `acr`,
   `amr`, or `sid` claim, nor an `audience`, `resource`, `scope`,
   `authorization_details`, or `requested_token_type` claim
   ({{excluded-claims}}, {{root-establishment}});

9. the assertion's `act` claim is present, conforms to the schema of
   {{assertion-claims}}, and identifies the current actor;

10. the request and the current actor are bound together:
    * the request is authenticated as an OAuth client that is the same
      entity as the current actor ({{client-identity}});
    * the `actor_token` was issued by a workload identity issuer the IdP
      trusts for the current actor's trust domain and tenant, is of an
      accepted type and currently valid, designates the IdP where its type
      defines an audience or applicability restriction, and authenticates
      the current actor;
    * the `actor_token` is sender-constrained to the key confirmed by the
      assertion's `cnf` ({{sender-constrained-presentation}});
    * that actor is the actor named in `act`; and
    * that actor is permitted by the chain's continuation authorization
      ({{root-establishment}}) to continue from the presented hop;

11. the request proves possession of the key confirmed by `cnf` with a DPoP
    proof {{RFC9449}} matching `cnf.jkt`
    ({{sender-constrained-presentation}});

12. `jti` has not been successfully consumed before for the assertion issuer;

13. `iat` and `exp` are valid NumericDate values, `iat` is not in the future
    beyond the IdP's permitted clock skew, `exp` is after `iat`, the assertion
    has not expired, and the assertion lifetime is no more than 300 seconds;

14. the requested `audience` and `resource`, every requested scope, and any
    requested authorization details {{RFC9396}} are authorized by the
    root-chain envelope's authorization basis, evaluated at the time of the
    request, and by IdP policy for the current actor (containment for
    authorization details is type-specific and deployment-defined, unlike
    the set containment of scope);

15. the requested output token type is
    `urn:ietf:params:oauth:token-type:id-jag`; and

16. the IdP can resolve, for the requested `audience`, both the
    audience-local subject and the current actor's client identifier
    ({{client-identity}}).

After all other validation succeeds, the IdP MUST, as part of issuing the
onward grant, atomically verify that the tuple (`iss`, `jti`) has not been
consumed and record it as consumed. At most one concurrent exchange using the
same tuple may succeed, and the IdP
MUST retain a successfully consumed tuple until at least the assertion's
`exp`, allowing for the IdP's permitted clock skew. A request that fails validation
before grant issuance does not consume the tuple; how an implementation
ensures this (for example, by releasing reservations on failure) is an
implementation detail.

These requirements imply per-chain state and, within the assertion validity
window, strongly consistent replay state at the IdP; the 300-second lifetime
bounds the replay-state retention window. Hop state is bounded differently:
the depth bound of rule 7 does not limit sibling fan-out or
self-continuation, so deployments SHOULD apply operational limits (for
example, continuation rate or hop count per chain) and SHOULD prune hop
state when its chain expires or is revoked.

A client that does not receive the response to an exchange cannot retry with
the same assertion: it obtains a fresh assertion from its Chain Authority and
repeats the exchange, so Chain Authority issuance SHOULD be inexpensive
relative to the exchange itself. The retry path is at-least-once: a retry
after a lost response mints a second, equivalent grant (short-lived,
sender-constrained to the same key, and bounded by the same authorization
basis) and a second hop, so it confers no additional authority. Equivalent
grants do not deduplicate the application operations they authorize;
operation idempotency remains the application's concern.

On success, the IdP records the continuation as a new pending hop whose
immutable parent is the presented handle, resolves the audience-local subject
for the requested RAS, and issues the onward ID-JAG with that `sub`. The
onward ID-JAG MUST carry the new hop's `identity_continuation_handle` claim
({{chain-id}}, rule 1); the target Resource Authorization Server binds it on
acceptance ({{ras-processing}}).

If validation fails, the IdP MUST return an OAuth error response as defined
by {{RFC8693}} and {{RFC6749}}. The IdP SHOULD use `invalid_request` for
malformed or internally inconsistent requests, including unsupported
parameter cardinality, the wrong `subject_token_type`, and an invalid or
impermissible `actor_token` (rule 10), which is an unacceptable token per
{{RFC8693}}. A request that fails proof of possession (rule 11) SHOULD
receive `invalid_dpop_proof` {{RFC9449}} where DPoP was used. For requests
outside the chain's authority, the IdP SHOULD use `invalid_target` for an
audience and resource pair, `invalid_scope` for a scope, and
`invalid_authorization_details` {{RFC9396}} for an `authorization_details`
value, in each case one not permitted by the authorization basis or by IdP
policy for the current actor (rule 14).

A presented `identity_continuation_handle` that is unknown, expired, revoked, or
otherwise not continuable (rule 7) makes the `subject_token` unacceptable,
and the IdP returns `invalid_request` as Section 2.2.2 of {{RFC8693}}
requires. Retrying with the same handle cannot succeed; continuing requires
a handle on a live branch or a new root delegation, which typically requires
the user. `invalid_target`, `invalid_scope`, and
`invalid_authorization_details` signal instead that this particular request
falls outside the chain's authority while the chain remains continuable, so
an unattended client can abandon only the current request. Because
`invalid_request` also covers malformed requests, the error code alone does
not distinguish a dead hop from a protocol error; a continuation-specific
error code that would restore that distinction is an open item
({{open-items}}).

# Onward ID-JAG {#onward-id-jag}

The following is a non-normative example of the onward ID-JAG issued by the
IdP:

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.travel.example/",
  "sub": "travel-local-subject",

  "client_id": "expense-service",
  "resource": "https://api.travel.example/",
  "scope": "trips.read",

  "identity_continuation_handle": "H1-Vt7bQ2mLpZ4xR9sKdEwa",

  "auth_time": 1710000000,
  "acr": "urn:example:loa:2",
  "amr": ["pwd", "mfa"],

  "act": {
    "iss": "https://expenses.example/",
    "sub": "expense-service",
    "act": {
      "iss": "https://expenses.example/",
      "sub": "expense-app"
    }
  },

  "cnf": {
    "jkt": "base64url-current-actor-key-thumbprint"
  },

  "iat": 1710000520,
  "exp": 1710000820,
  "jti": "idjag-travel-01"
}
~~~

The `act` chain in an onward ID-JAG is constructed by the IdP, not copied
from the assertion: it is the newly authenticated current actor placed atop
the presented hop's lineage, the actors the IdP itself authenticated along
that hop's parent path ({{chain-id}}, rule 8). Sibling branches never
contribute to one another's lineage, and any actor-chain depth bound set by
policy applies to the resulting chain ({{validation}}, rule 7). An actor
identical to the lineage entry it would be placed atop appears once: a
workload continuing from its own hop does not repeat in the chain.
Collapse affects only the constructed `act` chain; the IdP's hop record
retains each continuation. Policy
MAY limit the lineage depth exposed to a given audience, subject to audit
requirements ({{privacy}}).

Nested lineage in an onward ID-JAG is therefore authenticated, not
self-reported. In this example the chain is `expense-service` (authenticated
at this continuation to Travel) acting for a delegation rooted by
`expense-app` (authenticated at root issuance). A workload that was delegated
to only over an offline attenuated segment, and never performed an exchange,
appears in deployment audit records rather than in lineage; actor receipts
{{I-D.mcguinness-oauth-actor-receipts}} and actor-signed hop proofs
{{I-D.mcguinness-oauth-actor-proofs}} provide a verifiable form for such
records.

TravelRAS is a continuation-aware Resource Authorization Server
({{ras-processing}}): it validates the ID-JAG per
{{I-D.ietf-oauth-identity-assertion-authz-grant}}, applies its local
authorization policy, issues the access token, and binds the
`identity_continuation_handle` claim to that authorization state, from which its
domain's Transaction Token Service and Chain Authority later derive
continuation. The handle never enters the access token or reaches TravelAPI
(rule 4). The `client_id` is the current actor's client identifier for the
target Resource Authorization Server ({{client-identity}}); in this document's
examples the workload identifier serves as both the client identifier and the
`act.sub` value, so the two match.

# Continuation-Aware Resource Authorization Server {#ras-processing}

A Resource Authorization Server that accepts continuation-capable ID-JAGs is
continuation-aware. On receiving an ID-JAG that carries an
`identity_continuation_handle` claim, it MUST:

1. validate the ID-JAG per {{I-D.ietf-oauth-identity-assertion-authz-grant}};
2. authenticate the client presenting it;
3. verify the sender constraint, that is, proof of possession of the
   confirmed key;
4. apply its local authorization policy;
5. issue the resulting access token; and
6. bind `identity_continuation_handle` to the authorization state it establishes,
   recording whether continuation is permitted.

Binding and access-token issuance MUST be atomic: a hop is bound only when a
token was actually issued. The Resource Authorization Server MUST NOT place
`identity_continuation_handle` in the access token or expose it to the protected API
(rule 4); it makes the binding available only to its own trust domain's
Transaction Token Service or Chain Authority, and only privately
({{transaction-token-context}}). A non-normative internal record:

~~~
authorization:
  idp_issuer: https://idp.example/
  tenant: tenant-123
  local_subject: expense-local-subject
  client: expense-app
  continuation_handle: H0
  continuation_allowed: true
  effective_authorization: expenses.read
  confirmation_key: <thumbprint>
  status: active
~~~

## Hop Activation {#hop-activation}

RAS acceptance activates a hop. The IdP records a hop as pending when it issues
the ID-JAG carrying the handle; the hop becomes continuable only after the
target Resource Authorization Server has accepted and bound it as above. There
is no Resource-Authorization-Server-to-IdP callback: a valid Identity
Continuation Assertion from a Chain Authority mapped to that hop's target
Resource Authorization Server ({{root-establishment}}) is the evidence of
acceptance the IdP relies on. A Chain Authority issues only from accepted,
currently-active authorization state, so a handle copied from an
issued-but-rejected ID-JAG is not continuable: no mapped Chain Authority will
attest it. RAS-acceptance provenance is therefore exactly a trust in the
mapped Chain Authority, an audit and integrity property within an honest
domain rather than an IdP-verifiable proof of acceptance ({{security}}).

## A Gate, Not a Ceiling {#ras-gate}

RAS acceptance is a continuation gate, not an authorization ceiling. The IdP
continues to evaluate every downstream target solely against the root-chain
envelope and current-actor policy ({{validation}}, rule 14); a Resource
Authorization Server's local effective authorization neither narrows nor
widens what a later continuation may obtain. Cross-domain scope vocabularies
are not generally comparable; RAS-derived narrowing, if ever defined, would
require signed constraints and an explicit intersection model (an open item,
{{open-items}}).

## Durable Task Authorization {#task-provenance}

An unattended or scheduled continuation MUST root in durable authorization
state created when a Resource Authorization Server accepts a root ID-JAG, not
in a handle stored by a scheduler or other component. Concretely, the platform
workload obtains a root ID-JAG for a platform Resource Authorization Server
(for example a task service), redeems it there, and that Resource
Authorization Server binds the root hop to durable task-authorization state as
in {{ras-processing}}. The scheduler thereafter holds only a task identifier,
never the `identity_continuation_handle`; the platform's Transaction Token
Service derives the handle from authenticated task state when a run begins
({{transaction-token-context}}). Theft of the scheduler's stored task
identifier therefore yields no continuable credential: the handle is not
stored there, and continuation still requires a valid assertion from a mapped
Chain Authority ({{ras-processing}}) bound to the run's authenticated actor.

# Security Considerations {#security}

This profile assumes TLS-protected channels between all parties and an IdP that
correctly holds the per-audience subject map and the root-chain envelope. The
general OAuth security guidance of {{RFC9700}} applies. The adversaries
specifically considered are:

* a network or on-path attacker observing or replaying a Token Exchange
  request, addressed by sender-constraining the assertion and single-use
  consumption ({{sender-constrained-presentation}}, {{security-replay}});
* a compromised or curious intermediate workload that holds propagated chain
  context and attempts to broaden authority, impersonate the user, or inflate
  the authentication context, bounded by the root-chain envelope and the actor
  binding ({{security-assurance}}, {{security-envelope}});
* a compromised Chain Authority, which can issue assertions only for chains
  it is trusted for, and whose assertions still require an authenticated,
  permitted actor ({{security-chain-authority}});
* a party that can influence the client-to-actor mapping
  ({{client-identity}}), which binds client authentication to actor
  identity and, on a direct request without an `actor_token`, is the sole
  authenticator of the root actor ({{root-establishment}});
* a malicious or curious Resource Server or Resource Authorization Server
  attempting to correlate the user across boundaries, addressed by keeping
  `identity_continuation_handle` out of access tokens and external authorization claims and by
  its per-hop, per-domain freshness, so no handle links a user across SaaS
  boundaries ({{privacy}}).

The subsections below address each in turn.

## Sender Constraint and Proof of Possession

The Identity Continuation Assertion MUST NOT be accepted as a bearer token
{{RFC7800}}: presentation requires live proof of possession of the `cnf`
key, bound to the authenticated current actor
({{sender-constrained-presentation}}). A stolen assertion or actor token is
therefore unusable without the confirmed private key, and the short
single-use lifetime ({{security-replay}}) bounds the window even for the
key holder.

## Short Lifetime and Replay {#security-replay}

The 300-second lifetime ceiling and the atomic single-use consumption of
(`iss`, `jti`) are enforced at validation ({{validation}}, rule 13 and the
consume-and-record step following the numbered rules).
Because the assertion is consumed only at the IdP and never by a Resource
Server, its blast radius on replay is confined to the continuation exchange.

## Root Authentication Context {#security-assurance}

The authoritative `auth_time`, `acr`, and `amr` values come only from the
root-chain envelope recorded by the IdP. They are not accepted from the Chain
Authority. Continuing a chain MUST NOT strengthen or refresh that context, and
the IdP MUST copy it unchanged into an onward ID-JAG when the output profile
requires those claims.

## Envelope Enforcement and Offline Attenuation {#security-envelope}

The root-chain envelope is the authorization ceiling for a chain. The IdP
enforces that the requested target and scopes are permitted by the envelope's
authorization basis ({{validation}}, rule 14), which prevents a compromised
intermediate actor from broadening the chain beyond what the root delegation
and its governing consent and policy authorize.

Where an offline attenuated delegation stack feeds an Identity Continuation
Assertion, that stack's own specification provides monotonic attenuation,
bounded depth, and parent linkage along the offline segment (for example,
{{I-D.li-oauth-delegated-authorization}}). The Chain Authority validates
the segment before issuing ({{context-provenance}}); the IdP enforces only
the root-chain envelope.

The assertion is deliberately target-agnostic ({{excluded-claims}}): within
the envelope and IdP policy, the presenting actor chooses the target at
exchange time, so a compromised permitted actor can select among all
authorities the ceiling allows during the assertion's lifetime. This is a
tradeoff for the uniform request shape; optional issuance-time narrowing
constraints are an open item ({{open-items}}).

A related risk for a multi-user intermediary is association error rather
than broadening: a workload holding many valid handles that attaches the
wrong handle to a request continues the wrong user's chain, within that
chain's envelope. Deployments MUST preserve the association between an
inbound transaction and its handle ({{context-provenance}}); standardizing a
carrier for that association is an open item ({{open-items}}).

## Trust in the Chain Authority {#security-chain-authority}

The IdP MUST accept assertions only from Chain Authorities it trusts for the
relevant tenant ({{validation}}, rule 6). A compromised Chain Authority can issue
assertions only for chains it is trusted for, and a continuation still
requires an authenticated actor permitted by the chain's continuation
authorization ({{validation}}, rule 10), within the ceiling of
{{security-envelope}}.
Deployments SHOULD scope Chain Authority trust as narrowly as practical
and SHOULD monitor for anomalous continuation patterns.
`identity_continuation_handle` confidentiality is defense in depth, not
an authorization factor.

How the IdP establishes that trust (a Chain Authority's issuer identifier,
its authorized signing keys, and its tenant scope) is out of band and
deployment-specific, statically or through a federation or trust-framework
mechanism, as for any issuer whose assertions an Authorization Server
consumes {{RFC7523}}. Because the assertion carries no subject
({{assertion-claims}}), such a mechanism authorizes the Chain Authority
against the owning authority resolved from `identity_continuation_handle`,
not against a subject carried in the assertion.

## Trust in Actor Token Issuers

Validation rule 10 depends on the `actor_token` authenticating the current
actor, which makes the issuer of that token a trust anchor of equal weight to
the Chain Authority. The IdP MUST accept actor tokens only from workload
identity issuers it trusts for the actor's trust domain and tenant, scoped
and established exactly as it scopes and establishes Chain Authority trust
(a workload identity system such as the WIMSE architecture
{{I-D.ietf-wimse-arch}} can serve as the federation mechanism). An actor
token from an untrusted or out-of-scope issuer MUST be rejected regardless
of the validity of the accompanying assertion.

A Chain Authority and the workload identity issuer for its domain are
commonly the same operator, such as a Resource Authorization Server or
domain gateway performing both roles; this is an expected deployment
shape. It concentrates the two trust anchors in one party, so a compromise
of that operator affects both; the root-chain envelope and IdP actor
policy remain the effective bound in that event.

## Actor Chain Integrity

Lineage in an onward ID-JAG is IdP-authenticated, never self-reported
({{onward-id-jag}}): an assertion names only the current actor, and the IdP
MUST reject an assertion whose `act` names anyone else ({{validation}},
rule 9). Actors of an offline attenuated segment appear in the evidence
layer ({{onward-id-jag}}) rather than in lineage; asserting a verified
own-domain segment is reserved for a future extension ({{open-items}}).

## Token, Type, and Algorithm Confusion {#security-alg}

The IdP MUST verify the JOSE `typ` header value
`oauth-identity-continuation+jwt` and MUST NOT process an Identity Continuation
Assertion as any other kind of JWT, in accordance with {{RFC8725}}. The IdP
MUST reject an assertion whose JOSE `alg` header is `none`, and MUST restrict
accepted algorithms to a configured allowlist of asymmetric signature
algorithms. Symmetric (MAC) algorithms MUST NOT be accepted. The IdP MUST
select the verification key from its trust configuration for the assertion
`iss`. A `kid` header parameter MAY select among verification keys already
configured for that issuer. The IdP MUST NOT use an untrusted `jku`, `x5u`,
embedded `jwk`, or other assertion-supplied key material to establish trust or
retrieve a verification key. General OAuth security guidance in {{RFC9700}}
applies.

# Privacy Considerations {#privacy}

An `identity_continuation_handle` is observed by the participants of a single hop: the
accepting Resource Authorization Server, in the ID-JAG it validates, that
domain's Transaction Token Service and Chain Authority, and the IdP. It never
reaches an access token or the protected API ({{chain-id}}, rule 4). Because
each hop's handle is distinct and each Resource Authorization Server sees only
its own hop's value and resolves only its own pairwise subject, neither a
Resource Server nor a Resource Authorization Server can correlate the user
across SaaS boundaries from what it holds.

This property does not make the complete chain unlinkable. The IdP necessarily
correlates the chain, and workloads, Chain Authorities, or other control-plane
participants that receive the same `identity_continuation_handle` can correlate their observations
of that delegation. In particular, the actor chain carried in each onward
ID-JAG names the participating workloads, so colluding audiences can correlate
a transaction from actor lineage and timing alone, regardless of how
`identity_continuation_handle` is handled. Deployments requiring unlinkability across audiences
must weigh the audit value of deep actor lineage against this correlation
channel and MAY limit the nested lineage exposed to each audience, subject to
their audit requirements. Deployments SHOULD limit disclosure of `identity_continuation_handle` to
participants that require it to continue or administer the chain.

The content and entropy constraints of {{chain-id}}, rule 2, keep an
`identity_continuation_handle` from revealing the user or being guessed.

Continuation handles are hop-specific by construction ({{chain-id-privacy}}),
so audiences and branches do not share a correlatable identifier. A workload
co-located with a
Resource Server that receives chain context alongside a request
({{transaction-token-context}}) is, for that data, a control-plane participant subject
to the disclosure limits above.

# IANA Considerations {#iana}

## OAuth URI Registration

IANA is requested to register the following value in the "OAuth URI" registry
established by {{RFC6755}} and used for token type identifiers by {{RFC8693}}.

URN:
: urn:ietf:params:oauth:token-type:identity-continuation

Common Name:
: Token type URI for the Identity Continuation Assertion

Change Controller:
: IETF

Specification Document:
: This document, {{names}}

## Media Type Registration

IANA is requested to register the following media type in the "Media Types"
registry, in the manner described in {{RFC6838}}, corresponding to the JOSE
`typ` header value `oauth-identity-continuation+jwt`.

Type name:
: application

Subtype name:
: oauth-identity-continuation+jwt

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: binary; an Identity Continuation Assertion is a JWT {{RFC7519}}, which is a
  series of base64url-encoded values (some of which may be empty) separated by
  period ('.') characters.

Security considerations:
: See {{security}} of this document.

Interoperability considerations:
: N/A

Published specification:
: This document, {{names}}

Applications that use this media type:
: Applications using OAuth 2.0 Token Exchange {{RFC8693}} to perform identity
  continuation across SaaS boundaries.

Fragment identifier considerations:
: N/A

Additional information:
: <br>
  Deprecated alias names for this type: N/A<br>
  Magic number(s): N/A<br>
  File extension(s): N/A<br>
  Macintosh file type code(s): N/A

Person & email address to contact for further information:
: Karl McGuinness (public@karlmcguinness.com)

Intended usage:
: COMMON

Restrictions on usage:
: N/A

Author:
: Karl McGuinness

Change controller:
: IETF

## JSON Web Token Claims Registration

IANA is requested to register the following claim in the "JSON Web Token Claims"
registry established by {{RFC7519}}.

Claim Name:
: identity_continuation_handle

Claim Description:
: An opaque, IdP-generated reference to one hop of a
  delegation chain, used to correlate a continuation to its chain
  and parent hop and to resolve the per-audience subject.

Change Controller:
: IESG

Specification Document(s):
: This document, {{chain-id}}

## OAuth Authorization Server Metadata Registration

IANA is requested to register the following value in the "OAuth Authorization
Server Metadata" registry established by {{RFC8414}}.

Metadata Name:
: identity_continuation_supported

Metadata Description:
: Boolean value indicating support for the Identity Continuation Assertion
  profile

Change Controller:
: IESG

Specification Document(s):
: This document, {{metadata}}

Note: The token type URI `urn:ietf:params:oauth:token-type:id-jag` referenced by
this document is registered by
{{I-D.ietf-oauth-identity-assertion-authz-grant}} and is not registered here.

--- back

# Design Rationale {#rationale}

This appendix is non-normative. It explains why the Identity Continuation
Assertion is a distinct subject token and why a cross-boundary hop cannot
reuse an existing token. Both trace to per-boundary trust: each target trusts
only the IdP to name the user and scope authority, and where subjects are
pairwise only the IdP can mint the next audience's subject ({{motivation}},
{{core-principle}}).

## Relationship to ID-JAG {#rationale-idjag}

This document profiles ID-JAG {{I-D.ietf-oauth-identity-assertion-authz-grant}}
in two ways: the IdP embeds the `identity_continuation_handle` claim in a
continuation-capable ID-JAG ({{chain-id}}, rule 1), and the accepting Resource
Authorization Server binds that claim to authorization state
({{ras-processing}}). Both the issuance and the acceptance of the ID-JAG are
therefore profiled here.

The Identity Continuation Assertion is nonetheless a distinct artifact, not
itself an ID-JAG. It is the input to a Token Exchange, a `subject_token` whose
`aud` is the IdP and which carries no top-level `sub`, whereas an ID-JAG is the
output: an authorization grant whose `aud` is the target Resource Authorization
Server and which carries the resolved per-audience `sub` and, when
continuation-capable, the `identity_continuation_handle` claim. Presenting the
assertion to a Resource Authorization Server is meaningless; it is the input
that produces a chained ID-JAG. Viewed by function, the assertion is a further
identity-continuity credential ({{root-establishment}}): where a refresh token
continues a grant for its own client, the assertion continues the delegation
for downstream actors.

## Why Not a Transaction Token {#rationale-txn}

A Transaction Token {{I-D.ietf-oauth-transaction-tokens}} is the closest
neighbor, a short-lived signed JWT carrying delegation context, often issued
by a service that could also be the Chain Authority, but it sits at a
different layer. It is intra-domain: its `aud` is a trust
domain that it "MUST NOT be accepted outside", it is re-minted as it propagates,
and its `sub` is domain-local. The assertion instead crosses to the IdP, is
consumed only there, is single-use, and carries neither the target subject
nor the request context a Transaction Token carries. It is derived from a
Transaction Token at the boundary, not a profile of one.

## Why Not a Cross-Domain Propagation Token {#rationale-propagation}

This profile does use offline propagation between hops wherever it is sound:
that is the offline attenuated delegation layer, used when the subject does
not change ({{decision-rule}}). It cannot serve the boundary where the subject
changes: re-subjecting is a mint only the
IdP can perform, not an attenuation a holder can carry ({{core-principle}}). A
source-minted token cannot compute the next audience's pairwise subject;
only the IdP holds the map ({{motivation}}). The receiver trusts the IdP, not
the source issuer, to name the user; obtaining a grant from an
already-trusted authority is the model of {{I-D.ietf-oauth-identity-chaining}}
(for example {{I-D.fletcher-transaction-token-chaining-profile}}). An offline
token remains usable without consulting the IdP, whereas a continuation
checks current IdP state at every hop ({{lifecycle}}). And a stolen assertion
permits only an envelope-bounded IdP exchange ({{security}}). Built honestly,
such a token collapses into this profile. Direct propagation fits only where all domains share one global
subject, mutually trust each other's issuers (for example, a single
SPIFFE-style trust domain), and accept the loss of mid-chain revocation, a
different problem from the pairwise-subject case this profile addresses.
Delegated Authorization {{I-D.li-oauth-delegated-authorization}}, whose
client-issued tokens carry no subject at all, composes with this profile as
the intra-domain layer ({{decision-rule}}) and stops exactly where
re-subjecting begins.

## Alternative Topology: Resolution at the Target {#rationale-pull}

A pull topology was evaluated: the workload presents its reference to the
target RAS under a new grant type; the RAS resolves it at the IdP over a back
channel {{RFC7662}}; a variant returns the ID-JAG itself, so the target uses
unchanged ID-JAG processing. It removes the Chain Authority and moves one
token round trip from the workload to the target's back channel, but every
target RAS must implement the new grant type and sees the reference.
This document standardizes push because the target consumes an ordinary
ID-JAG unchanged, concentrating new behavior in the few (IdP, Chain
Authority) rather than the many (RAS). Pull remains a candidate companion
profile over the same envelope.

## Why a Signed Assertion Rather Than a Bare Grant Type {#rationale-grant-type}

A bare grant type would present the handle under client authentication, with
a sender-constrained `actor_token` and live proof of possession, but without
an assertion JWT or Chain Authority. It loses little where the continuing
domain has no offline segment and no domain-local policy. The signed assertion was retained for
what the Chain Authority attests first-hand and the IdP cannot: validating an
intra-domain offline segment before the boundary, and vouching for its own
workloads' identity and keys. Because the Chain Authority does not see the
target of the eventual exchange ({{excluded-claims}}), it gates only
continuation, never target or scope, which are solely the IdP's to authorize;
with all lineage IdP-constructed ({{onward-id-jag}}), the assertion carries
exactly what the Chain Authority can attest first-hand, the authenticated
current actor and its key binding. Where a deployment needs neither
capability, the Chain Authority reduces to a co-signature over the handle and
the current actor, and the bare grant type remains a candidate simplification.

# Worked Example (Same-IdP) {#example}

This appendix is non-normative. This and the following appendices cover the
three deployment shapes this profile serves: interactive SaaS-to-SaaS
chaining (this appendix), an unattended background agent
({{example-background}}), and a gateway topology with dynamically determined
upstream audiences ({{example-gateway}}). This appendix walks the canonical
flow of {{flow}} end-to-end for a single user: ExpenseApp invokes
ExpenseSaaS; ExpenseService, the workload handling that request, calls
TravelAPI to reach TravelSaaS; and TravelService, the TravelSaaS workload
that handles that call, in turn calls BookingAPI to complete the
itinerary. All parties trust one enterprise IdP at `https://idp.example/`.

Proof of possession uses DPoP. JWTs are shown as decoded payloads; JOSE
headers and signatures are omitted. Client authentication is required on
every exchange ({{client-identity}}) and is omitted from all example
requests in these appendices for brevity. The values are consistent with
the examples in {{assertion-claims}}, {{ras-processing}},
{{transaction-token-context}}, and {{onward-id-jag}}.

`identity_continuation_handle` never crosses a trust boundary except
inside an ID-JAG or an Identity Continuation Assertion ({{chain-id}}, rule
3): it does not appear in a Token Exchange response or an HTTP request
field, and ExpenseApp, ExpenseService, and TravelService never exchange it
directly with one another. Within a domain it travels only in that
domain's own chain context. Each accepting Resource Authorization Server
binds the handle it receives to the authorization state it establishes
({{ras-processing}}), and each domain's Transaction Token Service derives
the handle from that bound state for its own workloads
({{transaction-token-context}}).

The participants and values used throughout:

| Name | Value | Description |
|------|-------|-------------|
| User | (none) | The human, authenticated once at the IdP. |
| IdP | `https://idp.example/` | Trust anchor and Continuation Authorization Server. |
| ExpenseApp | `expense-app` | User-facing application; first-hop client. |
| ExpenseSaaS | `https://expenses.example/` | First SaaS the user invokes. |
| ExpenseRAS | `https://ras.expenses.example/` | Resource Authorization Server for ExpenseAPI; binds each hop it accepts to its authorization state ({{ras-processing}}). |
| ExpenseAPI | `https://api.expenses.example/` | Protected resource behind ExpenseRAS. |
| ExpenseService | `expense-service` | ExpenseSaaS workload that continues the chain and calls TravelAPI. |
| Expense TTS | `https://tts.expenses.example/` | ExpenseSaaS's Transaction Token Service; derives the handle ExpenseRAS bound and issues ExpenseService's local Transaction Token ({{transaction-token-context}}). |
| Expense Chain Authority | `https://ca.expenses.example/` | ExpenseSaaS's Chain Authority; issues the Identity Continuation Assertion that continues a hop ExpenseRAS accepted. |
| TravelSaaS | `https://travel.example/` | Downstream SaaS. |
| TravelRAS | `https://ras.travel.example/` | Resource Authorization Server for TravelAPI; binds each hop it accepts to its authorization state ({{ras-processing}}). |
| TravelAPI | `https://api.travel.example/` | Protected resource behind TravelRAS. |
| TravelService | `travel-service` | TravelSaaS workload that processes the inbound request and calls BookingAPI. |
| Travel TTS | `https://tts.travel.example/` | TravelSaaS's Transaction Token Service; derives the handle TravelRAS bound and issues TravelService's local Transaction Token ({{transaction-token-context}}). |
| Travel Chain Authority | `https://ca.travel.example/` | TravelSaaS's Chain Authority; issues the Identity Continuation Assertion that continues a hop TravelRAS accepted. |
| BookingSaaS | `https://booking.example/` | Third SaaS, reached in the third hop. |
| BookingRAS | `https://ras.booking.example/` | Resource Authorization Server for BookingAPI; binds each hop it accepts to its authorization state ({{ras-processing}}). |
| BookingAPI | `https://api.booking.example/` | Protected resource behind BookingRAS. |
| PartnerSaaS | `https://partner.example/` | SaaS outside the trust circle ({{example-federation-edge}}). |
| Handles | `kW4uJ8pTe2NxA6rQvD1zYs` (H0, root hop), `Uc9fB3mHs5LdK7gEnX2wRj` (H1, Travel hop), `Ht6mZ2pQe8VrKx4NcWy1Jd` (H2, Booking hop) | Continuation handles; one per hop, each a claim of its own continuation-capable ID-JAG ({{chain-id}}). |
| Subjects | `expense-local-subject`, `travel-local-subject`, `booking-local-subject` | The user's pairwise subject at ExpenseRAS, TravelRAS, and BookingRAS, respectively. |

At a glance, the message flow is:

~~~
User authenticates once at the IdP.

ExpenseApp -> IdP: exchange ID Token
IdP -> ExpenseApp: ID-JAG(ExpenseRAS) carrying H0
ExpenseApp -> ExpenseRAS: ID-JAG
ExpenseRAS -> ExpenseApp: AT1 (ExpenseRAS binds H0)
ExpenseApp -> ExpenseAPI: AT1
Expense TTS -> ExpenseService: Transaction Token
  (tctx.identity_continuation = H0)

ExpenseService -> Expense Chain Authority: request assertion for H0
Expense Chain Authority -> ExpenseService:
  Identity Continuation Assertion
ExpenseService -> IdP: assertion exchange + DPoP
IdP -> ExpenseService: ID-JAG(TravelRAS) carrying H1
ExpenseService -> TravelRAS: ID-JAG
TravelRAS -> ExpenseService: AT2 (TravelRAS binds H1)
ExpenseService -> TravelAPI: AT2
Travel TTS -> TravelService: Transaction Token
  (tctx.identity_continuation = H1)

TravelService -> Travel Chain Authority: request assertion for H1
Travel Chain Authority -> TravelService:
  Identity Continuation Assertion
TravelService -> IdP: assertion exchange + DPoP
IdP -> TravelService: ID-JAG(BookingRAS) carrying H2
TravelService -> BookingRAS: ID-JAG
BookingRAS -> TravelService: AT3 (BookingRAS binds H2)
TravelService -> BookingAPI: AT3
~~~

The subsections below detail each step; {{example-third-hop}} then repeats
the continuation pattern for a third hop to BookingRAS, this time from
within TravelSaaS.

## First Hop: Direct ID-JAG for ExpenseRAS {#example-first-hop}

ExpenseApp holds an ID Token for the authenticated user and exchanges it at the
IdP for an ID-JAG scoped to ExpenseRAS. The request is DPoP-bound to
ExpenseApp's key:

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof signed by the expense-app key>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
requested_token_type=urn:ietf:params:oauth:token-type:id-jag
audience=https://ras.expenses.example/
resource=https://api.expenses.example/
scope=expenses.read
subject_token=<id_token>
subject_token_type=urn:ietf:params:oauth:token-type:id_token
actor_token=<sender-constrained expense-app credential>
actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

The IdP authenticates the user from the ID Token, resolves the anchoring
session from its `sid` ({{root-establishment}}), and verifies that
ExpenseApp's actor credential is constrained to the DPoP key. The
authorization attached to that session
is continuation-capable: existing user consent and enterprise policy
permit continuation, authorize the immediate Expense target and the later
Travel and Booking targets, and designate the Travel and Booking workloads
as permitted continuers, so the IdP establishes a chain
({{root-establishment}}). It records the following target entries in the
root-chain envelope:

~~~
(https://ras.expenses.example/, https://api.expenses.example/)
    permitted scopes: expenses.read

(https://ras.travel.example/, https://api.travel.example/)
    permitted scopes: trips.read

(https://ras.booking.example/, https://api.booking.example/)
    permitted scopes: stays.book
~~~

The envelope also records the continuation authorization, the governing
authorization, and the chain's expiry; the root exchange does not authorize
the later targets merely by requesting the first ({{root-establishment}}).
A deployment whose onward targets are not knowable at establishment records
the ceiling without enumeration and evaluates each target at continuation
time ({{validation}}, rule 14).

The IdP creates a fresh root hop, H0, for this chain and embeds it as a
claim of the ID-JAG it is about to issue ({{chain-id}}, rule 1); the hop is
PENDING until a Resource Authorization Server accepts it ({{ras-processing}}).
The Token Exchange response carries no continuation-specific member: the
handle travels only inside the ID-JAG.

~~~ json
{
  "access_token": "<ID-JAG for ExpenseRAS>",
  "issued_token_type": "urn:ietf:params:oauth:token-type:id-jag",
  "token_type": "N_A",
  "expires_in": 300
}
~~~

The decoded ID-JAG for ExpenseRAS carries the user's ExpenseRAS-local
subject and the root hop's handle:

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.expenses.example/",
  "sub": "expense-local-subject",

  "client_id": "expense-app",
  "resource": "https://api.expenses.example/",
  "scope": "expenses.read",

  "auth_time": 1710000000,
  "acr": "urn:example:loa:2",
  "amr": ["pwd", "mfa"],

  "identity_continuation_handle": "kW4uJ8pTe2NxA6rQvD1zYs",

  "cnf": {
    "jkt": "base64url-expense-app-key-thumbprint"
  },

  "iat": 1710000005,
  "exp": 1710000305,
  "jti": "idjag-expense-01"
}
~~~

## ExpenseRAS Acceptance and the Expense-Domain Chain Context {#example-context}

ExpenseApp exchanges this ID-JAG at ExpenseRAS for an access token (AT1),
exactly as for any ID-JAG {{I-D.ietf-oauth-identity-assertion-authz-grant}}
(not shown), except that ExpenseRAS also recognizes the continuation grant
profile and processes `identity_continuation_handle` ({{ras-processing}}).
ExpenseRAS validates the ID-JAG, authenticates ExpenseApp, verifies the
DPoP proof, and applies its local policy; only if every check and the
access-token issuance itself succeed does it atomically bind H0 to the
authorization state behind AT1, moving the hop from PENDING to ACCEPTED. A
hop that never reaches ACCEPTED, for example one copied from an ID-JAG
that ExpenseRAS rejected, is not usable: no Chain Authority attests a hop
that its Resource Authorization Server never accepted.

ExpenseRAS keeps this association in a private, own-domain record; it is
never serialized into AT1, an external authorization claim, or anything
ExpenseAPI's callers observe:

~~~ json
{
  "identity_continuation_handle": "kW4uJ8pTe2NxA6rQvD1zYs",
  "status": "ACCEPTED",
  "authorization_state": "at1-authz-2f9c",
  "client_id": "expense-app",
  "bound_at": 1710000010
}
~~~

ExpenseApp calls ExpenseAPI with AT1. The Expense TTS, ExpenseSaaS's own
Transaction Token Service, resolves AT1 against the record ExpenseRAS just
created over their shared, own-domain interface ({{ras-processing}}),
derives H0 from it, and issues a local Transaction Token for
ExpenseService, the workload that will complete the request:

~~~ json
{
  "iss": "https://tts.expenses.example/",
  "aud": "https://expenses.example/",
  "sub": "expense-local-subject",
  "purp": "expense-report:complete",
  "txn": "txn-expense-88f2",

  "tctx": {
    "identity_continuation": {
      "iss": "https://idp.example/",
      "tenant": "acme-corp",
      "handle": "kW4uJ8pTe2NxA6rQvD1zYs"
    }
  },

  "iat": 1710000012,
  "exp": 1710000072,
  "jti": "tt-expense-0007"
}
~~~

The `tctx` object is the Transaction Token's namespaced context extension
point; `identity_continuation` is the member this profile defines within
it ({{transaction-token-context}}). `iss` and `tenant` name the IdP and
tenant whose chain state the handle indexes, the same pair a Chain
Authority needs to address the right token endpoint and apply the right
tenant-scoped policy when it later issues an assertion; `handle` is the
opaque hop reference itself. The Expense TTS mints this token for the
`https://expenses.example/` trust domain, not for any audience outside it
({{rationale-txn}}): as usual for a Transaction Token
{{I-D.ietf-oauth-transaction-tokens}}, a further ExpenseSaaS workload that
receives it re-mints its own Transaction Token for its next internal call
rather than forwarding this one unchanged, and every re-minted token
carries `tctx.identity_continuation` forward unchanged. No HTTP request
field and no Token Exchange response parameter carries H0; its only
cross-workload representations are this Transaction Token claim inside
ExpenseSaaS, and the ID-JAG and Identity Continuation Assertion that cross
a trust boundary ({{chain-id}}, rule 3).

## Obtaining the Identity Continuation Assertion {#example-ica}

ExpenseService needs to call TravelSaaS to complete the request. It
requests an Identity Continuation Assertion from `https://ca.expenses.example/`,
ExpenseSaaS's own Chain Authority, presenting the `identity_continuation_handle`
carried in its Transaction Token and proving control of its key so the
Chain Authority can bind `cnf` to it ({{assertion-issuance}}). The Chain
Authority is authorized to attest continuation from H0 because ExpenseRAS,
the Resource Authorization Server that accepted it, belongs to this Chain
Authority's domain, and the IdP's per-hop authority map designates this
Chain Authority for that domain ({{root-establishment}}); a Chain Authority
never attests a hop accepted by another domain's Resource Authorization
Server. The Chain Authority returns:

~~~ json
{
  "iss": "https://ca.expenses.example/",
  "aud": "https://idp.example/",
  "identity_continuation_handle": "kW4uJ8pTe2NxA6rQvD1zYs",

  "act": {
    "iss": "https://expenses.example/",
    "sub": "expense-service"
  },

  "cnf": {
    "jkt": "base64url-expense-service-key-thumbprint"
  },

  "iat": 1710000020,
  "exp": 1710000200,
  "jti": "continuation-assertion-01"
}
~~~

## Chained Exchange for the TravelRAS ID-JAG {#example-chained}

ExpenseService presents the assertion to the IdP as the `subject_token`,
DPoP-bound to its own key:

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof signed by the expense-service key>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
requested_token_type=urn:ietf:params:oauth:token-type:id-jag
audience=https://ras.travel.example/
resource=https://api.travel.example/
scope=trips.read
subject_token=<identity-continuation-assertion>
subject_token_type=<identity-continuation-token-type>
actor_token=<sender-constrained expense-service credential>
actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

The `subject_token_type` value above is
`urn:ietf:params:oauth:token-type:identity-continuation`. The IdP runs the
checks of {{validation}}: the DPoP key matches both the assertion's
`cnf.jkt` and the actor token's key confirmation, `expense-service` is the
actor named in `act`, H0 is CONTINUABLE, and the requested TravelRAS,
TravelAPI, and `trips.read` values match the Travel target entry in the
root-chain envelope. The IdP does not call ExpenseRAS back to confirm
acceptance: the valid assertion from ExpenseSaaS's mapped Chain Authority,
`https://ca.expenses.example/`, an authorized issuer for H0's domain, is
itself the evidence the IdP relies on that ExpenseRAS reached ACCEPTED
state for H0; that evidence is what makes H0 CONTINUABLE
({{ras-processing}}).

The IdP then resolves the user's TravelRAS-local subject, creates a fresh
hop H1 whose immutable parent is H0, and returns:

~~~ json
{
  "access_token": "<ID-JAG for TravelRAS>",
  "issued_token_type": "urn:ietf:params:oauth:token-type:id-jag",
  "token_type": "N_A",
  "expires_in": 300
}
~~~

The decoded ID-JAG for TravelRAS carries H1 and the newly constructed
`act` chain ({{onward-id-jag}}): `expense-service`, authenticated at this
exchange, placed atop the root actor `expense-app`. `travel-service` has
not yet performed an exchange, so it is not part of the lineage:

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.travel.example/",
  "sub": "travel-local-subject",

  "client_id": "expense-service",
  "resource": "https://api.travel.example/",
  "scope": "trips.read",

  "auth_time": 1710000000,
  "acr": "urn:example:loa:2",
  "amr": ["pwd", "mfa"],

  "identity_continuation_handle": "Uc9fB3mHs5LdK7gEnX2wRj",

  "act": {
    "iss": "https://expenses.example/",
    "sub": "expense-service",
    "act": {
      "iss": "https://expenses.example/",
      "sub": "expense-app"
    }
  },

  "cnf": {
    "jkt": "base64url-expense-service-key-thumbprint"
  },

  "iat": 1710000025,
  "exp": 1710000325,
  "jti": "idjag-travel-01"
}
~~~

## TravelRAS Acceptance and the Travel-Domain Chain Context {#example-use}

ExpenseService exchanges the TravelRAS ID-JAG at TravelRAS for an access
token (AT2), presenting a fresh DPoP proof with the same expense-service
key. TravelRAS recognizes the continuation grant profile just as
ExpenseRAS did: it validates the ID-JAG, authenticates ExpenseService,
and, on success, atomically binds H1 to the authorization state behind
AT2, exactly as {{example-context}} describes for ExpenseRAS and H0.

ExpenseService calls TravelAPI with AT2. The Travel TTS derives H1 from
that bound state and issues a local Transaction Token for TravelService,
the TravelSaaS workload that receives the request:

~~~ json
{
  "iss": "https://tts.travel.example/",
  "aud": "https://travel.example/",
  "sub": "travel-local-subject",
  "purp": "trip:process",
  "txn": "txn-travel-3a04",

  "tctx": {
    "identity_continuation": {
      "iss": "https://idp.example/",
      "tenant": "acme-corp",
      "handle": "Uc9fB3mHs5LdK7gEnX2wRj"
    }
  },

  "iat": 1710000032,
  "exp": 1710000092,
  "jti": "tt-travel-0004"
}
~~~

This Transaction Token stays inside TravelSaaS, minted for the
`https://travel.example/` trust domain, mirroring {{example-context}}.
`tctx.identity_continuation` carries the same `iss` and `tenant` as
ExpenseSaaS's did, because both hops are rooted at the same IdP and
tenant; only `handle` differs, from H0 to H1. The value that changes hop
to hop is the handle, not the routing information around it.

## Third Hop: TravelService Continues to BookingRAS {#example-third-hop}

Completing the itinerary requires a reservation at BookingSaaS
(`https://booking.example/`), whose API `https://api.booking.example/`
sits behind `https://ras.booking.example/`. TravelService, processing the
inbound request whose Transaction Token carries H1, requests an Identity
Continuation Assertion from `https://ca.travel.example/`, TravelSaaS's own
Chain Authority, authorized for H1 because TravelRAS accepted it
({{root-establishment}}), exactly as ExpenseService did in {{example-ica}}:

~~~ json
{
  "iss": "https://ca.travel.example/",
  "aud": "https://idp.example/",
  "identity_continuation_handle": "Uc9fB3mHs5LdK7gEnX2wRj",

  "act": {
    "iss": "https://travel.example/",
    "sub": "travel-service"
  },

  "cnf": {
    "jkt": "base64url-travel-service-key-thumbprint"
  },

  "iat": 1710000040,
  "exp": 1710000340,
  "jti": "continuation-assertion-02"
}
~~~

TravelService exchanges the assertion, DPoP-bound to its own key, for an
ID-JAG with `audience=https://ras.booking.example/`,
`resource=https://api.booking.example/`, and `scope=stays.book`, all
within the envelope's Booking target entry. The IdP creates a fresh hop
H2 whose immutable parent is H1 and constructs the onward `act` chain
({{onward-id-jag}}): `travel-service`, authenticated at this exchange,
placed atop the presented hop's lineage (`expense-service`, then
`expense-app`):

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.booking.example/",
  "sub": "booking-local-subject",

  "client_id": "travel-service",
  "resource": "https://api.booking.example/",
  "scope": "stays.book",

  "auth_time": 1710000000,
  "acr": "urn:example:loa:2",
  "amr": ["pwd", "mfa"],

  "identity_continuation_handle": "Ht6mZ2pQe8VrKx4NcWy1Jd",

  "act": {
    "iss": "https://travel.example/",
    "sub": "travel-service",
    "act": {
      "iss": "https://expenses.example/",
      "sub": "expense-service",
      "act": {
        "iss": "https://expenses.example/",
        "sub": "expense-app"
      }
    }
  },

  "cnf": {
    "jkt": "base64url-travel-service-key-thumbprint"
  },

  "iat": 1710000045,
  "exp": 1710000345,
  "jti": "idjag-booking-01"
}
~~~

TravelService, having performed this exchange, exchanges the ID-JAG at
BookingRAS for an access token (AT3), presenting a fresh DPoP proof with
the same key. BookingRAS binds H2 to AT3's authorization state, exactly as
ExpenseRAS and TravelRAS bound H0 and H1, and TravelService then calls
BookingAPI with AT3.

TravelService continues the chain itself, before crossing the boundary:
the current-domain actor is the one that obtains the next ID-JAG, not a
sibling workload it might hand the handle to ({{root-establishment}}).
Local fan-out within TravelSaaS ahead of a continuation can still use an
offline attenuated stack where the subject does not change
({{decision-rule}}); this hop does not need one.

## Reaching a Target Outside the Trust Circle {#example-federation-edge}

Suppose TravelSaaS must also call PartnerSaaS at `https://partner.example/`,
whose Resource Authorization Server does not trust `idp.example`. The chain
cannot continue there: the IdP holds no pairwise subject for that audience
and no authorization basis covers it, so a continuation request for that
target fails ({{validation}}, rules 14 and 16; `invalid_target`). This is
the profile's boundary, not a deployment error: continuation serves the set
of Resource Authorization Servers that trust the common IdP.

The crossing instead follows the identity-chaining model
{{I-D.ietf-oauth-identity-chaining}}: travel.example's own authorization
server (a distinct role from TravelSaaS's Chain Authority, which never
resolves target subjects) issues a grant that PartnerSaaS's server accepts
under a bilateral trust agreement, for example by exchanging TravelService's
Transaction Token, the same one that carries `tctx.identity_continuation`
inside TravelSaaS, under the Transaction Token chaining profile
{{I-D.fletcher-transaction-token-chaining-profile}}. The subject presented
to the partner is mapped by travel.example's authorization server under
that agreement, not by the IdP: the trust direction of
{{rationale-propagation}}, inverted by explicit federation.

The handle stays behind. PartnerSaaS is outside the trust circle, so no
ID-JAG or Identity Continuation Assertion names it as an audience, and no
Travel-domain Transaction Token reaches it ({{chain-id}}, rule 3;
{{transaction-token-context}}); the chain simply ends at the federation
edge. Audit continuity across the two legs is deployment-defined, for
example through the chaining profile's `txn` transaction identifier and
the evidence layer of {{assertion-claims}}.

# Background Agent Example (User-Scheduled Continuation) {#example-background}

This appendix is non-normative. It applies the continuation machinery of this
document to a background agent: the user is present once, when the task is
created, and absent at every run. Run-time authority still comes from a
fresh, sender-constrained assertion evaluated against the root-chain
envelope, exactly as in {{example}}; nothing about the IdP's validation
changes for a scheduled task.

What is different from an interactive chain is the rooting. The task's
authority is rooted in durable, platform-owned authorization state, created
by an explicit Resource Authorization Server for the platform's own
task-provisioning domain, a TaskRAS, when it accepts the root ID-JAG. This is
the pattern {{task-provenance}} requires of every background or scheduled
continuation: the platform's own RAS creates the durable task authorization,
bound to the resulting hop reference, before the Scheduler ever learns
anything about the task. The Scheduler is given only an opaque task
identifier; it never receives, stores, or transmits a handle. Without this
root, the example would fall back to a trusted sidecar handle store, exactly
what this architecture avoids.

The participants and values used throughout:

| Name | Value | Description |
|------|-------|-------------|
| User | (none) | The human, Alice; present at setup, absent at every run. |
| IdP | `https://idp.example/` | Trust anchor and Continuation Authorization Server. |
| BriefingAgent | `briefing-agent` | Agent platform workload that runs the scheduled task. |
| Platform | `https://platform.example/` | BriefingAgent's trust domain and actor issuer. |
| PlatformRAS (TaskRAS) | `https://ras.platform.example/` | Resource Authorization Server for the platform's own task-provisioning domain; accepts the root ID-JAG and creates durable task authorization bound to the resulting hop. |
| TaskAPI | `https://api.platform.example/tasks` | Protected resource behind PlatformRAS; the `resource` of the root ID-JAG. |
| Platform TTS | (internal) | Platform's Transaction Token Service; resolves a task identifier to its hop's authorization state for each run and carries that state intra-domain. |
| Chain Authority | `https://ca.platform.example/` | The platform's Chain Authority; issues Identity Continuation Assertions for its workloads, including the run-time continuation from the root hop. |
| CalendarRAS | `https://ras.calendar.example/` | Resource Authorization Server for the calendar service. |
| CalendarAPI | `https://api.calendar.example/` | Protected resource behind CalendarRAS. |
| Scheduler | (internal) | Platform component that triggers each run; holds only a task identifier. |
| MailRAS | `https://ras.mail.example/` | Resource Authorization Server for the mail upstream ({{example-dynamic}}). |
| MailAPI | `https://api.mail.example/` | Mail protected resource ({{example-dynamic}}). |
| Handles | `Pz6vTq1NcY4kM8bJf3RxWa` (H0, the root hop), `Qh2xNf9LpVb6tKdWs4RzYc` (this run's Calendar hop) | H0 persists across runs; each run's onward hop is a fresh child of H0. |
| Task Identifier | `task-123` | Opaque identifier; the only value the Scheduler ever stores. |
| Subjects | `alice-task-subject`, `alice-calendar-subject` | Alice's pairwise subject at PlatformRAS/TaskAPI and at CalendarRAS, respectively. |

## Setup (Alice Present)

Alice schedules "summarize my calendar every morning" and authorizes it in an
active session. `briefing-agent`, the platform workload, authenticates as
the OAuth client and performs the direct ID-JAG exchange of
{{token-exchange}}, targeting the platform's own TaskRAS rather than the
calendar service directly. Because the task must outlive Alice's session,
her consent takes the form OAuth already has for durable delegation: a
continuation-capable grant to the platform with refresh capability and an
explicit maximum lifetime. In an OpenID Connect deployment this is
typically a grant requested with the `offline_access` scope {{OIDC.Core}},
combined with the consent that makes it continuation-capable
({{root-establishment}}). The platform presents a refresh token from that
grant as the `subject_token`, so the chain's governing authorization is
anchored to the grant ({{root-establishment}}, {{lifecycle}}), with
`briefing-agent` as the permitted continuer. The IdP records two target
entries in the root-chain envelope: the root hop's own target, and the
task's actual need, authorized for later continuation:

~~~
(https://ras.platform.example/, https://api.platform.example/tasks)
    permitted scopes: task.manage

(https://ras.calendar.example/, https://api.calendar.example/)
    permitted scopes: calendar.read
~~~

The IdP records the envelope, with `briefing-agent` as the authenticated
root actor, and responds with the root ID-JAG. There is no separate
response parameter carrying the hop reference; it is a claim of the ID-JAG
itself:

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.platform.example/",
  "sub": "alice-task-subject",

  "client_id": "briefing-agent",
  "resource": "https://api.platform.example/tasks",
  "scope": "task.manage",

  "auth_time": 1712000000,
  "acr": "urn:example:loa:2",
  "amr": ["pwd", "mfa"],

  "identity_continuation_handle": "Pz6vTq1NcY4kM8bJf3RxWa",

  "cnf": {
    "jkt": "base64url-briefing-agent-key-thumbprint"
  },

  "iat": 1712000005,
  "exp": 1712000305,
  "jti": "idjag-platform-task-01"
}
~~~

`briefing-agent` redeems this ID-JAG at PlatformRAS, which recognizes the
continuation grant profile and processes it accordingly ({{ras-processing}}):
it validates the ID-JAG, authenticates `briefing-agent` as the OAuth
client, verifies the DPoP proof, applies its own local policy, and issues
an access token; on that success it atomically binds H0 to newly created,
durable task authorization, the ACCEPTED state of the hop. The request also
conveys the schedule and purpose by deployment-specific means, and
PlatformRAS records what it receives alongside the authorization it
creates. Only from ACCEPTED state can the platform's
Chain Authority ever attest a continuation of H0; a handle copied out of a
rejected or abandoned exchange is not continuable, because no Chain
Authority will attest a hop that never reached that state. PlatformRAS's
acceptance is a continuation gate, not an authorization ceiling: it
establishes that `briefing-agent` may continue from H0 at all, but the
root-chain envelope recorded above remains the sole ceiling for what H0
can reach, including the Calendar target and, later, Mail.

The durable task authorization this creates, keyed by the task identifier
PlatformRAS assigns, holds no bearer credential:

~~~
task_id:              task-123
  owner:               alice
  actor:               briefing-agent
    # the authorized continuer
  continuation_handle: Pz6vTq1NcY4kM8bJf3RxWa
    # H0; bound at PlatformRAS
  permitted_purpose:   morning-calendar-brief
  schedule:            "0 7 * * *"
  governing_grant:     grant-8f2c19a4
    # internal ref to the governing authorization
  expiry:              1719450000
    # advisory, local to PlatformRAS
  status:              active
~~~

The Scheduler stores only:

~~~
task_id: task-123
~~~

It never receives, stores, or transmits H0 or any other credential;
`task-123` identifies a row in PlatformRAS's own durable state and means
nothing outside the platform.

## Each Run (Alice Absent)

At a glance, each run proceeds as follows:

~~~
Scheduler      -> Platform:       run task-123 (task identifier only)
Platform TTS   -> Platform TTS:   resolve task-123 to its ACCEPTED
                                  PlatformRAS authorization state,
                                  deriving H0
Platform TTS   -> BriefingAgent:  authenticated run context; H0
                                  carried only by a Transaction
                                  Token, intra-domain
BriefingAgent  -> ChainAuthority: assertion for H0, bound to the
                                  briefing-agent key
ChainAuthority -> BriefingAgent:  Identity Continuation Assertion
BriefingAgent  -> IdP:            Token Exchange (assertion + DPoP)
IdP:                              validate per the rules of the
                                  validation section; resolve Alice's
                                  Calendar subject; mint a fresh
                                  child hop of H0
IdP            -> BriefingAgent:  ID-JAG(CalendarRAS) carrying that
                                  child hop as a claim
BriefingAgent  -> CalendarRAS:    ID-JAG -> access token (DPoP);
                                  CalendarRAS binds the child hop
BriefingAgent  -> CalendarAPI:    read calendar -> summarize
~~~

The Scheduler's trigger carries nothing but the task identifier. Platform
TTS looks up `task-123`, confirms that PlatformRAS's authorization state
for it is still ACCEPTED (not revoked, not expired), and starts the run
with H0 carried only as the intra-domain chain context of a Transaction
Token ({{transaction-token-context}}):

~~~
tctx.identity_continuation = {
  iss:    "https://idp.example/",
  tenant: "example-corp",
  handle: "Pz6vTq1NcY4kM8bJf3RxWa"
}
~~~

Here `iss` names the IdP that minted H0, not PlatformRAS, whose own
authorization state for H0 the Chain Authority checks separately below;
the value lets the Chain Authority know which IdP to address its
assertion to. That context stays inside the platform's own transaction
chain.
`briefing-agent`, running within it, reads H0 from this authenticated
run context rather than fetching or storing it independently, and no
message from the Scheduler ever carries it.

`briefing-agent` requests an Identity Continuation Assertion from the
platform's own Chain Authority for H0, proving control of its key. Before
issuing, the Chain Authority independently authenticates `briefing-agent`,
verifies its proof of possession, binds it as the actor for this
transaction, rejects any caller-supplied handle that does not match the
authenticated run context, and rechecks that PlatformRAS's authorization
state for H0 is still active: a Transaction Token is not itself a workload
credential, so trusting its contents alone would not be enough. The
assertion is the ordinary artifact of {{assertion-claims}}, issued by the
platform's own Chain Authority for its own workload ({{assertion-issuance}}):
`iss` is `https://ca.platform.example/`, `aud` is the IdP,
`identity_continuation_handle` is H0, the `act` claim is `briefing-agent`,
and `cnf` binds the agent's key.

The onward ID-JAG carries Alice's Calendar-local subject, an `act` chain of
`briefing-agent` (the root and continuing actor are the same workload
here), Alice's real authentication context from setup, and a fresh hop
reference for this run:

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.calendar.example/",
  "sub": "alice-calendar-subject",

  "client_id": "briefing-agent",
  "resource": "https://api.calendar.example/",
  "scope": "calendar.read",

  "auth_time": 1712000000,
  "acr": "urn:example:loa:2",
  "amr": ["pwd", "mfa"],

  "act": {
    "iss": "https://platform.example/",
    "sub": "briefing-agent"
  },

  "identity_continuation_handle": "Qh2xNf9LpVb6tKdWs4RzYc",

  "cnf": {
    "jkt": "base64url-briefing-agent-key-thumbprint"
  },

  "iat": 1712900001,
  "exp": 1712900301,
  "jti": "idjag-cal-alice-01"
}
~~~

The `identity_continuation_handle` above is this run's child of H0, not H0
itself: a fresh value the IdP mints for this continuation. A run tomorrow
presents H0 again and receives a different child of its own; both are
sibling hops sharing only H0 as their parent ({{chain-id}}). `briefing-agent`
redeems this ID-JAG at CalendarRAS for an access token; CalendarRAS
validates it, applies its own policy, issues the token, and atomically
binds the run's hop to its own authorization state ({{ras-processing}}),
the same acceptance-time binding PlatformRAS performed for H0 at setup.
`briefing-agent` then calls CalendarAPI, reads Alice's calendar, and
produces the summary.

Had this run also needed `https://api.mail.example/` behind
`https://ras.mail.example/` ({{example-dynamic}}), `briefing-agent` would
present H0 again for a second assertion and receive a second, independent
child hop bound at MailRAS. That hop and the Calendar hop above share only
their parent H0; neither carries the other's lineage.

## Points Worth Noticing

The `auth_time` is honest and inherited: it is fixed at the root exchange
in Setup and carried unchanged through every continuation, so it can
record when Alice actually authenticated days before this run
({{security-assurance}}). CalendarRAS decides whether that staleness is
acceptable for `calendar.read`; nothing presents the run as a fresh login.

A stolen copy of the durable task record, or of `task-123` itself, yields
only H0 and the record's other fields. H0 is useless to whoever took it:
without the platform's own agent key it cannot back a sender-constrained
assertion, and without a valid assertion from the platform's mapped Chain
Authority, issued only from PlatformRAS's currently ACCEPTED authorization
state, it cannot be presented to the IdP at all. The task record is not a
bearer credential; theft of it does not, by itself, authorize anything.

The chain fails closed. When Alice revokes the anchoring grant, the next
continuation attempt fails validation ({{validation}}, rule 7) and cannot
succeed by retrying: the platform's signal to seek re-authorization from
Alice. Three lifetimes are distinct here and should not be conflated: the
short redemption lifetime of any one ID-JAG; PlatformRAS's own local
authorization state for H0, including the advisory `expiry` in the task
record above; and the chain's authoritative continuation lifetime and
revocation, which live solely at the IdP ({{lifecycle}}). If the platform
wants advance warning before a chain lapses, that warning belongs in
authenticated task state it already controls, refreshed through its own
management interface or an optional claim, not in a Token Exchange
response parameter; the task record's `expiry` field is exactly this kind
of local, advisory value; it is never authoritative over the IdP's own
chain lifetime.

This pattern requires a user-present setup event to root the chain. Where
no such event exists (for example, an administratively mandated agent
acting for users who never authorized it), there is no delegation to
continue and this profile does not apply; such deployments need a
differently rooted authorization, such as administrative policy at the
IdP, which is out of scope for this document.

## A Dynamic Target {#example-dynamic}

Suppose the platform later extends the briefing to include unread mail,
which requires `https://api.mail.example/` behind
`https://ras.mail.example/`: a target nobody named when Alice created the
task. Under the target entries recorded in the setup above, a run's
continuation exchange presenting H0 for that audience fails, and the
chain is otherwise unaffected:

~~~
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "invalid_target"
}
~~~

For a deployment that expects dynamic targets, the envelope's basis is
Alice's standing consent (for example, a productivity read-access grant she
holds) and tenant policy, with no enumerated targets, and the IdP evaluates
it at continuation time ({{validation}}, rule 14). The same exchange then
succeeds only if read access to the mail service is within Alice's standing
consent and tenant policy permits `briefing-agent` to reach it. A request
for `mail.send`, outside that consent, fails with `invalid_scope`.

The envelope guardrail applies regardless of how a hop was rooted:
evaluation is bounded by the consent and policy in force when the chain
was established, never by whatever local policy PlatformRAS applied when
it accepted H0. If the tenant later broadens policy, existing chains do
not gain the new authority; if it narrows, they lose it at the next run.
Both failures are per-request (`invalid_target`, `invalid_scope`), and the
chain remains continuable for authorized targets, in contrast to a dead
chain, which fails every request ({{validation}}, rule 7).

# Gateway Example (Dynamic Upstream Audiences) {#example-gateway}

This appendix is non-normative. It applies the continuation machinery to a
gateway or proxy topology, common in AI tool-calling deployments: the runtime
that starts the user's session routes tool calls through a gateway that
aggregates upstream services, each protected by its own Resource
Authorization Server trusting the same IdP. The defining tension is that the
session runtime cannot know which upstream a given tool call will require,
while the gateway, which does know, holds no end-user credential addressed to
it. Neither party can be given the other's power without harm: the runtime
must not mint arbitrary cross-domain grants, and the gateway must not present
an identity assertion whose audience it is not. This example also retires the
client-supplied handle of an earlier design: agent-app never holds a
continuation handle to attach to a tool call, and the point in the flow where
that matters is called out below.

The participants and values used throughout:

| Name | Value | Description |
|------|-------|-------------|
| User | (none) | The human, Alice; her session runs in AgentApp. |
| IdP | `https://idp.example/` | Trust anchor and Continuation Authorization Server. |
| AgentApp | `agent-app` | Confidential agent runtime; hosts Alice's session and roots the chain. |
| AgentPlatform | `https://agent.example/` | AgentApp's trust domain and actor issuer. |
| Gateway | `tool-gateway` | Tool gateway workload; receives Alice's tool calls and continues the chain per call. |
| GatewayPlatform | `https://gateway.example/` | The gateway's trust domain and actor issuer; also the resource identifier of its tool-invocation surface, addressed as `resource` below. |
| GatewayRAS | `https://ras.gateway.example/` | Resource Authorization Server protecting the gateway. |
| Gateway TTS | (internal) | The gateway domain's Transaction Token Service; derives a hop's handle from GatewayRAS's bound authorization state for the local transaction context. |
| Chain Authority | `https://ca.gateway.example/` | The gateway domain's Chain Authority; issues assertions for its own workloads only. |
| Tenant | `tenant-gw-01` | Identifier scoping the gateway domain's Transaction Token context. |
| WikiRAS | `https://ras.wiki.example/` | Resource Authorization Server for the wiki upstream. |
| WikiAPI | `https://api.wiki.example/` | Upstream protected resource behind WikiRAS. |
| Handles | `Gm2sVe7XpB5tK9nLw4QzCd` (root/gateway hop, H0), `Kx9tRb3WnE6yPq2sHd8VfT` (wiki hop, H1) | Continuation handles; one per hop. |
| Subjects | `alice-gateway-subject`, `alice-wiki-subject` | Alice's pairwise subject at GatewayRAS and WikiRAS, respectively. |

At a glance, the message flow is:

~~~
Alice authenticates in agent-app.

AgentApp       -> IdP:            (1) direct exchange: Alice's ID
                                  Token
IdP            -> AgentApp:       (2) ID-JAG(GatewayRAS), carrying H0
                                  as a claim
AgentApp       -> GatewayRAS:     (3) ID-JAG
GatewayRAS     -> AgentApp:       (4) gateway access token;
                                  GatewayRAS binds H0
AgentApp       -> Gateway:        (5) tool call + access token (no
                                  handle)
Gateway TTS:                      (6) derives H0; issues
                                  tctx.identity_continuation

Gateway resolves the tool call to the wiki upstream.

Gateway        -> ChainAuthority: (7) request assertion for H0 (read
                                  from tctx)
ChainAuthority -> Gateway:        (8) Identity Continuation Assertion
Gateway        -> IdP:            (9) chained exchange (assertion +
                                  DPoP)
IdP            -> Gateway:        (10) ID-JAG(WikiRAS), carrying
                                  fresh H1
Gateway        -> WikiRAS:        (11) ID-JAG
WikiRAS        -> Gateway:        (12) wiki access token; WikiRAS
                                  binds H1
Gateway        -> WikiAPI:        (13) tool call executes
~~~

The subsections below detail each step.

## Root Exchange: The Runtime Roots the Chain

Alice authenticates in `agent-app`, a confidential client whose `client_id`
is the audience of her identity assertion, so `agent-app` performs the
direct exchange of {{token-exchange}} for the one audience it does know: the
gateway (`audience=https://ras.gateway.example/`,
`resource=https://gateway.example/`, `scope=gateway.invoke`) (steps 1 and
2). Alice's standing consent makes the session continuation-capable, and the
IdP establishes a chain ({{root-establishment}}); because tool routing is
dynamic, it records the envelope's authorization basis as Alice's standing
consent and tenant policy, with no enumerated targets ({{validation}}, rule
14), and `agent-app` as the authenticated root actor. Enterprise policy
registers `tool-gateway` as an actor permitted to continue chains rooted this
way ({{validation}}, rule 10). There is no response parameter for the
handle: the IdP embeds the root hop's reference directly as the
`identity_continuation_handle` claim of the issued ID-JAG, the same as it
will for every later continuation of this chain ({{chain-id}}, rule 1):

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.gateway.example/",
  "sub": "alice-gateway-subject",

  "client_id": "agent-app",
  "resource": "https://gateway.example/",
  "scope": "gateway.invoke",

  "identity_continuation_handle": "Gm2sVe7XpB5tK9nLw4QzCd",

  "auth_time": 1713000000,
  "acr": "urn:example:loa:2",
  "amr": ["pwd", "mfa"],

  "cnf": {
    "jkt": "base64url-agent-app-key-thumbprint"
  },

  "iat": 1713000005,
  "exp": 1713000305,
  "jti": "idjag-gateway-01"
}
~~~

`agent-app` redeems this ID-JAG at `ras.gateway.example` for an access token
(steps 3 and 4). GatewayRAS validates the ID-JAG, authenticates `agent-app`,
verifies its DPoP proof, and applies local policy; atomically with issuing
the access token, it binds H0 to the authorization state that token
represents, the accepted state from which its own domain can later continue
({{ras-processing}}). The binding is private to GatewayRAS's domain: it is
recorded for that domain's own Transaction Token Service and Chain Authority,
never returned to `agent-app`, and H0 itself never enters the access token
or any claim a protected resource would see ({{chain-id}}, rules 4 and 6).

`agent-app` invokes the gateway with that access token alone (step 5): no
handle, no header, no continuation-related value of any kind. This is the
point that closes off the earlier design's central weakness. A client that
never holds a handle cannot attach the wrong one, a stale one, or someone
else's, to its request; only GatewayRAS's own acceptance, not anything a
client presents, activates a hop. The gateway's Transaction Token Service
derives H0 from the authorization state GatewayRAS just bound and places it
in the local transaction context for every workload this tool call touches
inside the gateway's domain (step 6, {{transaction-token-context}}):

~~~ json
"tctx": {
  "identity_continuation": {
    "iss": "https://idp.example/",
    "tenant": "tenant-gw-01",
    "handle": "Gm2sVe7XpB5tK9nLw4QzCd"
  }
}
~~~

## Chained Exchange: The Gateway Continues

A tool call in Alice's session requires the wiki service. Only the gateway
knows this routing. The same `tool-gateway` workload, now acting as
continuer, reads H0 from its own transaction context, never from `agent-app`,
which supplied none, and requests an Identity Continuation Assertion from
`https://ca.gateway.example/`, proving control of its key so the Chain
Authority can bind `cnf` to it ({{assertion-issuance}}). The Chain Authority
independently authenticates `tool-gateway`, verifies that proof of
possession, and confirms that GatewayRAS's binding of H0 is still active and
accepted before issuing anything ({{context-provenance}}, {{ras-processing}});
a handle copied from a request GatewayRAS had rejected would never reach
that state, so no assertion could ever be issued for it. `tool-gateway` may
include a target hint with its request, but the hint only helps the Chain
Authority scope its own logging and issuance limits; it is never
authoritative over the IdP's target decision (steps 7 and 8):

~~~ json
{
  "iss": "https://ca.gateway.example/",
  "aud": "https://idp.example/",
  "identity_continuation_handle": "Gm2sVe7XpB5tK9nLw4QzCd",

  "act": {
    "iss": "https://gateway.example/",
    "sub": "tool-gateway"
  },

  "cnf": {
    "jkt": "base64url-tool-gateway-key-thumbprint"
  },

  "iat": 1713000390,
  "exp": 1713000690,
  "jti": "continuation-assertion-03"
}
~~~

`tool-gateway` presents the assertion to the IdP as the `subject_token`,
DPoP-bound to its key (step 9):

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof signed by the tool-gateway key>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
requested_token_type=urn:ietf:params:oauth:token-type:id-jag
audience=https://ras.wiki.example/
resource=https://api.wiki.example/
scope=wiki.read
subject_token=<identity-continuation-assertion>
subject_token_type=<identity-continuation-token-type>
actor_token=<sender-constrained tool-gateway credential>
actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

The `subject_token_type` value above is
`urn:ietf:params:oauth:token-type:identity-continuation`. The IdP verifies
that `tool-gateway` is a permitted continuer and that the DPoP key matches
both the assertion's `cnf.jkt` and the actor token's confirmation, and
evaluates the basis now: wiki read access is within Alice's standing
consent, and tenant policy permits the gateway to reach it
({{validation}}). It resolves Alice's wiki-local subject and, because this
onward grant is itself a continuation-capable ID-JAG, embeds a fresh hop
reference, H1, as its own `identity_continuation_handle` claim
({{chain-id}}, rule 1; {{validation}}) (step 10):

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.wiki.example/",
  "sub": "alice-wiki-subject",

  "client_id": "tool-gateway",
  "resource": "https://api.wiki.example/",
  "scope": "wiki.read",

  "identity_continuation_handle": "Kx9tRb3WnE6yPq2sHd8VfT",

  "auth_time": 1713000000,
  "acr": "urn:example:loa:2",
  "amr": ["pwd", "mfa"],

  "act": {
    "iss": "https://gateway.example/",
    "sub": "tool-gateway",
    "act": {
      "iss": "https://agent.example/",
      "sub": "agent-app"
    }
  },

  "cnf": {
    "jkt": "base64url-tool-gateway-key-thumbprint"
  },

  "iat": 1713000400,
  "exp": 1713000700,
  "jti": "idjag-wiki-01"
}
~~~

The gateway redeems this ID-JAG at `ras.wiki.example` for an access token and
calls the wiki API (steps 11 through 13). WikiRAS processes it exactly as
GatewayRAS processed the root ID-JAG: it validates the grant, authenticates
`tool-gateway`, and, atomically with issuing the wiki access token, binds H1
to the authorization state that token represents ({{ras-processing}}). H1
exists for the same reason H0 did: every continuation-capable ID-JAG carries
a fresh reference for its own hop, whether or not anything continues from it
({{chain-id}}, rule 1), so a wiki-domain workload could later obtain its own
assertion for H1 and reach some further upstream through the wiki domain's
own Chain Authority. This tool call needs no further hop, so that capability
simply goes unused, and H1 never leaves WikiRAS's own domain. A different tool call tomorrow reaches a different
upstream through the same machinery: a fresh assertion for a fresh hop, a
fresh continuation-time policy decision, no new consent ceremony, and the
same denial behavior as {{example-dynamic}} for targets or scopes outside
Alice's standing consent. Concurrent tool calls all derive from the same
GatewayRAS-bound H0 and become sibling hops with independent lineage; none
of them requires `agent-app` to do anything beyond the original access-token
request.

## Points Worth Noticing

The identity assertion's audience check is never weakened. Alice's assertion
is presented exactly once, by the client that is its audience. The gateway
never presents it; the gateway's authority is the Identity Continuation
Assertion bound to the gateway's own key, evaluated against the envelope.
Strict audience validation for identity assertions and a working proxy
topology coexist.

Unlike an earlier design in which the client conveyed the handle itself,
here `agent-app` never holds one: it cannot present the wrong one, a stale
one, or one belonging to another user's session, because it has nothing to
present. Only the accepting domain's own Transaction Token Service can ever
produce `tctx.identity_continuation` for a hop, and only from authorization
state its own Resource Authorization Server bound at acceptance
({{ras-processing}}, {{transaction-token-context}}).

WikiRAS does see `identity_continuation_handle`, but only H1, its own hop's
value, carried as a claim in the ID-JAG it validates; it never sees H0, the
gateway's hop, and nothing it does places H1 in an access token or in any
claim external to its own domain ({{chain-id}}, rule 4). The upstream
Resource Authorization Server otherwise consumes an ordinary continuation-
capable ID-JAG whose `act` chain states precisely what a proxy topology must
state: this gateway, acting for a delegation rooted by this runtime for this
user, for this audience and scope, under a policy decision made at
continuation time, with lineage authenticated by the IdP, not reported by
the gateway ({{onward-id-jag}}).

The runtime performed one exchange for one audience it already knew; the
gateway can request only what the envelope's basis permits, per target, per
hop, revocable at the IdP ({{lifecycle}}).

This example assumes a confidential runtime. Where the session runtime is a
public client, the continuation flow after root establishment is unchanged,
but bootstrapping the root exchange from a public client raises
considerations in the underlying ID-JAG profile (which recommends Token
Exchange for confidential clients) and is not addressed here.

# Open Items for Working Group Discussion {#open-items}

This appendix is non-normative. It lists design questions the author has
deliberately left open for working group discussion; feedback on any of them
is welcome.

\[\[ To be removed before publication as an RFC ]]

1. **Nested own-domain `act` segments (designed future extension).** In
   this version an assertion's `act` names only the current actor, and all
   lineage is IdP-authenticated ({{assertion-claims}}, {{onward-id-jag}}).
   The designed extension would let a Chain Authority additionally assert a
   verified intra-domain segment (actors of an offline attenuated stack
   within its own trust domain) as nested `act` values. The solution shape:
   own-domain actors only, with the verified leaf outermost; the IdP
   composes the segment atop the hop lineage, collapses an actor that
   appears in both so that each actor appears once, and applies depth
   policy to the composed chain. Enabling it is additive, lifting the single-level schema
   constraint of {{assertion-claims}}; an IdP at this version rejects
   nested `act` values deterministically. Should that extension ship, or is
   offline-actor audit better left to the evidence layer
   ({{I-D.mcguinness-oauth-actor-receipts}},
   {{I-D.mcguinness-oauth-actor-proofs}})?

2. **Signed assertion versus a recipient-bound direct profile.**
   {{rationale-grant-type}} documents an alternative in which the
   continuing workload presents the handle directly under client
   authentication, with no assertion, Chain Authority, or per-assertion
   replay state. Implementer review indicates that a raw handle is not an
   adequate subject token for such a profile: an actor credential proves
   who the actor is, not that this actor was authorized to receive and
   continue this particular chain. The sketched direct profile instead
   uses a recipient-bound continuation credential: when the IdP returns a
   hop's handle, it binds that handle to an intended successor (a specific
   actor, an actor class, a trust domain, or a confirmation key); a
   continuation presenting the handle succeeds only for that successor, and
   only together with client authentication, a sender-constrained actor
   token, and live proof of possession. Which profile should ship, and is the Chain Authority's
   remaining role (actor and key vouching plus a domain-local gate) worth
   its trust configuration where recipient binding is available?

3. **Pull topology.** {{rationale-pull}} documents resolution at the target
   Resource Authorization Server as a candidate companion profile that moves
   the integration cost from the IdP side to the target side.

4. **Mutual-TLS binding.** Deliberately not specified while the ID-JAG
   profile defines only DPoP-based binding
   ({{sender-constrained-presentation}}). Should the two profiles add
   mutual-TLS binding together?

5. **A client establishment parameter.** Establishment is
   authorization-driven,
   with no request parameter ({{root-establishment}}), because the
   first-hop client cannot generally predict downstream chaining. Should a
   future parameter let a client require establishment (failing fast when
   policy will not root a chain), suppress it (declining a handle it does
   not want), or negotiate lifetime, depth, or permitted continuers?

6. **Binding chain context to the local transaction.** Chain-context
   provenance ({{context-provenance}}) requires an authorized channel but
   does not bind a received handle to the inbound request, the prior actor,
   or the user context the receiver observed. A multi-user gateway holds
   many valid handles, and if it associates the wrong handle with a
   request, the IdP resolves the user from whichever chain the handle
   names. Should a future revision standardize one interoperable
   provenance profile? The sketched candidate carries the handle inside a
   Transaction Token {{I-D.ietf-oauth-transaction-tokens}} together with
   the parent actor and request context, so the Chain Authority verifies
   an integrity-protected association among chain, actor, and local
   transaction rather than relying on channel authentication alone;
   actor-signed hop proofs {{I-D.mcguinness-oauth-actor-proofs}} are an
   alternative carrier.

7. **Actor-token profile discovery.** This profile constrains accepted
   actor-token types, validity, and applicability
   ({{sender-constrained-presentation}}; {{validation}}, rule 10). How much
   of an IdP's accepted actor-token profile (types, issuers, proof
   methods), and of its other capabilities (confirmation methods, maximum
   assertion lifetime, an issuance endpoint, HTTP binding support, and
   error-detail support), should be advertised in its metadata
   ({{metadata}}) rather than learned out of band?

8. **A continuation-specific error code.** A dead hop ({{validation}},
   rule 7) returns `invalid_request` per Section 2.2.2 of {{RFC8693}}, the
   same code as a malformed request, so an unattended client cannot
   distinguish "seek re-authorization" from "fix the request" by the code
   alone. Should this profile register a continuation-specific error code
   (for example, `invalid_continuation`), or define a supplementary
   `error_details` member carrying a reason taxonomy (for example,
   `assertion_expired`, `handle_not_continuable`, `continuation_revoked`,
   `actor_not_permitted`, `target_outside_envelope`,
   `reauthentication_required`) so unattended clients get
   machine-actionable failure semantics without new top-level codes?

9. **A Chain Authority issuance profile.** How an actor obtains an
   assertion is out of scope ({{assertion-issuance}}), which limits
   end-to-end interoperability between independently implemented actors
   and Chain Authorities. The sketch is a token endpoint-style request:

   ~~~
   POST /identity-continuation-assertion HTTP/1.1
   DPoP: <proof>

   identity_continuation_handle=<handle>
   audience=https://idp.example/
   ~~~

   The authenticated workload identity and proof key determine `act` and
   `cnf`, never caller-supplied values, with errors, discovery, and retry
   behavior defined. Such a profile could also let the Chain Authority
   bind optional narrowing constraints into the assertion (an intended
   target, resource, or authorization-details digest) that the IdP would
   enforce as ceilings on the exchange, reducing assertion-repurposing
   risk without making the Chain Authority a target authority.

10. **A normative authorization-basis representation.** The envelope's
    authorization basis is internal IdP state, so conformance with its
    rule that a policy MAY narrow but MUST NOT broaden the authorization
    basis is observable only through
    behavior. Implementer review asks for a testable representation: for
    explicit targets, for example,

    ~~~ json
    { "targets": [ { "audience": "https://ras.travel.example/",
        "resource": "https://api.travel.example/",
        "scope": ["trips.read"] } ] }
    ~~~

    and, for dynamic ceilings, a concrete form such as an RAR
    authorization detail {{RFC9396}}, a policy-bound intent object, or an
    immutable policy-evaluation artifact. Should a future revision
    standardize one? A companion question is how continuation permission
    itself is represented in consent: a dedicated scope value is the
    obvious candidate, though establishment must remain possible without a
    client-requested scope ({{root-establishment}}).

# Acknowledgments
{:numbered="false"}

The author thanks the authors of OAuth Identity and Authorization Chaining
Across Domains and the Identity Assertion JWT Authorization Grant, on whose work
this profile builds.

# Document History
{:numbered="false"}

\[\[ To be removed from the final specification ]]

-00

* Initial revision
