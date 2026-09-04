---
title: "Identity Continuation Assertion for OAuth 2.0 Token Exchange"
abbrev: "Identity Continuation Assertion"
category: std

docname: draft-mcguinness-oauth-id-continuation-assertion-latest
submissiontype: IETF
number:
date:
consensus: false
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
  I-D.ietf-oauth-transaction-tokens:
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
  RFC8417:
  RFC8705:
  RFC9068:
  RFC9700:
  I-D.fletcher-transaction-token-chaining-profile:
  I-D.ietf-oauth-identity-chaining:
  I-D.ietf-wimse-arch:
  I-D.li-oauth-delegated-authorization:
  I-D.mcguinness-oauth-actor-receipts:
  I-D.mcguinness-oauth-actor-proofs:
  GRANT-MGMT:
    title: "Grant Management for OAuth 2.0"
    target: "https://openid.net/specs/oauth-v2-grant-management.html"
    date: false
    author:
      - org: "OpenID Foundation"

...

--- abstract

This document defines the Identity Continuation Assertion, a short-lived,
sender-constrained JWT used as an OAuth 2.0 Token Exchange subject token. It
lets an IdP Authorization Server (IdP) issue an onward Identity Assertion JWT
Authorization Grant (ID-JAG) when a user's request crosses service boundaries
after the user is no longer present. The profile targets deployments in which
several Resource Authorization Servers trust one IdP and use audience-local
subject identifiers that only the IdP can resolve. It complements offline
attenuation for intra-domain fan-out that does not change the subject.

--- middle

# Introduction

OAuth 2.0 {{RFC6749}} issues access tokens for use at protected resources, and
OAuth 2.0 Token Exchange {{RFC8693}} trades one token for another when a
request crosses a trust boundary. The Identity Assertion JWT Authorization
Grant (ID-JAG) {{I-D.ietf-oauth-identity-assertion-authz-grant}} applies Token
Exchange to identity: an IdP Authorization Server (IdP) mints a grant that
names the user for a single downstream audience. Each such exchange assumes
the subject's credential (an ID Token, refresh token, or SAML assertion) is
still available when the grant is minted.

In practice, many requests outlive that moment. An authenticated request may
cross several services after the user is no longer present, or reach an
audience the original credential never addressed. The first hop can still
present the subject's credential to obtain an ID-JAG, but a workload further
along the chain holds none of these credentials. This is hardest when each
Resource Authorization Server names the user by a pairwise, audience-local
subject that only the IdP can resolve, and that differs at every server: the
later workload has no way to name the user for the next audience. Only the IdP
can perform that mapping, so each continuation is a fresh mint from the IdP
rather than a reused or offline-attenuated token.

This document defines the Identity Continuation Assertion: a short-lived,
sender-constrained JWT that a later workload presents as the `subject_token`
of a Token Exchange request to obtain the next audience-scoped ID-JAG, with no
further user interaction. The assertion carries a continuation handle that
ties the request to the authorization state the IdP recorded when the chain
was established. A Resource Authorization Server (RAS) still trusts only the
IdP to name the user and scope authority; at each hop the IdP resolves
identity and re-checks the requested authority against the root-chain
envelope. Continuation therefore stays a fresh policy decision at every
boundary, never a bearer of standing authority.

This protocol assigns obligations to two roles in the trust domain a chain
continues from: a RAS that accepts an ID-JAG and binds the hop, and a
Continuation Assertion Issuer (CAI) that mints the assertion the workload
presents to the IdP. It defines their cross-domain artifacts and one request
for obtaining the assertion at a CAI token endpoint, leaving intra-domain
handle transport to the deployment ({{overview}}).

The profile stays deliberately narrow: it defines no new access-token format,
a Resource Server never consumes the Identity Continuation Assertion directly,
and a CAI never names the user for the target audience.

## When to Use This Profile Versus Offline Attenuation {#decision-rule}

Use this profile when a boundary re-mints the user's identity, that is:

* the next audience uses a pairwise subject only the IdP can resolve;
* the target trusts the IdP, not the previous issuer, to name the user; and
* current revocation and policy must be rechecked at every boundary.

Use offline attenuation, such as {{I-D.li-oauth-delegated-authorization}}, when
the subject and the trusted issuer both stay stable across the boundary and
offline delegation semantics are acceptable, for example intra-domain fan-out
under one workload identity. The two compose: offline attenuation inside a
trust domain, continuation where a boundary re-mints the subject.

## Relationship to ID-JAG and Identity Chaining

This document profiles Token Exchange {{RFC8693}}, JWT {{RFC7519}}, ID-JAG
{{I-D.ietf-oauth-identity-assertion-authz-grant}}, and OAuth Identity Chaining
{{I-D.ietf-oauth-identity-chaining}}; {{overview}} lists what it adds.

# Conventions and Definitions {#terms}

{::boilerplate bcp14-tagged}

This document uses the following terms:

IdP Authorization Server (IdP):
: The authority that authenticates the user, maps the user to each
  audience-local subject, and issues onward grants.

Resource Authorization Server (RAS):
: An Authorization Server that protects a particular API and trusts the
  IdP for subject resolution. It exchanges an ID-JAG for an API access token.
  {{I-D.ietf-oauth-identity-assertion-authz-grant}} abbreviates this role (AS).

Resource Server (RS):
: The server hosting the protected API. It never consumes an Identity
  Continuation Assertion or uses a continuation handle for authorization.

ID-JAG:
: An Identity Assertion JWT Authorization Grant
  {{I-D.ietf-oauth-identity-assertion-authz-grant}} issued for a target RAS.

Continuation-capable:
: Describes any of: an ID-JAG that carries the `identity_continuation_handle`
  claim ({{chain-id}}); a governing authorization under which server-side
  consent and tenant policy permit continuation ({{root-establishment}}); or
  the root exchange such an authorization governs.

Identity Continuation Assertion:
: A short-lived, sender-constrained JWT from a CAI, presented to
  the IdP as a Token Exchange `subject_token` to obtain an onward ID-JAG.

Chain:
: An IdP-held tree of hops under one governing authorization ({{lifecycle}}).

Continuation Assertion Issuer (CAI):
: The role trusted by the IdP to issue Identity Continuation Assertions for a
  tenant. It never resolves the target audience's user subject.

Current actor (presenting actor):
: The workload presenting the assertion to the IdP, named by `act` and
  authenticated by `actor_token`.

Root actor:
: The actor at the root of a chain: the authenticated OAuth client that
  obtains the first ID-JAG ({{client-identity}}); this is the Client of
  {{I-D.ietf-oauth-identity-assertion-authz-grant}} at the root exchange.
  Unlike a current actor, it need not present an `actor_token`.

Tenant:
: The administrative boundary within which the chain and CAI trust are
  configured; its determination is deployment-defined
  ({{security-trust-model}}).

Trust domain:
: An administrative and authentication boundary within which workloads can
  be directly authenticated, comparable to WIMSE
  {{I-D.ietf-wimse-arch}}. Its identifier is deployment-defined.

Intra-domain carrier:
: The server-derived, deployment-specific mechanism that surfaces an accepted
  hop's handle to authorized workloads within a trust domain
  ({{handle-propagation}}).

Authenticated context:
: The authenticated credential or state that selects exactly one RAS-bound
  authorization for a call ({{handle-propagation}}).

Continuation Handle (`identity_continuation_handle`):
: An opaque, unguessable, IdP-generated reference to one hop of a continuation
  chain; see {{chain-id}}.

Hop:
: A root or continuation record; a continuation record carries an immutable
  parent reference and a root record has none. Its lineage is its path to the
  root. A hop from which no workload continues is terminal; its branch ends
  there, while sibling branches may continue.

Governing authorization:
: The server-side consent and policy record, resolved from the root subject
  token, that anchors a chain and bounds every continuation under it
  ({{lifecycle}}).

Root-chain envelope:
: The state the IdP records at establishment and evaluates every continuation
  against, anchored to the chain's governing authorization ({{lifecycle}}); its
  dimensions and bounds are defined in {{root-establishment}}.

Audience-local (pairwise) subject:
: The subject identifier under which a particular RAS names the user. Distinct
  Resource Authorization Servers may name the same user with different
  identifiers; only the IdP holds the map between them.

Offline attenuation:
: Client-side attenuated delegation, in which a party narrows and forwards a
  credential without contacting the IdP; contrast the IdP-minted continuation
  this profile defines ({{decision-rule}}).

# The Identity Continuation Assertion {#assertion}

## Token Type and Media Type {#names}

The Identity Continuation Assertion has token type
`urn:ietf:params:oauth:token-type:identity-continuation` and media type
`application/oauth-identity-continuation+jwt` ({{iana}}). It is a JWT
{{RFC7519}} in JWS Compact Serialization. The CAI MUST set the JOSE `typ`
header to `oauth-identity-continuation+jwt` and MUST sign with an asymmetric
algorithm the IdP accepts ({{security-alg}}). The assertion MUST NOT be
encrypted (JWE) or use nested signing.

## Claims {#assertion-claims}

The following is a non-normative example of the Identity Continuation Assertion
claim set:

~~~ json
{
  "iss": "https://cai.expenses.example/",
  "aud": "https://idp.example/",
  "identity_continuation_handle": "kW4uJ8pTe2NxA6rQvD1zYs",

  "act": {
    "iss": "https://expenses.example/",
    "sub": "expense-service"
  },

  "cnf": {
    "jkt": "base64url-current-actor-key-thumbprint"
  },

  "iat": 1710000500,
  "exp": 1710000620,
  "jti": "k7Qm2Xp9Rf4sLc3vBw8aZ1"
}
~~~

The claims have the following meanings and requirements:

`iss`:
: REQUIRED. The CAI that issued the assertion; the IdP verifies its issuer
  trust per {{validation}}, rule 3, and its signature per rule 2.

`aud`:
: REQUIRED. A single string exactly matching the IdP issuer identifier, not
  its token endpoint URL.

`identity_continuation_handle`:
: REQUIRED. The hop being continued ({{chain-id}}).

`act`:
: REQUIRED. The current actor presenting the Token Exchange request, encoded
  as a single-level `act` claim per {{RFC8693}}, with a REQUIRED `iss` and
  `sub`, both non-empty strings. Additional members MAY carry further identity
  attributes but are non-authoritative and MUST NOT affect identity,
  authorization, lineage, or issuance; a recipient MUST ignore members it does
  not understand, and `exp`, `nbf`, `aud`, `scope`, `cnf`, and nested `act`
  MUST NOT be present.

`cnf`:
: REQUIRED. A confirmation claim {{RFC7800}} that binds the assertion to the
  presenting actor's key. It MUST contain exactly one method: `jkt`, the JWK
  SHA-256 thumbprint {{RFC7638}} of the DPoP key {{RFC9449}}.

`iat`, `exp`:
: REQUIRED. `exp` MUST follow `iat`, and `exp - iat` MUST NOT exceed 300
  seconds.

`jti`:
: REQUIRED. A replay-detection identifier that MUST be unique per `iss`
  during the assertion validity window and MUST contain at least 128 bits
  of entropy.

The assertion is a subject token whose subject the IdP resolves from the
referenced hop, not an {{RFC7523}} JWT-profile assertion. It MUST NOT contain
a top-level `sub`, `auth_time`, `acr`, `amr`, or `sid` claim (these come from
the root-chain envelope), nor the Token Exchange request parameters
`audience`, `resource`, `scope`,
`authorization_details`, or `requested_token_type` (these are supplied by
the request). The assertion's `aud` identifies the IdP, not the requested
target.

Other top-level claims MAY appear but MUST be ignored for validation,
authorization, and issuance.

# Continuation Handles (`identity_continuation_handle`) {#chain-id}

An `identity_continuation_handle` is an opaque, non-bearer reference to one
IdP-held hop of a chain. The IdP mints a fresh handle for each hop and carries
it in that hop's ID-JAG; continuing from a hop produces a child hop with its
own handle, recorded against the hop it continued from.

The following rules apply:

1. When it establishes or continues a chain ({{root-establishment}}), the IdP
   MUST embed a fresh `identity_continuation_handle` claim in the issued
   ID-JAG, for that root or child hop, and MUST NOT reuse a handle across hops.
   An ID-JAG carrying this claim is continuation-capable.

2. `identity_continuation_handle` MUST contain at least 128 bits of entropy,
   MUST NOT contain user-identifying information, and MUST consist of 22 to
   256 characters drawn from the base64url alphabet (`A`-`Z`, `a`-`z`,
   `0`-`9`, `-`, `_`).

3. Across a trust boundary, the handle is accepted only inside an ID-JAG (by
   the RAS) or an Identity Continuation Assertion (by the IdP), never as a
   standalone value. Inside the accepting RAS's domain, the CAI obtains it
   from RAS state or a carrier ({{handle-propagation}}); afterward it travels
   in the assertion.

4. A continuation-aware RAS binds the handle to the authorization state it
   establishes ({{ras-processing}}); RASes, Resource Servers, and CAIs MUST NOT
   modify the value. A hop is continuable only after that acceptance and binding
   ({{hop-activation}}); the IdP MUST use the handle only to resolve hop state,
   subject, and policy, never as authority.

