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
lets an Identity Provider (IdP) issue an onward Identity Assertion JWT
Authorization Grant (ID-JAG) when a user's request crosses service boundaries
after the user is no longer present. The profile targets deployments in which
several Resource Authorization Servers trust one IdP and use audience-local
subject identifiers that only the IdP can resolve. It complements offline
attenuation for intra-domain fan-out that does not change the subject.

--- middle

# Introduction

OAuth 2.0 {{RFC6749}} issues access to a specific audience, and OAuth 2.0
Token Exchange {{RFC8693}} exchanges one token for another when a request
crosses a trust boundary. The Identity Assertion JWT Authorization Grant
(ID-JAG) {{I-D.ietf-oauth-identity-assertion-authz-grant}} applies Token
Exchange to identity: an Identity Provider (IdP) mints an authorization grant
that names the user for a single downstream audience. Each of these exchanges
assumes the subject's credential, an ID Token, refresh token, or SAML
assertion, is present when the grant is minted.

Many requests outlive that moment. An authenticated request can cross several
services after the user is no longer present, or reach an audience the original
credential does not address. The first hop can still present the user's
credential to obtain an ID-JAG, but a later workload in the chain holds none of
those credentials. The difficulty is sharpest when Resource Authorization
Servers name the user with audience-local (pairwise) subject identifiers that
only the IdP can resolve, a different value at each server: the
later workload cannot name the user for the next audience at all. Only the IdP
can perform that mapping, so continuation is a fresh mint from the IdP, not a
reused or offline-attenuated token.

This document defines the Identity Continuation Assertion: a short-lived,
sender-constrained JWT that a later workload presents as the `subject_token` of
a Token Exchange request, in return for the next audience-scoped ID-JAG and
without another user interaction. The assertion carries a continuation handle
that binds the request to authorization state the IdP recorded when the chain
was established. Each Resource Authorization Server (RAS) trusts only the IdP
to name the user and scope authority. At every hop the IdP both resolves
identity and checks the requested authority against the root-chain envelope, so
continuation stays a fresh policy decision rather than a bearer of standing
authority.

This profile does not define a new access-token format, does not allow a
Resource Server to consume the Identity Continuation Assertion directly, and
does not allow a Chain Authority to name the user for the target audience.

This profile covers:

* a chain of applications, each fronted by its own RAS, for example an expense
  application that calls a travel service that calls a booking service;
* an unattended agent continuing a user's delegation; and
* an API gateway or agent runtime that roots one delegation and continues it
  to upstream services whose audiences are chosen per request rather than
  fixed in advance.

The worked example ({{example}}) follows this authorization path (not the API
call path):

~~~
ExpenseApp -> ExpenseRAS -> TravelRAS -> BookingRAS
~~~

Each trust domain from which the chain continues has three roles: the RAS that
accepts an ID-JAG and binds the hop; a trusted carrier, typically a Transaction
Token Service (TTS), that carries the hop reference to workloads inside the
domain; and a Chain Authority (CA) that issues the Identity Continuation
Assertion a workload presents to the IdP.
One party may operate all three within a domain ({{security-tts}}).

## Relationship to ID-JAG and Identity Chaining

This document profiles Token Exchange {{RFC8693}}, JWT {{RFC7519}}, ID-JAG
{{I-D.ietf-oauth-identity-assertion-authz-grant}}, and OAuth Identity Chaining
{{I-D.ietf-oauth-identity-chaining}}. It adds:

* the Identity Continuation Assertion subject-token type;
* an `identity_continuation_handle` claim in continuation-capable ID-JAGs;
* RAS binding of that claim to accepted authorization state;
* continuation-exchange validation rules;
* intra-domain Transaction Token context; and
* discovery metadata.

# Conventions and Definitions {#terms}

{::boilerplate bcp14-tagged}

This document uses the following terms:

Identity Provider (IdP):
: The authority that authenticates the user, maps the user to each
  audience-local subject, and issues onward grants.

Resource Authorization Server (RAS):
: An Authorization Server that protects a particular API and trusts the
  IdP for subject resolution. It exchanges an ID-JAG for an API access token.

Resource Server (RS):
: The protected API. It never consumes an Identity Continuation Assertion or
  uses a continuation handle for authorization. A co-located workload MAY
  receive the handle only as intra-domain context and MUST NOT place it in an
  access token or external authorization claim.

ID-JAG:
: An Identity Assertion JWT Authorization Grant
  {{I-D.ietf-oauth-identity-assertion-authz-grant}} issued for a target RAS.

Identity Continuation Assertion:
: A short-lived, sender-constrained JWT from a Chain Authority, presented to
  the IdP as a Token Exchange `subject_token` to obtain an onward ID-JAG.

Chain:
: An IdP-held tree of hops under one governing authorization ({{lifecycle}}).

Chain Authority (CA):
: The role trusted by the IdP to issue Identity Continuation Assertions for a
  tenant. It may be a RAS, TTS, gateway, or dedicated service, but never
  resolves the target audience's user subject.

Transaction Token Service (TTS):
: The service that, within a trust domain, derives a bound hop's continuation
  handle from Resource Authorization Server state into the intra-domain chain
  context its workloads carry ({{transaction-token-context}}).

Current actor (presenting actor):
: The workload presenting the assertion to the IdP, named by `act` and
  authenticated by `actor_token`.

Root actor:
: The actor at the root of a chain: the authenticated OAuth client that
  obtains the first ID-JAG ({{client-identity}}). Unlike a current actor,
  it need not present an `actor_token`.

Tenant:
: The administrative boundary within which the chain and Chain Authority
  trust are configured. Tenant determination is deployment-defined but MUST
  derive from authenticated material, not requester-supplied input.

Trust domain:
: An administrative and authentication boundary within which workloads can
  be directly authenticated, comparable to WIMSE
  {{I-D.ietf-wimse-arch}}. Its identifier is deployment-defined.

Continuation Handle (`identity_continuation_handle`):
: An opaque, unguessable, IdP-generated reference to one hop of a delegation
  chain; see {{chain-id}}.

Hop:
: A root or continuation record with an immutable parent reference. Its
  lineage is its path to the root.

Governing authorization:
: The server-side consent and policy record, resolved from the root subject
  token, that anchors a chain and bounds every continuation under it
  ({{lifecycle}}).

Root-chain envelope:
: The state the IdP records when it establishes a chain, and against which
  it evaluates every continuation. The envelope is anchored to the chain's
  governing authorization ({{lifecycle}}) and records, among its dimensions,
  the authorization basis and the continuation authorization defined below.
  Derived from authentication, consent, and tenant policy, it contains:

  * the authenticated user;
  * the authentication context (`auth_time`, `acr`, `amr`);
  * the authorization basis for onward targets;
  * the continuation authorization: the actors or trust domains permitted to
    continue the chain, and the basis on which that permission was
    established ({{root-establishment}});
  * any maximum actor-chain depth set by policy;
  * the chain's governing authorization ({{lifecycle}}); and
  * the chain's expiry.

  These dimensions are establishment-time ceilings; {{root-establishment}}
  defines how they are populated and bounded.

Audience-local (pairwise) subject:
: The subject identifier under which a particular RAS names the user. Distinct
  Resource Authorization Servers may name the same user with different
  identifiers; only the IdP holds the map between them.

Offline attenuation:
: Client-side attenuated delegation, in which a party narrows and forwards a
  credential without contacting the IdP; contrast the IdP-minted continuation
  this profile defines ({{decision-rule}}).

# Protocol Overview {#overview}

A continuation reuses the Token Exchange loop once per boundary: the root
exchange mints the first ID-JAG, and each later boundary mints the next from an
Identity Continuation Assertion. Handles H0 and H1 below name the successive
hops ({{chain-id}}).

~~~
  root credential
       |
       v
  [ IdP ]  mints ID-JAG(H0); owns the envelope and hop tree
       |
       v
  [ accepting RAS ]  redeems it, issues an access token, binds H0
       |
       v
  [ TTS or carrier ]  derives H0 into intra-domain request context
       |
       v
  [ Chain Authority ]  attests the accepted hop, actor, and key
       |
       v
  [ IdP ]  authorizes the next target, mints ID-JAG(H1)
~~~

The responsibilities never mix:

* the IdP owns the root-chain envelope and the hop tree, and alone resolves
  each audience-local subject;
* the accepting RAS decides whether the issued authorization was accepted and
  binds the hop to it ({{ras-processing}});
* a Transaction Token Service, or an equivalent trusted carrier, associates the
  accepted authorization with the current request inside the domain
  ({{transaction-token-context}});
* the Chain Authority attests the accepted hop, the current actor, and its key
  ({{assertion-issuance}}); and
