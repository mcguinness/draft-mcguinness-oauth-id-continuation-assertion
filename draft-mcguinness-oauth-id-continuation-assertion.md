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
 - identity continuation
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
 -
    fullname: "Aaron Parecki"
    organization: "Okta"
    email: "aaron@parecki.com"

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
  I-D.parecki-oauth-jwt-dpop-grant:
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
sender-constrained JSON Web Token (JWT) used as an OAuth 2.0 Token Exchange
subject token. It enables a workload acting on a user's behalf to obtain an
Identity Assertion JWT Authorization Grant (ID-JAG) for another service when
it lacks a suitable credential, including when the user is no longer
present.

A trusted issuer attests that a resource authorization server accepted an
earlier ID-JAG and that the resulting authorization remains active and
eligible for continuation. The workload exchanges this assertion at the
identity provider, which evaluates the requested access under the chain
authorization and current policy before issuing an onward ID-JAG. The
profile supports multi-hop access across resource authorization servers that
trust a common identity provider.

--- middle

# Introduction

The Identity Assertion JWT Authorization Grant (ID-JAG)
{{I-D.ietf-oauth-identity-assertion-authz-grant}} allows an application to
exchange the user's identity assertion at the IdP Authorization Server (IdP)
for a grant that it redeems at a target Resource Authorization Server (RAS)
for an access token. A service receiving that access token may need to call
a further service on the user's behalf, including when the user is no longer
present. It may hold neither the user's identity assertion nor another
credential accepted by the next authorization server.

This profile addresses deployments in which RASes trust a common IdP. The
receiving workload's incoming access token is not accepted at the next RAS.
When pairwise subject identifiers are used, the workload may also be unable
to determine the user's subject at that RAS. This profile enables multi-hop
access when the request's path is not known in advance,
such as at a Model Context Protocol (MCP) tool gateway ({{example-gateway}}).

This document defines the Identity Continuation Assertion, a short-lived,
sender-constrained JSON Web Token (JWT) {{RFC7519}} that the workload obtains
from a Continuation Assertion Issuer (CAI). The IdP trusts the CAI to attest
the following ({{assertion-issuance}}):

* The RAS accepted the referenced ID-JAG, established an authorization from
  it, and that authorization remains active and is eligible for continuation.
* The authenticated workload is associated with that authorization context.

The CAI binds the assertion to the key the workload proves to it
({{assertion-token-exchange}}). The accepting RAS may also perform the CAI
role.

The workload presents the assertion to the IdP as the subject token of an
OAuth 2.0 Token Exchange {{RFC8693}} request. The assertion identifies the
accepted authorization and the current actor; the continuation request, not
the assertion, names the target and the requested authority. The IdP
authenticates the workload, verifies possession of the assertion's bound key,
evaluates the requested access, and resolves the user's subject for the target
RAS. If authorized, it issues an onward ID-JAG that the workload redeems at
that RAS ({{token-exchange}}). The assertion conveys no downstream authority.

Each ID-JAG issued in a chain represents a hop. A root ID-JAG and the hops
descending from it form a chain. An opaque continuation handle identifies
each ID-JAG's hop within a chain ({{chain-id}}). The IdP records the
relationships between hops and associates each chain with the authorization
established at its root exchange. Each continuation is
evaluated under that authorization and current policy
({{chain-authorization}}). RAS acceptance enables continuation; incoming
access-token scopes do not automatically limit authority at another target.

A workload that continues once can reuse the resulting access token for
further calls, subject to the conditions in {{implementation}}.

This document extends ID-JAG, referred to as the base profile, and
complements OAuth Identity Chaining {{I-D.ietf-oauth-identity-chaining}}.
This profile does not replace mechanisms for narrowing an existing token
within one trust domain ({{decision-rule}}).

The IdP, the continuing workload, a RAS from which workloads continue, and
the CAI implement this extension. Root clients use the base exchange;
chain establishment requires a resolvable session anchor or, optionally, a
grant anchor ({{lifecycle-anchors}}), and without one the IdP issues an
ordinary ID-JAG without a handle. A terminal RAS needs only the base
profile's support for redeeming a DPoP-bound ID-JAG ({{onward-id-jag}}).
This document defines no new access-token format.

## Protocol Overview {#protocol-overview}

AgentApp calls a tool gateway on Alice's behalf; the gateway then calls a
wiki with a separate RAS. GatewayRAS also performs the CAI role. The figure
marks additions to the base profile as "new"; {{example-gateway}} supplies
the requests, responses, and tokens.

~~~
AgentApp     IdP       GatewayRAS    ToolGateway         WikiRAS
  |           |             |             |                 |
  | (1) exchange ID Token for ID-JAG      |                 |
  |---------->|             |             |                 |
  | ID-JAG with handle H0  [new claim]    |                 |
  |<----------|             |             |                 |
  | (2) present ID-JAG (jwt-bearer grant) |                 |
  |------------------------>|             |                 |
  | access token; RAS binds H0 to it  [new]                 |
  |<------------------------|             |                 |
  | (3) call: access token                |                 |
  |-------------------------------------->|                 |
  |           |             |             |                 |
  |           |             | (4) exchange access token: assertion [new]
  |           |             |<------------|                 |
  |           |             | assertion: H0 accepted, active, eligible [new]
  |           |             |------------>|                 |
  |           | (5) exchange assertion for next ID-JAG  [new]
  |           |<--------------------------|                 |
  |           | ID-JAG for wiki, handle H1 (child of H0)    |
  |           |-------------------------->|                 |
  |           |             |             | (6) present ID-JAG
  |           |             |             |---------------->|
  |           |             |             | access token; base profile
  |           |             |             |<----------------|
~~~

1. AgentApp exchanges Alice's ID Token for an ID-JAG at the IdP. The IdP
   records root hop H0 and its chain authorization, and includes H0's handle
   in the grant ({{root-establishment}}, {{chain-id}}).
2. AgentApp redeems the ID-JAG at GatewayRAS, which binds H0 to the resulting
   authorization ({{ras-processing}}).
3. AgentApp calls ToolGateway with the access token.
4. ToolGateway exchanges the token at GatewayRAS for an assertion attesting
   H0's acceptance, activity, eligibility, and association with the gateway
   ({{assertion-issuance}}).
5. ToolGateway exchanges the assertion at the IdP using its own credential
   and DPoP proof. The IdP authorizes the request, resolves Alice's wiki
   subject, and issues child hop H1 ({{validation}}).
6. ToolGateway redeems the ID-JAG at WikiRAS. If WikiRAS implements this
   profile it binds H1, enabling further continuation; otherwise it ignores
   the handle and the chain ends there.

# Conventions and Definitions {#terms}

{::boilerplate bcp14-tagged}

This document uses the following terms, listed alphabetically:

Actor-lineage depth:
: The number of entries in the actor lineage the IdP derives from its own
  hop records, after consecutive entries for the same actor are merged and
  before any narrowing of what the onward `act` discloses
  ({{onward-id-jag}}). Tenant policy bounds it per branch
  ({{lifecycle-limits}}).

Chain:
: An IdP-held tree of hops under one chain authorization; each hop's
  parent reference gives the tree its shape ({{onward-id-jag}}), and the
  authorization bounds its lifetime ({{lifecycle}}).

Chain authorization:
: The tenant's authorization decision recorded by the IdP at the root exchange
  and associated with the chain's lifecycle anchor ({{lifecycle-anchors}}).
  It determines which actors may continue and what authority they may obtain,
  subject to current policy ({{chain-authorization}}).

Continuation Assertion Issuer (CAI):
: The role the IdP trusts to issue Identity Continuation Assertions for a
  tenant and the RAS whose hops it attests ({{assertion-issuance}}).

Continuation Handle (`identity_continuation_handle`):
: An opaque, unguessable, IdP-generated reference to one hop of a chain
  ({{chain-id}}).

Continuation-capable:
: Describes an ID-JAG that carries the `identity_continuation_handle` claim
  ({{chain-id}}).

Current actor:
: The workload presenting the assertion to the IdP, named by `act`. Its
  canonical actor identity is the (`iss`, `sub`) pair that its authentication
  to the IdP resolves to ({{client-identity}}).

Durable chain:
: A chain anchored to a refresh token's OAuth grant rather than to the user's
  IdP session, so that it can continue after logout ({{lifecycle-anchors}}).

Hop:
: One link of a chain: the IdP's record of an ID-JAG it issued, with an
  immutable reference to its parent hop unless it is the root
  ({{onward-id-jag}}). Its hop lineage is its path to the root; a hop from
  which no workload continues is terminal.

ID-JAG:
: An Identity Assertion JWT Authorization Grant
  {{I-D.ietf-oauth-identity-assertion-authz-grant}} issued for a target RAS.

Identity Continuation Assertion:
: A short-lived, sender-constrained JWT from a CAI, presented to the IdP as a
  Token Exchange `subject_token` to obtain an onward ID-JAG ({{assertion}}).

IdP Authorization Server (IdP):
: The authority that authenticates the user, determines the user's subject
  identifier for each target RAS, and issues onward grants.

Pairwise subject:
: A subject identifier specific to a RAS or group of RASes, allowing the same
  user to have different identifiers at different audiences.

Resource Authorization Server (RAS):
: An Authorization Server that protects a particular API, trusts the IdP for
  subject resolution, and exchanges an ID-JAG for an API access token.
  {{I-D.ietf-oauth-identity-assertion-authz-grant}} abbreviates this role (AS).

Tenant:
: The administrative boundary within which the chain and CAI trust are
  configured; its determination is deployment-defined, derived from
  authenticated material, never from requester-supplied input
  ({{issuer-trust}}).

Trust domain:
: An administrative and authentication boundary within which workloads can be
  directly authenticated, comparable to Workload Identity in Multi System
  Environments (WIMSE) {{I-D.ietf-wimse-arch}}. Its identifier is
  deployment-defined.

Workload:
: A service that received a request on a user's behalf and may continue it to
  a further service.

# The Identity Continuation Assertion {#assertion}

A CAI issues the assertion ({{assertion-issuance}}); the IdP validates it
({{validation}}).

## Token Type and Media Type {#names}

The Identity Continuation Assertion has token type
`urn:ietf:params:oauth:token-type:identity-continuation` and media type
`application/oauth-identity-continuation+jwt` ({{iana}}). It is a JWT
{{RFC7519}} in JWS Compact Serialization.

The CAI MUST set the JOSE `typ` header to `oauth-identity-continuation+jwt`.
The CAI MUST sign the assertion with an asymmetric algorithm the IdP accepts
({{security-alg}}). The assertion MUST NOT be encrypted (JWE) or use nested
signing.

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

  "iat": 1710000020,
  "exp": 1710000200,
  "jti": "b8Rn5Yx1Qe4Nk2Wf6zVc9d"
}
~~~

The claims have the following meanings and requirements:

`iss`:
: REQUIRED. The CAI that issued the assertion; the IdP verifies its issuer
  trust per the issuer-trust rule of {{validation}}, and its signature per the
  well-formedness rule.

`aud`:
: REQUIRED. A single string exactly matching the IdP issuer identifier: not
  its token endpoint URL, and not the requested target.

`identity_continuation_handle`:
: REQUIRED. The hop being continued ({{chain-id}}).

`act`:
: REQUIRED. The current actor presenting the Token Exchange request, encoded
  as a single-level `act` claim per {{RFC8693}}:

  * `iss` and `sub` are REQUIRED, non-empty strings: the actor's canonical
    actor identity as {{client-identity}} defines it. Only `iss` and `sub`
    form the canonical actor identity under this document.
  * Additional members MAY carry further information about the actor but
    MUST NOT affect processing defined by this document unless another
    specification profiles their use.
  * A recipient MUST ignore members it does not understand.
  * `exp`, `nbf`, `aud`, `scope`, `cnf`, and a nested `act` MUST NOT be
    present; the IdP rejects an assertion whose `act` carries one (the
    well-formedness rule of {{validation}}).
  * The IdP compares both `iss` and `sub` with the canonical actor identity
    of the authenticated client ({{client-identity}}).

`cnf`:
: REQUIRED. A confirmation claim {{RFC7800}} binding the assertion to a key
  the current actor proves. `cnf` MUST contain exactly one confirmation
  method: `jkt`, the JWK SHA-256 thumbprint {{RFC7638}} of the DPoP key
  {{RFC9449}} ({{security-pop}}).

`iat`, `exp`:
: REQUIRED. `exp` MUST follow `iat`. The assertion is short-lived: a CAI
  SHOULD NOT issue a lifetime (`exp - iat`) longer than 300 seconds, and an
  IdP SHOULD accept lifetimes of up to 300 seconds. The IdP rejects a lifetime
  longer than the maximum it accepts ({{validation}}).