5. A hop's parent reference is immutable. The IdP MUST derive lineage only by
   walking parent references to the root; it does not maintain a single
   chain-wide actor history, so concurrent sibling continuations are
   independent branches.

# Chain Lifetime and Revocation {#lifecycle}

A chain is continuable only while active at the IdP. Each cross-boundary hop
is a fresh policy check. Revoking a hop stops its subtree at the next
continuation, fail-closed, but does not invalidate already-issued ID-JAGs or
access tokens; the revocation window is therefore bounded by their respective
lifetimes.

This is the deliberate difference from an offline-attenuated token, whose
minted child stays usable for its lifetime without contacting an authority.

Three independent lifetimes govern a continuation: the ID-JAG's short
redemption window; the access-token lifetime the accepting RAS sets, which
this profile does not constrain ({{ras-processing}}); and the IdP-held
continuation chain. Revoking the chain does not shorten an already-issued
access token, and an access token outliving the chain does not extend it.

~~~
ID-JAG redeem   |==|
access token    |===========|              RAS-set, independent
IdP-held chain  |=========================| IdP-held, spans hops
~~~

The governing authorization ({{terms}}) anchors to a lifecycle token: a refresh
token anchors to its OAuth grant, and `sid` or `SessionIndex` anchors to its
session. Rotation of a refresh token does not affect the grant anchor.

A chain ends when:

* the grant it is anchored to expires or is revoked;
* the session it is anchored to terminates; or
* continuation consent or policy for it is withdrawn.

A session-anchored chain MUST NOT outlive its session; only grant-anchored
chains may outlive logout. Ending a chain this way bounds only new
continuations; an ID-JAG already issued remains redeemable for its own
lifetime, since redemption is not a continuation.

The IdP MUST bound chain lifetime by the governing
authorization. It MUST support administrative revocation of an entire chain
and MAY revoke an individual hop's subtree, and MUST reject continuation on a
revoked, expired, or ended chain.

How an IdP surfaces chains to users and administrators for review and
revocation is deployment-specific; {{GRANT-MGMT}} describes OAuth grant
management for that purpose.

# Multi-Hop Cross-Domain Access {#access}

## Overview {#overview}

This document specifies:

* the `identity_continuation_handle` claim on continuation-capable ID-JAGs
  ({{chain-id}});
* the Identity Continuation Assertion and its use as a Token Exchange
  `subject_token` ({{assertion}}, {{token-exchange}});
* Token Exchange issuance of the assertion by a CAI that is an authorization
  server ({{assertion-token-exchange}});
* the IdP's validation of the continuation exchange ({{validation}});
* RAS binding of the handle from an accepted ID-JAG ({{ras-processing}}); and
* optional Authorization Server metadata, including
  `identity_continuation_issuers` ({{metadata}}).

How a workload obtains an assertion by other means, and how the bound handle
reaches a separate CAI inside the RAS's trust domain, are deployment-specific,
subject to the provenance rule in {{handle-propagation}}.

The accepting RAS may itself hold the CAI role (co-located), or a separately
mapped CAI may hold it (separate); see {{deployment-topologies}}. The IdP
accepts an assertion for a hop only from a CAI mapped to the hop's
accepting RAS ({{validation}}, rule 3), and the RAS itself is always
mapped.

A continuation reuses the Token Exchange loop once per boundary: the root
exchange mints the first ID-JAG, and each later boundary mints the next from an
Identity Continuation Assertion. Handles H0 and H1 below name the successive
hops ({{chain-id}}). The diagram shows the co-located topology; steps marked
new are this profile's additions to ID-JAG, and the last exchange shows the
base protocol resuming at a RAS that does not implement this extension
({{ras-processing}}).

~~~
  Client      Workload       IdP       RAS/CAI      Next RAS
  |           |              |            |              |
  | ID-JAG exchange          |            |              |
  |------------------------->|            |              |
  | ID-JAG (H0)   [new: handle claim]     |              |
  |<-------------------------|            |              |
  | jwt-bearer grant with the ID-JAG      |              |
  |-------------------------------------->|              |
  | access token  [new: RAS binds H0]     |              |
  |<--------------------------------------|              |
  |           |              |            |              |
  |           | request assertion [new: token exchange]  |
  |           |-------------------------->|              |
  |           | assertion [new: resolve H0, attest]      |
  |           |<--------------------------|              |
  |           | continuation exchange [new]              |
  |           |------------->|            |              |
  |           | ID-JAG (H1)  |            |              |
  |           |<-------------|            |              |
  |           | jwt-bearer grant, ordinary ID-JAG        |
  |           |----------------------------------------->|
  |           | access token, no binding; branch ends    |
  |           |<-----------------------------------------|
~~~

H1 then travels to the next boundary exactly as H0 did, and the loop repeats
until a hop reaches a terminal RAS. A hop is PENDING when the IdP issues its
ID-JAG, ACCEPTED once the RAS binds it, and CONTINUABLE while a mapped CAI
attests the still-active binding ({{hop-activation}}). The workload that
continues is a later party, not the original client that established the chain.

Each role validates only within its authority, and no artifact or role alone
authorizes continuation ({{security-trust-model}}). The sections that follow
trace artifact production and processing: the CAI issues the assertion
({{assertion-issuance}}), the IdP validates the exchange and issues the next
ID-JAG ({{token-exchange}}), the accepting RAS binds and processes it
({{ras-processing}}), and the domain surfaces the handle to later continuers
({{handle-propagation}}).

## Assertion Issuance {#assertion-issuance}

The CAI mints the Identity Continuation Assertion a workload presents to
continue a chain across a boundary. It MUST issue only for an actor in the
attested RAS's trust domain; actor authentication is deployment-specific. The
CAI MUST set the assertion's `aud` to the IdP recorded in the hop's RAS binding
({{ras-processing}}); it MUST NOT accept an IdP audience supplied by the
requester.

The current actor is a control-plane participant, not a bare-handle
transporter: the CAI obtains the handle from authenticated state associated
with the actor's transaction, and the actor separately proves the key placed in
`cnf`. The handle is advisory input, re-verified against RAS-bound state
({{hop-activation}}) by the preconditions below before any assertion issues.

### Preconditions {#assertion-preconditions}

The CAI MUST authenticate the actor and issue only after establishing that:

1. the handle came through an authenticated, confidential,
   integrity-protected chain path or equivalent authenticated state;

2. the current actor is authorized under CAI policy to continue the chain;

3. the current actor controls the key placed in `cnf`;

4. `act` names that actor and, if offline attenuation reached the actor, its
   delegation artifact is valid;

5. the actor is bound to the current transaction and the handle matches that
   transaction's RAS-bound state; and

6. a recheck against authoritative RAS state, whatever the carrier, confirms
   the authorization remains active and continuation remains permitted.

Possession of a handle or carrier token alone is insufficient. Target or
purpose hints can narrow CAI issuance but MUST NOT control the IdP's target
decision, and propagated context MUST NOT override the root-chain envelope.

### Request {#assertion-token-exchange}

A CAI that is an OAuth authorization server, including a RAS acting as its own
CAI, MAY issue assertions from its token endpoint using Token Exchange
{{RFC8693}} as profiled in this and the following subsections. A RAS acting as
its own CAI SHOULD support this profile, so that a workload in its domain has
one request to implement. Issuance by other means remains deployment-specific
({{handle-propagation}}). The requesting party is the current actor
({{terms}}), acting as an OAuth client of the CAI; this profile calls it the
client.

The client makes a Token Exchange request to the CAI's token endpoint with the
following parameters:

`grant_type`:
: REQUIRED. The value `urn:ietf:params:oauth:grant-type:token-exchange`.

`requested_token_type`:
: REQUIRED. The value
  `urn:ietf:params:oauth:token-type:identity-continuation`.

`subject_token`:
: REQUIRED. Either the access token the client received on the call it is
  continuing, or the Transaction Token that carries the hop's handle for that
  call.

`subject_token_type`:
: REQUIRED. `urn:ietf:params:oauth:token-type:access_token` or
  `urn:ietf:params:oauth:token-type:txn_token`
  {{I-D.ietf-oauth-transaction-tokens}}, matching the `subject_token`.

The `audience`, `resource`, `scope`, `actor_token`, and `actor_token_type`
parameters MUST NOT be included. The assertion carries no target or scope,
which are chosen at the IdP exchange; its `aud` is the IdP recorded in the
hop's binding; and the authenticated client is the actor named in `act`.

The client MUST authenticate to the token endpoint ({{RFC6749}}, Section 2.3),
and the request MUST include a DPoP proof {{RFC9449}}; the CAI binds the
assertion to the proven key in `cnf`. For example:

~~~
POST /token HTTP/1.1
Host: cai.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof of possession of the key to be placed in cnf>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:identity-continuation
&subject_token=<access token presented to the client>
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<JWT>
~~~

The CAI MUST verify that the `subject_token` is one of the following:

* an access token issued by a RAS the CAI is mapped to, unexpired, and valid
  for a protected resource that the authenticated client operates. The client
  is then a resource server exchanging a token it received, the scenario of
  the example in {{RFC8693}}, Section 2.3. A RAS acting as CAI resolves its
  own token; a separate CAI resolves it through introspection {{RFC7662}}; or
* a Transaction Token valid under Section 12.2 of
  {{I-D.ietf-oauth-transaction-tokens}} for the CAI's trust domain and
  carrying the hop's handle as chain context ({{handle-propagation}}). The
  Transaction Token is the carrier.

Either way, the authorization behind the token is the one the preconditions
({{assertion-preconditions}}) test: the token's integrity protection and the
authenticated request satisfy precondition 1, the authenticated client is the
actor of preconditions 2 and 4, the DPoP key is the key of precondition 3, and
the bound handle and live state satisfy preconditions 5 and 6.

### Successful Response {#assertion-response}

A successful response is a Token Exchange response ({{RFC8693}}, Section
2.2.1) in which `access_token` carries the Identity Continuation Assertion,
`issued_token_type` is
`urn:ietf:params:oauth:token-type:identity-continuation`, `token_type` is
`N_A`, and `expires_in` reflects the assertion's lifetime. This profile adds
one parameter:

`audience`:
: REQUIRED. The issuer identifier of the IdP to which the client presents the
  assertion, equal to the assertion's `aud`. Because the request carries no
  `audience`, this is how the client learns where the assertion goes. The
  client obtains that IdP's `token_endpoint` from its authorization server
  metadata ({{RFC8414}}), retrieved with the `oauth-authorization-server`
  well-known URI suffix under the issuer identifier, after confirming that the
  returned `issuer` exactly matches `audience`; where the IdP publishes no
  metadata, the client uses configuration bound to that issuer identifier
  ({{metadata}}).

The response MUST NOT include a `refresh_token`, which would let a client
obtain further assertions without presenting a token or passing the live
recheck.

~~~
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "issued_token_type": "urn:ietf:params:oauth:token-type:identity-continuation",
  "access_token": "<Identity Continuation Assertion, compact JWS>",
  "token_type": "N_A",
  "audience": "https://idp.example/",
  "expires_in": 120
}
~~~

### Error Response {#assertion-error-response}

On failure, the CAI returns an OAuth error ({{RFC6749}}, Section 5.2;
{{RFC8693}}, Section 2.2.2):

* `invalid_request` when the `subject_token` is invalid or unacceptable under
  policy, including when it is unknown, expired, revoked, not valid for a
  resource the client operates, has no bound handle, or names an authorization
  the client is not permitted to continue;
* `unauthorized_client` when the client is not permitted to use this grant
  type; and
* `invalid_dpop_proof` ({{RFC9449}}) for a failed proof.

## Token Exchange {#token-exchange}

An Identity Continuation Assertion is used as the `subject_token` of an OAuth
2.0 Token Exchange request {{RFC8693}}. The root exchange and a continuation
exchange use the same Token Exchange framework: a continuation exchange
substitutes an Identity Continuation Assertion for the root credential and
additionally supplies the actor authentication and DPoP proof described below.
The IdP establishes the chain; no request parameter asks it to do so
({{root-establishment}}). Before it can continue to a target, the current
actor needs a client registration or resolvable client identity at that
target's RAS; the onward ID-JAG `client_id` is that identifier
({{onward-id-jag}}).

### Request {#request}

The root exchange presents a normal subject token, such as an ID Token,
refresh token, or SAML assertion:

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof of possession of the cnf key>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:id-jag
&audience=https://ras.travel.example/
&resource=https://api.travel.example/
&scope=trips.read
&subject_token=<id_token | refresh_token | SAML assertion>
&subject_token_type=<normal-subject-token-type>
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<JWT>
~~~

On the root exchange, `actor_token` is OPTIONAL ({{root-establishment}}). The
root exchange and its ID-JAG conform to the base ID-JAG profile
({{I-D.ietf-oauth-identity-assertion-authz-grant}}) except where this document
extends it for continuation-capable issuance.

A continuation exchange presents an Identity Continuation Assertion and adds
the `actor_token` and a DPoP proof of the `cnf` key:

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof of possession of the cnf key>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:id-jag
&audience=https://ras.travel.example/
&resource=https://api.travel.example/
&scope=trips.read
&subject_token=<identity-continuation-assertion>
&subject_token_type=urn:ietf:params:oauth:token-type:identity-continuation
&actor_token=<sender-constrained-current-actor-credential>
&actor_token_type=<actor-token-type>
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<JWT>
~~~

