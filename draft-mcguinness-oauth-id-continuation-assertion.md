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
  RFC8725:
  RFC9110:
  RFC9396:
  RFC9449:
  I-D.ietf-oauth-identity-assertion-authz-grant:

informative:
  RFC6755:
  RFC6838:
  RFC8417:
  RFC8705:
  RFC8707:
  RFC9700:
  I-D.ietf-oauth-identity-chaining:
  I-D.ietf-oauth-transaction-tokens:
  I-D.ietf-wimse-arch:
  I-D.li-oauth-delegated-authorization:
  I-D.mcguinness-oauth-actor-receipts:
  I-D.mcguinness-oauth-actor-proofs:
  OIDC.FrontChannelLogout:
    title: "OpenID Connect Front-Channel Logout 1.0"
    target: "https://openid.net/specs/openid-connect-frontchannel-1_0.html"
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
Because each Resource Authorization Server trusts only the IdP to name the
user, because only the IdP holds the map from a root delegation to each
audience's subject, and because crossing a boundary that renames the user is a
re-issuance rather than an attenuation, the IdP remains the sole issuer of the
onward Identity Assertion JWT Authorization Grant (ID-JAG) at every
cross-boundary hop. The assertion is the
evidence carried into the exchange; the resulting grant is the authority. This
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
ExpenseSaaS calls TravelSaaS. TravelSaaS must call BookingSaaS.
Each SaaS API is protected by its own Resource Authorization Server.
All Resource Authorization Servers trust the same enterprise IdP for
SSO, delegation continuity, and audience-local subject resolution.
~~~

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
different. By the time TravelSaaS must call BookingSaaS, the user is no longer
present and TravelSaaS holds no end-user credential it could present to the
IdP; the only thing that crossed the ExpenseSaaS-to-TravelSaaS boundary is
chain context. TravelSaaS therefore cannot perform a normal identity assertion
exchange to obtain the next grant, even though the same IdP could mint it.

This document defines the missing input. An Identity Continuation Assertion
is a short-lived, sender-constrained, verifiable statement about the
in-flight delegation chain. A service presents it to the IdP in place of an
end-user credential, so that the IdP can resolve the next audience's subject
and issue the next onward grant. It is the evidence; the ID-JAG that comes back is the
authority.

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
attenuation. A holder can only narrow authority it already possesses; it cannot
synthesize a subject it was never issued. The end-user credential that authorized the first hop is not available to
downstream services, so before the IdP can mint the next grant it needs
fresh, sender-constrained evidence that the in-flight delegation is still live
and still represents this user. The Identity Continuation Assertion is that
evidence; it is used as a Token Exchange subject token to obtain an onward
ID-JAG, the output profile of this document.

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
type ({{names}}) for the continuation evidence, the `continuation` request
parameter and the `continuation_handle` and `chain_exp` response parameters
with their control-plane handling ({{root-establishment}}, {{chain-id}}),
the IdP validation rules for a continuation request ({{validation}}), an
OPTIONAL HTTP binding for conveying chain context ({{context-binding}}), and
an authorization server metadata signal ({{metadata}}). The onward ID-JAG
and the way a Resource Authorization Server consumes it are unchanged.

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
  Identity Continuation Assertion, no token it consumes carries a `continuation_handle`,
  and its authorization decisions never depend on one. A workload co-located
  with a Resource Server MAY receive a `continuation_handle` as control-plane context
  accompanying a request ({{context-binding}}).

ID-JAG:
: An Identity Assertion JWT Authorization Grant
  {{I-D.ietf-oauth-identity-assertion-authz-grant}}: the onward authorization
  grant the IdP issues for a target RAS.

Identity Continuation Assertion:
: A short-lived, sender-constrained JWT, defined by this document, issued by a
  Chain Authority and presented to the IdP as the `subject_token` of a Token
  Exchange request in order to obtain an onward authorization grant. Its token
  type is `urn:ietf:params:oauth:token-type:identity-continuation`.

Chain Authority:
: The party trusted by the Continuation Authorization Server to assert chain
  evidence for a given tenant and to issue Identity Continuation Assertions.
  It is a role, not a new deployed component: commonly an existing party the
  IdP already trusts (a Resource Authorization Server, a transaction-token
  service, or a domain gateway), though a deployment MAY run it as a
  dedicated service. It is never the authority for resolving the target
  audience's user subject identifier.

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

Continuation Handle (`continuation_handle`):
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
  * any maximum actor-chain depth set by policy; and
  * the chain's expiry.

  The authorization basis is the ceiling on onward targets captured at
  establishment from the user's consent and tenant policy; the IdP evaluates
  every continuation against it at the time of the request. The IdP records
  that ceiling (for example, a policy version or an evaluated maximal
  grant), and later changes MAY narrow, but MUST NOT broaden, what a
  continuation can obtain. A deployment whose onward targets are known at
  establishment may record the basis as explicit target entries, each
  binding one audience and resource to the scopes, and optionally the
  authorization details {{RFC9396}}, that may be requested for that target;
  dynamic fan-out, in which onward targets are not enumerable at
  establishment, needs no enumeration. See {{validation}} and {{lifecycle}}.

Audience-local (pairwise) subject:
: The subject identifier under which a particular RAS names the user. Distinct
  Resource Authorization Servers MAY name the same user with different
  identifiers; only the IdP holds the map between them.

# When to Use This Profile Versus Offline Attenuation {#decision-rule}

The deciding question is subject resolution, not cost.

* If the next audience requires a *different* subject identifier (pairwise
  subjects, where only the IdP holds the map), the IdP MUST mint that hop. An
  Identity Continuation Assertion to the IdP is used. Offline tokens cannot
  produce the next audience's `sub`.

* If the *same* subject identifier, or a key-based workload identity, suffices
  down a local fan-out, the parties SHOULD NOT round-trip the IdP. An offline
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
  "continuation_handle": "kW4uJ8pTe2NxA6rQvD1zYs",

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

`continuation_handle`:
: REQUIRED. The continuation handle being continued. It binds the request to a
  specific hop of a chain: the IdP resolves the chain, the parent hop, the
  user, and the target-audience subject from it. See {{chain-id}}.

`act`:
: REQUIRED. The current actor presenting the Token Exchange request, encoded
  as a single-level `act` claim per {{RFC8693}}. The `act` object contains a
  REQUIRED `iss` and a REQUIRED `sub`, both non-empty strings. Additional
  members MAY carry further identity attributes of the actor; a recipient
  MUST ignore members it does not understand, and the members `exp`, `nbf`,
  `aud`, `scope`, `cnf`, and nested `act` MUST NOT be present. The IdP MUST
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
  the key whose possession the actor demonstrates with Demonstrating Proof of
  Possession (DPoP) {{RFC9449}}. The Chain Authority binds the assertion to the key at issuance, and the
  presenting actor demonstrates live possession of that key on the Token
  Exchange request. The
  proof-of-possession model is specified in
  {{sender-constrained-presentation}}.

`iat`, `exp`:
: REQUIRED. The assertion lifetime MUST be short. `exp` MUST be after `iat`,
  and `exp - iat` MUST be no more than 300 seconds; see {{validation}}.

`jti`:
: REQUIRED. A unique identifier used by the IdP for replay detection. The value
  MUST be unique per issuer (`iss`) within the assertion validity window.

The assertion MUST NOT contain a top-level `sub`, `auth_time`, `acr`, or `amr`
claim. The IdP obtains the user and root authentication context from the
root-chain envelope indexed by `continuation_handle`, and identifies the current workload
from the `act` claim and the `actor_token`; repeating those values in
the assertion would only create competing sources of identity without adding
authority.

In deployments that already maintain a delegation or mission identifier, the
IdP MAY derive hop handles from that identifier internally (for example, by a
keyed one-way derivation), provided every value remains distinct per hop,
unguessable, and unlinkable, per {{chain-id}}.

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

These stay in the Token Exchange request ({{token-exchange}}) so that direct
and chained calls have an identical shape: the requested values always come
from the request, never from the assertion. The assertion's `aud` claim is
different: it identifies the Continuation Authorization Server that consumes
the assertion, not the requested target audience.

## Assertion Issuance and Key Binding {#assertion-issuance}

A presenting actor obtains an Identity Continuation Assertion from the Chain
Authority for a given `continuation_handle` ({{flow}}, step 7). The Chain
Authority:

* MUST bind the assertion to the requesting actor's key via `cnf`
  ({{assertion-claims}}), and MUST do so only after establishing that the
  requesting actor controls that key and is the actor recorded in the
  `act` claim; and

* SHOULD be in the requesting actor's trust domain. It can satisfy these
  requirements only for actors it can directly authenticate, so each trust
  domain normally operates or designates its own. Issuing assertions for
  actors in another trust domain is NOT RECOMMENDED unless the Chain
  Authority has an authenticated relationship with those actors and their
  keys.

How the actor authenticates to the Chain Authority and proves control of its
key is deployment-specific; it is typically the workload's existing mutually
authenticated credential. An assertion whose `cnf` is not bound to the
requesting actor's key is not presentable, because the IdP requires live
proof of possession of the confirmed key
({{sender-constrained-presentation}}).

## Chain-Context Provenance {#context-provenance}

The mechanism used to transport chain context between workloads is
deployment-specific, but its security properties are not. A Chain Authority
MUST issue an Identity Continuation Assertion only after establishing all of
the following:

1. the `continuation_handle` was received through an authenticated, confidential, and
   integrity-protected channel from a participant authorized to continue that
   chain, or was obtained from equivalent authenticated state held by the Chain
   Authority;

2. the requesting actor is authorized under Chain Authority policy to continue
   the chain;

3. the requesting actor controls the key placed in `cnf`; and

4. the actor placed in `act` is the authenticated requesting actor
   ({{assertion-claims}}), and, where an offline attenuated stack delegated
   to that actor, the Chain Authority has validated the delegation
   artifact's integrity and delegation rules (for example, an actor-signed
   hop-proof chain {{I-D.mcguinness-oauth-actor-proofs}}) before issuing.

Possession of `continuation_handle` alone is insufficient to satisfy these
requirements, and values received as propagated context, including actor
history or requested authority, MUST NOT override the root-chain envelope.
The transport MAY carry deployment-specific hints, but the assertion contains
only the claims of {{assertion-claims}}, and the IdP applies its own
root-chain and current-actor policy at the exchange.

## An HTTP Binding for Chain Context {#context-binding}

Interoperability between independently developed participants requires at
least one common representation for conveying `continuation_handle` with a request. This
document defines an OPTIONAL HTTP binding that participants SHOULD support
when they convey chain context over HTTP: the `Continuation-Handle` request header field
{{RFC9110}}, whose field value is the `continuation_handle`. The syntax of `continuation_handle`
({{chain-id}}, rule 2) confines it to characters valid in an HTTP field value.

~~~
Continuation-Handle: Yw5pD8kFq3RtN6vBx1LzHe
~~~

A sender MUST transmit this field only over a channel that satisfies
{{context-provenance}} (authenticated, confidential, and integrity-protected;
for example, mutual TLS between the workloads). A receiver MUST treat the
value per {{context-provenance}}: it identifies a chain but conveys no
authority. Participants SHOULD exclude the field value from logs.

The field is a singleton: a sender MUST NOT generate more than one
`Continuation-Handle` field in a message, and a receiver MUST reject a message
containing multiple instances, or a list-valued instance, as malformed,
rather than proceed as though no chain context were present.

Actor lineage and other chain context are conveyed by deployment-specific
means and are validated by the Chain Authority before issuance
({{context-provenance}}). Within a trust domain, `continuation_handle`
typically travels inside existing context propagation such as a Transaction
Token {{I-D.ietf-oauth-transaction-tokens}}. This binding serves the
inter-domain hop, which a Transaction Token cannot cross under its own
audience rules ({{rationale-txn}}).

# Continuation Handles (`continuation_handle`) {#chain-id}

A `continuation_handle` is an opaque, unguessable, IdP-generated reference to
one hop of a delegation chain. The IdP creates the root hop when it
establishes a chain ({{root-establishment}}) and a child hop for each
successful continuation; each child holds an immutable reference to the hop
whose handle was presented.

A chain is therefore a tree. Sibling continuations that present the same
handle, including concurrently, create sibling hops, and a hop's lineage is
the path from that hop to the root, independent of every other branch. From
the presented handle the IdP resolves the chain, the parent hop, and the
correct per-audience subject.

`continuation_handle` is a non-bearer reference, closer in spirit to a grant
identifier than to a token: it conveys no authority, and it is never
dereferenced into or presented in place of a token. Authority at each hop
comes from the sender-constrained assertion and the root-chain envelope. This
follows the pattern of the OpenID Connect `sid` claim
{{OIDC.FrontChannelLogout}} and the `txn` claim {{RFC8417}}: opaque,
issuer-generated identifiers that index server-side state. It differs in
being confined to the control plane: unlike `sid`, which a Resource Server
can see in an ID Token, `continuation_handle` MUST NOT appear in any token a
Resource Server consumes (rule 4; see also {{privacy}}).

The following rules apply:

1. The IdP MUST create a chain only when the root Token Exchange requests it
   ({{root-establishment}}), and MUST return a fresh continuation handle as a
   Token Exchange *response* parameter ({{response-param}}) for the root hop
   and for each continuation hop. Handle values MUST NOT be reused across
   hops.

2. `continuation_handle` MUST contain at least 128 bits of entropy, MUST NOT
   contain user-identifying information, and MUST consist of 22 to 256
   characters drawn from the base64url alphabet (`A`-`Z`, `a`-`z`, `0`-`9`,
   `-`, `_`), making it safe for use in HTTP field values
   ({{context-binding}}). Note that URLs are a poor carrier for handles,
   being prone to logging and referrer exposure.

3. `continuation_handle` lives in the control plane only: Token Exchange responses,
   propagated chain context, and Identity Continuation Assertions.

4. `continuation_handle` MUST NOT appear in any ID-JAG or access token that a Resource
   Server consumes. This preserves the audience-local subject property: a
   Resource Server sees only its own subject, audience, scope, and actor chain,
   and nothing that correlates the user across SaaS boundaries.

5. End-to-end audit correlation is performed at the IdP, which holds the hop
   graph for every chain. Per-Resource-Server logs use that server's own
   pairwise subject.

6. Resource Authorization Servers, Resource Servers, and Chain Authorities MUST
   NOT modify `continuation_handle`.

7. `continuation_handle` alone is not proof of authorization and MUST NOT be treated as a
   bearer credential. The IdP MUST use it only as a lookup handle for the hop,
   its chain state, subject resolution, and policy evaluation.

8. A hop's parent reference is immutable. The IdP MUST derive lineage solely
   by walking parent references from the presented hop to the root, and MUST
   NOT maintain or extend a single chain-wide actor history: concurrent
   sibling continuations are independent branches.

## Handle Freshness and Unlinkability {#chain-id-privacy}

Handles are hop-specific by construction: no two hops share a value (rule 1),
so audiences and branches do not acquire a shared identifier for the
delegation. A given handle is still observed by every participant on its
conveyance path (the party that received it, workloads that propagate it, the
Chain Authority, and the IdP), so unlinkability holds across hops, not within
a path, and actor lineage and timing remain correlation channels
({{privacy}}). Handles of a revoked chain, or of a hop subtree the IdP has
revoked ({{lifecycle}}), fail rule 7 of {{validation}} at the next exchange.

# Chain Lifetime and Revocation {#lifecycle}

A chain is continuable only while the IdP considers it active. Because every
cross-boundary hop is an exchange at the IdP ({{flow}}), the IdP is in the loop
at each hop and can stop a chain at any point.

Every chain has a governing authorization: the consent or policy grant under
which it was established, such as the user's session, a governing refresh
token, or a durable consent with an explicit maximum lifetime set by policy.
The IdP MUST bound the continuation lifetime of a chain by the lifetime of
its governing authorization, and MUST reject a continuation of an expired
chain ({{validation}}). The governing authorization is distinct from the
root authentication context: continuing a chain MUST NOT extend that
context, and `auth_time`, `acr`, and `amr` are fixed at root issuance and
inherited unchanged by onward grants ({{security-assurance}}).

The IdP MUST be able to revoke a chain, and MUST stop honoring continuation
for a revoked chain. Ending the governing authorization revokes the chain:
termination of the user's session does so when the session governs, as does
revocation of a governing refresh token or withdrawal of the underlying
policy. An IdP MAY additionally revoke an individual hop's subtree, stopping
continuation along one branch while the rest of the chain remains
continuable.

Because each onward grant requires a fresh exchange at the IdP, revocation
takes effect at the next hop, with no need to reach or invalidate chain
evidence already held by intermediate services. This is a deliberate
difference from an offline attenuated delegation token, where a minted child
token remains usable for its lifetime without contacting an authority.
Access tokens already issued by a Resource Authorization Server remain valid
for their own lifetimes and are governed by that server's revocation
mechanisms.

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
`subject_token_type`, and a root request that establishes a chain
additionally carries `continuation` ({{root-establishment}}).

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
actor_token=<sender-constrained-current-actor-credential>
actor_token_type=<actor-token-type>
~~~

## Establishing a Chain {#root-establishment}

A chain exists only if the root Token Exchange requests one. This document
defines the `continuation` Token Exchange request parameter: its presence in
a direct request asks the IdP to establish a chain for the delegation being
created, and the IdP MUST NOT establish a chain if the parameter is absent.
The parameter takes a single value, the string `true`. The IdP MUST reject a
request carrying more than one `continuation` parameter, any other value, or
a `continuation` parameter on a chained request ({{token-exchange}}) as
malformed ({{RFC6749}}, `invalid_request`). Future specifications may define
richer values to negotiate chain properties.

The chain's properties are not negotiated in this profile. The authorization
basis, the continuation authorization, any maximum actor-chain
depth, and the chain's expiry are determined by the user's authentication
and consent and by tenant policy, and are recorded in the root-chain
envelope.

When establishing a chain, the IdP MUST determine the root actor: the
authenticated OAuth client of the direct request, identified through the
authoritative client-to-actor mapping of {{client-identity}}. If the direct
request includes an `actor_token`, it MUST meet the requirements of
{{sender-constrained-presentation}} and MUST identify that same entity. The
IdP MUST record the root actor in the root hop only after this validation;
the root actor begins every branch's lineage ({{onward-id-jag}}).

The response returns the root hop's handle in `continuation_handle` and MAY
include the chain's expiry in `chain_exp` ({{response-param}}). If the IdP
cannot establish the requested chain, it MAY still issue the requested grant
without one; the absence of `continuation_handle` in the response tells the
client that no chain exists.

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
  accepted. These checks bind the authenticated workload identity, the
  assertion, and the demonstrated key to one actor.