* the IdP alone authorizes the next target against the envelope and mints the
  next ID-JAG ({{validation}}, {{onward-id-jag}}).

Each role validates the inputs within its authority; no artifact or role alone
authorizes continuation.

# When to Use This Profile Versus Offline Attenuation {#decision-rule}

Use this profile when a boundary re-mints the user's identity, that is:

* the next audience uses a pairwise subject only the IdP can resolve;
* the target trusts the IdP, not the previous issuer, to name the user; and
* current revocation and policy must be rechecked at every boundary.

Use offline attenuation, such as {{I-D.li-oauth-delegated-authorization}}, when
the subject and issuer trust stay stable across the boundary and offline
delegation semantics are acceptable, for example intra-domain fan-out under one
workload identity. The two compose: offline attenuation inside a trust domain,
continuation where a boundary re-mints the subject.

# The Identity Continuation Assertion {#assertion}

## Token Type and Media Type {#names}

The Identity Continuation Assertion is identified as follows:

~~~
Name:        Identity Continuation Assertion
Token type:  urn:ietf:params:oauth:token-type:identity-continuation
JOSE typ:    oauth-identity-continuation+jwt
~~~

The assertion is a signed JWT in JWS Compact Serialization {{RFC7519}}, with
media type `application/oauth-identity-continuation+jwt` ({{iana}}). It
MUST NOT be encrypted (JWE) or use nested signing. This profile relies on TLS
for confidentiality and defines only signed JWS Compact Serialization, keeping
a single interoperable representation. The IdP MUST verify the `typ` header per
{{RFC8725}}, which keeps the assertion from being consumed as another token
type.

## Claims {#assertion-claims}

The following is a non-normative example of the Identity Continuation Assertion
claim set:

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
    "jkt": "base64url-current-actor-key-thumbprint"
  },

  "iat": 1710000500,
  "exp": 1710000800,
  "jti": "k7Qm2Xp9Rf4sLc3vBw8aZ1"
}
~~~

The claims have the following meanings and requirements:

`iss`:
: REQUIRED. The Chain Authority issuer. The IdP MUST verify tenant trust and
  the signing key.

`aud`:
: REQUIRED. A single string exactly matching the IdP issuer identifier, not
  its token endpoint URL.

`identity_continuation_handle`:
: REQUIRED. The hop being continued ({{chain-id}}).

`act`:
: REQUIRED. The current actor presenting the Token Exchange request, encoded
  as a single-level `act` claim per {{RFC8693}}. The `act` object contains a
  REQUIRED `iss` and a REQUIRED `sub`, both non-empty strings. Additional
  members MAY carry further identity attributes but are non-authoritative and
  MUST NOT affect identity, authorization, lineage, or issuance; a recipient
  MUST ignore members it does not understand, and `exp`, `nbf`, `aud`, `scope`,
  `cnf`, and nested `act` MUST NOT be present. The IdP MUST reject a
  non-conforming `act`. The IdP, not the assertion, constructs lineage
  ({{onward-id-jag}}).

`cnf`:
: REQUIRED. A confirmation claim {{RFC7800}} that binds the assertion to the
  presenting actor's key. It MUST contain exactly one method: `jkt`, the JWK
  SHA-256 thumbprint {{RFC7638}} of the DPoP key {{RFC9449}}.

`iat`, `exp`:
: REQUIRED. `exp` MUST follow `iat`, and `exp - iat` MUST NOT exceed 300
  seconds, bounding how long a captured assertion could be replayed.

`jti`:
: REQUIRED. A replay-detection identifier that MUST be unique per `iss`
  during the assertion validity window and MUST contain at least 128 bits
  of entropy.

The assertion MUST NOT contain top-level `sub`, `auth_time`, `acr`, `amr`, or
`sid`; those values come from the root-chain envelope.

Other top-level claims MAY appear but MUST be ignored for validation,
authorization, and issuance.

Offline-segment evidence MAY be retained separately and SHOULD remain in the
control plane ({{I-D.mcguinness-oauth-actor-receipts}},
{{I-D.mcguinness-oauth-actor-proofs}}).

## Claims That Are Deliberately Excluded {#excluded-claims}

The assertion MUST NOT convey these Token Exchange request values:

~~~
audience (target)
resource
scope
authorization_details
requested_token_type
~~~

They remain request parameters. Assertion `aud` identifies the IdP, not the
requested target.

## Chain Authority Issuance {#assertion-issuance}

The Chain Authority MUST issue only for an actor in the attested RAS's trust
domain unless tenant configuration explicitly authorizes that external actor
and its keys. Keeping issuance in-domain prevents a handle-holding party from
bypassing the RAS-acceptance path. Actor authentication and the issuance
protocol are deployment-specific.

A presenting workload is a control-plane participant, not a
bare-handle-transporting application ({{handle-carriers}}): it reads the handle
from its
own intra-domain context and presents it to its Chain Authority, along with its
key and any narrowing hints. That handle is advisory input, not an authority
the workload asserts; the checks below re-verify it against RAS-bound state
before any assertion issues.

It MUST authenticate the actor and issue only after establishing that:

1. the handle came through an authenticated, confidential,
   integrity-protected chain path or equivalent authenticated state;

2. the presenting actor is authorized under Chain Authority policy to continue
   the chain;

3. the presenting actor controls the key placed in `cnf`; and

4. `act` names that actor and, if offline attenuation reached the actor, its
   delegation artifact is valid.

Possession of a handle or Transaction Token is insufficient. The Chain
Authority MUST bind the actor to the current transaction, verify that the
handle matches that transaction's RAS-bound state, and recheck authoritative,
uncached RAS state to confirm that the authorization remains active and
continuation remains permitted; a cached read could attest a hop the RAS has
since revoked. It MUST enforce per-transaction and per-actor
rate and fan-out limits with audit records. Target or purpose hints MAY narrow
Chain Authority issuance but MUST NOT control the IdP's target decision.
Propagated context MUST NOT override the root-chain envelope.

# Continuation Handles (`identity_continuation_handle`) {#chain-id}

An `identity_continuation_handle` is an opaque, non-bearer reference to one
IdP-held hop. H0 identifies the accepted source hop; an assertion presents H0
to continue from it, and on success the IdP creates a child H1 and places H1,
not H0, in the onward ID-JAG. The child's immutable parent is the presented
hop, so in the {{example}} chain the TravelRAS hop (H1) is a child of the
ExpenseRAS hop (H0). Reusing H0 for another permitted target creates a sibling
of H1, not a descendant; concurrent children are independent siblings.

The following rules apply:

1. When it establishes a chain ({{root-establishment}}), the IdP MUST embed a
   fresh hop reference as the `identity_continuation_handle` claim of the issued
   ID-JAG, for the root hop and for each continuation hop.
   Handle values MUST NOT be reused across hops. An ID-JAG that carries the
   `identity_continuation_handle` claim is continuation-capable.

2. `identity_continuation_handle` MUST contain at least 128 bits of entropy,
   MUST NOT contain user-identifying information, and MUST consist of 22 to
   256 characters drawn from the base64url alphabet (`A`-`Z`, `a`-`z`,
   `0`-`9`, `-`, `_`).

3. The handle crosses a trust boundary only inside an ID-JAG to the RAS or an
   Identity Continuation Assertion to the IdP, never standalone.

4. The handle MUST NOT appear in an access token or external Resource Server
   authorization claim. Authorized workloads MAY observe it only in
   intra-domain context subject to {{transaction-token-context}}.

5. The IdP performs end-to-end audit correlation; each RAS logs its local
   subject.

6. A continuation-aware Resource Authorization Server binds
   `identity_continuation_handle` to the authorization state it establishes
   ({{ras-processing}}); Resource
   Authorization Servers, Resource Servers, and Chain Authorities MUST NOT
   modify the value.

7. A hop is continuable only after RAS acceptance and binding
   ({{hop-activation}}). The handle conveys no authority; the IdP MUST use it
   only to resolve hop state, subject, and policy.

8. A hop's parent reference is immutable. The IdP MUST derive lineage solely
   by walking parent references from the presented hop to the root, and MUST
   NOT maintain or extend a single chain-wide actor history: concurrent
   sibling continuations are independent branches.

The IdP MAY derive handles from an internal delegation identifier using a
keyed one-way function if rules 1, 2, and 8 remain satisfied and the resulting
handles remain unlinkable.

## Continuation Handle Carriers {#handle-carriers}

A handle travels by one of three carriers, depending on context lifetime:

| Situation | Authoritative store | Application carries |
|---|---|---|
| Cross-domain hop ({{ras-processing}}) | IdP hop state | Assertion to the IdP, then ID-JAG to the RAS |
| Active request ({{transaction-token-context}}) | RAS authorization state | Access token; the TTS derives the context |
| Scheduled execution ({{task-provenance}}) | Durable task/RAS authorization | Opaque task identifier |