The requested `audience`, `resource`, `scope`, `requested_token_type`, and
any `authorization_details` {{RFC9396}} are supplied by the Token Exchange
request and never by the assertion ({{assertion-claims}}). A request can carry
multiple `resource` indicators {{RFC8707}}, which the IdP treats as an
order-independent set; the envelope-containment check ({{validation}}, rule 7)
applies to `authorization_details` and to scope alike. Client authentication
is required on every exchange ({{client-identity}}).

### Establishing a Chain {#root-establishment}

The IdP, not the client, establishes a chain, and MUST do so when the
governing authorization for a root exchange is continuation-capable;
advertised support ({{metadata}}) signals capability, not authority. To
establish, the IdP MUST include the root handle in the ID-JAG. The root
exchange MUST include a valid DPoP proof {{RFC9449}}, which the IdP MUST bind
to the ID-JAG in `cnf`. Absent the continuation authorization or a valid
proof, the IdP MUST NOT establish a chain or include an
`identity_continuation_handle`.

The root subject token MUST resolve to one of these lifecycle anchors:

* a refresh token's OAuth grant;
* an ID Token `sid` {{OIDC.FrontChannelLogout}} resolving to an active IdP
  session for that user and client; or
* a SAML `SessionIndex` {{SAML2.Core}} resolving to an active IdP session
  for that user and client.

The IdP MUST NOT root a chain from an unresolved anchor or an access token;
non-user-rooted authority is out of scope. `sid` and `SessionIndex` are used
only for resolution and MUST NOT enter assertions or chain context.

Server-side consent and policy make the governing authorization
continuation-capable and populate the root-chain envelope from authentication,
consent, and tenant policy:

* the authenticated user and authentication context (`auth_time`, `acr`,
  `amr`);
* the authorization basis for onward targets;
* the continuation authorization: the actors or trust domains permitted to
  continue the chain, and the basis on which that permission was established;
* any maximum actor-chain depth set by policy; and
* the governing authorization ({{lifecycle}}) and the chain's expiry.

Token claims cannot supply these values. Every dimension is an
establishment-time ceiling that later policy MAY narrow or revoke but MUST NOT
broaden; broadening requires a new chain, and consent granted afterward does
not widen the envelope.

The envelope takes one of two forms:

* an enumerated-target envelope, listing the permitted audiences and, where
  used, their resources, scopes, and authorization details {{RFC9396}}; or
* a policy-basis envelope, a stable authorization basis against which the IdP
  evaluates each requested target at request time.

Either form is fixed at establishment, not whatever the user could later
authorize.

The root actor is the authenticated OAuth client; its identity rests entirely
on the mapping in {{client-identity}}. Base ID-JAG's recommendation to use a
confidential client therefore applies to a continuation-capable root. An
optional `actor_token` MUST be valid, accepted for continuation, and
sender-constrained to the confirmed key, and MUST
identify that client and designate the IdP where applicable; the IdP records
the root actor and key only after this validation.

For every root or child hop, the IdP records the RAS audience placed in the
ID-JAG and any additional CAIs mapped to it; those mappings may be static
tenant configuration.

Establishment is at-least-once: retrying a lost response MAY create a second
chain. Revocation of the governing authorization applies to every chain rooted
in it, and the actor-chain depth bound is enforced per branch. Fan-out, rate,
or hop-count limits configured for a governing authorization apply across
every chain rooted in it, so sibling chains share one budget; a retried
establishment MUST NOT evade them.

### Presenter Authentication {#client-identity}

The client-to-actor mapping below applies to every exchange; the `act` and
`actor_token` matching and the DPoP proof apply to a continuation exchange, and
the root exchange's DPoP requirement is specified in {{root-establishment}}.

The current actor MUST authenticate as an OAuth client. Four rules govern how
that authentication maps to an actor identity:

* *Authoritative mapping.* The IdP MUST map the authenticated client to an
  actor identity; self-asserted mappings MUST NOT be accepted.
* *Root versus continuation.* On a root exchange, client authentication alone
  identifies the root actor ({{root-establishment}}). On a continuation
  exchange, the IdP MUST also match that identity to the assertion's `act` and
  the `actor_token`.
* *Dual-use JWT.* A sender-constrained JWT MAY serve as both client assertion
  and `actor_token` when it satisfies both profiles; for {{RFC7523}} its `sub`
  is the `client_id` and the IdP MUST authorize its issuer for that client.
  Otherwise the client authenticates separately.

The `actor_token` MUST NOT be bearer: for a JWT the IdP verifies `cnf.jkt`,
and for an opaque token it obtains equivalent confirmation from authoritative
metadata such as introspection {{RFC7662}}; its issuer, acceptance, sender
constraint, and applicability are checked by {{validation}} rule 5. The IdP
MUST compare the actor `iss` and `sub` as case-sensitive strings with no
transformation or canonicalization ({{RFC7519}}), across `actor_token`, `act`,
and the authenticated client, and identities in different tenants never
compare equal.

The actor MUST present a DPoP proof {{RFC9449}} for the key in `cnf.jkt`.
DPoP is the single mandatory confirmation method, so the target validates
confirmation identically to a directly issued ID-JAG, and this version defines
no mutual-TLS variant {{RFC8705}} ({{open-items}}). The onward ID-JAG MUST use
the same DPoP key; key rotation takes effect when the actor obtains a new
assertion and actor token bound to the new key.

Four signals bind the continuation request to the current actor. The
identities and key possession they establish must be mutually consistent:

| Signal | What it establishes |
|---|---|
| Client authentication | who is calling the IdP token endpoint |
| `actor_token` | the actor vouched for by its workload-identity issuer |
| Assertion `act` | the actor the CAI bound to the accepted hop |
| DPoP | live possession of the key binding all three to this request |

### Request Validation {#validation}

For a continuation exchange, the IdP MUST reject the request unless every rule
below holds; their order is not significant, though one rule's input may come
from another's resolution.

1. **Request parameters.**
   * exactly one each of `grant_type`, `subject_token`, `subject_token_type`,
     `requested_token_type`, `actor_token`, `actor_token_type`, and
     `audience`;
   * zero or more `resource`, and at most one each of `scope` and
     `authorization_details`, all OPTIONAL and, when present, evaluated by
     rule 7; and
   * `grant_type` is `urn:ietf:params:oauth:grant-type:token-exchange`,
     `subject_token_type` is
     `urn:ietf:params:oauth:token-type:identity-continuation`, and
     `requested_token_type` is `urn:ietf:params:oauth:token-type:id-jag`;

2. **Assertion well-formedness.**
   * the assertion is a JWT whose JOSE `typ` header is
     `oauth-identity-continuation+jwt`;
   * it contains exactly one value for each claim required by
     {{assertion-claims}} and none of the claims that section forbids;
   * `iss`, `aud`, `identity_continuation_handle`, and `jti` are non-empty
     strings, `act` and `cnf` are JSON objects with `cnf` naming exactly one
     confirmation method, and `iat` and `exp` are NumericDate numbers;
   * the signature validates under an acceptable algorithm ({{security-alg}},
     {{RFC8725}}) with the issuer's resolved signing keys ({{metadata}}); and
   * `aud` exactly matches the IdP's issuer identifier;

3. **Issuer trust.** The assertion `iss` is either the RAS issuer identifier
   the recorded ID-JAG `aud` names, as a string or one-element array, or an
   issuer identifier mapped to that RAS; it is trusted for the tenant and
   authorized to pair with the `actor_token` issuer for that tenant;

4. **Chain state.**
   * the handle identifies a RAS-accepted hop ({{hop-activation}}) on an
     active chain;
   * no ancestor subtree is revoked; and
   * the actor lineage that results from merging consecutive same-actor
     entries, as the onward `act` will ({{onward-id-jag}}), is within its
     depth bound, which counts lineage entries, not hops;

5. **Current actor and binding.**
   * `act` is present, conforms to the schema of {{assertion-claims}}, and
     identifies the current actor, which is the OAuth client the request is
     authenticated as ({{client-identity}});
   * the `actor_token_type` names a token type the IdP supports, and the
     `actor_token` has a trusted issuer for the actor's domain and tenant, is
     valid for that type, is accepted, designates the IdP where applicable,
     authenticates the actor, and is sender-constrained to the key confirmed
     by the assertion's `cnf` ({{client-identity}});
   * the request proves possession of the `cnf` key with a matching DPoP
     proof ({{client-identity}}, {{RFC9449}});
   * the actor is permitted by the chain's continuation authorization
     ({{root-establishment}}) to continue from the presented hop; and
   * the IdP can resolve, for the requested `audience`, both the
     audience-local subject and the actor's client identifier
     ({{onward-id-jag}});

6. **Freshness and replay.**
   * `iat` is within permitted future clock skew (which SHOULD NOT exceed 60
     seconds), `exp` follows `iat`, the assertion is unexpired, and its
     lifetime does not exceed 300 seconds; and
   * `jti` is not yet reserved for the assertion issuer, or is RESERVED or
     ISSUED under a fingerprint matching this request (permitting idempotent
     retry; see {{validation-replay}}); a RESERVED or ISSUED `jti` under a
     different fingerprint, or a FAILED `jti`, is rejected;

7. **Envelope containment.** The requested audience, resource, scopes, and
   authorization details are within the root-chain envelope as recorded at
   establishment and within current IdP actor policy; authorization-details
   containment uses the comparison rules defined for each authorization-detail
   type, since {{RFC9396}} defines no generic comparison, and a detail type
   whose rules the IdP does not implement is rejected.

### Replay Reservation and Retry {#validation-replay}

The reservation model gives a client idempotent recovery after a lost response
while preventing one assertion from authorizing more than one distinct request.

After validation, grant issuance reserves the assertion's (`iss`, `jti`), bound
to a fingerprint of the request it first authorizes, and records the
reservation as RESERVED, ISSUED, or FAILED (distinct from the hop states of
{{hop-activation}}). The fingerprint MUST cover `audience` as an exact string,
the `resource` values as an order-independent set, `scope` as an
order-independent set, the exact `authorization_details` JSON after form
decoding (a different serialization is a different request), the actor's `iss`
and `sub`, the confirmed key's `cnf.jkt` thumbprint, and a SHA-256 hash of the
exact `subject_token` after form decoding, which binds the fingerprint to the
specific assertion and its handle. Concurrent redemptions of one assertion
MUST NOT produce more than one grant. An identical retry
MUST return the same previously issued grant, not a new one, and a request that
does not match that fingerprint MUST be rejected. Replay uniqueness MUST be
keyed on (`iss`, `jti`); partitioning by tenant alone would let two assertion
issuers in one tenant collide on a reused `jti`.

The IdP MUST retain the reservation through `exp` plus the maximum permitted
clock skew, so an in-window retry is honored; a reservation that does not reach
ISSUED before `exp` becomes FAILED, which is final and requires a fresh
assertion.

After a lost response, a client MAY retry the same assertion to recover the
ISSUED result or obtain a fresh assertion. A fresh assertion may create an
equivalent grant and sibling hop but no additional authority. Application
idempotency remains out of scope. Realization guidance is in {{implementation}}.

### Successful Response {#success-response}

The Token Exchange response follows the base ID-JAG profile: the ID-JAG is
returned in `access_token`, with `token_type` `N_A`.

~~~
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "issued_token_type": "urn:ietf:params:oauth:token-type:id-jag",
  "access_token": "<continuation-capable ID-JAG, compact JWS>",
  "token_type": "N_A",
  "expires_in": 300
}
~~~

The hop reference is delivered as the ID-JAG's `identity_continuation_handle`
claim ({{chain-id}}, rule 1), a claim inside `access_token` and not a separate
Token Exchange response parameter; the accepting Resource Authorization Server
binds it ({{ras-processing}}), and a mapped CAI reaches it through RAS state
or intra-domain context ({{handle-propagation}}). There is likewise no
chain-expiry response parameter: chain lifetime is authoritative at the IdP
({{lifecycle}}), and a deployment needing advance warning conveys it through
task or authorization state, an optional ID-JAG claim, or a management API.

On success, the IdP records a PENDING child ({{hop-activation}}) of the
presented hop and issues an ID-JAG containing the resolved target `sub` and
fresh handle. An idempotent retry (rule 6; {{validation-replay}}) instead
returns the previously issued grant unchanged, creating no new hop or handle.

### Onward ID-JAG Construction {#onward-id-jag}