Actor identity comparison is exact: two actor identities are the same entity
only if their issuer and subject identifiers are octet-for-octet equal. The
IdP MUST apply this comparison when matching the `actor_token` to the
`act` claim and to the authenticated client ({{client-identity}}).
Comparison occurs within the tenant context already established for the
exchange ({{validation}}, rule 10); identities from different tenants are
never compared equal.

The onward ID-JAG the IdP issues is itself sender-constrained to the same key
through its own `cnf`, using the DPoP-based binding the ID-JAG profile defines
({{onward-id-jag}}). A mutual-TLS-bound variant {{RFC8705}} is deliberately
not specified: the ID-JAG profile currently defines only DPoP-based binding,
and a different confirmation method in the onward grant would break the
property that the target Resource Authorization Server processes an unchanged
ID-JAG. The presenting actor proves possession of the key again when it
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

* The requester MUST authenticate as an OAuth client. As with the direct
  ID-JAG exchange, this profile is intended for confidential clients.
* The IdP MUST verify that the authenticated client and the current actor
  are the same entity, comparing the client's actor identity (its `iss` and
  `sub` pair) with the `actor_token` and the `act` claim using the exact
  comparison of {{sender-constrained-presentation}}. The IdP learns the
  client's actor identity from an authoritative mapping, typically recorded
  in the client registration or resolvable from a shared client identifier
  namespace such as Client ID Metadata Documents.
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

## The `continuation_handle` Response Parameter {#response-param}

When the IdP establishes a chain ({{root-establishment}}) or continues one, it
MUST return a fresh continuation handle for the newly created hop as the
`continuation_handle` top-level member of the Token Exchange response (alongside
`access_token`, `issued_token_type`, and so forth). An authorized continuer obtains the next Identity Continuation
Assertion for that hop from its Chain Authority ({{assertion-issuance}}).
Each hop's handle is distinct and its parent reference is immutable
({{chain-id}}, rules 1 and 8). `continuation_handle` MUST NOT be carried as a claim
inside the issued ID-JAG (rule 4).

`continuation_handle` stays in the control plane: it is delivered to the party that
performed the Token Exchange and conveyed from there to the workload that
continues the chain, for example accompanying the request into the next service
or held by a Chain Authority in the originating trust domain. Keeping it out of
the ID-JAG prevents a cross-hop correlation handle from reaching every audience
that consumes a token ({{privacy}}). How it is conveyed is deployment-specific,
but the provenance and integrity requirements of {{context-provenance}} apply.

The IdP MAY also include `chain_exp`, a JSON number containing a NumericDate
{{RFC7519}} after which the chain ceases to be eligible for continuation
({{lifecycle}}), letting a continuing party anticipate expiry rather than
discover it by failure. `chain_exp` is advisory; the authoritative lifetime is
the chain state held by the IdP, which can revoke the chain before that time.

A non-normative response example is:

~~~ json
{
  "access_token": "<id-jag>",
  "issued_token_type": "urn:ietf:params:oauth:token-type:id-jag",
  "token_type": "N_A",
  "expires_in": 300,
  "continuation_handle": "Uc9fB3mHs5LdK7gEnX2wRj",
  "chain_exp": 1710086400
}
~~~

## Authorization Server Metadata {#metadata}

An IdP that supports this profile SHOULD signal it in its authorization server
metadata {{RFC8414}} with the following parameter:

`identity_continuation_supported`:
: OPTIONAL. Boolean value indicating support for the
  `urn:ietf:params:oauth:token-type:identity-continuation` subject token
  type, the `continuation` request parameter, and the `continuation_handle`
  and `chain_exp` Token Exchange response parameters. Default `false`.

Absent this signal, a client learns of support out of band or by attempting an
exchange.

# Same-IdP Core Flow {#flow}

The canonical same-IdP SaaS-to-SaaS flow proceeds as follows. Steps 6 through
12 repeat for each additional cross-boundary hop (for example, TravelRAS to
BookingRAS).

1. ExpenseApp authenticates the user at the IdP.

2. ExpenseApp requests an ID-JAG for ExpenseRAS via Token Exchange, including
   the `continuation` parameter to request a continuable chain
   ({{root-establishment}}).

3. The IdP issues an ID-JAG (`iss`=IdP, `aud`=ExpenseRAS, `sub`=the user's
   ExpenseRAS subject, with `scope` and `cnf`) and returns the root hop's
   continuation handle as the `continuation_handle` response parameter. The IdP records
   the root-chain envelope: the user, the authentication context
   (`auth_time`/`acr`/`amr`), the authorization basis, the
   continuation authorization (here, designated TravelSaaS and BookingSaaS
   workloads), any maximum actor-chain depth, the expiry, and the root hop
   with its authenticated actor (`expense-app`). `continuation_handle` is not a claim
   inside the ExpenseRAS ID-JAG.

4. ExpenseApp exchanges the ID-JAG at ExpenseRAS for an access token (AT1) and
   invokes ExpenseSaaS, conveying `continuation_handle` to ExpenseSaaS over an
   authenticated,
   confidential, and integrity-protected control-plane channel associated with
   the request. `continuation_handle` is not carried in the ID-JAG or AT1.

5. ExpenseService, the ExpenseSaaS workload handling the request, propagates
   chain context (`continuation_handle` and verified actor lineage) to
   TravelService over an authenticated, confidential, and integrity-protected
   channel. No audience-local subject is propagated across the SaaS boundary
   by default; local fan-out
   inside a domain MAY use an offline attenuated stack.

6. TravelService needs to call TravelAPI behind TravelRAS.

7. TravelService obtains an Identity Continuation Assertion from the Chain
   Authority for that `continuation_handle`.

8. TravelService presents the assertion to the IdP as the `subject_token` and
   requests an `id-jag` for `audience`=TravelRAS, `resource`=TravelAPI, and
   `scope`=`trips.read`.

9. The IdP validates the assertion, root-chain state, actor policy, the
   requested audience, resource, and scope, and the sender constraint.

10. The IdP resolves the user's subject identifier for TravelRAS.

11. The IdP records the continuation as a new hop whose parent is the
    presented handle, and issues a new ID-JAG (`iss`=IdP, `aud`=TravelRAS,
    `sub`=the same user's TravelRAS subject, an `act` chain constructed from
    the presented hop's lineage plus the newly authenticated current actor,
    and `cnf`), returning the new hop's fresh handle as the `continuation_handle`
    response parameter for any further hop. The TravelRAS ID-JAG carries no
    `continuation_handle` claim.

12. TravelService exchanges the new ID-JAG at TravelRAS for an access token
    (AT2). Steps 6 through 12 repeat for BookingRAS.

# IdP Validation for ID-JAG Output {#validation}

Before issuing an onward ID-JAG, the IdP MUST reject the Token Exchange request
unless all of the following hold:

1. the request contains exactly one each of `grant_type`, `subject_token`,
   `subject_token_type`, `requested_token_type`, `actor_token`, and
   `actor_token_type`; the `grant_type` is
   `urn:ietf:params:oauth:grant-type:token-exchange`, and the
   `subject_token_type` is
   `urn:ietf:params:oauth:token-type:identity-continuation`;

2. the request contains exactly one `audience` parameter and exactly one
   `resource` parameter;

3. the assertion is a JWT containing exactly one value for each required claim
   defined in {{assertion-claims}}; `iss`, `aud`, `continuation_handle`, and `jti` are
   non-empty strings; `act` and `cnf` are JSON objects; `iat` and `exp` are
   JSON numbers representing NumericDate values; and the JOSE `typ` header
   value is `oauth-identity-continuation+jwt`;

4. the assertion signature validates using a key authorized for the assertion
   issuer, and the JOSE `alg` is an asymmetric signature algorithm on the IdP's
   configured allowlist (the `none` algorithm MUST be rejected; see
   {{security-alg}});

5. the assertion `aud` exactly matches the IdP's issuer identifier;

6. the assertion `iss` is a trusted Chain Authority for this tenant;

7. `continuation_handle` identifies a known, active hop of an unexpired,
   unrevoked chain, on a branch within any actor-chain depth bound set by
   policy;

8. the assertion does not contain a top-level `sub`, `auth_time`, `acr`, or
   `amr` claim, nor an `audience`, `resource`, `scope`,
   `authorization_details`, or `requested_token_type` claim
   ({{excluded-claims}});

9. the assertion's `act` claim is present, conforms to the schema of
   {{assertion-claims}}, and identifies the current actor;

10. the request and the current actor are bound together:
    * the request is authenticated as an OAuth client that is the same
      entity as the current actor ({{client-identity}});
    * the `actor_token` was issued by a workload identity issuer the IdP
      trusts for the current actor's trust domain and tenant, and it
      authenticates the current actor;
    * that actor is the actor named in `act`; and
    * that actor is permitted by the chain's continuation authorization
      ({{root-establishment}}) to continue from the presented hop;

11. the request proves possession of the key confirmed by `cnf` with a DPoP
   proof {{RFC9449}} matching `cnf.jkt`, and the `actor_token` is
   sender-constrained to that same key
   ({{sender-constrained-presentation}});

12. `jti` has not been successfully consumed before for the assertion issuer;

13. `iat` and `exp` are valid NumericDate values, `iat` is not in the future
    beyond the IdP's permitted clock skew, `exp` is after `iat`, the assertion
    has not expired, and the assertion lifetime is no more than 300 seconds;