`nbf`:
: OPTIONAL. If present, processed as {{RFC7519}} specifies.

`jti`:
: REQUIRED. A replay-detection identifier that MUST be unique per `iss`
  during the assertion validity window, with negligible probability of
  collision. It SHOULD contain at least 128 bits of entropy.

The assertion MUST NOT contain:

* a top-level `sub`, `auth_time`, `acr`, `amr`, or `sid` claim; or
* the Token Exchange request parameters `audience`, `resource`, `scope`,
  `authorization_details`, or `requested_token_type` (these are supplied by
  the request).

The assertion occupies the `subject_token` role of Token Exchange {{RFC8693}}:
it carries no user subject, and the IdP resolves the user from the referenced
hop. It is not an {{RFC7523}} JWT-profile assertion.

Other top-level claims MAY appear but MUST be ignored for validation,
authorization, and issuance.

# Continuation Handles (`identity_continuation_handle`) {#chain-id}

An `identity_continuation_handle` is an opaque, non-bearer reference to one
IdP-held hop of a chain. The IdP generates a fresh handle for each hop and
includes it in that hop's ID-JAG. Each continuation creates a child hop with
its own handle and a reference to its parent. Every onward ID-JAG includes a
handle because the IdP does not know whether the target will continue the
chain. {{privacy}} describes the resulting correlation exposure.

A handle is non-secret but security-sensitive correlation state: it confers no
authority by itself, yet a handle together with a CAI trust path and the
actor's credential is a larger compromise than the actor's credential alone
({{security-pop}}); {{handle-propagation}} limits where it travels.

The following rules apply:

1. When it establishes or continues a chain ({{root-establishment}}), the IdP
   MUST embed a fresh `identity_continuation_handle` claim in the issued
   ID-JAG, for that root or child hop, and MUST NOT reuse a handle across
   hops.

2. `identity_continuation_handle` MUST provide at least 128 bits of
   unpredictability, MUST NOT contain user-identifying information, and MUST
   consist of characters drawn from the base64url alphabet (`A`-`Z`, `a`-`z`,
   `0`-`9`, `-`, `_`); it SHOULD NOT exceed 256 characters.

3. Across a trust boundary, the handle is accepted only inside an ID-JAG (by
   the RAS) or an Identity Continuation Assertion (by the IdP), never as a
   standalone value. Inside the accepting RAS's domain, the CAI obtains it
   from RAS state or a carrier ({{handle-propagation}}); afterward it travels
   in the assertion.

4. A continuation-aware RAS binds the handle to the authorization state it
   establishes ({{ras-processing}}), and a hop is continuable only after that
   acceptance and binding ({{hop-activation}}). RASes, Resource Servers, and
   CAIs MUST NOT modify the value. The IdP MUST use the handle only to resolve
   hop state, subject, and policy, never as authority.

# Multi-Hop Cross-Domain Access {#access}

This section specifies processing in protocol order: root exchange, RAS
acceptance, handle propagation, assertion issuance, and continuation exchange
({{protocol-overview}}). The following table locates each role's requirements.

| Role | Requirements |
|---|---|
| IdP | Establishing a Chain ({{root-establishment}}), Continuation Exchange ({{token-exchange}}), Chain Lifetime and Revocation ({{lifecycle}}), IdP metadata ({{metadata-idp}}), Issuer Trust Configuration ({{issuer-trust}}) |
| Continuation-aware RAS | RAS Processing ({{ras-processing}}), Handle Carriers ({{handle-propagation}}), RAS metadata ({{metadata-ras}}) |
| CAI | the assertion it issues ({{names}}, {{assertion-claims}}), Assertion Issuance and its subsections ({{assertion-issuance}}), Handle Carriers ({{handle-propagation}}), Separate CAI ({{separate-cai}}) |
| Continuing workload | Assertion Issuance Request, Client Authentication, and Successful Response ({{assertion-token-exchange}}, {{assertion-client-auth}}, {{assertion-response}}); Continuation Request and Client Authentication ({{request}}, {{client-identity}}); Successful Response ({{success-response}}) and Error Response and Recovery ({{error-response}}) |

## Establishing a Chain {#root-establishment}

A chain begins when the IdP issues a continuation-capable ID-JAG on a root
exchange.

### Root Exchange Request {#root-request}

The root exchange and its ID-JAG conform to the base ID-JAG profile
({{I-D.ietf-oauth-identity-assertion-authz-grant}}) except where this document
extends it to issue a continuation-capable ID-JAG. The request presents a
subject token supported by the base profile, such as an ID Token, refresh
token, or SAML assertion:

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:id-jag
&audience=https://ras.gateway.example/
&resource=https://gateway.example/
&scope=tools.invoke
&subject_token=<id_token | refresh_token | SAML assertion>
&subject_token_type=<normal-subject-token-type>
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<JWT>
~~~

Processing an `actor_token` on the root exchange is outside the scope of both
this document and the base profile
({{I-D.ietf-oauth-identity-assertion-authz-grant}}, Section 4.3.3), and the
root actor is the authenticated client ({{root-actor}}).

### Chain Establishment {#chain-establishment}

The IdP MUST establish a chain when tenant policy permits continuation for
the root exchange and the root subject token resolves to a
lifecycle anchor ({{lifecycle-anchors}}). To
establish a chain, the IdP MUST include the root handle in the ID-JAG. Absent
permission to continue, the IdP MUST NOT establish a chain or include an
`identity_continuation_handle`.

Tenant policy may also restrict establishment to particular clients, grants,
or targets.

The lifecycle anchor is the user's active IdP session or, for a durable chain,
a refresh token's OAuth grant. When the subject token resolves to no anchor, the
IdP issues the ID-JAG without a handle, as under the base profile. No request
parameter requests chain establishment. Advertised support ({{metadata-idp}})
indicates capability and does not authorize establishment.

For every hop it creates, root or child, the IdP MUST record the RAS audience
placed in the ID-JAG; the issuers it trusts to attest that RAS's hops are read
from current tenant configuration at each exchange ({{issuer-trust}}).

Establishment is at-least-once. Retrying a lost response MAY create a second
chain, and the limits of {{lifecycle-limits}} apply across every chain rooted
in one chain authorization.

### Root Actor {#root-actor}

The root actor is the authenticated OAuth client, identified by the mapping
in {{client-identity}}. The base profile's recommendation to use a confidential
client applies to a root exchange that establishes a chain.

This document places no proof-of-possession requirement on the root exchange.
The root client's obligations are those of the base profile, and sender
constraint becomes a requirement for an actor that continues
({{client-identity}}).

### Chain Authorization {#chain-authorization}

The IdP MUST associate each chain with the chain authorization under
which it was established. The IdP records:

* the authenticated user and tenant;
* the root actor;
* the authentication context (`auth_time`, `acr`, `amr`);
* the lifecycle anchor; and
* restrictions on which actors may continue and what authority they may obtain.

The representation is implementation-specific. The association, root facts,
and restrictions remain fixed for the chain's lifetime. Later requests or
policy changes cannot replace the authorization, change those facts, or relax
those restrictions.

The IdP authorizes each continuation under the recorded chain
authorization and current policy ({{validation}}). Policy can restrict
access but cannot exceed that authorization. Policy evaluation is outside
the scope of this document.

The root request's audience and scope describe the root ID-JAG. They do not
by themselves authorize or limit later targets. RAS-local permissions and CAI
attestation do not independently authorize onward access.

## Continuation-Aware RAS Processing {#ras-processing}

A continuation-aware RAS implements this extension and advertises the
continuation grant profile ({{metadata-ras}}). A RAS that does not implement
it processes an ordinary ID-JAG and ignores the handle. An ID-JAG that such a
RAS accepts cannot become a continuation source.

On accepting a continuation-capable ID-JAG, a continuation-aware RAS MUST:

1. accept the ID-JAG per {{I-D.ietf-oauth-identity-assertion-authz-grant}};
2. bind `identity_continuation_handle`, the ID-JAG's issuer, the tenant
   where the deployment conveys one, and any confirmed key to the
   authorization state it establishes, and record whether the authorization
   is eligible for continuation under the RAS's own policy; and
3. when the ID-JAG carries `cnf`, as every onward ID-JAG does
   ({{onward-id-jag}}) and a root ID-JAG may, issue the access token bound to
   the confirmed key with `token_type` `DPoP` ({{RFC9449}}, Section 5), never
   as a bearer token.

How the RAS receives or determines the tenant is deployment-specific; this
document defines no tenant claim.

Under the base profile, the RAS validates the grant, authenticates the client,
applies local authorization policy, and issues an access token. An ID-JAG
carrying `cnf` uses the DPoP-bound JWT grant, which verifies its sender
constraint ({{onward-id-jag}}). When a root ID-JAG lacks
`cnf` ({{root-establishment}}), the RAS's own policy decides whether that
access token is sender-constrained.

Three rules govern binding the handle:

* The RAS MUST bind the handle and issue the access token as one outcome: no
  access token without its binding, and no binding without a token.
* All successful redemptions of one continuation-capable ID-JAG, identified
  by its validated issuer and handle and qualified by the tenant binding
  where the RAS records one, MUST resolve to the same hop binding, so that no
  redemption creates a distinct continuation source. A matching handle under
  another issuer is a different grant, not a retry.
* The RAS exposes the binding, its record linking the handle to authorization
  state, only within its trust domain; the handle itself travels in the access
  token or another carrier as {{handle-propagation}} describes.

### Hop Acceptance {#hop-activation}

Two parties hold facts about a hop, neither carried on the wire:

| Fact | Held by | Meaning |
|---|---|---|
| Issuance | IdP | The IdP issued the ID-JAG and recorded the hop. |
| Acceptance | RAS | The RAS accepted the ID-JAG and bound the handle to the resulting authorization. |

The IdP learns of acceptance through a trusted CAI's attestation and maintains
no synchronized acceptance state ({{protocol-overview}}). The CAI attests only
hops the RAS has accepted. A RAS acting as CAI attests its own hops; a separate
CAI confirms acceptance and activity according to the RAS's authorization
semantics ({{assertion-issuance}}).

An issued hop cannot be continued without a trusted CAI's attestation. Unless
the CAI is compromised ({{security-trust-model}}), no trusted CAI attests an
issued-but-rejected ID-JAG, so continuation fails closed.

A hop is continuable while a CAI trusted for its RAS can attest it as accepted
and still active, and neither it nor any ancestor is revoked; whether a
particular continuation from it succeeds is decided by the validation rules of
the continuation exchange ({{validation}}).

RAS acceptance and recorded eligibility ({{ras-processing}}) are prerequisites
for CAI issuance. The IdP authorizes downstream access
under the chain authorization and current policy ({{validation}}).

## Handle Carriers Within the Domain {#handle-propagation}

Each call includes a credential or context that identifies exactly one
RAS-bound authorization. The CAI MUST use the handle bound to that
authorization, whether it reads RAS
state directly or receives a carrier derived from it. A requester chooses
which credential to present, but cannot supply or override its handle
separately. A session or subject alone is not enough to select the
authorization: doing so could attach another user's handle to the call.

When the RAS also acts as CAI, it reads the handle from its authorization
state. It may also include the handle in its access token
({{example-gateway}}). A separate CAI receives the handle through a carrier
derived from the RAS binding ({{ras-processing}}). A Transaction Token
{{I-D.ietf-oauth-transaction-tokens}} is one such carrier, an optional
intra-domain choice and not a dependency of this profile. The carrier is
accepted only within that trust domain ({{assertion-issuance}}).

The source of the authorization context depends on the type of call:

* For an ingress call, the access token, after the resource verifies its
  proof of possession where the token is sender-constrained.
* For a downstream call, a carrier forwarded from that ingress.
* For a scheduled run, the task named by an authenticated actor, resolved to
  the task authorization for which that actor is designated. The CAI MUST
  derive a scheduled continuation from durable RAS task authorization, not
  from a scheduler-held handle ({{security-authorization}}). The scheduler holds
  only a task identifier; each authenticated run re-derives the handle from
  active task state and still requires an assertion from a trusted CAI.

The CAI checks acceptance freshness for every carrier
({{assertion-preconditions}}).

A Resource Server has no obligations under this document. A carrier SHOULD NOT
expose the handle to a party with no role in continuation. Deployments keep
this non-secret but security-sensitive correlation state ({{chain-id}}) out
of logs, traces, and responses.

## Assertion Issuance {#assertion-issuance}