The onward ID-JAG conforms to the base ID-JAG profile
({{I-D.ietf-oauth-identity-assertion-authz-grant}}) except where this document
extends it: its `sub` is the IdP-issued pairwise subject for the target
audience, and `aud_sub` remains available under the base profile where the
target's native subject namespace differs. The IdP MUST copy the root
authentication context (`auth_time`, `acr`, `amr`) unchanged where
{{I-D.ietf-oauth-identity-assertion-authz-grant}} requires those claims;
continuation MUST NOT extend or strengthen it, for example by raising `acr` or
adding `amr` beyond the user's root authentication. The IdP constructs `act`
by placing the authenticated current actor atop the presented hop's lineage;
it never copies lineage from the assertion, and siblings do not contribute.
Consecutive identical actors merge into one entry, though the hop record
remains; policy MAY limit disclosed depth, narrowing what a target sees without
changing the depth bound the IdP enforces ({{validation}}, rule 4). Because
policy may narrow the disclosed lineage, a Resource Authorization Server
MUST NOT read the absence of a further nested `act` as proof that no earlier
actor exists. The following is a
non-normative example of the onward ID-JAG issued by the IdP:

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.travel.example/",
  "sub": "travel-pairwise-subject",

  "client_id": "expense-service",
  "resource": "https://api.travel.example/",
  "scope": "trips.read",

  "identity_continuation_handle": "Uc9fB3mHs5LdK7gEnX2wRj",

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

  "iat": 1710000025,
  "exp": 1710000325,
  "jti": "idjag-travel-01"
}
~~~

The onward ID-JAG's `client_id` is the current actor's identifier at the
target RAS.

### Error Response and Recovery {#error-response}

On failure, the IdP returns an OAuth error ({{RFC6749}}, {{RFC8693}}):

* it MUST return `invalid_continuation` ({{iana}}) only when the handle is
  permanently unusable: unknown, on an expired or ended chain, on a revoked
  hop or ancestor, or with continuation authorization withdrawn; and
* it SHOULD use `invalid_request` for a malformed, inconsistent, or
  unacceptable token, `invalid_dpop_proof` for a DPoP failure, and
  `invalid_target`, `invalid_scope`, or `invalid_authorization_details` for a
  request outside the envelope.

An `invalid_continuation` handle is permanently unusable: retrying it cannot
succeed.
Recovery requires establishing a new chain and succeeds only where the
governing authorization is still continuation-capable: a session-anchored chain
re-roots by re-authenticating the user, a grant-anchored chain from its
still-valid grant without the user, and a handle disabled by withdrawn
continuation authorization cannot re-root at all. The other errors
leave the chain still continuable, so a client abandons only the current
request.

## Continuation-Aware RAS Processing {#ras-processing}

A RAS from which continuation may occur implements this extension. A RAS that
does not implement it processes an ordinary ID-JAG and ignores the handle, so
an ID-JAG it accepts cannot become a continuation source.

A continuation-aware Resource Authorization Server, one that implements this
extension and advertises the continuation grant profile ({{metadata}}), MUST,
on accepting a continuation-capable ID-JAG:

1. accept the ID-JAG per {{I-D.ietf-oauth-identity-assertion-authz-grant}},
   which includes validating the grant, authenticating the presenting client,
   verifying the sender constraint, applying local authorization policy, and
   issuing an access token sender-constrained to the confirmed key; and
2. bind `identity_continuation_handle`, the ID-JAG's issuer and tenant, and
   the confirmed key to the authorization state it establishes, recording
   whether continuation is permitted.

The RAS MUST bind the handle and issue the access token as one outcome: no
access token without its binding, and no binding without a token. Repeated
redemption of one ID-JAG MUST bind to the same hop authorization record, so a
retry cannot create multiple records for one grant. The RAS exposes the binding
only within its trust domain ({{handle-propagation}}).

## Hop Activation {#hop-activation}

A hop moves through three states, spanning the IdP, RAS, and CAI. These are
conceptual states, not values carried on the wire. The IdP creates it PENDING.
Successful RAS binding makes it ACCEPTED. A mapped CAI attests a hop only once
it is ACCEPTED, so a PENDING hop yields no assertion and reaches no
continuation exchange. A fresh assertion from the mapped CAI lets the IdP
evaluate the hop as CONTINUABLE for one request. CONTINUABLE is not stored: it
means a trusted mapped CAI has freshly attested the hop as ACCEPTED and still
active, no ancestor is revoked, and the presenting actor is the one the
attestation names; rules 3 through 5 of {{validation}} establish those facts.

| State | Where it lives | Meaning |
|---|---|---|
| PENDING | IdP | the IdP issued the ID-JAG but has no acceptance evidence |
| ACCEPTED | RAS authorization state | the RAS redeemed the grant, authorized it, and bound the handle |
| CONTINUABLE | IdP, for one exchange | a mapped CAI freshly attested the still-active binding |

ACCEPTED is a state of the RAS's own authorization, not an IdP transition
delivered by callback; the IdP learns of it only through a CAI attestation.

The CAI assertion is trusted evidence of acceptance, not IdP-verifiable proof.
A RAS mapped to its own hop attests that state first-hand; any other mapped CAI
rechecks authoritative RAS state before attesting ({{assertion-issuance}}).
Absent CAI compromise ({{security-trust-model}}), an issued-but-rejected ID-JAG
cannot be continued because no mapped CAI may attest it. A mapped CAI is
mandatory; its absence fails closed.

Acceptance gates continuation but does not bound downstream authority: the IdP
evaluates later targets against the root envelope, and local RAS authorization
neither narrows nor widens it.

## Intra-Domain Handle Propagation {#handle-propagation}

When the accepting RAS holds the CAI role, it reads the handle directly
from its authorization state and needs no carrier. Otherwise, a carrier derived
from the RAS binding conveys the handle within the trust domain
({{ras-processing}}) and is accepted only within that domain
({{assertion-issuance}}).

One rule applies to direct reads and every carrier. Each call has an
authenticated context that selects one RAS-bound authorization. The handle
used for that call MUST be the handle bound to that authorization, whether read
directly from RAS state or from a carrier derived from it. A requester can
choose which credential to present, but cannot supply or override its handle
separately. A session or subject alone is not enough to select the
authorization, because doing so could attach another user's handle to the call.

The authenticated context selects the authorization as follows:

* For an ingress call, the sender-constrained access token after the resource
  verifies its proof of possession.
* For a downstream call, a carrier forwarded from that ingress.
* For a scheduled run, the task named by an authenticated actor, resolved to
  the task authorization for which that actor is designated
  ({{task-provenance}}).

The CAI's recheck against RAS state ({{assertion-issuance}}) covers staleness
for every carrier.

A Resource Server has no obligations under this specification, so a carrier
SHOULD NOT expose the handle to a party with no role in continuation.
Deployments also keep it out of logs, traces, and responses.

# Authorization Server Metadata {#metadata}

## IdP Authorization Server Metadata

An IdP that supports this profile SHOULD signal it in its authorization server
metadata {{RFC8414}} with the following parameter:

`identity_continuation_supported`:
: OPTIONAL. Boolean value indicating that the IdP accepts Identity Continuation
  Assertions of the
  `urn:ietf:params:oauth:token-type:identity-continuation` subject token type
  and issues continuation-capable ID-JAGs carrying the
  `identity_continuation_handle` claim. Default `false`. A continuation-capable
  ID-JAG is still the `urn:ietf:params:oauth:token-type:id-jag` type, so this
  flag advertises the capability rather than a new token type. It composes
  additively with base ID-JAG discovery: an IdP that sets this flag also lists
  `urn:ietf:params:oauth:token-type:id-jag` in its
  `identity_chaining_requested_token_types_supported`
  ({{I-D.ietf-oauth-identity-assertion-authz-grant}}), and the flag signals
  only the added acceptance of continuation assertions and issuance of
  handle-carrying ID-JAGs.

## Resource Authorization Server Metadata

A Resource Authorization Server advertises separately, by listing the grant
profile `urn:ietf:params:oauth:grant-profile:id-jag-continuation` in its
`authorization_grant_profiles_supported`
{{I-D.ietf-oauth-identity-assertion-authz-grant}}, that it recognizes a
continuation-capable ID-JAG and binds the `identity_continuation_handle`
claim to authorization state ({{ras-processing}}). This value is distinct
from the base ID-JAG grant profile, which signals only ordinary ID-JAG
processing and no handle binding.

Because a continuation-capable ID-JAG is an ID-JAG, a Resource Authorization
Server that advertises
`urn:ietf:params:oauth:grant-profile:id-jag-continuation` MUST also advertise
the base `urn:ietf:params:oauth:grant-profile:id-jag` profile and the
`urn:ietf:params:oauth:grant-type:jwt-bearer` grant type on which ID-JAG
depends ({{I-D.ietf-oauth-identity-assertion-authz-grant}}).

A Resource Authorization Server does not list itself as a CAI for its own
hops; it is mapped implicitly and remains subject to the IdP's issuer trust
({{security-trust-model}}). It MAY advertise additional Continuation Assertion
Issuers it authorizes to attest those hops, so the IdP can discover them rather
than be configured out of band:

`identity_continuation_issuers`:
: OPTIONAL. A JSON array of additional CAI issuer identifiers, each a
  `StringOrURI` {{RFC7519}}, that this Resource Authorization Server authorizes
  to attest hops it accepts; an empty array authorizes no additional CAIs.
  Values are compared with
  the assertion `iss` as exact, case-sensitive strings, and duplicates are
  ignored. The advertisement is a nomination only: the IdP MUST establish each
  issuer's identity and signing keys independently, and the advertisement
  alone MUST NOT establish key trust or override the IdP's tenant
  issuer-pairing policy. Because the IdP
  evaluates issuer trust and keys against its current trusted issuer and key
  state, removing an issuer or revoking its keys de-authorizes it for existing
  chains.
  Acceptance remains the IdP's decision.

When a CAI's issuer identifier is that of an OAuth authorization server, the
IdP obtains its signing keys from the `jwks_uri` in that server's metadata
({{RFC8414}}); a CAI without such a `jwks_uri`, like any other CAI, uses
authenticated configuration. A RAS nomination MUST NOT by itself trigger that
retrieval or authorize the issuer; the IdP applies its own issuer policy
first. The IdP MUST refresh remotely obtained keys under a bounded cache
policy, so a key removed from the JWK Set stops validating once the refresh
takes effect.

# Implementation Considerations {#implementation}

This section is non-normative. It describes ways an IdP can realize this
document's requirements; conformance depends only on the normative sections.

## Deployment Topologies {#deployment-topologies}

| Topology | CAI role held by | CAI handle source | Fits when |
|---|---|---|---|
| Co-located | the accepting RAS | read from RAS state | one operator runs the domain |
| Separate | a separately mapped CAI | intra-domain carrier | the RAS is shared infrastructure, the gateway is only an RS, or keys and audit need isolation |

Co-located is the default: it adds no configured CAI mapping and no self-entry
in `identity_continuation_issuers`. The RAS still advertises the continuation
profile ({{metadata}}), and the IdP still holds the RAS's issuer, key, tenant,
and issuer-pairing trust ({{security-trust-model}}). Both topologies produce
the same Identity Continuation Assertion and apply the same CAI requirements;
they differ only in how the CAI reaches the accepted hop's state. Either CAI
can issue the assertion from its token endpoint ({{assertion-token-exchange}}):
a co-located RAS takes the access token it issued as the `subject_token`, and a
separate CAI takes the token that carries the handle.

With a separate CAI, a deployment can propagate the handle in several ways:

* A Transaction Token {{I-D.ietf-oauth-transaction-tokens}} can carry the
  handle as request context ({{rationale-txn}}).
* A signed JWT access token {{RFC9068}} issued by the accepting RAS can carry
  the handle as a claim. The claim records issuance-time state and therefore
  cannot reflect a later revocation.
* For an opaque access token issued by the accepting RAS, its introspection
  response {{RFC7662}} can carry the handle as a member, generated when
  introspection occurs and present only when `active` is `true`.

The replay reservation ({{validation-replay}}) is typically held in strongly
consistent state: only one concurrent request reaches ISSUED, a concurrent
request under a matching fingerprint waits for or retries that result, and the
IdP retains and expires the reservation by the same clock it uses to evaluate
`exp`. A RAS can make the handle binding and token issuance of
{{ras-processing}} one outcome with a local transaction, or with a
compensating action that revokes a token whose binding did not commit.

Because the actor-chain depth bound counts merged lineage entries, an actor
that repeatedly continues as itself never trips it; the fan-out, rate, and
hop-count limits of {{root-establishment}} bound such retry-driven growth
instead, and the IdP prunes expired or revoked hop state.

The CAI accounts for retries separately from fan-out and keeps audit records of
its issuance and limit enforcement. The IdP performs end-to-end audit
correlation across a chain, while each RAS logs only its local subject.

An IdP can derive handles from an internal delegation identifier using a
keyed one-way function, provided the derived handles still satisfy rules 1, 2,
and 5 of {{chain-id}} and remain unlinkable.

An IdP can defer materializing chain state until the first continuation,
provided the handle still resolves to the same root and envelope; deferral
does not relax the reservation durability of {{validation-replay}}.

# Security Considerations {#security}

This profile assumes TLS, a correct IdP subject map and root-chain envelope,
and the OAuth guidance of {{RFC9700}}. It principally addresses these
adversaries:

* an on-path attacker replaying an assertion ({{security-replay}});
* a compromised intermediate workload broadening authority or continuing the
  wrong user's chain ({{security-envelope}});
* a compromised CAI or actor-token issuer
  ({{security-trust-model}}, {{security-actor-issuers}});
* a party influencing the client-to-actor mapping, which on a root exchange
  carrying no `actor_token` is the sole authenticator of the root actor
  ({{client-identity}});