An external or requesting application never selects or persists a bare handle
for transport; it carries an artifact from which trusted server-side state
derives the handle. An authorized intra-domain control-plane workload is
different: it reads the handle from that state and presents it to its Chain
Authority ({{assertion-issuance}}).

## Handle Freshness and Unlinkability {#chain-id-privacy}

Handles are unlinkable across hops but not among participants in one hop, and
revoked handles fail the next continuation exchange. {{privacy}} covers the
residual correlation channels.

# Chain Lifetime and Revocation {#lifecycle}

A chain is continuable only while active at the IdP. Each cross-boundary hop
is a fresh policy check. Revoking a hop stops its subtree at the next
continuation, fail-closed, but does not invalidate already issued ID-JAGs or
access tokens; the revocation window is therefore bounded by the ID-JAG's
short lifetime and by the access-token lifetime the accepting Resource
Authorization Server sets; this profile does not constrain that lifetime.

This is the deliberate difference from an offline-attenuated token, whose
minted child stays usable for its lifetime without contacting an authority.

Three lifetimes MUST NOT be conflated: the ID-JAG's short redemption window;
the access-token lifetime the accepting RAS sets independently
({{ras-processing}}); and the IdP-held continuation chain. Revoking the chain
does not shorten an already-issued access token, and an access token outliving
the chain does not extend it.

~~~
ID-JAG redeem   |==|
access token    |===========|              RAS-set, independent
IdP-held chain  |=========================| IdP-held, spans hops
~~~

The governing authorization ({{terms}}) anchors to a lifecycle token: a refresh
token anchors to its OAuth grant, and `sid` or `SessionIndex` anchors to its
session. Rotation of a refresh token does not affect the grant anchor. Grant
expiry or revocation ends the
chains anchored to that grant; session termination ends the chains anchored
to that session; and withdrawal of continuation consent or policy ends any
chain it governs. A session-anchored chain MUST NOT outlive its session; only
grant-anchored chains may outlive logout. Ending a chain this way bounds only
new continuations; an ID-JAG already issued remains redeemable for its own
lifetime, since
redemption is not a continuation.

The IdP MUST bound chain lifetime by the governing authorization and reject
expired chains.

`auth_time`, `acr`, and `amr` are fixed at root issuance; continuation MUST
NOT refresh them.

The IdP MUST revoke whole chains and MAY revoke an individual hop's subtree.
It MUST reject continuation from revoked state.

Issued access tokens remain governed by their RAS.

For a grant-anchored chain, the IdP MUST provide a user- or
administrator-facing interface showing the chain's root context, hop graph,
lineage, granted targets, expiry, and any recorded purpose; it MUST support
whole-chain revocation
and subtree revocation when offered. It SHOULD notify the user or
administrator at establishment and near expiry. The same interface is
RECOMMENDED for session-anchored chains. See {{GRANT-MGMT}}.

# Token Exchange Profile {#token-exchange}

An Identity Continuation Assertion is used as the `subject_token` of an OAuth
2.0 Token Exchange request {{RFC8693}}. A direct and a chained request use the
same Token Exchange framework: a chained request substitutes an Identity
Continuation Assertion for the root credential and additionally supplies the
actor authentication and DPoP proof described below. The IdP establishes the
chain; no request parameter asks it to do so ({{root-establishment}}).

## Direct ID-JAG Request

A direct request, in which the subject token is a normal subject token such as
an ID Token, refresh token, or SAML assertion:

~~~
grant_type=urn:ietf:params:oauth:grant-type:token-exchange
requested_token_type=urn:ietf:params:oauth:token-type:id-jag
audience=https://ras.travel.example/
resource=https://api.travel.example/
scope=trips.read
subject_token=<id_token | refresh_token | SAML assertion>
subject_token_type=<normal-subject-token-type>
actor_token=<sender-constrained-current-actor-credential> (OPTIONAL)
actor_token_type=<actor-token-type>                       (OPTIONAL)
~~~

On a direct request, `actor_token` is OPTIONAL ({{root-establishment}}). The
direct request and its ID-JAG conform to the base ID-JAG profile
({{I-D.ietf-oauth-identity-assertion-authz-grant}}) except where this document
extends it for continuation-capable issuance.

## Establishing a Chain {#root-establishment}

The IdP, not the client, establishes a chain. It MUST do so when a direct
ID-JAG exchange is governed by a continuation-capable governing authorization,
and MUST include the root handle in the ID-JAG. The exchange MUST include a
valid DPoP proof {{RFC9449}}, and the IdP MUST bind the resulting ID-JAG to that
key in `cnf`; without valid proof it MUST NOT include an
`identity_continuation_handle`. The IdP MAY defer materializing chain state
until the first continuation, provided the handle still resolves to the same
root and envelope; this does not relax the reservation durability of
{{validation-replay}}. Without continuation authorization, the IdP MUST
NOT establish a chain or include a handle. Advertised support ({{metadata}})
signals capability, not authority.

The root subject token MUST resolve to one of these lifecycle anchors:

* a refresh token's OAuth grant;
* an ID Token `sid` {{OIDC.FrontChannelLogout}} resolving to an active IdP
  session for that user and client; or
* a SAML `SessionIndex` {{SAML2.Core}} resolving to an active IdP session
  for that user and presenter.

The IdP MUST NOT root a chain from an unresolved anchor or an access token.
Non-user-rooted authority is out of scope. `sid` and `SessionIndex` are used
only for resolution and MUST NOT enter assertions or chain context.

Server-side consent and policy make the governing authorization
continuation-capable and populate the root-chain envelope of {{terms}} (the
authenticated user, authentication context, authorization basis, permitted
actors or trust domains, depth, governing authorization, and expiry). Token
claims cannot supply these values. Every dimension is an
establishment-time ceiling: later policy MAY narrow or revoke it but MUST NOT
broaden it; broadening requires a new chain. An envelope MAY enumerate exact
audience and resource pairs with their permitted scopes and authorization
details {{RFC9396}}; otherwise it records a
stable, policy-based basis, fixed at establishment, against which the IdP
evaluates each requested target at request time. A policy-based basis is not
whatever the user could authorize later: it is the enforceable record captured
at establishment, and consent granted afterward cannot broaden it.

The root actor is the authenticated OAuth client under the mapping in
{{client-identity}}. An optional `actor_token` MUST be valid, MUST be accepted
for continuation, and MUST designate the IdP where applicable. It MUST also be
sender-constrained to the confirmed key and MUST identify that client.
Only after validation does the IdP record the root actor and key. The root
actor's identity rests entirely on this client authentication
({{client-identity}}); base ID-JAG's recommendation to use a confidential
client therefore applies to a continuation-capable root.

For every root or child hop, the IdP records the target RAS and the Chain
Authorities mapped to it; the mapping MAY be static tenant configuration.
Only a mapped Chain Authority may attest that hop. A terminal RAS ignores the
handle; only a continuation-aware RAS can bind it and make the hop
continuable. Grant-profile advertisement is discovery only; a party that
requires onward continuation SHOULD consult it when available.

Establishment is at-least-once: retrying a lost response MAY create a second
chain. Revocation of the governing authorization applies to every chain rooted
in it, and the actor-chain depth bound is enforced per branch; the IdP MUST
enforce configured fan-out, rate, and hop-count limits as an aggregate keyed
to the governing authorization; a retried establishment MUST NOT evade these
limits.

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
never by the assertion ({{excluded-claims}}). Following
{{I-D.ietf-oauth-identity-assertion-authz-grant}}, `audience` identifies the
target Resource Authorization Server and `resource` ({{RFC8693}}, originally
defined in {{RFC8707}}) identifies the protected resource.

The request MAY also include `authorization_details` {{RFC9396}}, which the
ID-JAG profile supports in both the exchange and the issued grant; the
authorization-basis check ({{validation}}, rule 14) applies equally to it and
to scope. Client authentication is required on every exchange
({{client-identity}}) and is omitted from the example bodies for brevity.

## Sender-Constrained Presentation {#sender-constrained-presentation}

This section applies to a chained request; a direct request's DPoP requirement
is specified in {{root-establishment}}.

The actor MUST present a DPoP proof {{RFC9449}} for the key in
`cnf.jkt`. The IdP MUST verify the match and reject absent or invalid proof.

DPoP is the single mandatory confirmation method for interoperability: a
different confirmation method in the onward grant would make the target
validate that confirmation differently than for a directly issued ID-JAG. This
version therefore defines no mutual-TLS variant {{RFC8705}}; see {{open-items}}.