The CAI issues the Identity Continuation Assertion that a workload presents to
continue a chain across a boundary. The CAI MUST set the assertion's `aud` to
the IdP recorded in the hop's RAS binding ({{ras-processing}}). The CAI
MUST NOT accept an IdP audience supplied by the requester.

The CAI attests three facts about its own domain:

* The RAS accepted the ID-JAG for the hop.
* The authorization the RAS established from it is active and, by the RAS's
  own authorization semantics, eligible for continuation.
* The authenticated workload is associated with that authorization context,
  in which it received, or was designated to process, the request that hop
  authorized.

Whether that actor may continue, and to what, is the IdP's decision under the
chain authorization and current policy ({{validation}}).

### Assertion Issuance Request {#assertion-token-exchange}

A CAI that is an OAuth authorization server, including a RAS acting as its own
CAI, MAY issue assertions from its token endpoint using Token Exchange
{{RFC8693}} as profiled in this and the following subsections. Such a RAS
SHOULD support this method of issuance, so that a workload in its domain has
one request to implement. Issuance by other means remains deployment-specific
({{handle-propagation}}).

A workload obtains assertions from the token endpoint of the CAI its
deployment designates. For a RAS acting as its own CAI, this is the RAS's
token endpoint, discoverable through the RAS's metadata ({{metadata-ras}}); a
separate CAI is configured within the trust domain, which this document leaves
to the deployment ({{handle-propagation}}).

The current actor acts as an OAuth client of the CAI. It makes a Token
Exchange request to the CAI's token endpoint with the following parameters:

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
parameters MUST NOT be included: targets and scope are chosen at the IdP
exchange ({{assertion-claims}}), and the authenticated client is the actor
named in `act`.

The following example carries the client's authentication and its DPoP proof
({{assertion-client-auth}}):

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

### Client Authentication {#assertion-client-auth}

The client MUST authenticate to the CAI's token endpoint ({{RFC6749}}, Section
2.3). The client MUST include in the request a DPoP proof {{RFC9449}} of the
client's own key. The CAI MUST verify that proof and bind the assertion to the
proven key in `cnf`. The CAI's verification is defense in depth: the actor
proves possession of that key again to the IdP at the continuation exchange,
which is the security boundary for continuation ({{client-identity}}).

The CAI MUST issue only for an actor it is authoritative to associate with the
RAS-accepted authorization, typically one in the RAS's trust domain; actor
authentication is deployment-specific.

### Request Validation {#assertion-preconditions}

The CAI MUST verify that the `subject_token` is of the type its
`subject_token_type` declares ({{assertion-token-exchange}}) and is one of the
following:

* an access token the accepting RAS issued, that RAS being one whose hops the
  CAI attests, unexpired, and valid for a protected resource that the
  authenticated client operates, as determined from the CAI's registration or
  configuration of that client; or
* a Transaction Token valid for the CAI's trust domain under
  {{I-D.ietf-oauth-transaction-tokens}}, Section 12.2, carrying the `typ`
  header and issuer that specification defines, and carrying the hop's handle
  as chain context ({{handle-propagation}}, {{separate-cai}}).

A token of another type presented as `subject_token`, such as an ID-JAG or an
Identity Continuation Assertion, is unacceptable ({{assertion-error-response}}).

With an access token, the client is a resource server exchanging a token it
received, the scenario of the example in {{RFC8693}}, Section 2.3. A RAS acting
as CAI resolves its own token; a separate CAI resolves it as {{separate-cai}}
describes. In this exchange the `subject_token` identifies the RAS
authorization context from which continuation is requested; its subject is not
copied into the resulting assertion.

Either `subject_token` type supplies the facts below. The CAI MUST
authenticate the actor and issue only after establishing these facts:

1. The handle came through an authenticated, confidential,
   integrity-protected channel or equivalent authenticated state.

2. The current actor controls the key placed in `cnf`.

3. `act` names that actor. If offline attenuation reached the actor, the
   attenuated credential it received, the `subject_token` or the carrier that
   conveyed it, is valid ({{decision-rule}}).

4. The handle is the one bound to the authorization context the actor
   presents, its access token or carrier, and the actor is a party that
   context was issued or forwarded to ({{handle-propagation}},
   {{hop-activation}}). Selecting another intact context is the actor's
   choice; substituting a handle within a context is not. The handle is input
   to verify, never authority in itself ({{chain-id}}).

5. Evidence that is authoritative by the RAS's own authorization semantics,
   whatever the carrier, confirms that the authorization remains active and
   that its binding still records it as eligible for continuation. That
   evidence is either a recheck of RAS authorization state or, where the RAS's
   authorization is a self-contained short-lived token the RAS itself issued,
   that token's validity ({{separate-cai}}).

The subject token's integrity protection and the authenticated request
establish fact 1; the DPoP proof, fact 2; client authentication, fact 3, whose
attenuation condition the CAI establishes by validating the attenuated
credential it received; and the bound handle and the RAS's acceptance evidence,
facts 4 and 5.

{{security-pop}} traces the binding chain these facts form from ID-JAG to
assertion.

A live recheck SHOULD be used where the tenant requires withdrawal of a hop's
authorization to stop fresh assertions before the RAS's token would expire.
With self-contained evidence, the CAI stops issuing when that token expires,
as for any OAuth access token. Where the evidence is a self-contained token,
the assertion's `exp` SHOULD NOT exceed that token's expiry, so that an
assertion is not presentable after the evidence that supported it has lapsed
({{lifecycle-ending}}).

A domain may add its own conditions for issuing, for example limiting which of
its workloads may obtain assertions, but such conditions narrow issuance only.
Target or purpose hints can narrow CAI issuance, but neither they nor
propagated context expand the authority the IdP may issue ({{validation}}).

### Successful Response {#assertion-response}

A successful response is a Token Exchange response ({{RFC8693}}, Section
2.2.1) in which `access_token` carries the Identity Continuation Assertion,
`issued_token_type` is
`urn:ietf:params:oauth:token-type:identity-continuation`, `token_type` is
`N_A` (not applicable), and `expires_in` reflects the assertion's lifetime.
The `access_token` member is the {{RFC8693}} response container; the assertion
is not an OAuth access token, which `token_type` `N_A` signals. This document
adds one parameter:

`identity_continuation_authorization_server`:
: REQUIRED. A JSON string containing the issuer identifier ({{RFC8414}}) of
  the authorization server at which the client exchanges the assertion for an
  ID-JAG. The CAI MUST set its value to the assertion's `aud` claim. The client
  uses this parameter to identify the IdP without decoding the assertion.

The client obtains that IdP's `token_endpoint` from its authorization server
metadata ({{RFC8414}}), retrieved with the `oauth-authorization-server`
well-known URI suffix under the issuer identifier. Before sending the assertion
or its own credentials there, the client MUST confirm that the returned `issuer`
exactly matches `identity_continuation_authorization_server`. Where the IdP
publishes no metadata, the client uses configuration bound to that issuer
identifier ({{metadata}}).

The client SHOULD present the assertion only to an IdP it is configured to
trust; this parameter identifies the destination but does not establish trust.

The CAI MUST NOT include a `refresh_token` in the response, which would let a
client obtain further assertions without presenting a token or passing the
acceptance check.

~~~
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "issued_token_type": "urn:ietf:params:oauth:token-type:identity-continuation",
  "access_token": "<Identity Continuation Assertion, compact JWS>",
  "token_type": "N_A",
  "identity_continuation_authorization_server": "https://idp.example/",
  "expires_in": 180
}
~~~

### Error Response {#assertion-error-response}

On failure, the CAI returns an error response according to {{RFC6749}},
Section 5.2, and {{RFC8693}}, Section 2.2.2. This document specifies the
following error mappings:

* `invalid_request` when the `subject_token` is invalid or unacceptable under
  policy, including when it is unknown, expired, revoked, not of the type its
  `subject_token_type` declares, not valid for a resource the client operates,
  has no bound handle, or names an authorization whose binding does not record
  it as eligible for continuation, or when the request includes a parameter
  this document prohibits ({{assertion-token-exchange}});
* `unauthorized_client` when the client is not permitted to use this grant
  type; and
* `invalid_dpop_proof` ({{RFC9449}}) for a failed proof.

DPoP nonce processing and the `use_dpop_nonce` error apply unchanged from
{{RFC9449}}.

### Separate CAI {#separate-cai}

A separate CAI MUST obtain the handle and evidence of RAS acceptance and
continued eligibility from a source authoritative for that RAS within its
domain ({{deployment-topologies}}):

* the RAS's authorization state;
* its introspection response {{RFC7662}}; or
* a self-contained short-lived token issued by the RAS.

For an access-token `subject_token` ({{assertion-token-exchange}}), the CAI
uses introspection or validates a self-contained token.

The handle alone does not convey the originating IdP, tenant, or eligibility
for continuation. The introspection response's `active` member reports token
activity, not continuation eligibility ({{RFC7662}}, Section 2.2). A separate
CAI relying on introspection therefore obtains the eligibility and the
originating IdP and tenant from deployment-defined evidence or configuration,
as the requirement above already demands.

With a Transaction Token, the handle arrives through the carrier of
{{handle-propagation}}. Validating that token alone does not establish RAS
acceptance or eligibility for continuation.

The IdP accepts a separate CAI's attestation only where it trusts that CAI to
attest the accepting RAS's hops, from tenant configuration
({{issuer-trust}}, the issuer-trust rule of {{validation}}).

## Continuation Exchange {#token-exchange}

A continuation exchange is an OAuth 2.0 Token Exchange request {{RFC8693}}
whose `subject_token` is an Identity Continuation Assertion. It uses the same
Token Exchange framework as the root exchange ({{root-establishment}}),
substituting the assertion for the root credential and adding the actor
authentication and DPoP proof described below.

Before a chain can continue to a target, the current actor needs a client
registration or resolvable client identity at that target's RAS
({{onward-id-jag}}).

### Continuation Request {#request}

A continuation exchange presents an Identity Continuation Assertion, a DPoP
proof of the `cnf` key, and the actor's client authentication
({{client-identity}}):

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
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<JWT>
~~~

The request carries no `actor_token` or `actor_token_type`: the actor named in
`act` is the authenticated client ({{client-identity}}), and the IdP rejects a
request carrying either ({{validation}}, {{error-response}}).

The continuation request, never the assertion, supplies the requested
`audience`, `resource`, `scope`, `requested_token_type`, and any
`authorization_details` {{RFC9396}} ({{assertion-claims}}). A request can carry
multiple `resource` indicators {{RFC8707}}. The authorization rule
({{validation}}) applies to `authorization_details` and to scope alike.

### Client Authentication {#client-identity}

The current actor MUST authenticate as an OAuth client with a credential that
resolves to its canonical actor identity. The canonical actor identity is the
(`iss`, `sub`) pair the IdP holds for that client.

The `iss` member identifies the actor's identity authority, which determines
issuer pairing ({{validation}}, {{security-trust-model}}) independently of the
credential parameter. If the client registration specifies no other canonical
actor identity, `iss` is the IdP's issuer identifier and `sub` is the
`client_id`. This applies, for example, to a client registered without a
workload identity that authenticates with a client secret or mutual-TLS
certificate.

The IdP MUST derive that pair from its registration of the client or its
configuration of the client's credential issuer, not from the client
authentication method, and MUST NOT accept a self-asserted mapping.

For {{RFC7523}} client authentication, the IdP MUST authorize the assertion's
issuer for the authenticated client. The canonical actor identity remains the
pair derived from the IdP's configuration ({{rationale-client-id}}).

This mapping applies to every exchange. Client authentication identifies the
root actor; this profile adds no proof-of-possession requirement to the root
exchange ({{root-establishment}}). On a continuation exchange ({{request}}),
the IdP MUST also match the canonical actor identity to the assertion's `act`.

The comparison runs between actor identities, never between a raw OAuth client
identifier and an `act` value:

~~~
authenticated OAuth client
        |  authoritative mapping (IdP registration)
        v
canonical actor identity (iss, sub)
        ^
        |  equal
       act
~~~

The IdP MUST compare the actor `iss` and `sub` as case-sensitive strings with
no transformation or canonicalization ({{RFC7519}}): the assertion's `act` is
compared with the canonical actor identity, and identities in different tenants
never compare equal.

The actor MUST prove possession of the key in `cnf`; for the `jkt` method, that
proof is a DPoP proof {{RFC9449}}. The IdP MUST bind the onward ID-JAG to a key
the actor proves in the request. This version defines only DPoP confirmation;
the target validates it using {{RFC9449}}. Support for mutual-TLS confirmation
{{RFC8705}} remains an open question ({{open-items}}).