* a malicious Resource Server or audience attempting cross-domain correlation
  ({{privacy}}); and
* a faulty intra-domain carrier or RAS state lookup ({{security-envelope}},
  {{security-trust-model}}).

## Sender Constraint and Proof of Possession {#security-pop}

A continuation assertion names the actor the IdP will treat as the chain's
current holder. As a bearer token it would let any party that captured it, in
transit, from a log, or from a compromised intermediary, continue the chain as
that actor. The assertion MUST NOT be accepted as a bearer token {{RFC7800}};
every exchange requires live proof of possession of the `cnf` key via a DPoP
proof {{RFC9449}} ({{client-identity}}). A captured assertion is therefore
useless without the private key, and because the onward ID-JAG is bound to the
same key ({{client-identity}}), possession is demonstrated continuously across
the chain, not once at issuance.

## Durable Task Authorization {#task-provenance}

Scheduled continuation MUST derive from durable RAS task authorization, not a
scheduler-held handle, which would become a durable bearer-like credential
outside the per-call key proof and RAS binding that gate every other use. The
scheduler holds only a task identifier; each authenticated run re-derives the
handle from active task state and still requires an assertion from a mapped
CAI.

## Short Lifetime and Replay {#security-replay}

The 300-second ceiling and the single-use (`iss`, `jti`) reservation
({{validation-replay}}) confine replay to the IdP continuation exchange. The
request fingerprint bound by that reservation ties each assertion to the one
request it first authorized; without it, a resubmitted assertion could
authorize a second, different request within its window.

## Root Authentication Context {#security-assurance}

Downstream resources may gate access on authentication strength (`acr`) or
methods (`amr`); if continuation could raise those claims, an actor could reach
a step-up-gated resource the user never authenticated strongly enough for.
Authentication context therefore comes only from the root envelope, copied
unchanged into onward ID-JAGs ({{onward-id-jag}}).

## Envelope Enforcement and Offline Attenuation {#security-envelope}

The envelope bounds every target and authority. The CAI validates
any offline attenuation segment; the IdP still enforces only the envelope.
Because the assertion is target-agnostic, a permitted actor may select any
target within that ceiling.

Wrong-handle association can continue the wrong user's bounded chain. The
RAS-bound state establishes the authoritative association between the request
and the handle, whether the CAI reads it directly or through a carrier derived
from that state ({{handle-propagation}}). A handle a workload supplies is not
authoritative, and the CAI rejects substitution.

Keeping CAI issuance in-domain ({{assertion-issuance}})
prevents a handle-holding party from bypassing the RAS-acceptance path.

## Trust in Actor Token Issuers {#security-actor-issuers}

The `actor_token` authenticates the current actor to the IdP, so a rogue or
over-scoped actor-token issuer is an impersonation vector: a party controlling
one issuer could mint a token naming an actor in another domain or tenant and
continue that actor's chains. The IdP MUST accept actor tokens only from
issuers trusted for the actor's own domain and tenant, and MUST reject an
untrusted or out-of-scope issuer even when a valid CAI assertion accompanies
it. CAI attestation of the hop and actor-token authentication of the actor are
independent checks ({{security-trust-model}}); neither substitutes for the
other.

## Conjunctive Trust and Issuer Pairing {#security-trust-model}

A continuation requires all of these, and no one of them suffices alone:

* the CAI mapped to the presented hop's accepting Resource
  Authorization Server, which attests the chain-to-actor transition
  ({{validation}}, rule 3);
* the workload identity issuer trusted for the current actor's trust domain,
  which authenticates the actor through the `actor_token` ({{validation}},
  rule 5);
* live proof of possession of the confirmed key ({{validation}}, rule 5); and
* the IdP's own root-chain envelope and current-actor policy
  ({{validation}}, rule 7).

The IdP MUST authorize CAI and actor-token issuer pairings per tenant; separate
trust in each is insufficient. Tenant determination MUST derive from
authenticated material, not requester-supplied input. The IdP MUST scope CAI
trust by issuer, keys, tenant, and mapped RAS. The IdP MAY learn additional
mappings from the RAS's advertised `identity_continuation_issuers`
({{metadata}}). That advertisement is a nomination only: it scopes each RAS to
naming issuers for its own hops but does not establish issuer or key trust. The
IdP independently authenticates each CAI issuer and its signing keys
({{metadata}}), and tenant policy authorizes the resulting RAS, CAI, and
actor-token-issuer combination. Absent the advertisement, additional mappings
are configured out of band.

## Topology and Trust {#security-topology}

The accepting RAS's implicit CAI mapping establishes no issuer, key, tenant, or
issuer-pairing trust. The IdP sees one signed attestation in either topology,
so separating the CAI isolates keys and components but does not create a
protocol-level quorum. A workload that obtains the assertion by exchanging the
access token or Transaction Token it holds for the call
({{assertion-token-exchange}}) presents nothing
it did not already hold; what it gains is the CAI's attestation, gated by
policy and the live recheck. A compromised RAS can fabricate acceptance state in
either topology, since a separate CAI reads that state as authoritative; a
compromised separate CAI can additionally attest a hop the RAS refused. In
both topologies, RAS-local authorization revocation after issuance, which the
IdP cannot observe, leaves the assertion valid for its remaining lifetime;
a separate CAI adds any delay in RAS state reaching it.
The root envelope still bounds the result.

## Actor Chain Integrity {#security-actor-chain}

The `act` lineage records who has acted in the delegation. A compromised actor
could try to forge it, to hide its own identity, impersonate a more privileged
prior actor, or fabricate a delegation that never happened. This profile
denies that by construction: an assertion names only the current actor, and
the IdP builds the onward lineage itself by walking the hop's immutable parent
references ({{onward-id-jag}}), never by copying a chain the assertion
supplies. The IdP MUST reject any mismatch between the current actor and the
assertion's `act`.
Because lineage derives from IdP-held state rather than assertion input, a
party cannot rewrite history it does not control; offline-attenuation
segments, which the IdP does not observe, do not enter lineage.

## Token, Type, and Algorithm Confusion {#security-alg}

An attacker may try to pass one token type off as another, downgrade the
signature algorithm, or steer verification to a key it controls. The IdP MUST
verify `typ`, reject `alg=none` and symmetric algorithms, and allowlist
asymmetric algorithms. It MUST select keys from trusted issuer configuration;
`kid` MAY select among them. It MUST NOT trust assertion `jku`, `x5u`, embedded
`jwk`, or other supplied key material.

## Metadata Disclosure {#security-metadata}

Advertising `identity_continuation_issuers` ({{metadata}}) in publicly readable
authorization server metadata reveals which additional CAIs a Resource
Authorization Server authorizes to attest its hops, and can thereby disclose
federation topology, tenant relationships, and deployment structure, the same
disclosure concern the base ID-JAG profile raises for issuer-specific
metadata. A deployment whose CAI
relationships are sensitive SHOULD omit the advertisement and convey the
nomination out of band or through
access-controlled discovery.

# Privacy Considerations {#privacy}

A hop's `identity_continuation_handle` is visible to its ID-JAG client, the
accepting Resource Authorization Server, the CAI, the workload that receives
the assertion, and the IdP. Where a deployment propagates it, the carrier and
its recipients, including any audience of an access token that carries it,
see it as well ({{handle-propagation}}).

Handles are opaque, high-entropy, and hop-specific ({{chain-id}}), so they do
not provide a common cross-RAS identifier for a user.

The chain is not unlinkable: the IdP correlates it, participants sharing a
handle can correlate that hop, and actor lineage and timing may correlate
transactions across audiences. For example, an observer comparing ID-JAGs
issued to two audiences within one short window and carrying the same
actor-chain shape may infer they belong to one user's transaction, even without
a shared handle.

The onward ID-JAG's `act` chain also names the prior actors to the accepting
RAS outright, with no correlation needed; {{onward-id-jag}} lets policy limit
the disclosed depth, and deployments may limit the lineage exposed to each
audience.

# IANA Considerations {#iana}

## OAuth Extensions Error Registration

IANA is requested to register the following error in the "OAuth Extensions
Error Registry" established by {{RFC6749}}.

Error Name:
: invalid_continuation

Error Usage Location:
: token endpoint response

Related Protocol Extension:
: Identity Continuation Assertion for OAuth 2.0 Token Exchange

Change Controller:
: IETF

Specification Document(s):
: This document, {{error-response}}

## OAuth Parameters Registration

IANA is requested to update the registration of the `audience` parameter in
the "OAuth Parameters" registry established by {{RFC6749}}. {{RFC8693}}
registered the parameter for the token request; this document adds the token
response usage location and leaves the change controller unchanged, so that
the entry reads as follows.

Parameter name:
: audience

Parameter usage location:
: token request, token response

Change controller:
: IESG

Specification Document(s):
: Section 2.1 of {{RFC8693}}; this document, {{assertion-response}}

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

IANA is also requested to register the following grant-profile value in the
same registry.

URN:
: urn:ietf:params:oauth:grant-profile:id-jag-continuation

Common Name:
: Grant profile identifier for a continuation-capable ID-JAG, whose accepting
  Resource Authorization Server binds the `identity_continuation_handle` claim
  to authorization state

Change Controller:
: IETF

Specification Document:
: This document, {{metadata}}, {{ras-processing}}

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
: binary; the `+jwt` structured syntax suffix {{RFC8417}} registers this
  encoding. An Identity Continuation Assertion is a JWT {{RFC7519}}, a series of
  base64url-encoded values (some of which may be empty) separated by period
  ('.') characters.

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
  continuation chain, used to correlate a continuation to its chain
  and parent hop and to resolve the per-audience subject. This claim
  appears in an Identity Continuation Assertion and in a continuation-capable
  ID-JAG, and its value may also travel in intra-domain chain context,
  including an access token the accepting Resource Authorization Server
  issues ({{handle-propagation}}).

Change Controller:
: IETF

Specification Document(s):
: This document, {{chain-id}}

## OAuth Token Introspection Response Registration

IANA is requested to register the following value in the "OAuth Token
Introspection Response" registry established by {{RFC7662}}.

Name:
: identity_continuation_handle

Description:
: The continuation handle bound to the introspected token's authorization,
  present only when `active` is `true` and returned only to resources within
  the issuing server's trust domain ({{handle-propagation}})

Change Controller:
: IESG

Specification Document(s):
: This document, {{handle-propagation}}

## OAuth Authorization Server Metadata Registration

IANA is requested to register the following values in the "OAuth Authorization
Server Metadata" registry established by {{RFC8414}}.

Metadata Name:
: identity_continuation_supported

Metadata Description:
: Boolean value indicating support for the Identity Continuation Assertion
  profile

Change Controller:
: IETF

Specification Document(s):
: This document, {{metadata}}

Metadata Name:
: identity_continuation_issuers

Metadata Description:
: Array of issuer identifiers for additional Continuation Assertion Issuers
  (CAIs) a Resource Authorization Server authorizes to attest hops it accepts
  (a nomination; the IdP establishes issuer key trust independently)

Change Controller:
: IETF

Specification Document(s):
: This document, {{metadata}}

Note: The token type URI `urn:ietf:params:oauth:token-type:id-jag` referenced by
this document is registered by
{{I-D.ietf-oauth-identity-assertion-authz-grant}} and is not registered here.

Note: The `authorization_grant_profiles_supported` metadata parameter and the
base `urn:ietf:params:oauth:grant-profile:id-jag` value referenced by this
document are defined and registered by
{{I-D.ietf-oauth-identity-assertion-authz-grant}} and are not registered here;
this document registers only the
`urn:ietf:params:oauth:grant-profile:id-jag-continuation` value.

--- back

# Design Rationale {#rationale}

This non-normative appendix records the principal design choices.

## Relationship to ID-JAG {#rationale-idjag}

The assertion is the Token Exchange input: its audience is the IdP and it has
no top-level `sub`. The resulting ID-JAG is the target Resource Authorization
Server's grant and contains the IdP-resolved subject and, when applicable, a
continuation handle. The artifacts therefore have different issuers,
audiences, subjects, and consumers.

## Why Not a Transaction Token {#rationale-txn}

A Transaction Token {{I-D.ietf-oauth-transaction-tokens}} carries request
context within one trust domain. The assertion crosses from that domain to
the IdP, is single-use, and carries neither the target subject nor general
request context. It may be derived from Transaction Token context, but is not
a Transaction Token profile.

## Why Not a Cross-Domain Propagation Token {#rationale-propagation}

The choice follows {{decision-rule}}: a pairwise-subject boundary can be
crossed only by the IdP, which the target trusts to name the user, and IdP
exchange permits current-state and envelope checks at every hop. Direct
propagation instead fits deployments with a global subject,
shared issuer trust, and no need for mid-chain IdP revocation, such as a
single SPIFFE-style trust domain (one workload-identity namespace with no
pairwise-subject boundary to cross). Delegated Authorization
{{I-D.li-oauth-delegated-authorization}}, whose client-issued tokens carry no
subject, composes with this profile as the intra-domain layer and stops where
re-issuance to a new subject begins.

## Alternative Topology: Resolution at the Target {#rationale-pull}