14. the requested `audience` and `resource`, every requested scope, and any
    requested authorization details {{RFC9396}} are authorized by the
    root-chain envelope's authorization basis, evaluated at the time of the
    request, and by IdP policy for the current actor;

15. the requested output token type is
    `urn:ietf:params:oauth:token-type:id-jag`; and

16. the IdP can resolve the target-audience subject for the requested
    `audience`.

After all other validation succeeds, the IdP MUST, as part of issuing the
onward grant, atomically verify that the tuple (`iss`, `jti`) has not been
consumed and record it as consumed. At most one concurrent exchange using the
same tuple may succeed, and the IdP
MUST retain a successfully consumed tuple until at least the assertion's
`exp`, allowing for its permitted clock skew. A request that fails validation
before grant issuance does not consume the tuple; how an implementation
ensures this (for example, by releasing reservations on failure) is an
implementation detail.

These requirements imply per-chain state and, within the assertion validity
window, strongly consistent replay state at the IdP; the 300-second lifetime
bounds the retention window.

A client that does not receive the response to an exchange cannot retry with
the same assertion: it obtains a fresh assertion from its Chain Authority and
repeats the exchange, so Chain Authority issuance SHOULD be inexpensive
relative to the exchange itself. The retry path is at-least-once: a retry
after a lost response mints a second, equivalent grant and hop, short-lived,
sender-constrained to the same key, and bounded by the same authorization
basis, so it confers no additional authority.

On success, the IdP records the continuation as a new hop whose immutable
parent is the presented handle, resolves the audience-local subject for the
requested RAS, and issues the onward ID-JAG with that `sub`. The onward ID-JAG
MUST NOT carry a `continuation_handle` claim.

If validation fails, the IdP MUST return an OAuth error response as defined by
{{RFC8693}} and {{RFC6749}}. The IdP SHOULD use `invalid_request` for malformed
or internally inconsistent requests, including requests with unsupported
parameter cardinality or the wrong `subject_token_type`; `invalid_target` for
a requested audience and resource pair not authorized for the chain;
`invalid_scope` for a requested scope not permitted by the chain's
authorization basis or by IdP policy for the current actor; and
`invalid_authorization_details` {{RFC9396}} for a malformed
`authorization_details` value or one not permitted by that basis.

The IdP SHOULD use `invalid_grant` when the `continuation_handle` is unknown,
expired, revoked, or otherwise no longer eligible for continuation (rule 7).
This is a deliberate, profile-specific deviation from Section 2.2.2 of
{{RFC8693}}, which would direct `invalid_request` for an unacceptable
`subject_token`: for this token type the assertion itself may be entirely
valid while the referenced chain is dead, and the semantics of
`invalid_grant` (an expired or revoked grant) match the condition. The chain
itself is dead, retrying cannot succeed, and continuing requires establishing
a new root delegation, which typically requires the user. `invalid_target` and
`invalid_scope` instead signal that this particular request falls outside the
chain's authorization basis while the chain may remain continuable for other
targets. This distinction lets an unattended client decide between escalating
for re-authorization and abandoning only the current request.

# Onward ID-JAG {#onward-id-jag}