A request contains one DPoP proof, so the assertion and resulting ID-JAG are
bound to the same key. Client authentication may use an independent
credential. Other confirmation methods could support different keys
({{open-items}}). Key rotation takes effect when the actor obtains an
assertion bound to the new key.

### Request Validation {#validation}

Where the IdP offers idempotent retry, a presentation whose (`iss`, `jti`)
matches an ISSUED reservation, as {{idempotent-retry}} defines, is processed
under that section; an IdP that does not offer retry rejects every second
presentation ({{validation-replay}}). For a first presentation, the IdP MUST
reject the request unless every rule below holds. {{error-response}} specifies
error precedence when multiple rules fail.

1. **Request parameters.**
   * exactly one each of `grant_type`, `subject_token`, `subject_token_type`,
     `requested_token_type`, and `audience`, and no `actor_token` or
     `actor_token_type` ({{RFC8693}}, Section 2.1);
   * zero or more `resource`, treated as an order-independent set, and at
     most one each of `scope` and `authorization_details`, all OPTIONAL and,
     when present, evaluated by the authorization rule; and
   * `grant_type` is `urn:ietf:params:oauth:grant-type:token-exchange`,
     `subject_token_type` is
     `urn:ietf:params:oauth:token-type:identity-continuation`, and
     `requested_token_type` is `urn:ietf:params:oauth:token-type:id-jag`;

2. **Assertion well-formedness.**
   * the assertion is a JWT whose JOSE `typ` header is
     `oauth-identity-continuation+jwt`;
   * it is a JWS in Compact Serialization, and is neither a JWE nor a nested
     JWT ({{names}});
   * it carries exactly one value for each claim required by
     {{assertion-claims}} and none of the claims that section forbids;
   * `iss`, `aud`, `identity_continuation_handle`, and `jti` are non-empty
     strings, `act` and `cnf` are JSON objects with `cnf` naming exactly one
     confirmation method, and `iat`, `exp`, and any `nbf` are NumericDate
     numbers;
   * `act` carries none of the members {{assertion-claims}} forbids (`exp`,
     `nbf`, `aud`, `scope`, `cnf`, and a nested `act`);
   * `aud` exactly matches the IdP's issuer identifier;
   * the signature validates with the issuer's resolved signing keys
     ({{metadata}});
   * `alg` is neither `none` nor a symmetric algorithm and is on the IdP's
     allowlist ({{security-alg}}, {{RFC8725}}); and
   * the signing key comes from trusted issuer configuration, not from a
     `jku`, `x5u`, or embedded `jwk` header, though `kid` MAY select among
     the configured keys;

3. **Issuer trust.**
   * the assertion `iss` is either the accepting RAS itself, identified by the
     ID-JAG `aud` recorded for the hop as a string or one-element array, or
     another issuer the IdP trusts to attest that RAS's hops, recorded from
     tenant configuration ({{issuer-trust}}); and
   * in either case, that issuer is trusted for the chain's tenant, recorded
     at establishment, and authorized to pair, for that tenant, with the
     actor's identity authority ({{client-identity}});

4. **Chain state.**
   * the handle identifies a hop the IdP issued, on an active chain, that the
     assertion attests as accepted: a valid assertion from a CAI the IdP
     trusts for that hop's RAS is itself that attestation; no claim in the
     assertion carries it ({{hop-activation}});
   * neither the presented hop nor any ancestor is revoked;
   * the actor lineage that results from merging consecutive same-actor
     entries, as the onward `act` will ({{onward-id-jag}}), is within its
     actor-lineage depth bound, which counts lineage entries, not hops; and
   * the continuation is within the fan-out, rate, and hop-count limits of
     the chain authorization ({{lifecycle-limits}});

5. **Current actor and binding.**
   * `act` is present, conforms to the schema of {{assertion-claims}}, and
     identifies the current actor, the canonical actor identity of the
     authenticated client ({{client-identity}});
   * the request proves possession of the `cnf` key with a matching DPoP
     proof ({{client-identity}}, {{RFC9449}}); the assertion is never
     accepted as a bearer token ({{RFC7800}}); and
   * the IdP can resolve, for the requested `audience`, both the user's subject
     identifier and the actor's client identifier ({{onward-id-jag}});

6. **Freshness and replay.**
   * `iat` is within the IdP's permitted clock skew, `exp` follows `iat`, the
     assertion is unexpired within that same skew, and any `nbf` has passed
     within it ({{RFC7519}}, Section 4.1.5). One skew value applies to `iat` as
     future skew, to `exp` as past skew, and to reservation retention
     ({{validation-replay}}); it SHOULD NOT exceed 60 seconds;
   * the assertion's lifetime does not exceed the maximum the IdP accepts
     ({{assertion-claims}}); and
   * `jti` is not yet reserved for the assertion issuer ({{validation-replay}});
     where the IdP offers idempotent retry, a presentation matching an ISSUED
     reservation is recovery ({{idempotent-retry}}), not a continuation
     exchange, and any other reserved `jti` is rejected;

7. **Authorization.** The IdP MUST issue an ID-JAG only if the chain
   authorization associated with the referenced hop ({{chain-authorization}})
   and current policy permit the authenticated actor to continue from that hop
   and permit the authority represented by the resulting ID-JAG. This
   evaluation includes the audience, resources, scopes, and authorization
   details, including any default scope or other defaults the IdP applies
   under its policy. An omitted `scope` uses a policy default or results in
   `invalid_scope` ({{RFC6749}}, Section 3.3). An omitted `resource` does not
   by itself require a default. The IdP MUST reject the
   request if it cannot establish that authorization.

   The issued ID-JAG carries the `scope`, `resource`, and
   `authorization_details` values that express the granted authority.
   Target or purpose hints reaching the CAI ({{assertion-preconditions}})
   MUST NOT control the IdP's target decision, and propagated context MUST NOT
   expand the authority permitted by the chain authorization and current
   policy. The IdP evaluates each authorization detail according to its type
   ({{RFC9396}}) and rejects a type whose authorization semantics it does not
   implement.

### Successful Response {#success-response}

The response to a continuation exchange follows the base ID-JAG profile: the
IdP returns the ID-JAG in `access_token`, with `token_type` `N_A` (not
applicable; {{RFC8693}}, Section 2.2.1). The IdP MUST NOT include a
`refresh_token`: a renewable credential would let the workload obtain further
grants without fresh CAI attestation, or root a new chain through the
refresh-token anchor, outside the hop's revocation dependencies.

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

On success, the IdP MUST record a child hop ({{hop-activation}}) of the
presented hop before returning an ID-JAG carrying the resolved target `sub`
and the child hop's fresh handle. An idempotent retry (the freshness rule;
{{idempotent-retry}}) instead returns the previously issued grant unchanged,
creating no new hop or handle.

The hop reference is delivered as the ID-JAG's `identity_continuation_handle`
claim ({{chain-id}}), a claim inside `access_token` and not a separate Token
Exchange response parameter; the accepting RAS binds it ({{ras-processing}}),
and the CAI reaches it through RAS state or intra-domain context
({{handle-propagation}}).

There is no chain-expiry response parameter: chain lifetime is authoritative
at the IdP ({{lifecycle}}), and a deployment needing advance warning conveys
it through task or authorization state, an optional ID-JAG claim, or a
management API.

### Onward ID-JAG Construction {#onward-id-jag}

The onward ID-JAG conforms to the base ID-JAG profile
({{I-D.ietf-oauth-identity-assertion-authz-grant}}) except where this document
extends it: its `sub` is the IdP-issued subject identifier for the target
audience, and `aud_sub` remains available under the base profile where the
target's native subject namespace differs.

Because the onward ID-JAG carries `cnf`, the actor redeems it with the
DPoP-bound JWT grant, `urn:ietf:params:oauth:grant-type:jwt-dpop`
({{I-D.parecki-oauth-jwt-dpop-grant}}), and a DPoP proof of the bound key, as
the base profile specifies for a key-bound ID-JAG
({{I-D.ietf-oauth-identity-assertion-authz-grant}}, Section 9.8.1.2.1). The
target RAS needs no continuation support to redeem this grant. A RAS that
does not implement this profile may issue a bearer access token under the
base profile. A continuation-aware RAS follows the binding requirements of
{{ras-processing}}, even if no further continuation occurs.

Where the recorded root authentication context contains `auth_time`, `acr`, or
`amr`, the IdP MUST include them in the onward ID-JAG unchanged. Continuation
MUST NOT extend or strengthen the authentication context, for example by
raising `acr` or adding `amr` beyond the user's root authentication.

The IdP constructs `act` as follows:

* It places the authenticated current actor atop the presented hop's lineage,
  whose origin is the authenticated client of the root exchange even if
  the root ID-JAG carries no `act` ({{root-actor}}); it never copies lineage
  from the assertion, and siblings do not contribute.
* A hop's parent reference is immutable. The IdP MUST derive lineage from
  that hop's ancestry to the root, excluding sibling branches. Storage and
  traversal methods are implementation-specific.
* Consecutive identical actors, those whose canonical actor identities are
  equal under the comparison rules of {{client-identity}}, both `iss` and
  `sub`, merge into one entry, though the hop record
  remains; policy MAY limit disclosed depth, narrowing what a target sees
  without changing the actor-lineage depth bound the IdP enforces (the
  chain-state rule of {{validation}}).
* A RAS MUST NOT read the absence of a further nested `act` as proof that no
  earlier actor exists, because policy may narrow what is disclosed: `act` is
  the disclosed actor lineage, not the authoritative history, which only the
  IdP's hop records hold.

The onward ID-JAG's `client_id` identifies the current actor's OAuth client
at the target RAS and may differ from its client identifier at the IdP.
If the IdP cannot resolve that target client identity, the request fails
with `invalid_target` ({{error-response}}).

The following is a non-normative example of the onward ID-JAG issued by the
IdP:

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

### Error Response and Recovery {#error-response}

On failure, the IdP returns an error response ({{RFC6749}}, Section 5.2;
{{RFC8693}}, Section 2.2.2). On a first presentation, when more than one rule
of {{validation}} fails, the IdP MUST return the code for the earliest
failure. Request-parameter and well-formedness failures (including signature
verification) come first. No chain-state code is returned before issuer trust
for the hop's RAS is established, nor before the client authentication and key
proof of the current-actor rule succeed, so no chain-state code reaches a
caller that has not authenticated as the current actor and proved the `cnf`
key. Among chain-state failures a permanently unusable hop precedes a limit.
The contents of an unverified assertion never determine the response, and a
missing hop is `invalid_request`, so a caller learns nothing about which
handles exist.

For recovery, client authentication, key proof, and fingerprint matching
precede chain-state checks ({{idempotent-retry}}).

On a first presentation or matching ISSUED recovery, the IdP MUST return
`invalid_continuation` ({{iana}}) when the handle identifies an issued hop
that is permanently unusable: its chain has expired or ended,
the hop or an ancestor is revoked, or the tenant has withdrawn the chain's
permission to continue ({{lifecycle-ending}}). The IdP MUST NOT return
`invalid_continuation` in any other case.

For other failures, the IdP MUST use the following error codes:

* `invalid_request`: a malformed, inconsistent, or unacceptable token, including
  an assertion that fails the well-formedness rule of {{validation}} (which
  covers signature verification) or the issuer-trust rule; an `act` that does
  not equal the authenticated client's canonical actor identity
  ({{client-identity}}); an unknown handle; a
  lifetime above the accepted maximum; prohibited `actor_token` or
  `actor_token_type` parameters; or a reserved assertion that cannot be
  processed as idempotent recovery ({{idempotent-retry}}), other than a
  matching ISSUED recovery whose hop is permanently unusable, which is
  `invalid_continuation`.
* `invalid_dpop_proof`: a DPoP failure.
* `unauthorized_client`: the chain authorization or current policy does
  not permit this actor to continue from the hop. Other actors may still
  continue; withdrawal of the chain's permission is `invalid_continuation`.
* `invalid_grant`: continuation would exceed actor-lineage depth, fan-out,
  hop-count, or rate limits ({{lifecycle-limits}}). A rate-limited request can
  be retried after the policy-defined window; other limit failures require a
  different request.
* `invalid_target`: the requested `audience` or a requested `resource` is not
  permitted by the chain authorization or current policy, or the IdP cannot
  resolve the user's subject identifier or the actor's client identifier at
  the target.
* `invalid_scope`: a requested scope is not permitted by the chain
  authorization or current policy, or `scope` is omitted and no policy
  default exists ({{RFC6749}}, Section 3.3).