The request MUST include a valid, accepted `actor_token` identifying the
actor in `act`. It MUST be sender-constrained to the same key and MUST NOT be
bearer. For a JWT, the IdP verifies `cnf.jkt`; for an opaque token, it obtains
equivalent confirmation from authoritative metadata such as introspection
{{RFC7662}}. Any audience or applicability restriction MUST designate the
IdP.

The IdP MUST compare the actor `iss` and `sub` as case-sensitive strings with
no transformation or canonicalization ({{RFC7519}}), across `actor_token`,
`act`, and the authenticated client. Identities in different tenants never
compare equal.

The onward ID-JAG MUST use the same DPoP key. The actor proves possession
again at the target RAS.

Key rotation takes effect when the actor obtains a new assertion and actor
token bound to the new key.

## Client Identity and Authentication {#client-identity}

The current actor MUST authenticate as an OAuth client, and the IdP MUST map
that client authoritatively to an actor identity; self-asserted mappings
MUST NOT be accepted. On a continuation exchange the IdP MUST also match that
identity to the assertion's `act` and the `actor_token`; at root establishment
neither is present, so client authentication alone identifies the root actor.

A sender-constrained JWT MAY serve as both client assertion and `actor_token`
when it satisfies both profiles. For {{RFC7523}}, its `sub` is the
`client_id`, and the IdP MUST authorize its issuer for that client. Otherwise
the client authenticates separately.

The onward ID-JAG `client_id` is the current actor's identifier at the target
RAS. The actor therefore needs a registration or resolvable client identity
at each target, as required by ID-JAG.

Four signals identify the actor on a continuation exchange, and all four, with
the confirmed key, must agree:

| Signal | What it establishes |
|---|---|
| Client authentication | who is calling the IdP token endpoint |
| `actor_token` | the actor vouched for by its workload-identity issuer |
| Assertion `act` | the actor the Chain Authority bound to the accepted hop |
| DPoP | live possession of the key binding all three to this request |

The apparent redundancy is conjunctive trust: a mismatch on any one, or a key
not proven live, fails the exchange ({{validation}}, rules 9 through 11).

## Continuation Handle Delivery {#handle-delivery}

The IdP delivers the hop reference in the issued ID-JAG's
`identity_continuation_handle` claim ({{chain-id}}, rule 1;
{{onward-id-jag}}), not as a separate Token Exchange response parameter. Each
hop's handle is distinct and its parent reference is immutable ({{chain-id}},
rules 1 and 8). The accepting Resource Authorization Server binds it to
authorization state ({{ras-processing}}); the current domain then surfaces it
to continuers through intra-domain chain context
({{transaction-token-context}}).

There is no advisory chain-expiry response parameter. Chain lifetime is
authoritative at the IdP ({{lifecycle}}); a deployment that needs advance
warning of expiry conveys it through authenticated task or authorization
state, an optional ID-JAG claim, or a management API, not through the
Token Exchange response.

## Request Validation {#validation}

The IdP MUST reject the request unless every rule below holds. Their order is
not significant, though one rule's input may come from another's resolution:
the tenant used to check Chain Authority trust comes from resolving the
presented handle.

1. the request contains exactly one each of `grant_type`, `subject_token`,
   `subject_token_type`, `requested_token_type`, `actor_token`, and
   `actor_token_type`; the `grant_type` is
   `urn:ietf:params:oauth:grant-type:token-exchange`, and the
   `subject_token_type` is
   `urn:ietf:params:oauth:token-type:identity-continuation`;

2. the request contains exactly one `audience` and one `resource` parameter,
   and at most one `scope` and one `authorization_details`; `scope` and
   `authorization_details` are OPTIONAL, each evaluated by rule 14 when present;

3. the assertion is a JWT containing exactly one value for each required claim
   defined in {{assertion-claims}}; `iss`, `aud`,
   `identity_continuation_handle`, and `jti` are non-empty strings; `act` and
   `cnf` are JSON objects, with `cnf` containing exactly one confirmation
   method; `iat` and `exp` are JSON numbers representing NumericDate values;
   and the JOSE `typ` header is `oauth-identity-continuation+jwt`;

4. the assertion signature validates using a key authorized for the assertion
   issuer, and the JOSE `alg` is an asymmetric signature algorithm on the IdP's
   configured allowlist (the `none` algorithm MUST be rejected; see
   {{security-alg}});

5. the assertion `aud` exactly matches the IdP's issuer identifier;

6. assertion `iss` is trusted for the tenant, mapped to the hop's accepting RAS,
   and authorized to pair with the `actor_token` issuer for that tenant;

7. the handle identifies a RAS-accepted hop ({{hop-activation}}) on an active
   chain, no ancestor
   subtree is revoked, and the actor lineage that results from collapsing
   consecutive same-actor entries, as the onward `act` will ({{onward-id-jag}}),
   is within its depth bound; the bound counts lineage entries, not hops;

8. the assertion does not contain a top-level `sub`, `auth_time`, `acr`,
   `amr`, or `sid` claim, nor an `audience`, `resource`, `scope`,
   `authorization_details`, or `requested_token_type` claim
   ({{assertion-claims}}, {{excluded-claims}});

9. the assertion's `act` claim is present, conforms to the schema of
   {{assertion-claims}}, and identifies the current actor;

10. the request and actor are bound:
    * the request is authenticated as an OAuth client that is the same
      entity as the current actor ({{client-identity}});
    * the `actor_token_type` names a token type the IdP supports, and the
      `actor_token` has a trusted issuer for the actor's domain and tenant, is
      valid for that type, is accepted, designates the IdP where applicable,
      and authenticates the actor;
    * the `actor_token` is sender-constrained to the key confirmed by the
      assertion's `cnf` ({{sender-constrained-presentation}});
    * that actor is the actor named in `act`; and
    * that actor is permitted by the chain's continuation authorization
      ({{root-establishment}}) to continue from the presented hop;

11. the request proves possession of the key confirmed by `cnf` with a DPoP
    proof {{RFC9449}} matching `cnf.jkt`
    ({{sender-constrained-presentation}});

12. `jti` is not yet reserved for the assertion issuer, or is RESERVED or
    ISSUED under a fingerprint matching this request (permitting idempotent
    retry; see the reservation rules in {{validation-replay}}); a RESERVED or
    ISSUED `jti` under
    a different fingerprint, or a FAILED `jti`, is rejected;

13. `iat` and `exp` are valid NumericDates, `iat` is within permitted future
    clock skew (which SHOULD NOT exceed 60 seconds), `exp` follows `iat`, the
    assertion is unexpired, and its lifetime does not exceed 300 seconds;

14. requested audience, resource, scopes, and authorization details are
    within the root-chain envelope as recorded at establishment and within
    current IdP actor policy; authorization-details containment uses the
    comparison rules defined for each authorization-detail type, since
    {{RFC9396}} defines no generic comparison, and a detail type whose rules
    the IdP does not implement is rejected;

15. the requested output token type is
    `urn:ietf:params:oauth:token-type:id-jag`; and

16. the IdP can resolve, for the requested `audience`, both the
    audience-local subject and the current actor's client identifier
    ({{client-identity}}).

## Replay Reservation and Retry {#validation-replay}

The reservation model gives a client idempotent recovery after a lost response
while preventing one assertion from authorizing more than one distinct request.
It fixes each assertion's outcome to a single request fingerprint.

After validation, grant issuance MUST atomically reserve (`iss`, `jti`) and
bind it to a fingerprint containing audience and resource as exact strings,
scope as an order-independent set, the exact `authorization_details` JSON as
received after form decoding (different serializations are different
requests), the actor (its `iss` and `sub`), the confirmed key (its `cnf.jkt`
thumbprint), and a SHA-256 hash of the exact `subject_token` value after form
decoding, which binds the fingerprint to the specific assertion and its handle.

The record states are RESERVED, ISSUED, and FAILED, distinct from the hop
states of {{hop-activation}}. Reservation MUST occur
only after target and policy validation. Once reserved, the tuple MUST NOT be
released for another fingerprint. An identical retry MUST return the
same previously issued grant, not a new one; a different fingerprint MUST be
rejected. Only one concurrent request can reach ISSUED; a concurrent request
under a matching fingerprint waits for or retries that result. The IdP MUST
retain
the tuple through `exp` plus the maximum permitted clock skew, using the same
clock used to evaluate `exp`. A reservation that does not reach ISSUED before
`exp` becomes FAILED; a FAILED tuple is terminal and requires a fresh
assertion.

Replay uniqueness MUST use (`iss`, `jti`), not an unbound tenant partition;
partitioning by tenant alone would let two assertion issuers in one tenant
collide on a reused `jti`.