The following is a non-normative example of the onward ID-JAG issued by the
IdP:

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.travel.example/",
  "sub": "travel-local-subject",

  "client_id": "travel-service",
  "resource": "https://api.travel.example/",
  "scope": "trips.read",

  "auth_time": 1710000000,
  "acr": "urn:example:loa:2",
  "amr": ["pwd", "mfa"],

  "act": {
    "iss": "https://travel.example/",
    "sub": "travel-service",
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
policy applies to the resulting chain ({{validation}}, rule 7).

Nested lineage in an onward ID-JAG is therefore authenticated, not
self-reported. In this example the chain is `travel-service` (authenticated
at this continuation) acting for a delegation rooted by `expense-app`
(authenticated at root issuance). Intermediate workloads that never
performed an exchange, such as `expense-service`, appear in deployment audit
records instead; actor receipts {{I-D.mcguinness-oauth-actor-receipts}} and
actor-signed hop proofs {{I-D.mcguinness-oauth-actor-proofs}} provide a
verifiable form for such records.

TravelRAS processes this as an ordinary ID-JAG per
{{I-D.ietf-oauth-identity-assertion-authz-grant}}. It does not need to
understand the Identity Continuation Assertion, and it never sees `continuation_handle`.
The `client_id` is the current actor's client identifier for the target
Resource Authorization Server ({{client-identity}}); in this document's
examples the workload identifier serves as both, so it matches the outer
`act.sub`.

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
  binding ({{security-assurance}}, the envelope enforcement below);
* a compromised Chain Authority, which can issue assertions only for chains
  it is trusted for, and whose assertions still require an authenticated,
  permitted actor (the Chain Authority trust section below);
* a malicious or curious Resource Server or Resource Authorization Server
  attempting to correlate the user across boundaries, addressed by keeping
  `continuation_handle` out of every data-plane token ({{privacy}}).

The subsections below address each in turn.

## Sender Constraint and Proof of Possession

The Identity Continuation Assertion MUST be sender-constrained via `cnf`
{{RFC7800}} and MUST NOT be accepted as a bearer token. The IdP verifies live
proof of possession of the confirmed key and binds the assertion, that key, and
the authenticated current actor together ({{sender-constrained-presentation}}).
Consequently a stolen assertion or stolen actor token cannot be replayed by a
party that does not hold the confirmed private key, and the short single-use
lifetime ({{security-replay}}) bounds the window even for the key holder.

## Short Lifetime and Replay {#security-replay}

The assertion lifetime MUST be no more than 300 seconds ({{validation}}). The
IdP MUST atomically consume (`iss`, `jti`) with successful grant issuance and
MUST permit at most one successful exchange for that tuple ({{validation}}).
Because the assertion is consumed only at the IdP and never by a Resource
Server, its blast radius on replay is confined to the continuation exchange.

## Root Authentication Context {#security-assurance}

The authoritative `auth_time`, `acr`, and `amr` values come only from the
root-chain envelope recorded by the IdP. They are not accepted from the Chain
Authority. Continuing a chain MUST NOT strengthen or refresh that context, and
the IdP MUST copy it unchanged into an onward ID-JAG when the output profile
requires those claims.

## Envelope Enforcement and Offline Attenuation

The root-chain envelope is the authorization ceiling for a chain. The IdP
enforces that the requested target and scopes are permitted by the envelope's
authorization basis ({{validation}}, rule 14), which prevents a compromised
intermediate actor from broadening the chain beyond what the root delegation
and its governing consent and policy authorize.

Where an offline attenuated delegation stack feeds an Identity Continuation
Assertion, that stack's own specification provides and verifies monotonic
attenuation, bounded delegation depth, and parent linkage along the offline
segment (for example, {{I-D.li-oauth-delegated-authorization}},
whose client-issued tokens are key-linked, monotonically downscoped, and
depth-bounded). The Chain Authority validates the segment before issuing
({{context-provenance}}); the IdP itself enforces only the root-chain envelope
described above.

## Trust in the Chain Authority

The IdP MUST accept assertions only from Chain Authorities it trusts for the
relevant tenant ({{validation}}). A compromised Chain
Authority can issue assertions for chains it is trusted for; a continuation
additionally requires an authenticated actor permitted by the chain's
continuation authorization. Deployments SHOULD scope Chain Authority trust as narrowly as practical
and SHOULD monitor for anomalous continuation patterns. Because the IdP
enforces the root-chain envelope, a compromised Chain Authority cannot obtain
a target or scope outside the chain's authorization basis. Neither possession
of a `continuation_handle` nor trusted-Chain-Authority status alone authorizes
continuation: the IdP independently requires that the authenticated current
actor is permitted by IdP policy to continue the specific chain
({{validation}}, rule 10). `continuation_handle` confidentiality is defense in depth, not
an authorization factor.

How the IdP establishes that trust (a Chain Authority's issuer identifier, its
authorized signing keys, and its tenant scope) is out of band and
deployment-specific, as for any issuer whose assertions an Authorization Server
consumes {{RFC7523}}. It MAY be established statically or through a federation
or trust-framework mechanism. Because the Identity Continuation Assertion
carries no subject ({{assertion-claims}}), such a mechanism resolves the root
subject and its owning authority from `continuation_handle` and authorizes the Chain
Authority against that resolved authority, rather than against a subject carried
in the assertion. The root-chain envelope continues to bound a trusted Chain
Authority's authority regardless of how trust is established.

## Trust in Actor Token Issuers

Validation rule 10 depends on the `actor_token` authenticating the current
actor, which makes the issuer of that token a trust anchor of equal weight to
the Chain Authority. The IdP MUST accept actor tokens only from workload
identity issuers it trusts for the actor's trust domain and tenant, scoped
exactly as it scopes Chain Authority trust. How that trust is established (the
issuer, its keys, and its domain scope) is out of band and
deployment-specific, whether through static configuration, a federation, or a
workload identity system such as the WIMSE architecture
{{I-D.ietf-wimse-arch}}. An actor token from an untrusted or out-of-scope
issuer MUST be rejected regardless of the validity of the Identity
Continuation Assertion that accompanies it.

## Actor Chain Integrity

Actor lineage in an onward ID-JAG is constructed by the IdP from the actors
it authenticated at the root exchange and at each continuation, so lineage
is authenticated rather than self-reported ({{onward-id-jag}}). An assertion
names only the current actor, and the IdP MUST reject an assertion whose
`act` names anyone else ({{validation}}, rule 9). The IdP bases
authorization decisions on the root-chain state, the authenticated current
actor, and the requested `audience`, `resource`, and `scope`. Actors of an
offline attenuated segment appear in deployment audit records and the
evidence layer rather than in lineage; asserting a verified own-domain
segment is reserved for a future extension ({{open-items}}).

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

A `continuation_handle` correlates the control-plane participants on its
conveyance path, and the rules in {{chain-id}} keep it out of the data plane. Because `continuation_handle`
MUST NOT appear in any ID-JAG or access token consumed by a Resource Server,
and because each RAS sees only its own pairwise subject, a Resource Server or
RAS that receives only its audience-local data-plane tokens cannot directly
correlate the user across SaaS boundaries from those tokens.

This property does not make the complete chain unlinkable. The IdP necessarily
correlates the chain, and workloads, Chain Authorities, or other control-plane
participants that receive the same `continuation_handle` can correlate their observations
of that delegation. In particular, the actor chain carried in each onward
ID-JAG names the participating workloads, so colluding audiences can correlate
a transaction from actor lineage and timing alone, regardless of how
`continuation_handle` is handled. Deployments requiring unlinkability across audiences
must weigh the audit value of deep actor lineage against this correlation
channel and MAY limit the nested lineage exposed to each audience, subject to
their audit requirements. Deployments SHOULD limit disclosure of `continuation_handle` to
participants that require it to continue or administer the chain.

`continuation_handle` MUST NOT contain user-identifying information and MUST carry at least
128 bits of entropy ({{chain-id}}, rule 2), so that a `continuation_handle` neither
reveals the user nor can be guessed.

Continuation handles are hop-specific by construction ({{chain-id-privacy}}),
so audiences and branches do not share a correlatable identifier. A handle's
conveyance path still observes its value, and a workload co-located with a
Resource Server that receives chain context alongside a request
({{context-binding}}) is, for that data, a control-plane participant subject
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
: continuation_handle

Claim Description:
: Continuation Handle: an opaque, IdP-generated reference to one hop of a
  delegation chain, used to correlate a continuation to its chain
  and parent hop and to resolve the per-audience subject.

Change Controller:
: IESG

Specification Document(s):
: This document, {{chain-id}}

## OAuth Parameters Registration

IANA is requested to register the following values in the "OAuth Parameters"
registry established by {{RFC6749}}. The `continuation` parameter is used in
the OAuth 2.0 Token Exchange {{RFC8693}} request; `continuation_handle` and `chain_exp`
are used in the response.

Parameter name:
: continuation

Parameter usage location:
: token request

Change controller:
: IETF

Specification document(s):
: This document, {{root-establishment}}

Parameter name:
: continuation_handle

Parameter usage location:
: token response

Change controller:
: IETF

Specification document(s):
: This document, {{response-param}}

Parameter name:
: chain_exp

Parameter usage location:
: token response

Change controller:
: IETF

Specification document(s):
: This document, {{response-param}}

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

## HTTP Field Name Registration

IANA is requested to register the following entry in the "Hypertext Transfer
Protocol (HTTP) Field Name Registry" defined by {{RFC9110}}.

Field Name:
: Continuation-Handle

Status:
: permanent

Reference:
: This document, {{context-binding}}

Note: The token type URI `urn:ietf:params:oauth:token-type:id-jag` referenced by
this document is registered by
{{I-D.ietf-oauth-identity-assertion-authz-grant}} and is not registered here.

--- back

# Design Rationale {#rationale}

This appendix is non-normative. It explains why the Identity Continuation
Assertion is defined as a distinct Token Exchange subject token, rather than as
a profile of an existing artifact, and why a cross-boundary hop cannot be served
by propagating an existing token. The recurring answer is per-boundary trust:
each target trusts only the IdP to name the user and to scope authority over
its resources, and where subjects are pairwise, only the IdP can even mint the
next audience's subject ({{motivation}}, {{core-principle}}).

## Why Not a Profile of ID-JAG {#rationale-idjag}

An ID-JAG {{I-D.ietf-oauth-identity-assertion-authz-grant}} and an Identity
Continuation Assertion sit on opposite sides of the same Token Exchange and play
opposite roles, so neither can be a profile of the other:

* An ID-JAG is the **output**: an authorization grant. An Identity Continuation
  Assertion is the **input**: evidence presented as a `subject_token`.

* An ID-JAG's `aud` is the target Resource Authorization Server; an Identity
  Continuation Assertion's `aud` is the IdP. The two are validated by different
  parties under different rules.

* An ID-JAG carries the user's resolved per-audience subject; an Identity
  Continuation Assertion does not carry a top-level `sub` ({{assertion-claims}}).
  Profiling ID-JAG would force a `sub` to mean precisely what this profile
  forbids.

* `continuation_handle` is excluded from every ID-JAG ({{chain-id}}, rule 4), yet it is the
  defining claim of an Identity Continuation Assertion. "An ID-JAG carrying
  `continuation_handle`" is self-contradictory.

An Identity Continuation Assertion is also not a grant: presenting it to a
Resource Authorization Server is meaningless. It is the input that *produces* a
chained ID-JAG, not a kind of ID-JAG.

## Why Not a Profile of a Transaction Token {#rationale-txn}

Transaction Tokens {{I-D.ietf-oauth-transaction-tokens}} are the closest
neighbor (a short-lived signed JWT carrying delegation context, often issued by
a service that could also act as the Chain Authority), but they sit at a
different layer. A Transaction Token is an *intra-domain* object:
its `aud` identifies a trust domain and it "MUST NOT be accepted outside" it, it
is consumed by workloads inside the domain and re-minted as it propagates, and
its `sub` is domain-local. An Identity Continuation Assertion instead crosses to
the IdP: its `aud` is the IdP, it is consumed only there, it is single-use, and
it carries neither the target subject (the IdP resolves it) nor the request
context (`tctx`, `rctx`, `scope`) a Transaction Token carries. It is therefore
*derived from* a Transaction Token at the boundary, not a profile of one.

## Why Not a Cross-Domain Propagation Token {#rationale-propagation}

A natural question is whether the cross-boundary hop could be served by simply
propagating a token across domains (an "inter-domain propagation token")
consumed directly by the next domain with no IdP round-trip. This profile does
use offline propagation between hops wherever it is sound: that is the offline
attenuated delegation layer, used when the subject does not change
({{decision-rule}}). It
cannot serve the boundary where the subject changes. Crossing that boundary
re-subjects the user, and re-subjecting is a mint that only the IdP can perform,
not an attenuation a propagation token can carry: a holder narrows authority it
already has, it does not synthesize a subject it was never issued
({{core-principle}}). Four consequences make this concrete:

1. **Subject resolution.** A propagation token is minted by the source side,
   which does not know and cannot compute the next audience's pairwise subject;
   only the IdP holds the map ({{motivation}}). Such a token can only carry the
   source domain's subject (wrong at the next domain, and a privacy leak), no
   subject (the user is unidentifiable), or a single global subject (which
   pairwise subjects preclude by design). No amount of signing supplies data the
   issuer does not have.

2. **Trust direction.** The receiving domain trusts the IdP, not the source
   domain's issuer, to name the user and scope authority over its resources.
   Direct propagation asks domain B to accept domain A's signature as authority
   over domain B. Obtaining a grant from an authority that domain B already
   trusts is the model of {{I-D.ietf-oauth-identity-chaining}}.

3. **Revocation.** An offline propagation token is usable for its lifetime
   without contacting an authority, removing the ability to revoke mid-chain.
   Because each Identity Continuation Assertion hop is an exchange at the IdP,
   revocation takes effect at the next hop ({{lifecycle}}).

4. **Blast radius.** A token consumed directly by the next Resource
   Authorization Server is a broad, comparatively long-lived credential. An
   Identity Continuation Assertion is narrowly scoped to `aud` = IdP,
   single-use, and sender-constrained
   ({{sender-constrained-presentation}}). Its theft permits only a request for a
   grant within the root-chain envelope.

Built honestly, a cross-domain propagation token collapses into this profile: to
be usable at the next domain its subject must be re-resolved, and the only party
that can do so is the IdP, so the token must be presented to the IdP, which is
exactly an Identity Continuation Assertion. Where all domains share a single
global subject for the user, mutually trust each other's issuers (for example, a
single SPIFFE-style trust domain), and accept the loss of mid-chain revocation,
direct inter-domain propagation is appropriate; that is a different problem from
the pairwise-subject, per-boundary-trust case this profile addresses.
Delegated Authorization {{I-D.li-oauth-delegated-authorization}} illustrates
the boundary from the other side: its client-issued delegation tokens are
forbidden to carry a subject at all, so that stack composes with this profile
as the intra-domain layer of {{decision-rule}} and stops exactly where
re-subjecting begins.

## Alternative Topology: Resolution at the Target {#rationale-pull}

This profile uses a push topology: the continuing workload carries continuation
evidence to the IdP and then carries the resulting ID-JAG to the target
Resource Authorization Server. A pull topology was also evaluated. In it, the
workload presents its chain reference directly to the target Resource
Authorization Server under a new grant type, together with proof of possession
of its key, and that server resolves the reference at the IdP over a back
channel in the style of token introspection {{RFC7662}}: the IdP performs the
same envelope, actor-policy, and proof-of-possession checks, resolves its
audience-local subject, and the Resource Authorization Server issues the
access token directly. A variant returns the ID-JAG itself in the back-channel
response, so the Resource Authorization Server validates it with unchanged
ID-JAG processing and only the delivery path differs.