* `invalid_authorization_details`: a requested authorization detail is not
  permitted by the chain authorization or current policy, or its type is one
  the IdP does not implement.

DPoP nonce processing and the `use_dpop_nonce` error apply unchanged from
{{RFC9449}}.

Recovery from `invalid_continuation` requires a new root exchange, authorized
under current policy and resolving to an active anchor
({{root-establishment}}). Establishing a new chain does not reactivate the old
chain's handles.

Other errors apply to the request and do not establish that the chain is
permanently unusable. An actor-lineage depth rejection
concerns the particular continuation whose resulting lineage would exceed the
bound, not the hop itself; a continuation that merges into an existing lineage
entry, or a later policy raising the bound, may still succeed.

After a lost response, the client MAY retry the same assertion where the IdP
offers idempotent retry ({{idempotent-retry}}), or obtain a fresh assertion.
Expired acceptance evidence can prevent fresh assertion issuance; shared
fan-out or hop-count limits can prevent a subsequent exchange
({{lifecycle-limits}}). A fresh assertion may create an equivalent grant and
sibling hop but no additional authority.

A client cannot distinguish on the wire a presentation whose request does not
match its reservation's fingerprint from one whose reservation is RESERVED or
FAILED: all three return `invalid_request` ({{idempotent-retry}}). The client
can retry the original request within the retry window, since pending issuance
may complete, or obtain a fresh assertion. Other `invalid_request` failures,
such as malformed requests or prohibited parameters, require correcting or
abandoning the request.

### Replay Reservation and Retry {#validation-replay}

The IdP MUST issue at most one grant per assertion, including under concurrent
presentations: it reserves the assertion's (`iss`, `jti`) as an atomic
first-writer decision once validation succeeds and before it issues the grant,
and MUST retain the reservation through `exp` plus the permitted clock skew
(one value, per the freshness rule of {{validation}}). Uniqueness is keyed on
(`iss`, `jti`), since partitioning by tenant alone would let two assertion
issuers in one tenant collide on a reused `jti`. The reservation MUST be
visible to every IdP instance that accepts assertions for that issuer, and an
instance that cannot reach that shared state MUST reject the request rather
than issue. Without idempotent retry this needs only the set of (`iss`, `jti`)
values presented within that window.

A request that fails validation creates no reservation and does not modify
any existing reservation.

The IdP MUST reject a second presentation of a reserved assertion unless it
offers idempotent retry.

A consumed assertion is not equivalent to a fresh one. With live acceptance
evidence, the CAI refuses a fresh assertion once the hop's authorization has
lapsed ({{assertion-preconditions}}); with self-contained evidence, issuance
continues until the RAS token expires, and single-use still keeps each consumed
assertion from authorizing more than its one grant within that tail
({{security-pop}}). Single-use therefore bounds staleness and multiplicity; it
does not itself establish that the hop was accepted.

#### Idempotent Retry {#idempotent-retry}

An IdP MAY offer idempotent retry by binding the reservation to a fingerprint
of the request first authorized and recording the reservation as RESERVED,
ISSUED, or FAILED (distinct from the hop facts of {{hop-activation}}). These
names describe the states an IdP distinguishes, not a required representation.
The IdP MUST reject a presentation whose (`iss`, `jti`) matches a reservation
but whose request does not match that reservation's fingerprint. A presentation
matching the fingerprint of an ISSUED reservation within the IdP's retry window
is processed as recovery, defined below. The fingerprint MUST cover:

* `audience` as an exact string;
* the `resource` values as an order-independent set;
* `scope` as an order-independent set;
* the exact `authorization_details` JSON after form decoding (a different
  serialization is a different request);
* the actor's `iss` and `sub`;
* the confirmation key in `cnf` (its `jkt` thumbprint in this version); and
* a SHA-256 hash of the exact `subject_token` after form decoding, which binds
  the fingerprint to the specific assertion and its handle.

An IdP that offers idempotent retry MUST retain the ISSUED reservation, its
fingerprint, and its result for a retry window it defines, alongside the
reservation retention of {{validation-replay}}; recovery is available only
within that window and only while the issued grant is unexpired. The window is a
deployment choice and MAY be advertised or documented.

For recovery, the IdP MUST verify:

* the requester's client authentication and canonical actor identity;
* the DPoP proof of the `cnf` key;
* the request fingerprint; and
* that the presented hop and every ancestor remain unrevoked and the chain
  has not ended, including through anchor expiry or withdrawal of the chain's
  permission to continue ({{lifecycle-ending}}).

If these checks succeed and the issued grant is unexpired, the IdP MUST return
that grant unchanged. Otherwise, it MUST reject the request under
{{error-response}}. The IdP never extends or reissues the grant; after its
`exp`, the client obtains a fresh assertion.

Recovery applies no other validation or policy checks. Changes to actor or
target authorization, or to issuer trust, do not prevent recovery of an
already-issued grant. Recovery creates no hop and consumes no fan-out,
hop-count, or rate budget. Throttling repeated requests is a deployment choice.

A presentation matching a RESERVED reservation, whose first presentation has
not completed, MUST be rejected with `invalid_request`; the client can retry
it once that first presentation completes ({{error-response}}). A reservation
that does not reach ISSUED before `exp` plus the permitted clock skew becomes
FAILED, which is final: a presentation matching a FAILED reservation MUST be
rejected with `invalid_request`, and the client obtains a fresh assertion.

Application idempotency is outside the scope of this document.
{{implementation}} provides implementation guidance; discovery of optional
recovery support remains an open question ({{open-items}}).

# Chain Lifetime and Revocation {#lifecycle}

This profile supports session-anchored chains and, optionally, durable chains
anchored to a refresh token's grant. A session-anchored chain ends with the
session; a grant-anchored chain can support an unattended agent after logout
({{example-background}}). The ending rules and limits apply to both forms.
A reader implementing session anchors only can skip the grant-specific
processing and background example.

A chain is continuable only while active at the IdP. Each continuation is a
fresh policy check, and revoking a hop stops its descendants at the next
continuation, fail-closed; an offline-attenuated token, by contrast, stays
usable for its lifetime without contacting an authority.

Revocation does not invalidate an already-issued ID-JAG, so the revocation
window is the ID-JAG's remaining redemption window plus the lifetime of the
access token a late redemption obtains; any refresh token the RAS issues,
which the base profile recommends against, extends it further.

Three independent lifetimes govern a continuation:

* the ID-JAG's short redemption window;
* the access-token lifetime the accepting RAS sets, which this profile does
  not constrain; and
* the IdP-held continuation chain.

Revoking the chain does not shorten an already-issued access token, and an
access token outliving the chain does not extend it.

~~~
ID-JAG redeem   |==|
access token    |===========|              RAS-set, independent
IdP-held chain  |=========================| IdP-held, spans hops
~~~

## Anchors {#lifecycle-anchors}

The root subject token resolves to one of these anchors
({{root-establishment}}):

* an ID Token `sid` {{OIDC.FrontChannelLogout}} resolving to an active IdP
  session for that user and client;
* a SAML `SessionIndex` {{SAML2.Core}} resolving to an active IdP session
  for that user and client; or
* for a durable chain, a refresh token's OAuth grant.

The IdP MUST NOT root a chain from an unresolved anchor or an access token;
non-user-rooted authority is out of scope. `sid` and `SessionIndex` are used
only for resolution and MUST NOT enter assertions or chain context.

Support for grant anchors is OPTIONAL; a session-anchored implementation is
complete. Without it, base processing of a
refresh-token subject is unchanged; any otherwise-authorized ID-JAG omits the
continuation handle ({{chain-establishment}}). Rotation of a refresh token
does not affect the grant anchor.

## Ending a Chain {#lifecycle-ending}

A chain ends when:

* the session it is anchored to terminates;
* the grant it is anchored to expires or is revoked; or
* tenant policy withdraws permission to continue it.

A chain MUST NOT outlive its anchor ({{lifecycle-anchors}}): a session-anchored
chain ends with the session, and only a grant-anchored chain may outlive
logout. An authorization that a session produced but that has its own
lifecycle, such as a refresh token's grant, is a grant anchor, so logout ends
a chain only when the session itself is the anchor. Ending a chain this way
bounds only new continuations; an ID-JAG already issued remains redeemable for
its own lifetime, since redemption is not a continuation.

The IdP has these duties over chain lifetime:

* it MUST bound chain lifetime by the chain authorization;
* it MUST support administrative revocation of an entire chain and MAY
  revoke an individual hop's subtree; and
* it MUST reject continuation on a revoked, expired, or ended chain.

The IdP ends a chain when it observes the withdrawal, whether through a
policy event or at the next continuation attempt, and an ended chain stays
ended: restoring the policy that withdrew permission does not revive it, its
handles remain permanently unusable ({{error-response}}), and a new chain is
needed.

RAS-local withdrawal stops fresh assertions once the CAI observes it;
self-contained acceptance evidence may remain usable until its expiry
({{assertion-preconditions}}). Previously issued assertions remain subject to
the IdP's current chain, trust, and authorization checks ({{validation}}).
RAS-local withdrawal does not revoke descendants accepted at other RASes;
stopping their continuation requires IdP revocation of the chain or affected
subtree.

How an IdP surfaces chains to users and administrators for review and
revocation is deployment-specific; {{GRANT-MGMT}} describes OAuth grant
management for that purpose.

## Limits {#lifecycle-limits}

Fan-out is the number of child hops continued from one hop and is counted
per hop. Hop count is the total number of hops in a chain across all
branches. Rate is the number of newly issued continuations, not attempts,
within a window tenant policy defines. Hop count and rate are counted per
chain and, where the tenant configures one budget for a chain
authorization, aggregated across every chain rooted in it, so sibling chains
share that budget; a retried establishment ({{root-establishment}}) MUST NOT
evade it. Revocation of the chain authorization applies to every chain
rooted in it. The actor-lineage depth bound, set by tenant policy, is
enforced per branch.

The IdP MUST enforce a finite hop-count limit on every chain, either the
tenant's configured value or the IdP's default, so that a workload
continuing as itself cannot extend a chain without bound. Fan-out and rate
limits remain optional tenant controls. Whether the profile should fix a
default is open ({{open-items}}).

# Authorization Server Metadata and Trust Configuration {#metadata}

An IdP and a Resource Authorization Server advertise support in their
metadata; the IdP configures the CAIs and actor identity authorities it trusts.

## IdP Authorization Server Metadata {#metadata-idp}

An IdP that supports this profile SHOULD signal it in its authorization server
metadata {{RFC8414}} with the following parameter:

`identity_continuation_supported`:
: OPTIONAL. Boolean, default `false`, indicating that the IdP accepts the
  `urn:ietf:params:oauth:token-type:identity-continuation` subject token type
  and issues continuation-capable ID-JAGs. Such an ID-JAG is still the
  `urn:ietf:params:oauth:token-type:id-jag` type; an IdP that sets this flag
  also lists that type in `identity_chaining_requested_token_types_supported`
  ({{I-D.ietf-oauth-identity-chaining}}). This flag adds only the continuation
  capability.

## Resource Authorization Server Metadata {#metadata-ras}

A Resource Authorization Server that binds the
`identity_continuation_handle` claim to authorization state MUST advertise
support by listing the grant profile
`urn:ietf:params:oauth:grant-profile:id-jag-continuation` in
its `authorization_grant_profiles_supported`
{{I-D.ietf-oauth-identity-assertion-authz-grant}}. This value indicates that the
server recognizes continuation-capable ID-JAGs and performs that binding
({{ras-processing}}). The base ID-JAG grant profile indicates ordinary grant
processing without handle binding.

A Resource Authorization Server that advertises
`urn:ietf:params:oauth:grant-profile:id-jag-continuation` MUST also advertise
the base `urn:ietf:params:oauth:grant-profile:id-jag` profile and the
`urn:ietf:params:oauth:grant-type:jwt-bearer` grant type on which ID-JAG
depends ({{I-D.ietf-oauth-identity-assertion-authz-grant}}). It MUST also
advertise `urn:ietf:params:oauth:grant-type:jwt-dpop`, since every onward
ID-JAG it redeems carries `cnf` ({{onward-id-jag}}).

## Issuer Trust Configuration {#issuer-trust}

The IdP MUST authorize CAI and actor identity authority pairings per tenant;
separate trust in each is insufficient ({{security-trust-model}}). The IdP
MUST scope CAI trust by issuer, keys, tenant, and the RAS it attests for.
Tenant determination MUST derive from authenticated material, not
requester-supplied input.