A pull design would have each target resolve a reference at the IdP over a
back channel {{RFC7662}}. It requires a new target-side grant and per-request
back channel. The selected push design reuses the ID-JAG grant path, adding
only handle binding at continuation-source RASes; the CAI supplies
acceptance evidence. Pull remains a possible companion profile.

## Why a Signed Assertion Rather Than a Bare Grant Type {#rationale-grant-type}

The signed assertion lets the CAI attest the authenticated actor,
key, accepted hop, and any intra-domain policy checks that the IdP cannot
observe. It also keeps the IdP's trust coarse: the IdP trusts a CAI mapped to
the accepting Resource Authorization Server (the audience the ID-JAG names),
scoped per tenant. That CAI abstracts the actors, keys, and continuation
policy within its trust domain, so the IdP authorizes mapped CAIs rather than
the individual actors behind them. The assertion does not authorize
target or scope. Where that domain-local attestation is unnecessary, a
recipient-bound direct grant remains a possible simplification, but it forces
the IdP to authenticate every actor and hold per-actor policy directly, which
does not scale across domains.

## Why Asymmetric Signing Only {#rationale-alg}

This profile requires asymmetric signing and forbids encryption and nested
signing ({{names}}), tighter than RFC 8725 {{RFC8725}}, which also permits
verified symmetric algorithms. The restriction is deliberate: asymmetric
verification avoids distributing a shared secret across domains and the
key-confusion risk a symmetric key between the CAI and IdP would create
({{security-alg}}); TLS on every hop and the assertion's minimal contents make
encryption unnecessary; and a single compact signed form removes an
interoperability choice between issuers and verifiers.

# Examples {#examples}

This non-normative appendix illustrates three deployment shapes: a tool
gateway that selects its upstream at request time ({{example-gateway}}),
interactive application chaining across three domains ({{example}}), and an
unattended background agent ({{example-background}}). The gateway example
comes first because it is the simplest deployment: one domain runs RAS and
CAI together, and the handle travels in the access token.

Message sequences are vertical lifelines with time flowing downward. The
payload and state blocks below them are tagged "On the wire" when they cross a
trust boundary,
"Intra-domain context" when they travel only within one trust domain, and
"Server-side state" when they are never transmitted. Continuation handles
are written H0, H1, and so on, one per hop. Each example identifies its
deployment topology before describing the flow. JWTs are shown as decoded
payloads; JOSE headers and signatures are omitted. Except where shown, client
authentication is omitted. Proof of possession uses DPoP.

## Gateway Example (Co-located RAS and CAI) {#example-gateway}

An agent runtime, AgentApp, calls a tool gateway on behalf of Alice. The
gateway decides at request time which upstream API a tool call needs, here a
wiki, so AgentApp knows the gateway's audience but not the eventual upstream,
and the gateway holds no assertion of Alice's identity addressed to it. This
example shows the gateway obtaining an audience-specific ID-JAG for the wiki
without ever seeing Alice's credential. It is the simplest deployment of this
profile and the one a gateway vendor implements.

Topology: co-located. GatewayRAS is the Resource Authorization Server (RAS)
that accepts the ID-JAG and also the Continuation Assertion Issuer (CAI) for
the hops it accepts, so it needs no separate CAI mapping. It carries the
handle in the access tokens it issues.

All parties trust one enterprise IdP at `https://idp.example/`, tenant
`tenant-123`. Alice has pairwise subjects at GatewayRAS and WikiRAS, which
only the IdP can map.

* Agent domain (`agent.example`): client `agent-app`, the confidential runtime
  that holds Alice's session and roots the chain.
* Gateway domain (`gateway.example`): GatewayRAS, the gateway's authorization
  server at `https://ras.gateway.example/`, and the workload `tool-gateway`,
  the Resource Server at `https://gateway.example/` that AgentApp calls.
* Wiki domain (`wiki.example`): WikiRAS at `https://ras.wiki.example/`, an
  ordinary ID-JAG authorization server in front of WikiAPI at
  `https://api.wiki.example/`. It is terminal in this chain.

H0 is the root hop, bound at GatewayRAS; H1 is the wiki hop, which nobody
continues.

~~~
AgentApp        IdP       GatewayRAS/CAI     ToolGateway     WikiRAS/API
    |            |               |                |               |
    |-ID Token-->|               |                |               |
    |<-ID-JAG H0-|               |                |               |
    |-jwt-bearer: ID-JAG + DPoP->| bind H0        |               |
    |<--access token, H0 claim---|                |               |
    |----------tool call with AT + DPoP---------->|               |
    |            |               |<--exchange AT--|               |
    |            |               |-assertion H0-->|               |
    |            |<-assertion, actor token, DPoP--|               |
    |            |-----------ID-JAG H1----------->|               |
    |            |               |                |-ID-JAG, DPoP->|
    |            |               |                |   (no binding)|
    |            |               |                |<---wiki AT----|
    |            |               |                |-call WikiAPI->|
    |<-------------------result-------------------|               |
~~~

### Root Exchange {#example-gateway-root}

AgentApp exchanges Alice's ID Token, whose `sid` anchors the chain to her IdP
session ({{root-establishment}}), for an ID-JAG addressed to GatewayRAS, the
one audience it knows:

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof signed by the agent-app key>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:id-jag
&audience=https://ras.gateway.example/
&resource=https://gateway.example/
&scope=tools.invoke
&subject_token=<id_token>
&subject_token_type=urn:ietf:params:oauth:token-type:id_token
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<agent-app JWT>
~~~

Server-side consent and tenant policy make the governing authorization
continuation-capable, so the IdP establishes a chain and embeds H0
({{root-establishment}}). Because the upstreams are unknown at root time, the
root-chain envelope records an authorization basis, Alice's standing consent
and tenant policy, rather than enumerated targets, and enterprise policy
permits `tool-gateway` to continue it.

On the wire (decoded ID-JAG for GatewayRAS):

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.gateway.example/",
  "sub": "gateway-pairwise-subject",

  "client_id": "agent-app",
  "resource": "https://gateway.example/",
  "scope": "tools.invoke",

  "auth_time": 1710000200,
  "acr": "urn:example:loa:2",
  "amr": ["pwd", "mfa"],

  "identity_continuation_handle": "Qm7zXu2VtL9pKe4RaW1nHc",

  "cnf": {
    "jkt": "base64url-agent-app-key-thumbprint"
  },

  "iat": 1710000205,
  "exp": 1710000505,
  "jti": "idjag-gateway-01"
}
~~~

### GatewayRAS Binds H0 and Issues the Access Token {#example-gateway-bind}

AgentApp redeems the ID-JAG at GatewayRAS with the jwt-bearer grant, exactly
as for any ID-JAG, proving the same key the ID-JAG is bound to:

~~~
POST /token HTTP/1.1
Host: ras.gateway.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof signed by the agent-app key>

grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
&assertion=<the ID-JAG above>
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<agent-app JWT>
~~~

GatewayRAS advertises the continuation grant profile ({{metadata}}) and so
recognizes the handle. It binds H0, the issuing IdP, the tenant it associates
with that IdP, and the confirmed key to the authorization state behind the
access token, in the same outcome that issues the token ({{ras-processing}}).

Server-side state at GatewayRAS:

~~~ json
{
  "identity_continuation_handle": "Qm7zXu2VtL9pKe4RaW1nHc",
  "status": "ACCEPTED",
  "authorization_state": "gw-authz-7a1e",
  "idp": "https://idp.example/",
  "tenant": "tenant-123",
  "client_id": "agent-app",
  "cnf_jkt": "base64url-agent-app-key-thumbprint",
  "bound_at": 1710000210
}
~~~

GatewayRAS issues the access token as a signed JWT that carries H0 as a claim,
the access-token carrier of {{deployment-topologies}}.

On the wire (decoded access token):

~~~ json
{
  "iss": "https://ras.gateway.example/",
  "aud": "https://gateway.example/",
  "sub": "gateway-pairwise-subject",
  "client_id": "agent-app",
  "scope": "tools.invoke",
  "identity_continuation_handle": "Qm7zXu2VtL9pKe4RaW1nHc",
  "cnf": {
    "jkt": "base64url-agent-app-key-thumbprint"
  },
  "iat": 1710000210,
  "exp": 1710000810,
  "jti": "at-gw-0001"
}
~~~

AgentApp calls the gateway with this token and a DPoP proof. It chooses which
token to present, and with it which authorization, but cannot alter the handle
inside the token or pair it with another token's key.

### ToolGateway Obtains the Assertion {#example-gateway-ica}

ToolGateway validates the access token and its DPoP binding as any resource
server would. Resolving the tool call, it selects the wiki as the upstream. To
continue Alice's chain there it needs an Identity Continuation Assertion,
which it obtains by exchanging the access token it just received at
GatewayRAS's token endpoint, authenticating as the OAuth client `tool-gateway`
and proving its own key ({{assertion-token-exchange}}):

~~~
POST /token HTTP/1.1
Host: ras.gateway.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof signed by the tool-gateway key>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:identity-continuation
&subject_token=<the access token above>
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<tool-gateway JWT>
~~~

The access token is bound to the `agent-app` key; the DPoP proof on this
request proves the `tool-gateway` key, which GatewayRAS places in the
assertion's `cnf`, and is not matched against the token's own `cnf`. No
`audience`, `resource`, or `scope` is sent: the upstream is chosen at the IdP
exchange, not here ({{assertion-token-exchange}}).

GatewayRAS confirms the token is its own, unexpired, and addressed to
`https://gateway.example/`, the resource `tool-gateway` is registered to
operate; reads H0 from it and rechecks that the binding is still active;
confirms that policy permits `tool-gateway` to continue; and issues the
assertion bound to the proven key.

On the wire (issuance response):

~~~ json
{
  "issued_token_type": "urn:ietf:params:oauth:token-type:identity-continuation",
  "access_token": "<the assertion below, compact JWS>",
  "token_type": "N_A",
  "audience": "https://idp.example/",
  "expires_in": 120
}
~~~

The `audience` tells ToolGateway which IdP the assertion is for; it is where
the request for the next ID-JAG goes.

On the wire (decoded assertion):

~~~ json
{
  "iss": "https://ras.gateway.example/",
  "aud": "https://idp.example/",
  "identity_continuation_handle": "Qm7zXu2VtL9pKe4RaW1nHc",

  "act": {
    "iss": "https://gateway.example/",
    "sub": "tool-gateway"
  },

  "cnf": {
    "jkt": "base64url-tool-gateway-key-thumbprint"
  },

  "iat": 1710000230,
  "exp": 1710000350,
  "jti": "ica-gw-0001"
}
~~~

Neither AgentApp nor ToolGateway supplies H0 as input to issuance, so neither
can substitute another handle ({{handle-propagation}}).

### ToolGateway Continues to WikiRAS {#example-gateway-continue}

ToolGateway resolves the token endpoint of `https://idp.example/`, the
`audience` it was given, from that IdP's metadata and presents the assertion
there as the `subject_token` of a
continuation exchange, with its actor credential and a DPoP proof of the same
key, requesting an ID-JAG for WikiRAS. `tool-gateway` is a registered client
of the IdP; its sender-constrained credential serves as both client assertion
and `actor_token` under the dual-use rule of {{client-identity}}:

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof signed by the tool-gateway key>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:id-jag
&audience=https://ras.wiki.example/
&resource=https://api.wiki.example/
&scope=wiki.read
&subject_token=<identity-continuation-assertion>
&subject_token_type=urn:ietf:params:oauth:token-type:identity-continuation
&actor_token=<sender-constrained tool-gateway credential>
&actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

The IdP validates the exchange ({{validation}}). The assertion is signed by
GatewayRAS, the RAS recorded for H0's hop, so its issuer is trusted without a
separate CAI mapping. H0 names an accepted hop on an active chain. The `act`
claim names `tool-gateway`, the authenticated client, and the DPoP proof
matches `cnf`. The wiki target falls within the recorded basis, Alice's
standing consent and tenant policy, which is how a target nobody enumerated
at root time is admitted; `wiki.read` is evaluated against that basis, not
against the root `tools.invoke` scope. The IdP resolves Alice's wiki subject
and issues the ID-JAG with H1 and `tool-gateway` atop `agent-app` in the
lineage.

On the wire (decoded ID-JAG for WikiRAS):

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.wiki.example/",
  "sub": "wiki-pairwise-subject",

  "client_id": "tool-gateway",
  "resource": "https://api.wiki.example/",
  "scope": "wiki.read",

  "auth_time": 1710000200,
  "acr": "urn:example:loa:2",
  "amr": ["pwd", "mfa"],

  "identity_continuation_handle": "Zp3kRt8VbN2wLq6YsD4mXe",

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

  "iat": 1710000235,
  "exp": 1710000535,
  "jti": "idjag-wiki-01"
}
~~~

A target outside the basis fails with `invalid_target`; the chain stays
continuable and only that tool call fails ({{error-response}}).
{{example-dynamic}} shows one.

### WikiRAS Redeems an Ordinary ID-JAG {#example-gateway-terminal}