The IdP needs strongly consistent replay state. The actor-chain depth bound
counts collapsed lineage entries, so an actor that repeatedly continues as
itself collapses to one entry each time and never trips that bound. To bound
such growth, the IdP MUST enforce a configured limit on fan-out, rate,
and hop count, aggregated per governing authorization ({{root-establishment}}),
and MUST prune expired or revoked hop state.

After a lost response, a client MAY retry the same assertion to recover the
ISSUED result or obtain a fresh assertion. A fresh assertion may create an
equivalent grant and sibling hop but no additional authority. Application
idempotency remains out of scope. The Chain Authority SHOULD account for
retries separately from fan-out while preventing retry claims from bypassing
limits; issuance SHOULD be inexpensive relative to the exchange.

## Success and Error Responses {#validation-response}

On success, the IdP records a PENDING child ({{hop-activation}}) of the
presented hop and issues an ID-JAG containing the resolved target `sub` and
fresh handle. An idempotent retry (rule 12; {{validation-replay}}) instead
returns the previously issued grant unchanged, creating no new hop or handle.

On failure, the IdP MUST return an OAuth error {{RFC6749}}, {{RFC8693}}. It
SHOULD use `invalid_request` for malformed, inconsistent, or unacceptable
tokens; `invalid_dpop_proof` for DPoP failure; and `invalid_target`,
`invalid_scope`, or `invalid_authorization_details` for requests outside the
envelope.

The IdP MUST return `invalid_continuation` ({{iana}}) when the presented handle
cannot support this continuation, distinguishing a dead hop from the
`invalid_request` of a malformed request. Such a handle is terminal: retrying
it cannot succeed. Recovery requires establishing a new chain and succeeds only
where the governing authorization is still continuation-capable: a
session-anchored chain re-roots by re-authenticating the user, a grant-anchored
chain from its still-valid grant without the user; a handle disabled by
withdrawn continuation authorization cannot re-root at all. Target-specific errors (`invalid_target`,
`invalid_scope`,
`invalid_authorization_details`) leave the chain otherwise continuable, so a
client abandons only the current request.

## Onward ID-JAG {#onward-id-jag}

The onward ID-JAG conforms to the base ID-JAG profile
({{I-D.ietf-oauth-identity-assertion-authz-grant}}) except where this document
extends it: its `sub` is the IdP-issued pairwise subject for the target
audience, and `aud_sub` remains available under the base profile where the
target's native subject namespace differs. The following is a non-normative
example of the onward ID-JAG issued by the IdP:

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

The IdP constructs `act` by placing the authenticated current actor atop the
presented hop's lineage; it never copies lineage from the assertion. Siblings
do not contribute. Consecutive identical actors collapse to one entry, though
the hop record remains; policy MAY limit disclosed depth, narrowing what a
target sees without changing the depth bound the IdP enforces
({{validation}}, rule 7).

The target RAS validates the ID-JAG, issues its access token, and, if
continuation-aware, binds the handle. The ID-JAG `client_id` is the current
actor's identifier at that RAS.

## Authorization Server Metadata {#metadata}

An IdP that supports this profile SHOULD signal it in its authorization server
metadata {{RFC8414}} with the following parameter:

`identity_continuation_supported`:
: OPTIONAL. Boolean value indicating that the IdP accepts Identity Continuation
  Assertions of the
  `urn:ietf:params:oauth:token-type:identity-continuation` subject token type
  and issues continuation-capable ID-JAGs carrying the
  `identity_continuation_handle` claim. Default `false`.

A Resource Authorization Server advertises separately, by listing the grant
profile `urn:ietf:params:oauth:grant-profile:id-jag-continuation` in its
`authorization_grant_profiles_supported`
{{I-D.ietf-oauth-identity-assertion-authz-grant}}, that it recognizes a
continuation-capable ID-JAG and binds the `identity_continuation_handle`
claim to authorization state ({{ras-processing}}). This value is distinct
from the base ID-JAG grant profile, which signals only ordinary ID-JAG
processing and no handle binding. Because a continuation-capable ID-JAG is an
ID-JAG, a Resource Authorization Server that advertises
`urn:ietf:params:oauth:grant-profile:id-jag-continuation` MUST also advertise
the base `urn:ietf:params:oauth:grant-profile:id-jag` profile and the
`urn:ietf:params:oauth:grant-type:jwt-bearer` grant type on which ID-JAG
depends ({{I-D.ietf-oauth-identity-assertion-authz-grant}}). These are
distinct capabilities: the IdP signal covers continuation issuance and
acceptance; the Resource Authorization Server profile covers handle binding.

Absent these signals, a party learns of support out of band or by attempting an
exchange.

# Continuation-Aware Resource Authorization Server {#ras-processing}

Only a RAS from which continuation occurs implements this extension. A
terminal RAS processes an ordinary ID-JAG and ignores the handle; because no
later continuation uses the terminal hop, the RAS need not bind its handle.

A continuation-aware Resource Authorization Server, one that implements this
extension and advertises the continuation grant profile ({{metadata}}), MUST,
on accepting a continuation-capable ID-JAG:

1. validate the ID-JAG per {{I-D.ietf-oauth-identity-assertion-authz-grant}};
2. authenticate the client presenting it;
3. verify the sender constraint, that is, proof of possession of the
   confirmed key;
4. apply its local authorization policy;
5. issue an access token sender-constrained to the confirmed key; and
6. bind `identity_continuation_handle` to the authorization state it
   establishes, recording whether continuation is permitted.

Binding and token issuance MUST be atomic, so no access token is issued
without its binding and no binding is recorded without a token. Repeated
redemption of one ID-JAG MUST bind to the same hop authorization record, so a
retry cannot create multiple records for one grant. The RAS MUST NOT place the
handle in an access token, external authorization claim, or protected-API
authorization input. It exposes the binding only privately within its trust
domain.

## Hop Activation {#hop-activation}

A hop moves through three states. The IdP creates it PENDING. Successful RAS
binding makes it ACCEPTED. A mapped Chain Authority attests a hop only once it
is ACCEPTED, so a PENDING hop yields no assertion and reaches no continuation
exchange. A fresh assertion from the mapped Chain Authority
lets the IdP evaluate the hop as CONTINUABLE for one request; CONTINUABLE is
not stored but holds only while rules 6, 7, and 9 to 11 of {{validation}} hold
for
that request. There is no RAS callback.

| State | Where it lives | Meaning |
|---|---|---|
| PENDING | IdP | the IdP issued the ID-JAG but has no acceptance evidence |
| ACCEPTED | RAS authorization state | the RAS redeemed the grant, authorized it, and bound the handle |
| CONTINUABLE | IdP, for one exchange | a mapped Chain Authority freshly attested the still-active binding |

ACCEPTED is a state of the RAS's own authorization, not an IdP transition
delivered by callback; the IdP learns of it only through a Chain Authority
attestation.

The Chain Authority assertion is trusted evidence of acceptance, not
IdP-verifiable proof: the IdP has no channel back to the RAS to confirm
acceptance directly ({{rationale-pull}}), so it relies on the mapped Chain
Authority having rechecked authoritative RAS state before attesting
({{assertion-issuance}}). A compromised mapped Chain Authority can thus attest a
hop that its Resource Authorization Server refused, or for which it denied
continuation, overriding that server's local decision; the envelope still
bounds the result, but the accept-and-continue gate is only as trustworthy as
the
mapped Chain Authority. Absent such compromise, an issued-but-rejected ID-JAG
cannot be continued because no mapped Chain Authority may attest it. A mapped
Chain Authority is mandatory; its absence fails closed. Acceptance gates
continuation but does not bound downstream authority ({{ras-gate}}).

## A Gate, Not a Ceiling {#ras-gate}

RAS acceptance is a gate, not a downstream ceiling. The IdP evaluates later
targets against the root envelope; local RAS authorization neither narrows nor
widens it. Cross-domain scope vocabularies are not generally comparable, so
RAS-derived narrowing, if ever defined, would need signed constraints and an
explicit intersection model.

## Durable Task Authorization {#task-provenance}

Scheduled continuation MUST root in durable RAS authorization, not a
scheduler-held handle: a scheduler holding the handle would turn it into a
durable, bearer-like credential outside the per-call key proof and RAS binding
that gate every other use. The scheduler holds only a task identifier; each
authenticated run derives the handle from active task state and still
requires an assertion from a mapped Chain Authority.

# Transaction Token Chain Context {#transaction-token-context}

A trusted intra-domain carrier associates the accepted hop with the current
request. Such a carrier is server-derived, bound to the current credential,
key, and RAS authorization, non-overridable by the requester, confined to the
trust domain, and re-derived when replaced, as the rules below require. A
Transaction Token is the standardized realization of these properties; an
equivalent carrier is permitted only where all of them hold
({{security-envelope}}).