When a CAI's issuer identifier is that of an OAuth authorization server, the
IdP obtains its signing keys from the `jwks_uri` in that server's metadata
({{RFC8414}}); a CAI without such a `jwks_uri`, like any other CAI, uses
authenticated configuration. The IdP MUST refresh remotely obtained keys
under a bounded cache policy, so a key removed from the JWK Set stops
validating once the refresh takes effect.

The IdP evaluates issuer trust and keys against its current trusted issuer
and key state, so removing an issuer or revoking its keys de-authorizes it
for existing chains: continuation from such a CAI's assertions is refused at
the next continuation exchange (the issuer-trust rule of {{validation}}).

CAI trust is provisioned and withdrawn through deployment-specific
configuration at the IdP.

# Implementation Considerations {#implementation}

This non-normative section provides implementation guidance. Conformance
depends on the normative sections.

A workload continues once per authorization context and target, then reuses
the resulting access token while valid and sufficient for the requested
access. Reuse requires the same user, tenant, actor, key, and source hop;
a sibling branch has different lineage and revocation dependencies. Renewal
requires a fresh assertion satisfying {{assertion-preconditions}}.

Each continuation depends on IdP availability and, for a separate CAI,
access to RAS acceptance evidence. Offline attenuation can avoid those
exchanges where existing subject and issuer trust suffice ({{decision-rule}}).

The IdP retains hop records for the chain's lifetime and replay state for
the periods in {{validation-replay}} and {{idempotent-retry}}. Ancestry caches
and audit indexes can supplement immutable parent records without changing
lineage or revocation checks. Handles can be derived with a keyed one-way
function if they satisfy {{chain-id}} and remain unlinkable.

A RAS can couple handle binding and token issuance with a local transaction
or compensate for failed binding by revoking the token. CAI audit records
cover issuance and limits; the IdP correlates the chain, while each RAS logs
its local subject. Retries and same-actor continuation remain subject to
{{validation-replay}} and {{lifecycle-limits}}.

Failure paths worth testing:

* Concurrent redemption of one ID-JAG.
* A crash between issuing an ID-JAG and recording its hop.
* A policy change between continuations.
* A credential from the wrong tenant.
* Local revocation after assertion issuance.

An IdP can defer materializing chain state until first continuation if the
handle still resolves to the same root and chain authorization and replay
rules remain satisfied. Replacing the hop tree with self-verifying handles
remains an open question ({{open-items}}).

## Deployment Topologies {#deployment-topologies}

The topologies differ in who performs the CAI role and obtains hop state:

| Topology | CAI role held by | Source | Fits when |
|---|---|---|---|
| Co-located | the accepting RAS | RAS state | one operator runs the domain |
| Separate | a separate CAI the IdP trusts for the RAS | domain carrier | the RAS is shared infrastructure, the gateway is only a Resource Server, or keys and audit need isolation |

Both use the same assertion, issuance requirements, and number of exchanges.
Co-location reduces configuration; separation requires a carrier and access
to acceptance evidence. IdP trust and RAS advertisement follow {{issuer-trust}}
and {{metadata-ras}}.

Carrier choices follow {{handle-propagation}}:

* A Transaction Token carries the handle as request context.
* A signed JWT access token {{RFC9068}} carries issuance-time state; the
  CAI's acceptance check addresses subsequent changes.
* An opaque token's introspection response {{RFC7662}} can supply the handle
  while `active` is `true`.

{{example-gateway}} uses the RAS's access token; {{example}} uses a separate
CAI and Transaction Tokens.

# Security Considerations {#security}

This profile assumes TLS, correct IdP subject mapping and authorization,
and the OAuth guidance of {{RFC9700}}. The following sections address token
capture, compromised workloads and authorities, forged lineage, and token
confusion. {{privacy}} covers correlation and disclosure.

## Sender Constraint and Proof of Possession {#security-pop}

Using a captured assertion requires both authentication as its `act` actor
and DPoP proof of its `cnf` key ({{client-identity}}, {{validation}}). The
onward ID-JAG and a continuation-aware RAS's access token bind to that key.
A RAS that does not implement continuation follows the base profile's token
binding rules ({{onward-id-jag}}).

For a sender-constrained incoming grant, synchronous continuation follows
this binding sequence:

1. The redeemer proves the ID-JAG's key at the RAS, which binds the handle to
   authorization state and its access token to the key ({{ras-processing}}).
2. The receiving workload verifies the caller's DPoP proof as an {{RFC9449}}
   resource server.
3. The workload authenticates to the CAI and proves its own key, which the
   CAI places in the assertion's `cnf` ({{assertion-preconditions}}).

For forwarded context, Transaction Tokens, and scheduled tasks, fact 4 of
{{assertion-preconditions}} replaces the call-boundary check. The workload
proves its own key, not the incoming token's key: the access token belongs
to the caller and the assertion to the callee. The CAI does not compare those
keys ({{assertion-token-exchange}}).

This profile adds no proof-of-possession requirement at the root. A root RAS
may issue a bearer access token, whose theft lets an attacker induce an
honest workload to
continue under its own key. Workload sender constraint does not prevent that
use. Although the attacker weakens only the caller's link, the consequences
can reach any downstream target permitted by the chain authorization and
current policy; the root token's resource does not bound onward authority.
Sender constraint at ingress prevents use without the bound key.

Assertion lifetime bounds the replay window, and single-use confines an
assertion to one issued grant ({{validation-replay}}). Otherwise a workload
whose RAS-local authorization had lapsed could keep obtaining grants from an
unexpired assertion after the CAI would refuse a fresh one. Optional recovery
returns the same grant and does not reopen issuance.

## Authorization Enforcement {#security-authorization}

The IdP evaluates the chain's own authorization, even if the user or actor
could obtain access under another ({{chain-authorization}}). RAS acceptance
and CAI attestation supply no onward authority. The CAI checks any offline
attenuation segment ({{assertion-preconditions}}).

Broad permissions increase the impact of a compromised workload. Policy
changes affect running chains subject to recorded restrictions:

* A fixed target restriction prevents general tenant policy from admitting
  a later-added target.
* Where targets are left to current policy, adding one widens access for
  every active chain governed by that authorization.
* Adding a permitted continuer likewise admits an actor to those chains,
  subject to recorded actor restrictions and {{validation}}.

A substituted handle could continue the wrong user's chain. The CAI's
RAS-bound evidence associates the authorization with the actor; possession of
an unrelated handle cannot bypass that check ({{handle-propagation}},
{{assertion-preconditions}}). Scheduled continuation likewise derives from
RAS task state; a scheduler-held handle would create a durable bearer-like
credential outside that binding.

Copying root authentication context unchanged prevents an actor from raising
`acr` or `amr` to bypass a target's step-up requirements ({{onward-id-jag}}).
Targets evaluating those claims should also evaluate `auth_time`: a
grant-anchored chain can retain old authentication context while the user
is absent ({{lifecycle-anchors}}).

## Trust in Actor Identity Authorities {#security-actor-issuers}

A rogue or over-scoped identity authority could impersonate an actor in
another domain or tenant. The IdP obtains the authority from trusted client
or credential-issuer configuration and checks its tenant-specific pairing
with the CAI ({{client-identity}}, {{validation}}). CAI attestation does not
substitute for actor authentication.

## Conjunctive Trust and Issuer Pairing {#security-trust-model}

Continuation requires all four independent elements ({{validation}}):

* a CAI trusted to attest the accepting RAS's hops;
* an identity authority trusted for the actor's domain and tenant;
* proof of possession of the confirmed key; and
* permission under the chain authorization and current policy.

The IdP configures the issuer pairings as {{issuer-trust}} specifies.

## Topology and Trust {#security-topology}

Using the RAS's own issuer identifier does not establish trust in its keys,
tenant, or issuer pairings. A separate CAI isolates keys and components but
adds no quorum: the IdP receives one attestation ({{security-trust-model}}).

Exchanging a token already held by the workload adds the CAI's attestation,
subject to policy and acceptance checks. A compromised RAS can fabricate
acceptance state in either topology; a compromised separate CAI can also
attest a hop the RAS refused. Chain authorization and current policy still
bound the result. {{lifecycle-ending}} covers RAS-local withdrawal and delayed
observation by a separate CAI.

## Actor Chain Integrity {#security-actor-chain}

An actor could try to hide itself, impersonate a prior actor, or fabricate
a delegation. The IdP prevents this by matching the assertion's sole actor
to the authenticated client and deriving lineage from its own hop records
({{validation}}, {{onward-id-jag}}).

The disclosed `act` may be minimized and excludes offline segments the IdP
does not observe. It therefore cannot support a rule such as "deny if an
actor ever participated"; the IdP's hop records are authoritative.

## Token, Type, and Algorithm Confusion {#security-alg}

The well-formedness rule of {{validation}} prevents token-type substitution,
signature-algorithm downgrade, and attacker-directed key selection.

# Privacy Considerations {#privacy}

A handle is visible to the ID-JAG client, accepting RAS, CAI, continuing
workload, and IdP. Propagating it exposes it to carrier recipients, including
the audiences of an access token carrying it ({{handle-propagation}}).

Opaque, hop-specific handles provide no common cross-RAS user identifier,
but the chain remains correlatable: the IdP sees the chain, shared handles
link hop participants, and matching actor lineage and timing can link
transactions across audiences. The onward `act` also directly identifies
prior actors to the RAS; policy can limit disclosed depth ({{onward-id-jag}}).
For example, grants issued close together with matching actor lineage may
reveal a shared user transaction even when their handles differ.

# IANA Considerations {#iana}

Note: The token type URI `urn:ietf:params:oauth:token-type:id-jag` referenced by
this document is registered by
{{I-D.ietf-oauth-identity-assertion-authz-grant}} and is not registered here.

Note: The `authorization_grant_profiles_supported` metadata parameter and the
base `urn:ietf:params:oauth:grant-profile:id-jag` value referenced by this
document are defined and registered by
{{I-D.ietf-oauth-identity-assertion-authz-grant}} and are not registered here;
this document registers only the
`urn:ietf:params:oauth:grant-profile:id-jag-continuation` value.

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

IANA is requested to register the following parameter in the "OAuth
Parameters" registry established by {{RFC6749}}.

Parameter name:
: identity_continuation_authorization_server

Parameter usage location:
: token response

Change controller:
: IETF

Specification Document(s):
: This document, {{assertion-response}}

## OAuth URI Registration

IANA is requested to register the following value in the "OAuth URI" registry
established by {{RFC6755}} and used for token type identifiers by {{RFC8693}}.

URN:
: urn:ietf:params:oauth:token-type:identity-continuation

Common Name:
: Token type URI for the Identity Continuation Assertion

Change Controller:
: IETF

Specification Document(s):
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

Specification Document(s):
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
: Karl McGuinness, Aaron Parecki

Change controller:
: IETF

## JSON Web Token Claims Registration

IANA is requested to register the following claim in the "JSON Web Token Claims"
registry established by {{RFC7519}}.

Claim Name:
: identity_continuation_handle

Claim Description:
: An opaque, IdP-generated reference to one hop of a
  continuation chain, used to resolve the referenced hop and its chain state.
  This claim appears in an Identity Continuation Assertion and in a
  continuation-capable ID-JAG, and its value may also travel in intra-domain
  chain context, including an access token the accepting Resource
  Authorization Server issues ({{handle-propagation}}).

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
  present only when `active` is `true`; {{handle-propagation}} constrains its
  exposure within the issuing server's trust domain

Change Controller:
: IETF

Specification Document(s):
: This document, {{handle-propagation}}

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

--- back

# Design Rationale {#rationale}

This non-normative appendix records the principal design choices.

## IdP-Mediated Continuation {#decision-rule}

A common IdP resolves target subjects and checks current authorization and
chain state at each continuation ({{validation}}). RAS-local withdrawal
remains subject to acceptance-evidence freshness ({{lifecycle-ending}}).

Offline attenuation, such as {{I-D.li-oauth-delegated-authorization}}, suits
boundaries where existing subject and issuer trust suffice. Deployments can
attenuate within a domain and continue across boundaries needing IdP
resolution. A target outside the common IdP's trust requires another
agreement and profile, such as
{{I-D.fletcher-transaction-token-chaining-profile}}.

## CAI Attestation and ID-JAG Redemption {#rationale-grant-type}

An ID-JAG authorizes the `client_id` client to redeem at its `aud` RAS and,
when sender-constrained, binds redemption to that client's key. Receiving
it as the audience does not authorize the RAS to change the client or key
({{I-D.ietf-oauth-identity-assertion-authz-grant}}).