The pull topology has real advantages. It removes the Chain Authority
entirely, because the target's Authorization Server asks the IdP it already
trusts and no additional assertion issuer exists. It removes one token round
trip per hop. It reduces the cross-boundary artifact to a single opaque
reference, shrinking the deployment-specific transport surface. And issuing a
fresh reference per resolution mirrors the per-hop handles of the push
topology ({{chain-id}}).

Its cost falls on the many-party side of the ecosystem: every target Resource
Authorization Server must implement the new grant type and the back-channel
resolution call, and the reference becomes visible to each Resource
Authorization Server it is presented to. This document standardizes the push
topology because it preserves the deployment property that drives ID-JAG
adoption: the target Resource Authorization Server consumes an ordinary
ID-JAG and requires no changes. Push concentrates new behavior in the IdP and
the Chain Authority, which are few; pull distributes it to Resource
Authorization Servers, which are many. The pull topology remains a candidate
companion profile should the ecosystem prefer to move the integration cost to
the target side; the two topologies consume the same root-chain envelope and
differ only in who carries the evidence to the IdP.

## Why a Signed Assertion Rather Than a Bare Grant Type {#rationale-grant-type}

A dedicated grant type was also evaluated: the continuing workload would
present `continuation_handle` as a request parameter directly to the IdP, authenticated
by its sender-constrained `actor_token` and live proof of possession, with no
assertion JWT, no Chain Authority issuer trust, and no per-assertion replay
state. Where the continuing domain has no offline delegation segment and no
domain-local continuation policy, that reduction loses little: the IdP
independently authenticates the actor and enforces the envelope either way.

The signed assertion was retained for the work the Chain Authority performs
that the IdP cannot: verifying the intra-domain offline segment before the
boundary is crossed, and vouching first-hand for its own workloads' identity,
keys, and provenance. The Chain Authority does not see the target of the
eventual exchange ({{excluded-claims}}) and therefore gates only continuation
itself, never target or scope, which are solely the IdP's to authorize. With
all lineage constructed by the IdP ({{onward-id-jag}}), the
assertion carries exactly what the Chain Authority can attest first-hand: the
authenticated current actor and its key binding. Deployments needing neither
capability reduce the Chain Authority to
a co-signature over `continuation_handle` and the current actor; the bare grant type
remains a candidate simplification should working group feedback favor it.

# Worked Example (Same-IdP) {#example}

This appendix is non-normative. This and the following appendices cover the
three deployment shapes this profile serves: interactive SaaS-to-SaaS
chaining (this appendix), an unattended background agent
({{example-background}}), and a gateway topology with dynamically determined
upstream audiences ({{example-gateway}}). This appendix walks the canonical
flow of {{flow}} end-to-end for a single user: ExpenseApp invokes
ExpenseSaaS, and ExpenseService, the workload handling that request, calls
TravelSaaS, whose TravelService must call TravelAPI to finish the request. All parties trust one
enterprise IdP at `https://idp.example/`.
Proof of possession uses DPoP. JWTs are shown as decoded payloads; JOSE headers
and signatures are omitted. Client authentication is required on every
exchange ({{client-identity}}) and is omitted from all example requests in
these appendices for brevity. The values are consistent with the examples in
{{assertion-claims}}, {{response-param}}, and {{onward-id-jag}}.

The participants and values used throughout:

| Name | Value | Description |
|------|-------|-------------|
| User | (none) | The human, authenticated once at the IdP. |
| IdP | `https://idp.example/` | Trust anchor and Continuation Authorization Server. |
| ExpenseApp | `expense-app` | User-facing application; first-hop client. |
| ExpenseSaaS | `https://expenses.example/` | First SaaS the user invokes. |
| ExpenseService | `expense-service` | ExpenseSaaS workload that calls TravelSaaS. |
| ExpenseRAS | `https://ras.expenses.example/` | Resource Authorization Server for ExpenseAPI. |
| ExpenseAPI | `https://api.expenses.example/` | Protected resource behind ExpenseRAS. |
| TravelSaaS | `https://travel.example/` | Downstream SaaS. |
| TravelService | `travel-service` | TravelSaaS workload that calls TravelAPI. |
| TravelRAS | `https://ras.travel.example/` | Resource Authorization Server for TravelAPI. |
| TravelAPI | `https://api.travel.example/` | Protected resource behind TravelRAS. |
| Chain Authority | `https://ca.travel.example/` | TravelSaaS's Chain Authority; issues the Identity Continuation Assertion for TravelSaaS workloads. |
| BookingSaaS | `https://booking.example/` | Third SaaS, reached in the third hop. |
| BookingWorker | `booking-worker` | TravelSaaS workload delegated offline for the Booking hop. |
| BookingRAS | `https://ras.booking.example/` | Resource Authorization Server for BookingAPI. |
| BookingAPI | `https://api.booking.example/` | Protected resource behind BookingRAS. |
| Handles | `kW4uJ8pTe2NxA6rQvD1zYs` (root hop), `Uc9fB3mHs5LdK7gEnX2wRj` (Travel hop) | Continuation handles; one per hop. |
| Subjects | `expense-local-subject`, `travel-local-subject`, `booking-local-subject` | The user's pairwise subject at ExpenseRAS, TravelRAS, and BookingRAS, respectively. |

At a glance, the message flow is:

~~~
User authenticates once at the IdP.

ExpenseApp -> IdP: exchange ID Token
IdP -> ExpenseApp: ID-JAG(ExpenseRAS) + handle
ExpenseApp -> ExpenseRAS: ID-JAG
ExpenseRAS -> ExpenseApp: AT1
ExpenseApp -> ExpenseSaaS: API request + handle

ExpenseService -> TravelSaaS: protected request + chain context
TravelSaaS -> TravelService: verified chain context

TravelService -> ChainAuthority: request assertion
ChainAuthority -> TravelService: Identity Continuation Assertion
TravelService -> IdP: assertion exchange + DPoP
IdP -> TravelService: ID-JAG(TravelRAS) + handle
TravelService -> TravelRAS: ID-JAG
TravelRAS -> TravelService: AT2
TravelService -> TravelAPI: API call
~~~

The subsections below detail each step.

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
continuation=true
subject_token=<id_token>
subject_token_type=urn:ietf:params:oauth:token-type:id_token
actor_token=<sender-constrained expense-app credential>
actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

The IdP authenticates the user from the ID Token and verifies that ExpenseApp's
actor credential is constrained to the DPoP key. The `continuation` parameter
requests a continuable chain ({{root-establishment}}). For this example,
existing user consent and enterprise policy authorize the immediate Expense
target and the later Travel and Booking targets, and enterprise policy
designates the Travel and Booking workloads as permitted continuers. The IdP
establishes the chain and records the following target entries in the
root-chain envelope:

~~~
(https://ras.expenses.example/, https://api.expenses.example/)
    permitted scopes: expenses.read

(https://ras.travel.example/, https://api.travel.example/)
    permitted scopes: trips.read

(https://ras.booking.example/, https://api.booking.example/)
    permitted scopes: stays.book
~~~

The envelope also records the continuation authorization and the chain's
expiry. The root exchange does not authorize the later targets merely by
requesting the first: all three entries must already be supported by the
authentication, consent, and policy from which the IdP constructs the
envelope. This example records the basis as explicit target entries; a
deployment whose onward targets are not knowable at establishment records
the consent-and-policy ceiling without enumeration and evaluates the Travel
target at continuation time ({{validation}}, rule 14). The
IdP then responds:

~~~ json
{
  "access_token": "<ID-JAG for ExpenseRAS>",
  "issued_token_type": "urn:ietf:params:oauth:token-type:id-jag",
  "token_type": "N_A",
  "expires_in": 300,
  "continuation_handle": "kW4uJ8pTe2NxA6rQvD1zYs"
}
~~~

The decoded ID-JAG for ExpenseRAS carries the user's ExpenseRAS-local subject:

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

  "cnf": {
    "jkt": "base64url-expense-app-key-thumbprint"
  },

  "iat": 1710000005,
  "exp": 1710000305,
  "jti": "idjag-expense-01"
}
~~~

ExpenseApp exchanges this ID-JAG at ExpenseRAS for an access token (AT1) and
invokes ExpenseSaaS, exactly as for any ID-JAG
{{I-D.ietf-oauth-identity-assertion-authz-grant}} (not shown). ExpenseApp
conveys `continuation_handle` to ExpenseSaaS over an authenticated,
confidential, and integrity-protected control-plane channel associated with
that API request.
The `continuation_handle` does not appear in the ID-JAG or AT1.

## Crossing the Boundary into TravelSaaS {#example-context}

To complete the request, ExpenseService calls TravelSaaS, propagating the
chain context to TravelService over an authenticated, confidential, and
integrity-protected channel:

~~~
continuation_handle   = kW4uJ8pTe2NxA6rQvD1zYs
actor chain = expense-service (ExpenseSaaS) <- expense-app
~~~

The user's ExpenseRAS-local subject (`expense-local-subject`) is not propagated
across the boundary. The IdP retains the authoritative authentication context
and root-chain envelope; those values are not supplied by TravelService or
repeated in the Identity Continuation Assertion.

## Obtaining the Identity Continuation Assertion {#example-ica}

TravelService needs to call TravelAPI behind TravelRAS. It requests an Identity
Continuation Assertion from the Chain Authority for this `continuation_handle`, proving
control of its key so the Chain Authority can bind `cnf` to it
({{assertion-issuance}}). The Chain Authority returns the assertion shown in
{{assertion-claims}}: `iss` is the Chain Authority, `aud` is the IdP,
`continuation_handle` is the value above, the `act` names `travel-service`, and
`cnf.jkt` is TravelService's key thumbprint.

## Chained Exchange for the TravelRAS ID-JAG {#example-chained}

TravelService presents the assertion to the IdP as the `subject_token`,
DPoP-bound to TravelService's key:

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof signed by the travel-service key>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
requested_token_type=urn:ietf:params:oauth:token-type:id-jag
audience=https://ras.travel.example/
resource=https://api.travel.example/
scope=trips.read
subject_token=<identity-continuation-assertion>
subject_token_type=<identity-continuation-token-type>
actor_token=<sender-constrained travel-service credential>
actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

The `subject_token_type` value above is
`urn:ietf:params:oauth:token-type:identity-continuation`. The IdP runs the
checks of {{validation}}: the DPoP key matches both the assertion's `cnf.jkt`
and the actor token's key confirmation, `travel-service` is the actor
named in `act`, the `continuation_handle` is active, and the requested TravelRAS, TravelAPI, and
`trips.read` values match the Travel target entry in the root-chain envelope.
It then resolves the user's TravelRAS-local subject and returns:

~~~ json
{
  "access_token": "<ID-JAG for TravelRAS>",
  "issued_token_type": "urn:ietf:params:oauth:token-type:id-jag",
  "token_type": "N_A",
  "expires_in": 300,
  "continuation_handle": "Uc9fB3mHs5LdK7gEnX2wRj"
}
~~~

The decoded ID-JAG for TravelRAS is the one shown in {{onward-id-jag}}: same
user, but `sub` is now `travel-local-subject`, and it carries no `continuation_handle`.
The response returns a fresh handle for the newly created Travel hop; the
Booking continuation in {{example-offline-segment}} presents that handle, so
its lineage builds on the Travel hop rather than on any sibling branch.

## Using the TravelRAS ID-JAG {#example-use}

TravelService exchanges the TravelRAS ID-JAG at TravelRAS for an access token
(AT2), presenting a fresh DPoP proof with the same `travel-service` key, and
calls TravelAPI. TravelRAS processes the ID-JAG as an ordinary ID-JAG and
never sees the Identity Continuation Assertion or the `continuation_handle`.

## Third Hop After Offline Delegation {#example-offline-segment}

Completing the itinerary requires a reservation at BookingSaaS
(`https://booking.example/`), whose API `https://api.booking.example/` sits
behind `https://ras.booking.example/`. Inside TravelSaaS, `travel-service`
does not make this call itself: it fans out offline to `booking-worker`,
another travel.example workload, using an offline attenuated delegation token
with no IdP round trip ({{decision-rule}}). The user's subject changes at the
Booking boundary, so that hop is a continuation.

`booking-worker` requests an assertion from `https://ca.travel.example/`,
which validates the offline delegation before issuing
({{context-provenance}}) and names `booking-worker` as the current actor:

~~~ json
{
  "iss": "https://ca.travel.example/",
  "aud": "https://idp.example/",
  "continuation_handle": "Uc9fB3mHs5LdK7gEnX2wRj",

  "act": {
    "iss": "https://travel.example/",
    "sub": "booking-worker"
  },

  "cnf": {
    "jkt": "base64url-booking-worker-key-thumbprint"
  },

  "iat": 1710000900,
  "exp": 1710001200,
  "jti": "continuation-assertion-02"
}
~~~

`booking-worker` exchanges the assertion, DPoP-bound to its own key, for an
ID-JAG with `audience=https://ras.booking.example/`,
`resource=https://api.booking.example/`, and `scope=stays.book`, all within
the envelope's Booking target entry. The IdP constructs the onward `act`
chain ({{onward-id-jag}}): `booking-worker`, authenticated at this exchange,
placed atop the presented hop's lineage (`travel-service`, then
`expense-app`):

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.booking.example/",
  "sub": "booking-local-subject",

  "client_id": "booking-worker",
  "resource": "https://api.booking.example/",
  "scope": "stays.book",

  "auth_time": 1710000000,
  "acr": "urn:example:loa:2",
  "amr": ["pwd", "mfa"],

  "act": {
    "iss": "https://travel.example/",
    "sub": "booking-worker",
    "act": {
      "iss": "https://travel.example/",
      "sub": "travel-service",
      "act": {
        "iss": "https://expenses.example/",
        "sub": "expense-app"
      }
    }
  },

  "cnf": {
    "jkt": "base64url-booking-worker-key-thumbprint"
  },

  "iat": 1710000920,
  "exp": 1710001220,
  "jti": "idjag-booking-01"
}
~~~

`booking-worker`, having performed this exchange, now anchors a new hop of
the chain, and any continuation presenting that hop's handle builds on its
lineage. BookingRAS processes an ordinary ID-JAG and receives an
authenticated lineage, including an offline hop that its own domain could
never have verified.

# Background Agent Example (User-Scheduled Continuation) {#example-background}

This appendix is non-normative. It applies the continuation machinery of this
document to a background agent: the user is present once, when the task is
created, and absent at every run. The example is the same protocol as
{{example}}, time-shifted; nothing new is defined. The property that makes it
work is that the platform stores only the non-bearer `continuation_handle`; no refresh
token or other user credential is vaulted, and run-time authority comes from a
fresh, sender-constrained assertion evaluated against the root-chain envelope.

The participants and values used throughout:

| Name | Value | Description |
|------|-------|-------------|
| User | (none) | The human, Alice; present at setup, absent at every run. |
| IdP | `https://idp.example/` | Trust anchor and Continuation Authorization Server. |
| BriefingAgent | `briefing-agent` | Agent platform workload that runs the scheduled task. |
| Platform | `https://platform.example/` | BriefingAgent's trust domain and actor issuer. |
| Chain Authority | `https://ca.platform.example/` | The platform's Chain Authority; issues assertions for its workloads. |
| CalendarRAS | `https://ras.calendar.example/` | Resource Authorization Server for the calendar service. |
| CalendarAPI | `https://api.calendar.example/` | Protected resource behind CalendarRAS. |
| Handle | `Pz6vTq1NcY4kM8bJf3RxWa` | Root hop handle, stored with the task. |
| Subject | `alice-calendar-subject` | Alice's pairwise subject at CalendarRAS. |

## Setup (Alice Present)

Alice schedules "summarize my calendar every morning" and authorizes it in an
active session. The platform performs the direct ID-JAG exchange of
{{token-exchange}} with Alice's session as the `subject_token` and
`briefing-agent` as the authenticated actor, including the `continuation`
parameter ({{root-establishment}}). Because the task will outlive
Alice's session, the consent captured here establishes an explicit maximum
chain lifetime as the chain's governing authorization ({{lifecycle}}), with
`briefing-agent` as the permitted continuer, and the
envelope's authorized target entry is exactly the task's need:
(`https://ras.calendar.example/`, `https://api.calendar.example/`,
`calendar.read`). The IdP records the envelope, with `briefing-agent` as the
authenticated root actor, and responds with the grant plus:

~~~ json
{
  "continuation_handle": "Pz6vTq1NcY4kM8bJf3RxWa",
  "chain_exp": 1714592000
}
~~~

The platform stores the task. This record contains no credential:

~~~
task: morning-calendar-brief
  user:     alice
  agent:    briefing-agent           # the authorized continuer
  continuation_handle: Pz6vTq1NcY4kM8bJf3RxWa  # non-bearer
  chain_exp: 1714592000
  schedule: "0 7 * * *"
~~~

## Each Run (Alice Absent)

At a glance, each run proceeds as follows:

~~~
Scheduler      -> BriefingAgent:  run task, here is the handle
BriefingAgent  -> ChainAuthority: assertion for the stored handle,
                                  bound to the briefing-agent key
ChainAuthority -> BriefingAgent:  Identity Continuation Assertion
BriefingAgent  -> IdP:            Token Exchange (assertion + DPoP)
IdP:                              validate per the rules of the
                                  validation section; resolve Alice's
                                  Calendar subject; mint
IdP            -> BriefingAgent:  ID-JAG(CalendarRAS) + handle
BriefingAgent  -> CalendarRAS:    ID-JAG -> access token (DPoP)
BriefingAgent  -> CalendarAPI:    read calendar -> summarize
~~~

The assertion is the ordinary artifact of {{assertion-claims}}, issued by the
platform's own Chain Authority for its own workload
({{assertion-issuance}}): `iss` is `https://ca.platform.example/`, `aud` is
the IdP, `continuation_handle` is the stored value, the `act` claim is `briefing-agent`,
and `cnf` binds the agent's key. The onward ID-JAG carries Alice's
Calendar-local subject, an `act` chain of `briefing-agent` (the root and
continuing actor are the same workload here), and Alice's real authentication
context from setup:

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

  "cnf": {
    "jkt": "base64url-briefing-agent-key-thumbprint"
  },

  "iat": 1712900001,
  "exp": 1712900301,
  "jti": "idjag-cal-alice-01"
}
~~~