Within a trust domain, a TTS derives the handle from RAS-bound authorization
state and places it in Transaction Token context
{{I-D.ietf-oauth-transaction-tokens}}:

~~~ json
"tctx": {
  "identity_continuation": {
    "iss": "https://idp.example/",
    "tenant": "tenant-123",
    "handle": "kW4uJ8pTe2NxA6rQvD1zYs"
  }
}
~~~

The `identity_continuation` object has the following members:

* `iss` (REQUIRED): the exact IdP issuer identifier.
* `handle` (REQUIRED): the hop's continuation handle.
* `tenant` (REQUIRED except for a single-tenant IdP issuer, identified by
  `iss`): the tenant.

A recipient MUST ignore unknown members. A malformed or repeated object MUST
be treated as carrying no chain context.

The requester MUST NOT supply or override this member. Before deriving, the
protected endpoint or TTS MUST validate live proof of possession of the
confirmed key presented on the current call. It MUST then derive the member
from the authorization record bound to that verified credential, key, and RAS
state, never from a session or subject, which could otherwise bind the wrong
user's authorization state to this call. The token MUST
NOT be accepted outside its trust domain and is normally forwarded
unmodified. A replacement token MUST re-derive the member from the same
RAS-bound state.

Authorized intra-domain workloads MAY read the handle. They MUST NOT place it
in access tokens, external authorization claims, responses, webhooks, errors,
or calls to non-participants, and SHOULD omit it from logs and traces. The
handle conveys no authority.

# Security Considerations {#security}

This profile assumes TLS, a correct IdP subject map and root-chain envelope,
and the OAuth guidance of {{RFC9700}}. It addresses these adversaries:

* an on-path attacker replaying an assertion ({{security-replay}});
* a compromised intermediate workload broadening authority or continuing the
  wrong user's chain ({{security-envelope}});
* a compromised Chain Authority or actor-token issuer
  ({{security-trust-model}});
* a party influencing the client-to-actor mapping, which on a direct request
  carrying no `actor_token` is the sole authenticator of the root actor
  ({{client-identity}});
* a malicious Resource Server or audience attempting cross-domain correlation
  ({{privacy}}); and
* a faulty or co-located Transaction Token Service ({{security-tts}}).

## Sender Constraint and Proof of Possession

The assertion MUST NOT be accepted as bearer {{RFC7800}}. It requires live
proof of the actor's `cnf` key.

## Short Lifetime and Replay {#security-replay}

The 300-second ceiling and atomic reservation of (`iss`, `jti`)
({{validation-replay}}) limit replay to the IdP continuation exchange.

## Root Authentication Context {#security-assurance}

Authentication context comes only from the root envelope. Continuation MUST
NOT extend or strengthen it, for example by presenting a higher `acr` or added
`amr` than the user performed at root; the IdP MUST copy it unchanged into
onward ID-JAGs ({{onward-id-jag}}) when
{{I-D.ietf-oauth-identity-assertion-authz-grant}} requires those claims.

## Envelope Enforcement and Offline Attenuation {#security-envelope}

The envelope bounds every target and authority. The Chain Authority validates
any offline attenuation segment; the IdP still enforces only the envelope.
Because the assertion is target-agnostic, a permitted actor may select any
target within that ceiling.

Wrong-handle association can continue the wrong user's bounded chain. The TTS
establishes the authoritative association between the request and the handle by
deriving it from the current credential's RAS-bound state; a handle a workload
supplies is not authoritative, and the Chain Authority rejects substitution.
Another carrier MAY be used only if it provides the same properties
({{transaction-token-context}}): server-derived; bound to
the credential, key, and RAS authorization; non-overridable; domain-confined;
and re-derived when replaced.

## Trust in the Transaction Token Service {#security-tts}

A faulty TTS can splice a valid wrong-user hop into a transaction, and every
downstream check at the Chain Authority and IdP still sees a well-formed
continuation. The TTS MUST key derivation to the presented credential, not a
session or subject, and SHOULD be monitored independently.

One operator MAY run the RAS, TTS, and Chain Authority. Where independent
acceptance evidence matters, deployments SHOULD separate them or audit the
binding-to-attestation path.

## Trust in the Chain Authority {#security-chain-authority}

The IdP MUST scope Chain Authority trust by issuer, keys, tenant, and mapped
RAS. Deployments SHOULD minimize that scope and monitor anomalies. Trust is
established out of band or through federation, as in {{RFC7523}}. Handle
confidentiality provides defense in depth, not authorization.

## Trust in Actor Token Issuers {#security-actor-issuers}

The IdP MUST accept actor tokens only from issuers trusted for the actor's
domain and tenant. An untrusted or out-of-scope issuer MUST be rejected even
with a valid Chain Authority assertion.

## Conjunctive Trust and Issuer Pairing {#security-trust-model}

A continuation requires all of these, and no one of them suffices alone:

* the Chain Authority mapped to the presented hop's accepting Resource
  Authorization Server, which attests the chain-to-actor transition
  ({{validation}}, rule 6);
* the workload identity issuer trusted for the current actor's trust domain,
  which authenticates the actor through the `actor_token` ({{validation}},
  rule 10);
* live proof of possession of the confirmed key ({{validation}}, rule 11); and
* the IdP's own root-chain envelope and current-actor policy
  ({{validation}}, rule 14).

The IdP MUST authorize Chain Authority and actor-token issuer pairings per
tenant; separate trust in each is insufficient.

The anchors MAY be co-located, with this resulting blast radius:

| Compromised | What it yields | What still bounds it |
|---|---|---|
| Chain Authority | Can attest mapped hops | Still needs permitted actor, key proof, and envelope |
| Actor issuer | Can mint actor identities | Needs mapped CA, key proof, and envelope |
| Chain Authority + actor issuer | Can fabricate an actor transition | Still envelope-bounded |
| RAS + TTS + Chain Authority | Can fabricate acceptance and attestation | Still envelope-bounded |

No compromise listed above yields authority beyond the root-chain envelope.
Co-locating anchors trades away the defense in depth that the conjunction
otherwise provides. If the IdP is also co-located, even the envelope backstop
becomes organizational rather than protocol-separated.

## Actor Chain Integrity {#security-actor-chain}

Lineage is IdP-constructed. An assertion names only the current actor; the IdP
MUST reject any mismatch. Offline-segment actors do not enter lineage.

## Token, Type, and Algorithm Confusion {#security-alg}

The IdP MUST verify `typ`, reject `alg=none` and symmetric algorithms, and
allowlist asymmetric algorithms. It MUST select keys from trusted issuer
configuration; `kid` MAY select among them. It MUST NOT trust assertion
`jku`, `x5u`, embedded `jwk`, or other supplied key material.

# Privacy Considerations {#privacy}

A hop's `identity_continuation_handle` is visible only to its ID-JAG client,
accepting Resource Authorization Server, and IdP, plus the domain's
Transaction Token Service, Chain Authority, and authorized workloads. It MUST
NOT enter an access token, external authorization claims, or protected-API
authorization input ({{chain-id}}, rule 4). A workload receiving it as
intra-domain context is a control-plane participant.

Handles are opaque, high-entropy, and hop-specific ({{chain-id}},
{{chain-id-privacy}}). Resource Authorization Servers therefore cannot use
them to correlate a user across SaaS boundaries.

The chain is not unlinkable: the IdP correlates it, participants sharing a
handle can correlate that hop, and actor lineage and timing may correlate
transactions across audiences. For example, an observer comparing ID-JAGs
issued to two audiences within one short window and carrying the same
actor-chain shape may infer they belong to one user's transaction, even without
a shared handle. The onward ID-JAG's `act` chain also names the prior actors to
the accepting RAS outright, with no correlation needed; {{onward-id-jag}} lets
policy limit the disclosed depth. Deployments SHOULD disclose handles only to
participants that continue or administer the chain. They MAY limit lineage
exposed to each audience, subject to audit requirements.

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
: This document, {{validation-response}}

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
  delegation chain, used to correlate a continuation to its chain
  and parent hop and to resolve the per-audience subject. This claim
  appears in an Identity Continuation Assertion and in a continuation-capable
  ID-JAG, and its value may also travel in intra-domain chain context; it
  MUST NOT be placed in an access token or a Resource Server's external
  authorization claims ({{chain-id}}, rules 3 and 4).

Change Controller:
: IETF

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
pairwise-subject boundary to cross) ({{decision-rule}}). Delegated Authorization
{{I-D.li-oauth-delegated-authorization}}, whose client-issued tokens carry no
subject, composes with this profile as the intra-domain layer and stops where
re-issuance to a new subject begins.