Similarly, a workload receives an access token as the protected resource.
The token authorizes the caller and may bind to the caller's key. Verifying
that proof gives the workload neither the key nor a credential for its own
exchange at the IdP ({{security-pop}}).

The CAI supplies the missing attestation: it associates the workload with an
accepted, active authorization eligible for continuation and binds an
IdP-addressed assertion to the workload's key ({{assertion-preconditions}}).
Direct exchange of either earlier token would need additional rules for
that authority and binding ({{open-items}}).

Issuance reuses Token Exchange {{RFC8693}}. A Transaction Token can carry the
issuance context but does not itself prove RAS acceptance. The assertion
omits the user subject, which the IdP resolves for the onward ID-JAG.
Returning an ID-JAG preserves the target's redemption interface; reference
resolution and recipient-bound credentials remain alternatives ({{open-items}}).

Asymmetric signing avoids sharing secrets between CAIs and IdPs; one compact
JWS format reduces implementation choices ({{names}}). TLS protects transport
but does not conceal the assertion from the carrying workload.

## Per-Hop Handles {#rationale-handles}

Per-hop handles distinguish sibling branches, allowing correct ancestry and
subtree revocation without carrying user identity or ancestry in the handle
({{chain-id}}, {{onward-id-jag}}). Their visibility still permits correlation
({{privacy}}).

IdP-held state enables current checks at the cost of state retention and IdP
availability. Stateless hop commitments remain open ({{open-items}}).

## Actor Identity and Target Client Identity {#rationale-client-id}

In an assertion, `act` identifies the actor associated with the accepted
context; it grants no downstream authority. The actor's canonical identity
preserves lineage across credentials and target-specific registrations,
while the onward `client_id` preserves the target's ID-JAG redemption
interface ({{client-identity}}, {{onward-id-jag}}).

The `may_act` claim ({{RFC8693}}, Section 4.4) can inform actor authorization
but supplies neither acceptance evidence nor target subject resolution.

## Authorization Boundary {#rationale-boundary}

RAS scopes have audience-specific semantics. Cross-target restrictions
therefore belong in chain authorization and IdP policy
({{chain-authorization}}, {{security-authorization}}).

The profile binds continuation to accepted authorization and carries identity
and lineage. Whether requested work serves the user's or tenant's purpose
remains deployment policy; no purpose claim or agent authorization model is
defined.

Single-use limits each assertion to one grant; its lifetime bounds when it
can authorize new continuation ({{validation-replay}}).

# Examples {#examples}

These non-normative examples show a gateway, a SaaS chain, and a background
agent. The gateway is the baseline; authorization outcomes illustrate
deployment policy, not conformance requirements.

* "On the wire" crosses a trust boundary; "Intra-domain context" stays within
  a domain; "Server-side state" is not transmitted.
* H0, H1, and so on identify hops. Diagrams run downward in time.
* JWTs are decoded payloads without headers or signatures; client
  authentication is omitted except where shown. Key proofs use DPoP.
* The gateway uses a bearer root token; the SaaS root RAS chooses sender
  constraint. Onward continuation-aware RASes bind tokens as {{ras-processing}}
  requires.

## Gateway Example (Co-located RAS and CAI) {#example-gateway}

AgentApp calls ToolGateway for Alice. The gateway selects WikiAPI at request
time without receiving Alice's identity assertion. GatewayRAS also acts as
CAI and carries handles in access tokens.

All parties trust `https://idp.example/`, tenant `tenant-123`. The IdP maps
Alice's pairwise subjects at GatewayRAS and WikiRAS.

* Agent domain: confidential client `agent-app` roots the chain.
* Gateway domain: GatewayRAS at `https://ras.gateway.example/` protects
  `tool-gateway` at `https://gateway.example/` and binds root H0.
* Wiki domain: WikiRAS at `https://ras.wiki.example/` protects
  `https://api.wiki.example/`. It uses the base profile; H1 is terminal.

~~~
AgentApp        IdP       GatewayRAS/CAI     ToolGateway     WikiRAS/API
    |            |               |                |               |
    |-ID Token-->|               |                |               |
    |<-ID-JAG H0-|               |                |               |
    |-jwt-bearer: ID-JAG-------->| bind H0        |               |
    |<--access token, H0 claim---|                |               |
    |----------tool call with AT----------------->|               |
    |            |               |<--exchange AT--|               |
    |            |               |-assertion H0-->|               |
    |            |<--assertion + DPoP-------------|               |
    |            |-----------ID-JAG H1----------->|               |
    |            |               |                |-ID-JAG, DPoP->|
    |            |               |                |   (no binding)|
    |            |               |                |<---wiki AT----|
    |            |               |                |-call WikiAPI->|
    |<-------------------result-------------------|               |
~~~

### Provisioning {#example-gateway-provisioning}

The deployment uses these registrations and trust settings:

**IdP** ({{client-identity}}, {{issuer-trust}}, {{root-establishment}}):

* Register confidential clients `agent-app` and `tool-gateway` in
  `tenant-123`, with canonical identities (`https://agent.example/`,
  `agent-app`) and (`https://gateway.example/`, `tool-gateway`).
* Authorize `https://gateway.example/` to issue ToolGateway's credential and
  pair it with GatewayRAS as CAI. Trust GatewayRAS's issuer and signing keys.
* Permit ToolGateway to continue with read access to productivity tools;
  advertise continuation support and record its client identity at WikiRAS.

**GatewayRAS:** trust the IdP, advertise continuation, and register
`tool-gateway` as operating `https://gateway.example/`.

**ToolGateway:** provision a DPoP key and a gateway-domain client credential.

**WikiRAS:** trust the IdP, register ToolGateway, and support jwt-dpop.

ToolGateway uses this client assertion at the IdP; its GatewayRAS exchange
uses a separate assertion addressed there ({{example-gateway-ica}}):

~~~ json
{
  "iss": "https://gateway.example/",
  "sub": "tool-gateway",
  "aud": "https://idp.example/",
  "iat": 1710000230,
  "exp": 1710000290,
  "jti": "tg-cred-01"
}
~~~

Here `iss` and `sub` match its configured canonical actor identity.

### Root Exchange {#example-gateway-root}

AgentApp exchanges Alice's ID Token at the IdP for an ID-JAG addressed to
GatewayRAS. The token's `sid` anchors the chain to her session
({{root-establishment}}):

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded

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

The chain authorization admits read access to services currently classified
as productivity tools. Adding or removing a wiki from that class changes
subsequent access without replacing the chain ({{security-authorization}}).

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

  "iat": 1710000205,
  "exp": 1710000505,
  "jti": "idjag-gateway-01"
}
~~~

### GatewayRAS Binds H0 and Issues the Access Token {#example-gateway-bind}

AgentApp redeems the ID-JAG with jwt-bearer. GatewayRAS chooses a bearer
access token; the root exchange requires no sender constraint:

~~~
POST /token HTTP/1.1
Host: ras.gateway.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
&assertion=<the ID-JAG above>
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<agent-app JWT>
~~~

GatewayRAS binds H0, the IdP, and tenant while issuing the token
({{ras-processing}}). With no ID-JAG `cnf`, there is no key to record.

Server-side state at GatewayRAS:

~~~ json
{
  "identity_continuation_handle": "Qm7zXu2VtL9pKe4RaW1nHc",
  "status": "ACCEPTED",
  "continuation_eligible": true,
  "authorization_state": "gw-authz-7a1e",
  "idp": "https://idp.example/",
  "tenant": "tenant-123",
  "client_id": "agent-app",
  "bound_at": 1710000210
}
~~~

On the wire (signed JWT access token carrying H0):

~~~ json
{
  "iss": "https://ras.gateway.example/",
  "aud": "https://gateway.example/",
  "sub": "gateway-pairwise-subject",
  "client_id": "agent-app",
  "scope": "tools.invoke",
  "identity_continuation_handle": "Qm7zXu2VtL9pKe4RaW1nHc",
  "iat": 1710000210,
  "exp": 1710000810,
  "jti": "at-gw-0001"
}
~~~

AgentApp calls the gateway with this token; it cannot alter the embedded
handle. {{security-pop}} describes bearer-ingress exposure, and {{example}}
shows a sender-constrained root.

### ToolGateway Obtains the Assertion {#example-gateway-ica}

ToolGateway validates the incoming token, selects WikiAPI, and exchanges the
token at GatewayRAS using its client credential and its own DPoP key:

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

GatewayRAS checks resource ownership, token validity, and H0's active,
eligible authorization ({{assertion-preconditions}}). The assertion binds to
ToolGateway's proven key, not any incoming-token key ({{security-pop}}).

On the wire (response naming the IdP for continuation):

~~~ json
{
  "issued_token_type": "urn:ietf:params:oauth:token-type:identity-continuation",
  "access_token": "<the assertion below, compact JWS>",
  "token_type": "N_A",
  "identity_continuation_authorization_server": "https://idp.example/",
  "expires_in": 120
}
~~~

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
  "jti": "n3Vt7Jq2Xd5PbR9wLc1eKs"
}
~~~

### ToolGateway Continues to WikiRAS {#example-gateway-continue}

ToolGateway discovers the named IdP's token endpoint as {{assertion-response}}
specifies, then requests a WikiRAS ID-JAG using the assertion, its client
credential, and a DPoP proof:

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
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<tool-gateway client assertion>
~~~

The IdP applies {{validation}}: GatewayRAS is trusted for H0, the chain is
active, ToolGateway's identity and key match, and chain authorization and
current policy permit `wiki.read`. The root `tools.invoke` scope grants no
independent wiki authority.

The IdP resolves Alice's wiki subject and creates H1 with ToolGateway atop
AgentApp in the lineage.

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

An excluded target returns `invalid_target` without ending the chain
({{example-dynamic}}).

### WikiRAS Redeems an Ordinary ID-JAG {#example-gateway-terminal}

ToolGateway redeems at WikiRAS using jwt-dpop and the same key. WikiRAS
ignores H1, issues the access token, and records no continuation binding.
ToolGateway calls WikiAPI and returns the result to AgentApp.

Further calls reuse the token under {{implementation}}; another upstream
requires a new assertion and sibling hop under H0.

### What a Gateway Implements {#example-gateway-checklist}

The gateway domain adds two capabilities:

* GatewayRAS advertises continuation, binds and carries H0, and issues
  assertions ({{ras-processing}}, {{assertion-issuance}}, {{metadata-ras}}).
* ToolGateway exchanges its incoming token for an assertion, then exchanges
  that assertion at the IdP using its own identity and key.

{{example-gateway-provisioning}} supplies the registrations. AgentApp and
WikiRAS use the base profile; Alice's identity assertion never reaches the
gateway.

## SaaS Chain Example (Separate CAI and Transaction Token Carrier) {#example}

ExpenseApp calls ExpenseAPI, which continues to TravelAPI and then
BookingAPI. All domains trust `https://idp.example/`; the IdP maps the user's
pairwise subjects and each domain's canonical workload identities.

* Expense: `expense-app`, ExpenseRAS, Expense TTS, Expense CAI, and
  `expense-service` behind ExpenseAPI.
* Travel: TravelRAS, Travel TTS, Travel CAI, and `travel-service`.
* Booking: BookingRAS and BookingAPI, using the base profile.

Separate CAIs use Transaction Token carriers. H0 binds at ExpenseRAS, H1 at
TravelRAS; BookingRAS leaves H2 unbound.

Root exchange and H0 propagation:

~~~
ExpenseApp       IdP        ExpenseRAS      Expense TTS  ExpenseService
     |            |              |               |              |
     |-ID Token-->|              |               |              |
     |<-ID-JAG H0-|              |               |              |
     |-jwt-bearer: ID-JAG, DPoP->| bind H0       |              |
     |<-----------AT1------------|               |              |
     |---------------call ExpenseAPI: AT1 + DPoP--------------->|
     |            |              |               |<-----AT1-----|
     |            |              |<-resolve AT1--|              |
     |            |              |---bound H0--->|              |
     |            |              |               |-TT with H0-->|
~~~

Continuation to Travel; Booking repeats the pattern:

~~~
ExpenseService    Expense CAI     IdP       TravelRAS  Travel TTS/API
       |               |           |            |             |
       |-exchange TT-->|           |            |             |
       |<--assertion---|           |            |             |
       |--assertion, actor, DPoP-->|            |             |
       |<--------ID-JAG H1---------|            |             |
       |-------jwt-dpop: ID-JAG + DPoP--------->| bind H1     |
       |<------------------AT2------------------|             |
       |-------------call TravelAPI: AT2 + DPoP-------------->|
       |               |           |            | TT with H1 to workload