ToolGateway redeems the ID-JAG at WikiRAS with the jwt-bearer grant and a DPoP
proof of the same key. WikiRAS implements nothing from this profile: it
validates the ID-JAG as the base profile requires, ignores the handle, and
issues an access token without binding H1 ({{ras-processing}}). ToolGateway
calls WikiAPI as Alice's wiki subject and returns the result to AgentApp. Each
further tool call repeats the exchange of {{example-gateway-ica}} and creates
a sibling hop under H0 (H2, and so on).

### What a Gateway Implements {#example-gateway-checklist}

The gateway domain adds two things to an ordinary OAuth deployment:

* GatewayRAS binds the handle when it redeems a continuation-capable ID-JAG,
  places it in the access token, and issues assertions from its token endpoint
  by Token Exchange.
* `tool-gateway` exchanges the access token it received for an assertion, then
  presents that assertion, its actor credential, and a DPoP proof to the IdP
  for the next ID-JAG.

Outside the domain nothing changes. AgentApp performs a base ID-JAG exchange
and never shares Alice's credential, and WikiRAS runs the base profile
unmodified.

## Three-Hop Interactive Example {#example}

This section walks a three-domain flow with separate CAIs and a Transaction
Token carrier end-to-end for a single user: ExpenseApp invokes ExpenseSaaS;
ExpenseService, the workload handling
that request, calls TravelAPI to reach TravelSaaS; and TravelService, the
TravelSaaS workload that handles that call, in turn calls BookingAPI to
complete the itinerary. All parties trust one enterprise IdP at
`https://idp.example/`.

Topology: separate CAI with a Transaction Token carrier.

The authorization path, the sequence of accepting RASes, differs from that
API-call path:

~~~
ExpenseApp -> ExpenseRAS -> TravelRAS -> BookingRAS
~~~

Across a trust boundary the handle is accepted only inside an ID-JAG or
Identity Continuation Assertion; within a domain it travels only as derived
chain context.

Participants are grouped by trust domain:

* Expense domain (`expenses.example`): client `expense-app`, workload
  `expense-service`, and ExpenseRAS / Expense TTS / Expense CAI, in front of
  ExpenseAPI.
* Travel domain (`travel.example`): workload `travel-service`, and
  TravelRAS / Travel TTS / Travel CAI, in front of TravelAPI.
* Booking domain (`booking.example`): BookingRAS and BookingAPI only. It is
  terminal in this chain, an ordinary ID-JAG Resource Authorization Server
  that needs no continuation support ({{ras-processing}}).
* Outside the trust circle: PartnerSaaS (`partner.example`), reached in
  {{example-federation-edge}}.

The user has a pairwise subject at each RAS, which only the IdP can map.
Handles are H0 at ExpenseRAS, H1 at TravelRAS, and H2 in the ID-JAG addressed
to terminal BookingRAS, which never binds it.

The root hop establishes the chain and the Expense domain's accepted
authorization:

~~~
 ExpenseApp        IdP          ExpenseRAS       ExpenseAPI/TTS
     |               |               |                 |
     |--ID Token---->|               |                 |
     |<-ID-JAG(H0)---|               |                 |
     |------------------ID-JAG------>|                 |
     |<-------------------AT1--------| bind H0         |
     |------------------request + AT1 + DPoP---------->|
     |               |               |<-resolve AT1----|
     |               |               |--bound H0------>|
     |               |               |    derive H0 into tctx
~~~

Each continuation repeats one exchange. ExpenseService obtains the Travel
grant before crossing the boundary:

~~~
 ExpenseService  Expense CAI        IdP        TravelRAS TravelAPI/TTS
       |              |             |             |             |
       |-exchange TT->|             |             |             |
       |<-assertion---|             |             |             |
       |--------------------------->|             |             |
       |     assertion + DPoP       |             |             |
       |<---------------------------| ID-JAG(H1)  |             |
       |----------------------------------------->|             |
       |                 ID-JAG                   |             |
       |<-----------------------------------------| AT2; bind H1|
       |-----------------request + AT2 + DPoP------------------>|
       |              |             |             |<-resolve AT2|
       |              |             |             |--bound H1-->|
       |              |             |             | derive into TT
~~~

{{example-third-hop}} repeats the pattern from TravelSaaS to terminal
BookingRAS.

### Root ID-JAG for ExpenseRAS {#example-first-hop}

ExpenseApp holds an ID Token for the authenticated user and exchanges it at the
IdP for an ID-JAG scoped to ExpenseRAS. The request is DPoP-bound to
ExpenseApp's key.

On the wire (request):

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof signed by the expense-app key>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:id-jag
&audience=https://ras.expenses.example/
&resource=https://api.expenses.example/
&scope=expenses.read
&subject_token=<id_token>
&subject_token_type=urn:ietf:params:oauth:token-type:id_token
&actor_token=<sender-constrained expense-app credential>
&actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

The IdP resolves the ID Token's `sid` to the anchoring session and verifies
ExpenseApp's actor credential and DPoP key. Existing consent and enterprise
policy permit continuation to Expense, Travel, and Booking by the designated
workloads, so the IdP records this root-chain envelope:

Server-side state:

~~~
(https://ras.expenses.example/, https://api.expenses.example/)
    permitted scopes: expenses.read

(https://ras.travel.example/, https://api.travel.example/)
    permitted scopes: trips.read

(https://ras.booking.example/, https://api.booking.example/)
    permitted scopes: stays.book
~~~

The envelope also records the governing authorization, permitted continuers,
and expiry. A deployment with unknown onward targets records an
authorization-basis ceiling instead and evaluates each target at continuation
time ({{validation}}, rule 7).

The IdP creates a fresh root hop, H0, for this chain and embeds it as a
claim of the ID-JAG it is about to issue ({{chain-id}}, rule 1); the hop is
PENDING until a Resource Authorization Server accepts it ({{hop-activation}}).
The Token Exchange response carries the ID-JAG and no continuation-specific
response member; H0 travels inside the ID-JAG.

The decoded ID-JAG for ExpenseRAS carries the user's ExpenseRAS-local
subject and the root hop's handle.

On the wire (decoded ID-JAG):

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.expenses.example/",
  "sub": "expense-pairwise-subject",

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

### ExpenseRAS Acceptance and the Expense-Domain Chain Context {#example-context}

ExpenseApp exchanges this ID-JAG at ExpenseRAS for an access token (AT1),
exactly as for any ID-JAG {{I-D.ietf-oauth-identity-assertion-authz-grant}}
(not shown), except that ExpenseRAS also recognizes the continuation grant
profile and processes `identity_continuation_handle` ({{ras-processing}}).
ExpenseRAS validates the ID-JAG, authenticates ExpenseApp, verifies the
DPoP proof, and applies its local policy; only if every check and the
access-token issuance itself succeed does it atomically bind H0 to the
authorization state behind AT1, moving the hop from PENDING to ACCEPTED. A
hop that never reaches ACCEPTED, for example one copied from an ID-JAG
that ExpenseRAS rejected, is not usable: no CAI attests a hop
that its Resource Authorization Server never accepted.

ExpenseRAS keeps this association in a private internal record. This
deployment does not put it in AT1; the Expense TTS derives it from this record
for the authenticated request.

Server-side state:

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
Transaction Token Service, resolves AT1 against the record that ExpenseRAS
just created over their shared, own-domain interface ({{ras-processing}}),
derives H0 from it, and issues a local Transaction Token for
ExpenseService, the workload that will complete the request.

Intra-domain context (decoded Transaction Token):

~~~ json
{
  "iss": "https://tts.expenses.example/",
  "aud": "https://expenses.example/",
  "sub": "expense-pairwise-subject",
  "txn": "txn-expense-88f2",
  "scope": "expense-report:complete",
  "req_wl": "expense-api",

  "tctx": {
    "identity_continuation": {
      "iss": "https://idp.example/",
      "tenant": "tenant-123",
      "handle": "kW4uJ8pTe2NxA6rQvD1zYs"
    }
  },

  "iat": 1710000012,
  "exp": 1710000072,
  "jti": "tt-expense-0007"
}
~~~

The Expense TTS derives this context from AT1's authorization record; neither
ExpenseApp nor ExpenseService supplies H0 ({{handle-propagation}}). The
Transaction Token remains inside `expenses.example` and is normally forwarded
unchanged within that domain. The `tctx.identity_continuation` encoding
shown is illustrative and deployment-defined; this document standardizes no
carrier schema.

### Obtaining the Identity Continuation Assertion {#example-ica}

ExpenseService obtains an assertion covering H0 by presenting its Transaction
Token as the `subject_token` at Expense CAI's token endpoint
({{assertion-token-exchange}}); the response's `audience`,
`https://idp.example/`, tells it where to request the next ID-JAG.
Before issuing, Expense CAI authenticates ExpenseService, verifies its key,
confirms that H0 belongs to the transaction that ExpenseService is serving,
and rechecks that ExpenseRAS's authorization remains active. The IdP's per-hop
map designates Expense CAI to attest hops accepted by ExpenseRAS
({{assertion-issuance}}, {{root-establishment}}).

On the wire (decoded assertion):

~~~ json
{
  "iss": "https://cai.expenses.example/",
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
  "jti": "b8Rn5Yx1Qe4Nk2Wf6zVc9d"
}
~~~

### Continuation Exchange for the TravelRAS ID-JAG {#example-chained}

ExpenseService presents the assertion to the IdP as the `subject_token`,
DPoP-bound to its own key.

On the wire (request):

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof signed by the expense-service key>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:id-jag
&audience=https://ras.travel.example/
&resource=https://api.travel.example/
&scope=trips.read
&subject_token=<identity-continuation-assertion>
&subject_token_type=urn:ietf:params:oauth:token-type:identity-continuation
&actor_token=<sender-constrained expense-service credential>
&actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

The IdP runs the checks of {{validation}}: the DPoP key matches both the
assertion's `cnf.jkt` and the actor token's key confirmation; `expense-service`
is the
actor named in `act`; H0 is CONTINUABLE; and the requested TravelRAS,
TravelAPI, and `trips.read` values match the Travel target entry in the
root-chain envelope. The IdP does not call ExpenseRAS to confirm acceptance.
Instead, the assertion from ExpenseSaaS's mapped CAI,
`https://cai.expenses.example/`, is the evidence that H0 reached ACCEPTED state
and is CONTINUABLE ({{hop-activation}}).

The IdP resolves the user's TravelRAS pairwise subject and creates H1 as a
child of H0. The decoded ID-JAG carries H1 and the newly constructed
`act` chain ({{onward-id-jag}}): `expense-service`, authenticated at this
exchange, placed atop the root actor `expense-app`. `travel-service` has
not yet performed an exchange, so it is not part of the lineage.

On the wire (decoded ID-JAG):

~~~ json
{
  "iss": "https://idp.example/",
  "aud": "https://ras.travel.example/",
  "sub": "travel-pairwise-subject",

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

### TravelRAS Acceptance and the Travel-Domain Chain Context {#example-use}

ExpenseService exchanges the TravelRAS ID-JAG at TravelRAS for an access
token (AT2), presenting a fresh DPoP proof with the same expense-service
key. TravelRAS recognizes the continuation grant profile just as
ExpenseRAS did: it validates the ID-JAG, authenticates ExpenseService,
and, on success, atomically binds H1 to the authorization state behind
AT2, exactly as {{example-context}} describes for ExpenseRAS and H0.

ExpenseService calls TravelAPI with AT2. The Travel TTS derives H1 from
that bound state and issues a local Transaction Token for TravelService,
the TravelSaaS workload that receives the request. Its chain-context member
differs from the Expense token only in the hop handle:

Intra-domain context (excerpt):

~~~ json
"tctx": {
  "identity_continuation": {
    "iss": "https://idp.example/",
    "tenant": "tenant-123",
    "handle": "Uc9fB3mHs5LdK7gEnX2wRj"
  }
}
~~~

The token remains inside `travel.example`; H1 replaces H0 because TravelRAS,
not ExpenseRAS, is now the accepted authorization from which continuation
will occur.

### TravelService Continues to BookingRAS {#example-third-hop}

TravelService needs a reservation from BookingSaaS. Processing the request
whose Transaction Token carries H1, it obtains the same assertion shape as
{{example-ica}} from Travel CAI, now naming H1, `travel-service`, and
TravelService's confirmed key.

TravelService exchanges the assertion, DPoP-bound to its own key, for an
ID-JAG with `audience=https://ras.booking.example/`,
`resource=https://api.booking.example/`, and `scope=stays.book`, all
within the envelope's Booking target entry. The IdP creates a fresh hop
H2 whose immutable parent is H1 and constructs the onward `act` chain
({{onward-id-jag}}): `travel-service`, authenticated at this exchange,
placed atop the presented hop's lineage (`expense-service`, then
`expense-app`).

On the wire (selected claims from the decoded ID-JAG):

~~~ json
{
  "aud": "https://ras.booking.example/",
  "sub": "booking-pairwise-subject",
  "client_id": "travel-service",
  "resource": "https://api.booking.example/",
  "scope": "stays.book",
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
  }
}
~~~

TravelService redeems the ID-JAG at BookingRAS for an access token (AT3),
presenting a fresh DPoP proof with the same key. Because Booking is terminal,
BookingRAS follows the ordinary ID-JAG profile: it ignores H2, issues AT3, and
does not bind the hop ({{ras-processing}}). Only ExpenseRAS and TravelRAS,
the Resource Authorization Servers from which continuation occurs, implement
the binding extension. TravelService then calls BookingAPI with AT3.

TravelService itself is the current-domain actor that obtains the next
ID-JAG; it does not pass the handle to a sibling workload
({{handle-propagation}}).

### Reaching a Target Outside the Trust Circle {#example-federation-edge}

Suppose TravelSaaS must also call PartnerSaaS at `https://partner.example/`,
whose Resource Authorization Server does not trust `idp.example`. The chain
cannot continue there: the IdP holds no pairwise subject for that audience
and no authorization basis covers it, so a continuation request for that
target fails ({{validation}}, rules 5 and 7; `invalid_target`). This is
the profile's boundary, not a deployment error: continuation serves the set
of Resource Authorization Servers that trust the common IdP.

A separate identity-chaining profile can cross that boundary under a
bilateral trust agreement. For example, TravelService can present its
Transaction Token to the Travel-domain authorization server under
{{I-D.fletcher-transaction-token-chaining-profile}}, which issues a minimized
grant for PartnerSaaS. The Transaction Token and continuation handle stay in
the Travel domain; neither is sent to PartnerSaaS.

## Background Agent Example (User-Scheduled Continuation) {#example-background}

The user is present when the task is created and absent at every run. Unlike
the interactive example, the root hop is bound to durable, platform-owned
task authorization. The Scheduler stores only an opaque task identifier;
each run derives fresh context from the active authorization
({{task-provenance}}).

Topology: separate CAI with a Transaction Token carrier.

* Platform domain (`platform.example`): workload `briefing-agent`, and
  PlatformRAS (the platform's own TaskRAS) / Platform TTS / Platform CAI,
  in front of TaskAPI (`https://api.platform.example/tasks`);
  the Scheduler is an internal platform component, holding only the task
  identifier, that triggers each run.
* Calendar domain (`calendar.example`): CalendarRAS only, in front of
  CalendarAPI. It is terminal in every run.
* Mail domain (`mail.example`): MailRAS in front of MailAPI, reached only
  in the dynamic-target scenario below ({{example-dynamic}}); likewise
  terminal.

The Scheduler stores only `task-123`. H0 remains bound to the PlatformRAS
task authorization across runs; each run receives a fresh child of H0 for
its terminal target.

### Setup (Alice Present)

Alice authorizes "summarize my calendar every morning." Because the task must
outlive her session, `briefing-agent` uses a refresh token from a
continuation-capable grant as the root exchange's subject token. The chain
is therefore anchored to that grant, not Alice's current session
({{root-establishment}}, {{lifecycle}}). The root ID-JAG targets the
PlatformRAS; the envelope records both that root target and the
Calendar target needed by the task.

Server-side state (root envelope excerpt):

~~~
(https://ras.platform.example/, https://api.platform.example/tasks)
    permitted scopes: task.manage

(https://ras.calendar.example/, https://api.calendar.example/)
    permitted scopes: calendar.read
~~~

The response and RAS-binding pattern match {{example-first-hop}} and
{{example-context}}; the request differs by using a refresh token to obtain a
grant-anchored chain. PlatformRAS binds H0 to the durable task authorization.

PlatformRAS keys the resulting durable task authorization by its assigned
task identifier. The record holds no bearer credential.

Server-side state (PlatformRAS task authorization):

~~~
task_id:              task-123
owner:                alice
actor:                briefing-agent
continuation_handle:  Pz6vTq1NcY4kM8bJf3RxWa  # H0
permitted_purpose:    morning-calendar-brief
schedule:             "0 7 * * *"
governing_grant:      grant-8f2c19a4  # internal reference
expiry:               1719450000  # local, not IdP lifetime
status:               active
~~~

Server-side state (Scheduler):

~~~
task_id: task-123
~~~

The Scheduler never receives, stores, or transmits H0 or any user, chain, or
bearer credential;
`task-123` identifies a row in PlatformRAS's own durable state and means
nothing outside the platform.

### Each Run (Alice Absent)

Each run first authenticates the trigger and derives H0 from active task
state:

~~~
 Scheduler   BriefingAgent      Platform TTS
     |              |                 |
     |---trigger--->|                 | task-123
     |              |-task-123+proof->|
     |              |                 | verify proof + task; derive H0
     |              |<-fresh TT(H0)---|
~~~

BriefingAgent then performs a fresh continuation to terminal CalendarRAS:

~~~
 BriefingAgent    Platform CAI       IdP         CalendarRAS
       |               |             |               |
       |--exchange TT->|             |               |
       |<-assertion----|             |               |
       |---------------------------->|               |
       |      assertion + DPoP       |               |
       |<----------------------------| ID-JAG(child) |
       |-------------------------------------------->|
       |                 ID-JAG                      |
       |<--------------------------------------------| access token
       |               |             |      no binding (terminal)
~~~

The task identifier is not a secret and does not authorize a run. The
Scheduler's trigger authenticates and carries only `task-123`; BriefingAgent
then authenticates to the Platform TTS and proves possession of its key, and
the TTS, after confirming `task-123` is active and BriefingAgent is its
designated actor, derives H0 into fresh intra-domain context
({{handle-propagation}}). Neither the Scheduler nor BriefingAgent
selects H0.

BriefingAgent exchanges that Transaction Token at Platform CAI's token
endpoint ({{assertion-token-exchange}}) and presents the assertion to the IdP
the response's `audience` names. Before issuing, Platform CAI
authenticates `briefing-agent`, verifies its key and transaction, and rechecks
that PlatformRAS's H0 authorization remains
active. The assertion and onward ID-JAG have the shapes shown in
{{example-ica}} and {{example-chained}}.

Each run presents H0 and receives a different child. CalendarRAS is terminal,
so it issues the access token without binding that child. A later run's child
is a sibling, not a descendant, of the earlier run's child
({{chain-id}}, {{ras-processing}}).

Had this run also needed `https://api.mail.example/` behind
`https://ras.mail.example/` ({{example-dynamic}}), `briefing-agent` would
present H0 again for a second assertion and receive a second, independent
child for MailRAS. MailRAS is also terminal and does not bind it. The Mail
and Calendar children share H0 as their parent; neither carries the other's
lineage.

### A Dynamic Target {#example-dynamic}

Suppose the platform later extends the briefing to include unread mail,
which requires `https://api.mail.example/` behind
`https://ras.mail.example/`: a target nobody named when Alice created the
task. Under the target entries recorded in the setup above, a run's
continuation exchange presenting H0 for that audience fails, and the
chain is otherwise unaffected.

On the wire (response):

~~~
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "error": "invalid_target"
}
~~~

For a deployment that expects dynamic targets, the envelope's basis is
Alice's standing consent as recorded when the chain was established (for
example, a productivity read-access grant) and tenant policy, with no
enumerated targets; the IdP evaluates each dynamic target against that
recorded basis at continuation time ({{validation}}, rule 7). A scope granted
only later does not extend this chain. The same exchange succeeds only if
read access to the mail service is within Alice's standing consent and tenant
policy permits `briefing-agent` to reach it. A request for `mail.send`,
outside that consent, fails with `invalid_scope`.

The establishment-time envelope remains the ceiling: later policy may narrow
or revoke it but cannot broaden it. A target-specific failure leaves the
chain continuable for other authorized targets.

### Points Worth Noticing

* Stealing `task-123` reveals no handle and does not authorize a trigger.
* Stealing the internal task record exposes H0, but H0 alone is insufficient:
  continuation still requires the agent key and an assertion from Platform
  CAI while the PlatformRAS authorization remains active.
* The ID-JAG, local task authorization, and IdP-held chain have distinct
  lifetimes ({{lifecycle}}).

This pattern requires a user-present setup event to root the chain. Where
no such event exists (for example, an administratively mandated agent
acting for users who never authorized it), there is no delegation to
continue and this profile does not apply; such deployments need a
differently rooted authorization, such as administrative policy at the
IdP, which is out of scope for this document.

# Open Items for Working Group Discussion {#open-items}

This non-normative appendix lists unresolved design questions.

\[\[ To be removed before publication as an RFC ]]

1. **Signed assertion versus a recipient-bound direct profile.**
   Could the IdP bind a continuation credential to an intended actor, actor
   class, trust domain, or key and accept it with client authentication,
   sender-constrained `actor_token`, and live key proof? Are the CAI's actor/key
   attestation and domain-local gate worth the added trust configuration
   ({{rationale-grant-type}})?

2. **Mutual-TLS binding.** Should this profile and ID-JAG add mutual-TLS
   binding together ({{client-identity}})?

3. **A client establishment parameter.** Should a client be able to require
   or suppress chain establishment, or negotiate lifetime, depth, or
   permitted continuers ({{root-establishment}})?

4. **Authorization-basis representation.** Should the envelope expose a
   testable representation of the authorization ceiling, for example:

   ~~~ json
   { "targets": [ { "audience": "https://ras.travel.example/",
       "resource": "https://api.travel.example/",
       "scope": ["trips.read"] } ] }
   ~~~

   Dynamic ceilings might instead use an authorization detail {{RFC9396}},
   policy-bound intent, or immutable policy artifact. Should continuation
   permission have a dedicated consent scope even though establishment can
   occur without a client-requested scope?

Further questions are tracked in the project's issue list rather than expanded
here: nested own-domain `act` segments and offline-actor audit
({{I-D.mcguinness-oauth-actor-receipts}},
{{I-D.mcguinness-oauth-actor-proofs}}); a pull topology with target-side
resolution ({{rationale-pull}}); IdP discovery metadata for accepted
actor-token types, issuers, and proof methods ({{metadata}}); a non-user root
profile ({{decision-rule}}); and RAS-derived narrowing with a signed
intersection model ({{hop-activation}}).

# Acknowledgments
{:numbered="false"}

The author thanks the authors of the OAuth Identity and Authorization Chaining
Across Domains and the Identity Assertion JWT Authorization Grant, on whose work
this profile builds.

# Document History
{:numbered="false"}

\[\[ To be removed before publication as an RFC ]]

-02

* Added the `audience` parameter to the assertion issuance response, so the
  client learns which IdP to present the assertion to, and registered its
  token response usage.
* Rewrote the gateway example as the first, end-to-end example for gateway
  implementers: co-located RAS and CAI, handle carried in the access token,
  assertion obtained by Token Exchange, and an ordinary ID-JAG redemption at
  the terminal RAS.
* Defined Token Exchange issuance of the assertion at the token endpoint of a
  CAI that is an authorization server, with the access token or Transaction
  Token the workload holds as the subject token, resolving the CAI-issuance
  open item; the gateway example uses it.
* Made the retry-limit and replay-concurrency requirements self-contained
  rather than pointing into Implementation Considerations, defined
  CONTINUABLE and continuation-capable without circularity, and moved the
  per-target client registration prerequisite to the Token Exchange
  introduction.
* Annotated the overview diagram with this profile's additions and the
  terminal hop.
* Implicitly mapped the accepting RAS as a CAI for its own hops; metadata now
  lists only additional CAIs. Added a scope statement to the Overview and a
  non-normative Deployment Topologies subsection holding the carrier
  realizations, added a Topology and Trust security subsection, and switched
  the gateway example to the co-located topology.
* Removed the prohibition on carrying the handle in an access token, along
  with the carrier requirements that restated the CAI preconditions. One rule
  remains: the handle surfaced for a call is the one the RAS bound to the
  authorization that the call's authenticated context selects, read from RAS
  state or from a carrier derived from it. The accepting RAS's own access
  token is now a permitted carrier, and the handle is also registered as an
  introspection response member.

-01

* Renamed Chain Authority to Continuation Assertion Issuer and the
  direct/chained exchanges to root/continuation exchanges, and aligned with the
  base ID-JAG profile: terminology (IdP Authorization Server), Token Exchange
  request/response formatting, and `resource` cardinality (zero or more, per
  RFC 8707).
* Restructured for clarity and scope: grouped request validation into seven
  rules; split the response into success, onward ID-JAG construction, and
  errors; added a non-normative Implementation Considerations section; and
  demoted the intra-domain carrier and other deployment guidance out of
  normative text.
* Bound the originating IdP and tenant to the accepted hop, so the CAI derives
  the assertion audience from that binding rather than requester input;
  restricted CAI issuance to the RAS trust domain.
* Made the RAS `identity_continuation_issuers` advertisement a nomination only
  (the IdP establishes issuer trust and keys independently, resolving an
  authorization-server CAI's keys from its `jwks_uri`) and added a Metadata
  Disclosure security consideration.
* Tightened the security model: relocated the replay-fingerprint and
  authentication-context requirements into the protocol sections with their
  rationale in Security; narrowed `invalid_continuation` to permanently
  unusable handles; distinguished the ID-JAG, assertion, and access-token
  lifetimes; clarified that a depth-limited `act` is not proof of complete
  lineage; and made chain revocation testable.
* Rewrote the Introduction; corrected the examples and cross-references,
  expanded the root-chain envelope and design rationale, softened the
  handle-correlation claim, trimmed the open items, and marked the draft an
  individual submission.

-00

* Initial revision