## Alternative Topology: Resolution at the Target {#rationale-pull}

A pull design would have each target resolve a reference at the IdP over a
back channel {{RFC7662}}. It requires a new target-side grant and per-request
back channel. The selected push design reuses the ID-JAG grant path, adding
only handle binding at continuation-source RASes; the Chain Authority supplies
acceptance evidence. Pull remains a possible companion profile.

## Why a Signed Assertion Rather Than a Bare Grant Type {#rationale-grant-type}

The signed assertion lets the Chain Authority attest the authenticated actor,
key, accepted hop, and any intra-domain policy checks that the IdP cannot
observe. It does not authorize target or scope. Where that domain-local
attestation is unnecessary, a recipient-bound direct grant remains a possible
simplification.

# Examples {#examples}

This non-normative appendix illustrates three deployment shapes: interactive
application chaining ({{example}}), an unattended background agent
({{example-background}}), and a gateway with dynamically selected upstream
audiences ({{example-gateway}}).

Message sequences are vertical lifelines with time flowing downward. The
payload and state blocks below them are tagged "On the wire" when they cross a
trust boundary,
"Intra-domain context" when they travel only within one trust domain, and
"Server-side state" when they are never transmitted. Continuation handles
are written H0, H1, and H2, one per hop.

## Worked Example (Same-IdP) {#example}

This section walks the canonical same-IdP flow end-to-end for a single
user: ExpenseApp invokes ExpenseSaaS; ExpenseService, the workload handling
that request, calls TravelAPI to reach TravelSaaS; and TravelService, the
TravelSaaS workload that handles that call, in turn calls BookingAPI to
complete the itinerary. All parties trust one enterprise IdP at
`https://idp.example/`.

Proof of possession uses DPoP. JWTs are shown as decoded payloads; JOSE
headers, signatures, and client authentication are omitted. The handle
crosses a trust boundary only inside an ID-JAG or Identity Continuation
Assertion and travels within a domain only as derived chain context.

Participants are grouped by trust domain; all trust the IdP at
`https://idp.example/`. Each domain from which continuation occurs has three
logical roles: a Resource Authorization Server that binds the accepted hop, a
Transaction Token Service (TTS) that derives its chain context, and a Chain
Authority that attests continuation. A deployment may co-locate those roles.

* Expense domain (`expenses.example`): client `expense-app`, workload
  `expense-service`, and ExpenseRAS / Expense TTS / Expense CA, in front of
  ExpenseAPI.
* Travel domain (`travel.example`): workload `travel-service`, and
  TravelRAS / Travel TTS / Travel CA, in front of TravelAPI.
* Booking domain (`booking.example`): BookingRAS and BookingAPI only. It is
  terminal in this chain, an ordinary ID-JAG Resource Authorization Server
  that needs no continuation support ({{ras-processing}}).
* Outside the trust circle: PartnerSaaS (`partner.example`), reached in
  {{example-federation-edge}}.

The user has a pairwise subject at each RAS, which only the IdP can map.
Handles are H0 at ExpenseRAS, H1 at TravelRAS, and H2 at BookingRAS.

The root hop establishes the chain and the Expense domain's accepted
authorization:

~~~
 ExpenseApp        IdP          ExpenseRAS       ExpenseAPI/TTS
     |               |               |                 |
     |--ID Token---->|               |                 |
     |<-ID-JAG(H0)---|               |                 |
     |------------------ID-JAG------>|                 |
     |<-------------------AT1--------| bind H0         |
     |------------------request + AT1 + DPoP--------->|
     |               |               |<-resolve AT1----|
     |               |               |--bound H0------>|
     |               |               |    derive H0 into tctx
~~~

Each continuation repeats one exchange. ExpenseService obtains the Travel
grant before crossing the boundary:

~~~
 ExpenseService  Expense CA        IdP        TravelRAS TravelAPI/TTS
       |              |             |             |             |
       |-request H0-->|             |             |             |
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

### First Hop: Direct ID-JAG for ExpenseRAS {#example-first-hop}

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
requested_token_type=urn:ietf:params:oauth:token-type:id-jag
audience=https://ras.expenses.example/
resource=https://api.expenses.example/
scope=expenses.read
subject_token=<id_token>
subject_token_type=urn:ietf:params:oauth:token-type:id_token
actor_token=<sender-constrained expense-app credential>
actor_token_type=urn:ietf:params:oauth:token-type:jwt
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
time ({{validation}}, rule 14).

The IdP creates a fresh root hop, H0, for this chain and embeds it as a
claim of the ID-JAG it is about to issue ({{chain-id}}, rule 1); the hop is
PENDING until a Resource Authorization Server accepts it ({{ras-processing}}).
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
that ExpenseRAS rejected, is not usable: no Chain Authority attests a hop
that its Resource Authorization Server never accepted.

ExpenseRAS keeps this association in a private internal record. It is never
serialized into AT1, an external authorization claim, or anything that
ExpenseAPI's callers observe.

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
ExpenseApp nor ExpenseService supplies H0. The Transaction Token remains
inside `expenses.example` and is normally forwarded unchanged within that
domain. A replacement token requires the TTS to re-derive the member
({{transaction-token-context}}).

### Obtaining the Identity Continuation Assertion {#example-ica}

ExpenseService asks its own Chain Authority for an assertion covering H0.
Before issuing, Expense CA authenticates ExpenseService, verifies its key,
confirms that H0 belongs to the transaction that ExpenseService is serving,
and rechecks that ExpenseRAS's authorization remains active. The IdP's per-hop
map designates Expense CA to attest hops accepted by ExpenseRAS
({{assertion-issuance}}, {{root-establishment}}).

On the wire (decoded assertion):

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
  "jti": "b8Rn5Yx1Qe4Nk2Wf6zVc9d"
}
~~~

### Chained Exchange for the TravelRAS ID-JAG {#example-chained}

ExpenseService presents the assertion to the IdP as the `subject_token`,
DPoP-bound to its own key.

On the wire (request):

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

The IdP runs the checks of {{validation}}: the DPoP key matches both the
assertion's `cnf.jkt` and the actor token's key confirmation; `expense-service`
is the
actor named in `act`; H0 is CONTINUABLE; and the requested TravelRAS,
TravelAPI, and `trips.read` values match the Travel target entry in the
root-chain envelope. The IdP does not call ExpenseRAS to confirm acceptance.
Instead, the assertion from ExpenseSaaS's mapped Chain Authority,
`https://ca.expenses.example/`, is the evidence that H0 reached ACCEPTED state
and is CONTINUABLE ({{ras-processing}}).

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

### Third Hop: TravelService Continues to BookingRAS {#example-third-hop}

TravelService needs a reservation from BookingSaaS. Processing the request
whose Transaction Token carries H1, it obtains the same assertion shape as
{{example-ica}} from Travel CA, now naming H1, `travel-service`, and
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
({{root-establishment}}).

### Reaching a Target Outside the Trust Circle {#example-federation-edge}

Suppose TravelSaaS must also call PartnerSaaS at `https://partner.example/`,
whose Resource Authorization Server does not trust `idp.example`. The chain
cannot continue there: the IdP holds no pairwise subject for that audience
and no authorization basis covers it, so a continuation request for that
target fails ({{validation}}, rules 14 and 16; `invalid_target`). This is
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

* Platform domain (`platform.example`): workload `briefing-agent`, and
  PlatformRAS (the platform's own TaskRAS) / Platform TTS / Platform CA,
  in front of TaskAPI (`https://api.platform.example/tasks`);
  the Scheduler is an internal platform component, holding only the task
  identifier, that triggers each run.
* Calendar domain (`calendar.example`): CalendarRAS only, in front of
  CalendarAPI. It is terminal in every run, an ordinary ID-JAG Resource
  Authorization Server that needs no continuation support
  ({{ras-processing}}).
* Mail domain (`mail.example`): MailRAS in front of MailAPI, reached only
  in the dynamic-target scenario below ({{example-dynamic}}); likewise
  terminal.

The Scheduler stores only `task-123`. H0 remains bound to the PlatformRAS
task authorization across runs; each run receives a fresh child of H0 for
its terminal target.

### Setup (Alice Present)

Alice authorizes "summarize my calendar every morning." Because the task must
outlive her session, `briefing-agent` uses a refresh token from a
continuation-capable grant as the direct exchange's subject token. The chain
is therefore anchored to that grant, not Alice's current session
({{root-establishment}}, {{lifecycle}}). The root ID-JAG targets the
platform's TaskRAS; the envelope records both that root target and the
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
     |              |-task-123 + key->|
     |              |                 | verify key + task; derive H0
     |              |<-fresh TT(H0)---|
~~~

BriefingAgent then performs a fresh continuation to terminal CalendarRAS:

~~~
 BriefingAgent    Platform CA       IdP         CalendarRAS
       |               |             |               |
       |--request H0-->|             |               |
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
scheduler's trigger authenticates and carries only `task-123`; the Briefing
Agent then authenticates to the Platform TTS and proves possession of its key,
and the TTS, after confirming `task-123` is active and the Briefing Agent is
its designated actor, derives H0 into fresh intra-domain context
({{transaction-token-context}}). Neither the scheduler nor the agent selects H0.

Before issuing, Platform CA authenticates `briefing-agent`, verifies its key
and transaction, and rechecks that PlatformRAS's H0 authorization remains
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

{
  "error": "invalid_target"
}
~~~

For a deployment that expects dynamic targets, the envelope's basis is
Alice's standing consent as recorded when the chain was established (for
example, a productivity read-access grant) and tenant policy, with no
enumerated targets; the IdP evaluates each dynamic target against that
recorded basis at continuation time ({{validation}}, rule 14). A scope granted
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
  CA while the PlatformRAS authorization remains active.