~~~

### Root Exchange (H0 at ExpenseRAS) {#example-first-hop}

ExpenseApp exchanges an ID Token whose `sid` anchors the chain:

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:id-jag
&audience=https://ras.expenses.example/
&resource=https://api.expenses.example/
&scope=expenses.read
&subject_token=<id_token>
&subject_token_type=urn:ietf:params:oauth:token-type:id_token
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<expense-app JWT>
~~~

Chain authorization and current policy permit the illustrated access.

Server-side state (IdP authorization):

~~~
(https://ras.expenses.example/, https://api.expenses.example/)
    permitted scopes: expenses.read

(https://ras.travel.example/, https://api.travel.example/)
    permitted scopes: trips.read

(https://ras.booking.example/, https://api.booking.example/)
    permitted scopes: stays.book
~~~

On the wire (decoded ID-JAG with root H0):

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

  "iat": 1710000005,
  "exp": 1710000305,
  "jti": "idjag-expense-01"
}
~~~

### ExpenseRAS Binds H0; Expense TTS Carries It {#example-context}

ExpenseRAS redeems the ID-JAG for AT1 and binds H0, the IdP, tenant, and
continuation eligibility ({{ras-processing}}). AT1 does not carry the handle.

Server-side state at ExpenseRAS:

~~~ json
{
  "identity_continuation_handle": "kW4uJ8pTe2NxA6rQvD1zYs",
  "status": "ACCEPTED",
  "continuation_eligible": true,
  "authorization_state": "at1-authz-2f9c",
  "idp": "https://idp.example/",
  "tenant": "tenant-123",
  "client_id": "expense-app",
  "bound_at": 1710000010
}
~~~

On the ExpenseAPI call, Expense TTS derives H0 from that binding and assigns
`expense-service` through authenticated routing state. The token's `req_wl`
names the requester, `expense-api`; it does not establish that assignment.
Neither ExpenseApp nor ExpenseService supplies H0.

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

This token stays within `expenses.example`; `tctx.identity_continuation` is
illustrative, not a standardized carrier schema.

### ExpenseService Obtains the Assertion {#example-ica}

ExpenseService exchanges the Transaction Token at Expense CAI with client
authentication and its own DPoP key:

~~~
POST /token HTTP/1.1
Host: cai.expenses.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof signed by the expense-service key>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:identity-continuation
&subject_token=<the Transaction Token above>
&subject_token_type=urn:ietf:params:oauth:token-type:txn_token
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<expense-service JWT>
~~~

The CAI verifies the token, checks ExpenseService's routing assignment, and
rechecks active, eligible authorization with ExpenseRAS before issuing
({{assertion-preconditions}}). The IdP's tenant configuration trusts this
CAI for ExpenseRAS ({{issuer-trust}}).

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

### Continuation to TravelRAS (H1) {#example-chained}

ExpenseService requests a TravelRAS ID-JAG using its assertion, client
credential, and DPoP proof:

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
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<expense-service client assertion>
~~~

The IdP applies {{validation}}, including trust in Expense CAI for ExpenseRAS,
the actor/key binding, and authorization for Travel. It relies on the
assertion rather than contacting ExpenseRAS, resolves the Travel subject,
and creates H1 with ExpenseService atop ExpenseApp.

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

### The Pattern Repeats to BookingRAS (H2) {#example-third-hop}

ExpenseService redeems at TravelRAS for AT2. TravelRAS binds H1; Travel TTS
carries it to TravelService. Only the handle changes within this illustrated
`tctx.identity_continuation` object:

Intra-domain context:

~~~ json
"tctx": {
  "identity_continuation": {
    "iss": "https://idp.example/",
    "tenant": "tenant-123",
    "handle": "Uc9fB3mHs5LdK7gEnX2wRj"
  }
}
~~~

TravelService obtains an assertion from Travel CAI and continues to Booking
with its own credential and key. Chain authorization and policy permit
`stays.book`; the IdP creates H2 under H1 and extends lineage.

On the wire (selected ID-JAG claims):

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

BookingRAS redeems under the base profile, ignores H2, and issues AT3 for
TravelService's BookingAPI call.

### What Differs from the Gateway Example {#example-differences}

Compared with the gateway:

* The IdP configures a separate CAI for each continuing RAS; conjunctive trust
  applies in either topology ({{issuer-trust}}).
* TTS-derived carriers, rather than access tokens, carry handles.
* Two continuations produce three actor entries, including the root actor.
* ExpenseRAS chooses sender constraint at the root, so ExpenseApp proves its
  key at redemption and on its API call.

## Background Agent Example (Scheduled Continuation) {#example-background}

Alice's daily calendar briefing uses a grant-anchored chain that survives
logout. PlatformRAS binds H0 to task state; Platform TTS and a separate CAI
support `briefing-agent`, whose canonical identity is
(`https://platform.example/`, `briefing-agent`). The Scheduler holds only a
task identifier.

CalendarRAS is terminal for each run. MailRAS illustrates a later target
excluded by the grant ({{example-dynamic}}).

### Setup: Anchoring the Chain to a Grant {#example-background-setup}

BriefingAgent exchanges a refresh token at the IdP for a root ID-JAG
addressed to PlatformRAS. The grant permits Platform and Calendar access
for Alice's daily briefing;
current policy agrees. The grant anchors H0 beyond logout ({{lifecycle}}).

Server-side state (IdP grant authorization):

~~~
(https://ras.platform.example/, https://api.platform.example/tasks)
    permitted scopes: task.manage

(https://ras.calendar.example/, https://api.calendar.example/)
    permitted scopes: calendar.read
~~~

PlatformRAS binds H0 to durable task state, keyed by its own task identifier:

Server-side state (PlatformRAS):

~~~
task_id:                      task-123
owner:                        alice
actor:                        briefing-agent
identity_continuation_handle: Pz6vTq1NcY4kM8bJf3RxWa  # H0
permitted_purpose:            morning-calendar-brief
schedule:                     "0 7 * * *"
governing_grant:              grant-8f2c19a4  # internal reference
expiry:                       1719450000  # local, not IdP lifetime
status:                       active
continuation_eligible:        true
~~~

Server-side state (Scheduler):

~~~
task_id: task-123
~~~

The Scheduler holds neither H0 nor a user credential; `task-123` is only a
local state reference.

### Each Run: Deriving H0 from Task State {#example-background-run}

With no user present, the platform derives H0 from task state:

~~~
 Scheduler   BriefingAgent      Platform TTS
     |              |                 |
     |---trigger--->|                 | task-123
     |              |-task-123+proof->|
     |              |                 | verify proof + task; derive H0
     |              |<-fresh TT(H0)---|
~~~

The trigger authenticates but `task-123` authorizes nothing. BriefingAgent
authenticates and proves its key to the TTS, which confirms the active task
and designated actor before deriving H0 ({{handle-propagation}}). Neither the
Scheduler nor agent chooses the handle.

### Each Run: Continuing to CalendarRAS {#example-background-continue}

Each run issues an assertion, continues, and redeems:

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

Platform CAI applies {{assertion-preconditions}} to the active task state.
BriefingAgent exchanges the assertion at the named IdP with its credential
and key; the tokens follow {{example-ica}} and {{example-chained}}.

The root and current actor are both BriefingAgent, so `act` merges them.
CalendarRAS issues an access token without binding the child hop; each run
creates a sibling under H0.

### A Newly Requested Target {#example-dynamic}

Adding mail to the briefing requires MailRAS. The grant excludes it, so
continuation fails even if general tenant policy permits mail access:

~~~
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "error": "invalid_target"
}
~~~

Under an authorization leaving productivity targets to current policy
({{example-gateway-root}}), adding Mail could instead permit `mail.read`
without a new chain. An excluded `mail.send` scope returns `invalid_scope`.
These request failures leave other authorized continuations available.

### What Differs from the SaaS Chain Example {#example-background-differences}

Compared with the SaaS chain:

* An optional grant anchor permits continuation after logout.
* Each run derives H0 from active task state and obtains a fresh assertion.
  Stealing the task record reveals H0 but not the agent's key.
* Run-specific hops are siblings under H0.

The example starts with Alice present. Other roots still require an
authorized exchange resolving to an active session or supported grant anchor;
administrative policy alone supplies no anchor ({{root-establishment}},
{{lifecycle-anchors}}).

# Open Items for Working Group Discussion {#open-items}

This non-normative appendix identifies questions for Working Group review.

\[\[ To be removed before publication as an RFC ]]

1. **Acceptance attestation.** Could a recipient-bound design, in which the IdP
   binds a continuation credential to an intended actor, actor class, trust
   domain, or key, or a target-resolved design replace the CAI assertion while
   still supplying evidence of RAS acceptance and of the actor's association
   with that context, which today only a party in the RAS's domain can provide
   ({{rationale-grant-type}})?

2. **Sender constraint.** Should mutual-TLS confirmation be defined jointly
   with ID-JAG? Should the assertion and ID-JAG permit different proven keys,
   and how should supported methods be advertised ({{client-identity}})?

3. **Client establishment control.** Should root clients be able to require or
   suppress chain establishment ({{root-establishment}})? The authors' current
   position is that establishment remains a tenant policy decision, so that
   existing clients need no change to participate in a chain.

4. **Stateless handles.** Can self-verifying handles preserve ancestry and
   subtree revocation within the recommended size bound, and at what privacy
   and operational cost ({{chain-id}}, {{onward-id-jag}})?

5. **Idempotent recovery.** Should optional support be discoverable, and if so
   should an IdP advertise that it offers recovery, the retry window, or both
   ({{idempotent-retry}})?

6. **Limits.** The document recommends that an assertion's lifetime not exceed
   300 seconds, recommends that IdPs accept lifetimes of up to 300 seconds and
   leaves the maximum to deployment, and requires a finite hop-count limit
   without fixing a default ({{assertion-claims}}, {{lifecycle-limits}}).
   Should the 300-second recommendations become requirements, should the
   profile fix a hop-count default, and should the accepted maximum be
   advertised in metadata?

The project issue tracker also records WG questions on authorization bounds
(#106), acceptance freshness (#107), actor identity evidence (#108), CAI
discovery (#109), bearer ingress (#110), and acceptance accountability (#41):
https://github.com/mcguinness/draft-mcguinness-oauth-id-continuation-assertion/issues

Further extension topics include client-requested limits or permitted actors,
intra-domain actor lineage and audit ({{I-D.mcguinness-oauth-actor-receipts}},
{{I-D.mcguinness-oauth-actor-proofs}}), non-user roots, and RAS-derived
narrowing ({{hop-activation}}).

# Acknowledgments
{:numbered="false"}

The authors thank the Working Group participants who developed the OAuth
Identity and Authorization Chaining Across Domains and the Identity Assertion
JWT Authorization Grant specifications, on whose work this profile builds.

# Document History
{:numbered="false"}

\[\[ To be removed before publication as an RFC ]]

-02

* Replaced the fixed root envelope of -01 with chain authorization based on
  recorded root facts and current tenant policy. Clarified that the CAI
  attests acceptance, eligibility, and actor association; the IdP authorizes
  onward access.
* Removed `actor_token` processing; client authentication determines the
  canonical actor identity. Distinguished disclosed actor lineage from the
  IdP's hop lineage.
* Defined assertion issuance using Token Exchange at the CAI, with an access
  token or Transaction Token and DPoP. Added the
  `identity_continuation_authorization_server` response parameter and client
  validation against the IdP's metadata issuer.
* Simplified hop state to IdP issuance and RAS acceptance, with continuation
  eligibility attested by the CAI using live or self-contained evidence.
* Removed the root exchange's proof-of-possession requirement. Specified
  DPoP-bound redemption of onward ID-JAGs and access-token binding at
  continuation-aware RASes.
* Clarified chain lifetime, withdrawal, and limits; made grant-anchor support
  optional and specified base-profile behavior when no anchor can be resolved.
* Defined optional recovery of an issued grant after a lost response while
  preserving single-use assertions. Clarified error codes and their precedence.
* Relaxed assertion lifetime, `jti` entropy, and handle-length requirements;
  permitted `nbf`; recommended capping assertion expiry at self-contained
  evidence expiry; and registered the handle as an introspection response
  member.
* Removed RAS nomination of CAIs through `identity_continuation_issuers`,
  clarified IdP configuration of CAI trust, and lifted the -01 restriction of
  CAI issuance to the RAS's trust domain, which is now typical rather than
  required.
* Reorganized the protocol description and security considerations; revised
  the introduction, examples, design rationale, and open items.
* Added Aaron Parecki as an author.

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