## Points Worth Noticing

The `auth_time` is honest and inherited: it records when Alice actually
authenticated at setup, possibly days before this run
({{security-assurance}}). CalendarRAS decides whether that staleness is
acceptable for `calendar.read`; nothing presents the run as a fresh login.

Each run consumes a fresh, single-use assertion; between runs the platform
holds nothing presentable. Theft of the stored record yields a `continuation_handle`
that is useless without the agent's key ({{chain-id}}). Each run presents the
stored root handle and creates its own hop, receiving that hop's fresh handle
for any onward hops within the run; sibling runs are independent branches of
the chain.

The chain fails closed. When Alice's authorization is revoked or `chain_exp`
passes, the next exchange returns `invalid_grant` ({{validation}}): the
signal to seek re-authorization from Alice rather than retry. `chain_exp`
lets the platform anticipate expiry and prompt her before the task silently
stops, and, where the IdP
provides the management interfaces encouraged by {{lifecycle}}, the chain is
visible and revocable there.

This pattern requires a user-present setup event to root the chain. Where no
such event exists (for example, an administratively mandated agent acting
for users who never authorized it), there is no delegation to continue and
this profile does not apply; such deployments need a differently rooted
authorization, such as administrative policy at the IdP, which is out of
scope for this document.

## A Dynamic Target {#example-dynamic}

Suppose the platform later extends the briefing to include unread mail,
which requires `https://api.mail.example/` behind
`https://ras.mail.example/`: a target nobody named when Alice created the
task. Under the target entries recorded in the setup above, the agent's
exchange for that audience fails, and the chain is otherwise unaffected:

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
it at continuation time ({{validation}}, rule 14). The same exchange then succeeds if, and only
if, read access to the mail service is within Alice's standing consent and
if tenant policy permits `briefing-agent` to reach it. A request for
`mail.send`, outside that consent, fails with `invalid_scope`.

The envelope guardrail applies: evaluation is bounded by the consent and
policy in force when the chain was established. If the tenant later broadens
policy, existing chains do not gain the new authority; if it narrows, they
lose it at the next run. Both failures are per-request (`invalid_target`,
`invalid_scope`), and the chain remains continuable for authorized targets,
in contrast to `invalid_grant`, which reports that the chain itself is no
longer continuable ({{validation}}).

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
an identity assertion whose audience it is not.

The participants and values used throughout:

| Name | Value | Description |
|------|-------|-------------|
| User | (none) | The human, Alice; her session runs in AgentApp. |
| IdP | `https://idp.example/` | Trust anchor and Continuation Authorization Server. |
| AgentApp | `agent-app` | Confidential agent runtime; hosts Alice's session and roots the chain. |
| AgentPlatform | `https://agent.example/` | AgentApp's trust domain and actor issuer. |
| Gateway | `tool-gateway` | Tool gateway workload; continues the chain per tool call. |
| GatewayPlatform | `https://gateway.example/` | The gateway's trust domain and actor issuer. |
| GatewayRAS | `https://ras.gateway.example/` | Resource Authorization Server protecting the gateway. |
| Chain Authority | `https://ca.gateway.example/` | The gateway platform's Chain Authority; issues assertions for its workloads. |
| WikiRAS | `https://ras.wiki.example/` | Resource Authorization Server for the wiki upstream. |
| WikiAPI | `https://api.wiki.example/` | Upstream protected resource behind WikiRAS. |
| Handle | `Gm2sVe7XpB5tK9nLw4QzCd` | Root hop handle. |
| Subject | `alice-wiki-subject` | Alice's pairwise subject at WikiRAS. |

At a glance, the message flow is:

~~~
Alice authenticates in agent-app.

AgentApp -> IdP:            (1) direct exchange: Alice's ID Token
IdP -> AgentApp:            (2) ID-JAG(GatewayRAS) + handle
AgentApp -> GatewayRAS:     (3) ID-JAG
GatewayRAS -> AgentApp:     (4) gateway access token
AgentApp -> Gateway:        (5) tool call + Continuation-Handle

Gateway resolves the tool call to the wiki upstream.

Gateway -> ChainAuthority:  (6) request assertion for the handle
ChainAuthority -> Gateway:  (7) Identity Continuation Assertion
Gateway -> IdP:             (8) chained exchange (assertion + DPoP)
IdP -> Gateway:             (9) ID-JAG(WikiRAS) + handle
Gateway -> WikiRAS:         (10) ID-JAG
WikiRAS -> Gateway:         (11) wiki access token
Gateway -> WikiAPI:         (12) tool call executes
~~~

The subsections below detail each step.

## Root Exchange: The Runtime Establishes the Chain

Alice authenticates in `agent-app`, a confidential client whose `client_id`
is the audience of her identity assertion, so `agent-app` performs the
direct exchange of {{token-exchange}}, including the `continuation` parameter
({{root-establishment}}), for the one audience it does
know: the gateway (`audience=https://ras.gateway.example/`) (steps 1 and 2).
Because tool routing is dynamic, the IdP records the envelope's
authorization basis as Alice's standing consent and tenant policy, with no
enumerated targets ({{validation}}, rule 14), and `agent-app` as the
authenticated root actor. Enterprise policy registers `tool-gateway` as an
actor permitted to continue chains rooted this way ({{validation}}, rule 10).
The response returns `continuation_handle` `Gm2sVe7XpB5tK9nLw4QzCd`.

`agent-app` exchanges its gateway ID-JAG at `ras.gateway.example` for an
access token (steps 3 and 4) and invokes the gateway, conveying the `continuation_handle`
with the request per the HTTP binding ({{context-binding}}) (step 5):

~~~
Continuation-Handle: Gm2sVe7XpB5tK9nLw4QzCd
~~~

## Chained Exchange: The Gateway Continues

A tool call in Alice's session requires the wiki service. Only the gateway
knows this routing. `tool-gateway` requests an Identity Continuation
Assertion from `https://ca.gateway.example/`, proving control of its key so
the Chain Authority can bind `cnf` to it ({{assertion-issuance}}) (steps 6
and 7):

~~~ json
{
  "iss": "https://ca.gateway.example/",
  "aud": "https://idp.example/",
  "continuation_handle": "Gm2sVe7XpB5tK9nLw4QzCd",

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
DPoP-bound to its key (step 8):

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
evaluates
the basis now: wiki read access is within Alice's standing
consent, and tenant policy permits the gateway to reach it. It resolves
Alice's wiki-local subject and mints (step 9):

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.wiki.example/",
  "sub": "alice-wiki-subject",

  "client_id": "tool-gateway",
  "resource": "https://api.wiki.example/",
  "scope": "wiki.read",

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

The gateway exchanges this at `ras.wiki.example` for an access token and
calls the wiki API (steps 10 through 12). A different tool call tomorrow
reaches a different upstream through the same machinery: a fresh assertion, a
fresh continuation-time policy decision, no new consent ceremony, and the
same denial behavior as {{example-dynamic}} for targets or scopes outside
Alice's standing consent. Concurrent tool calls present the same stored root
handle and become sibling hops with independent lineage.

## Points Worth Noticing

The identity assertion's audience check is never weakened. Alice's assertion
is presented exactly once, by the client that is its audience. The gateway
never presents it; the gateway's authority is a continuation assertion bound
to the gateway's own key, evaluated against the envelope. Strict audience
validation for identity assertions and a working proxy topology coexist.

The upstream Resource Authorization Server consumes an ordinary ID-JAG whose
`act` chain states precisely what a proxy topology needs stated: this
gateway, acting for a delegation rooted by this runtime for this user, for
this audience and scope, under a policy decision made at continuation time.
The lineage is authenticated by the IdP, not reported by the gateway
({{onward-id-jag}}).

Neither party ever holds the other's power. The runtime performed one
exchange for one audience it already knew; the gateway can request only what
the envelope's basis permits, per target, per hop, revocable at
the IdP ({{lifecycle}}).

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
   own-domain actors only, the verified leaf outermost, the IdP composing
   the segment atop the hop lineage and collapsing an actor that appears in
   both so each appears once, with depth policy applied to the composed
   chain. Enabling it is additive, lifting the single-level schema
   constraint of {{assertion-claims}}; an IdP at this version rejects
   nested `act` values deterministically. Should that extension ship, or is
   offline-actor audit better left to the evidence layer
   ({{I-D.mcguinness-oauth-actor-receipts}},
   {{I-D.mcguinness-oauth-actor-proofs}})?

2. **Signed assertion versus a bare grant type.** {{rationale-grant-type}}
   documents the alternative in which the continuing workload presents the
   handle directly under client authentication, with no assertion, Chain
   Authority, or per-assertion replay state. Which do implementers prefer,
   and is the Chain Authority's remaining role (actor and key vouching plus
   a domain-local gate) worth its trust configuration?

3. **Pull topology.** {{rationale-pull}} documents resolution at the target
   Resource Authorization Server as a candidate companion profile that moves
   the integration cost from the IdP side to the target side.

4. **Mutual-TLS binding.** Deliberately not specified while the ID-JAG
   profile defines only DPoP-based binding
   ({{sender-constrained-presentation}}). Should the two profiles add
   mutual-TLS binding together?

5. **Richer `continuation` values.** The request parameter is a flag, and
   negotiation of lifetime, depth, or permitted continuers is reserved for
   future values ({{root-establishment}}). Is client-side negotiation of
   chain properties needed at all?

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