* The ID-JAG, local task authorization, and IdP-held chain have distinct
  lifetimes ({{lifecycle}}).

This pattern requires a user-present setup event to root the chain. Where
no such event exists (for example, an administratively mandated agent
acting for users who never authorized it), there is no delegation to
continue and this profile does not apply; such deployments need a
differently rooted authorization, such as administrative policy at the
IdP, which is out of scope for this document.

## Gateway Example (Dynamic Upstream Audiences) {#example-gateway}

AgentApp knows the gateway audience but not the eventual upstream. The
gateway knows the upstream but holds no end-user assertion addressed to it.
This flow lets the gateway obtain an audience-specific grant without
weakening the original assertion's audience check.

* AgentPlatform domain (`agent.example`): client `agent-app` only, the
  confidential runtime that hosts Alice's session and roots the chain; it
  has no Resource Authorization Server of its own in this example.
* Gateway domain (`gateway.example`): workload `tool-gateway`, and
  GatewayRAS / Gateway TTS / Gateway CA, in front of the gateway's
  own tool-invocation surface (`resource=https://gateway.example/`),
  scoped under tenant `tenant-gw-01`.
* Wiki domain (`wiki.example`): WikiRAS only, in front of WikiAPI. It is
  terminal in this chain, an ordinary ID-JAG Resource Authorization
  Server that needs no continuation support ({{ras-processing}}).

Alice has pairwise subjects at GatewayRAS and WikiRAS, which only the IdP can
map. H0 is the root hop bound at GatewayRAS; H1 is the terminal Wiki hop.

The runtime roots the chain at the gateway:

~~~
 AgentApp          IdP          GatewayRAS       GatewayAPI/TTS
     |               |               |                 |
     |--ID Token---->|               |                 |
     |<-ID-JAG(H0)---|               |                 |
     |------------------ID-JAG------>|                 |
     |<--------------gateway AT------| bind H0         |
     |------------tool request + AT + DPoP----------->|
     |               |               |<-resolve AT-----|
     |               |               |--bound H0------>|
     |               |               |    derive H0 into tctx
~~~

After resolving the tool request to Wiki, the gateway continues the chain:

~~~
 ToolGateway       Gateway CA        IdP          WikiRAS/API
      |                 |             |                 |
      |--request H0---->|             |                 |
      |<-assertion------|             |                 |
      |------------------------------>|                 |
      |       assertion + DPoP        |                 |
      |<------------------------------| ID-JAG(H1)      |
      |------------------------------------------------>|
      |                ID-JAG to WikiRAS                |
      |<------------------------------------------------| wiki AT
      |--------------------call WikiAPI with AT-------->|
      |                 |             |        no binding (terminal)
~~~

### Root Exchange: The Runtime Roots the Chain

AgentApp performs a direct exchange for the one audience it knows: GatewayRAS.
The eventual upstreams are not known at root time, so, unlike the worked
example whose envelope enumerated each onward target, this envelope records an
authorization-basis ceiling, Alice's standing consent and tenant policy, with
no enumerated targets; enterprise policy permits `tool-gateway` to continue it
({{root-establishment}}, {{validation}}, rule 14). GatewayRAS accepts the
ID-JAG and binds H0 exactly as ExpenseRAS bound H0 in {{example-context}}.

AgentApp then invokes the gateway with its access token and no continuation
input. AgentApp can read H0 in its ID-JAG, but it cannot supply or select the
handle used for this call. Gateway TTS derives H0 from the authorization
that GatewayRAS bound to the presented access token
({{transaction-token-context}}).

### Chained Exchange: The Gateway Continues

Resolving the tool call, the gateway selects Wiki as the upstream, a target no
one enumerated when AgentApp rooted the chain. ToolGateway reads H0 from its
transaction context, obtains an assertion from Gateway CA, and presents it to
the IdP as in {{example-chained}}, now requesting
`audience=https://ras.wiki.example/`, `resource=https://api.wiki.example/`, and
`scope=wiki.read`.

Because the envelope enumerates no targets, the IdP evaluates this dynamically
chosen target against the recorded basis, Alice's standing consent and tenant
policy at establishment ({{validation}}, rule 14). Wiki read access is within
that basis and enterprise policy permits `tool-gateway` to reach it, so the
exchange succeeds and the IdP constructs the onward lineage with `tool-gateway`
atop `agent-app`. A target hint from the gateway informs issuance limits and
logging only; the IdP, not the gateway, decides whether a target is in the
envelope.

WikiRAS is terminal and redeems the resulting ID-JAG without binding H1. Each
permitted tool call repeats this exchange and creates a sibling hop under H0; a
target outside the basis fails with `invalid_target` as in {{example-dynamic}}.

### Points Worth Noticing

* AgentApp alone presents Alice's root credential (the ID Token); the gateway
  never holds or presents it.
* Gateway TTS, not AgentApp, selects H0 from GatewayRAS-bound state.
* The IdP evaluates every dynamically selected target against the root
  envelope and constructs the gateway's actor lineage.

# Open Items for Working Group Discussion {#open-items}

This non-normative appendix lists unresolved design questions.

\[\[ To be removed before publication as an RFC ]]

1. **Nested own-domain `act` segments.** Should a future version let a Chain
   Authority add verified own-domain actors to `act`, with the leaf outermost
   and the IdP deduplicating and depth-limiting the composed lineage? Or should
   offline-actor audit remain in the evidence layer
   ({{I-D.mcguinness-oauth-actor-receipts}},
   {{I-D.mcguinness-oauth-actor-proofs}})?

2. **Signed assertion versus a recipient-bound direct profile.**
   Could the IdP bind a continuation credential to an intended actor, actor
   class, trust domain, or key and accept it with client authentication,
   sender-constrained `actor_token`, and live key proof? Is the Chain
   Authority's actor/key attestation and domain-local gate worth its added
   trust configuration ({{rationale-grant-type}})?

3. **Pull topology.** Should target-side resolution be defined as a companion
   profile ({{rationale-pull}})?

4. **Mutual-TLS binding.** Should this profile and ID-JAG add mutual-TLS
   binding together ({{sender-constrained-presentation}})?

5. **A client establishment parameter.** Should a client be able to require
   or suppress chain establishment, or negotiate lifetime, depth, or
   permitted continuers ({{root-establishment}})?

6. **Other chain-context carriers.** Should this profile standardize an
   alternative to Transaction Tokens that derives the handle from RAS-bound
   state and is not requester-supplied or overridable? Actor-signed hop proofs
   are one candidate {{I-D.mcguinness-oauth-actor-proofs}}.

7. **Discovery.** Which accepted actor-token types, issuers, proof methods,
   confirmation methods, lifetime limits, endpoints, bindings, and error
   capabilities should IdP metadata advertise ({{metadata}})?

8. **Chain Authority issuance.** Should the document define an interoperable
   token-endpoint-style issuance request ({{assertion-issuance}})?

   ~~~
   POST /identity-continuation-assertion HTTP/1.1
   DPoP: <proof>

   identity_continuation_handle=<handle>
   audience=https://idp.example/
   ~~~

   The authenticated workload and proof key would determine `act` and `cnf`.
   The profile could also define errors, discovery, retry, and optional
   target/resource constraints enforced by the IdP as ceilings.

9. **Authorization-basis representation.** Should the envelope expose a
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

10. **A non-user root.** Should a sibling profile root continuation in
    tenant- or workload-scoped authorization while retaining this profile's
    envelope, revocation, and boundary-crossing model, or is that out of
    scope ({{decision-rule}})?

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
