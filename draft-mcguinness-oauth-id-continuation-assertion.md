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
several Resource Authorization Servers trust one IdP and use pairwise subject
identifiers that only the IdP can resolve. It complements offline
attenuation for intra-domain fan-out that does not change the subject.

--- middle

# Introduction

OAuth 2.0 {{RFC6749}} issues access tokens for use at protected resources, and
OAuth 2.0 Token Exchange {{RFC8693}} trades one token for another when a
request crosses a trust boundary. The Identity Assertion JWT Authorization
Grant (ID-JAG) {{I-D.ietf-oauth-identity-assertion-authz-grant}} applies Token
Exchange to identity: an IdP Authorization Server (IdP) accepts the user's ID
Token, refresh token, or SAML assertion and mints a grant that names the user
for a single downstream audience.

Many requests outlive that exchange. A request may cross several services
after the user is no longer present, or reach an audience the credential never
addressed, as at a tool gateway for agents, including Model Context Protocol
(MCP) gateways, that selects its upstream at request time
({{example-gateway}}). The first hop can present the user's credential to
obtain an ID-JAG; a workload further along the chain holds no such credential.
When each Resource Authorization Server (RAS) knows the user by a different
pairwise subject that only the IdP can resolve, that workload cannot even name
the user for the next audience.

This document defines the Identity Continuation Assertion: a short-lived,
sender-constrained JWT that a later workload presents as the `subject_token`
of a Token Exchange request to obtain the next audience-scoped ID-JAG, with no
further user interaction. The assertion carries a continuation handle that
ties the request to the chain state the IdP recorded when the chain was
established, including an envelope: the targets, continuing parties, depth,
and lifetime that tenant policy permits. At each hop the IdP resolves the
user's identity for the new audience and checks the requested target against
that envelope, not against the scopes of the token the workload received.

Three properties are at the core of the protocol:

1. Only the IdP names the user for a new audience.
2. Only an accepted authorization may be continued.
3. Only an authenticated actor that the IdP permits to continue from that
   accepted authorization, and that proves possession of its own key, may
   request continuation.

The IdP establishes the first by minting every ID-JAG itself, and the RAS the
second by binding the handle when it accepts a grant. For the third, a
Continuation Assertion Issuer (CAI) trusted for the RAS's hops attests that the
actor holds an accepted, still-active authorization, and the actor's own
client authentication and proof of possession establish who is asking. The
IdP's envelope then decides whether that actor may continue. The actor need
not be the party the incoming token was bound to: a gateway continues from a
token bound to the application that called it, proving its own key.

Continuation is therefore a fresh IdP decision at every boundary
({{validation}}), not a reused token or attenuated delegation in which each
service forwards a subset of what it received, and the assertion confers no
standing authority.
What it establishes is continuity of delegated identity and of authorization
ancestry. Whether an onward action is part of the work the user or tenant
sanctioned is a policy question the IdP answers through the envelope and,
where a deployment uses them, authorization details {{RFC9396}}
({{rationale-boundary}}).

This profile is an opt-in extension to ID-JAG. A root client whose subject
token resolves to a grant or session keeps its exchange unchanged
({{root-establishment}}); this profile asks nothing new of it, not even sender
constraint. A RAS from
which no workload continues runs unmodified ID-JAG, which this document calls
the base profile, and need not know that a chain exists ({{ras-processing}}).
Support is needed only where continuation proceeds: at the continuing
workload, its RAS and CAI, and the IdP. Chain state, envelope enforcement, and
replay protection sit at the IdP, which already resolves the pairwise subject
and holds the tenant policy; a continuation-aware RAS adds only the binding of
its own hops and the assertion that a workload may continue them, so an
existing ID-JAG ecosystem can add multi-hop access where a request's path is
not known in advance.

This document profiles Token Exchange, JWT {{RFC7519}}, and ID-JAG, and
complements OAuth Identity Chaining {{I-D.ietf-oauth-identity-chaining}}
({{rationale-idjag}}). Its scope is deliberately narrow. It assigns
obligations to two roles in the trust domain a chain continues from: the RAS
that accepts an ID-JAG and binds the hop, and the CAI that mints the
assertion. It defines their cross-domain artifacts and one request for
obtaining the assertion at a CAI token endpoint, leaving intra-domain handle
transport to the deployment ({{handle-propagation}}). It defines no new
access-token format, a Resource Server never consumes the assertion directly,
and a CAI never names the user for the target audience. {{protocol-overview}}
walks through one continuation end to end.

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

## Protocol Overview {#protocol-overview}

This section walks through one continuation in the deployment of
{{example-gateway}}. Alice uses an agent application, AgentApp, which calls a
tool gateway on her behalf, and the gateway must in turn call a wiki API on
her behalf. AgentApp holds Alice's ID Token; the gateway holds nothing of
Alice's that the wiki's authorization server would accept. This profile lets
the gateway obtain a grant for the wiki anyway, without ever seeing Alice's
credential, by proving to the IdP that it is continuing a request the IdP has
already granted.

Four parties take part:

* The IdP is the authorization server Alice signed in to. It issues ID-JAGs,
  and it alone can map Alice to the pairwise subject identifier by which each
  downstream service knows her.
* A Resource Authorization Server (RAS) is the authorization server in front
  of an API. It accepts an ID-JAG and issues an access token for that API. The
  gateway and the wiki each have one.
* The workload is a service that received a request for Alice and must now
  call a further API for her. Here it is the gateway.
* A Continuation Assertion Issuer (CAI) vouches, within the gateway's trust
  domain, that the gateway holds an accepted, still-active authorization for
  Alice's request and is the party asking to continue it. In the
  baseline deployment, and in this walk-through, the gateway's RAS also
  performs this role, so no separate component is needed.

The steps are numbered as in the figure. Steps marked "as in ID-JAG" are
unchanged from the base profile; the parts marked "new" are what this profile
adds.

~~~
AgentApp     IdP       Gateway RAS     Gateway          Wiki RAS
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
  |           |             | assertion: H0 active, may continue [new]
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

1. As in ID-JAG, AgentApp exchanges Alice's ID Token at the IdP for an ID-JAG
   addressed to the gateway's RAS. New: the IdP records this grant as the
   first hop of a chain; each ID-JAG it issues in the chain is one hop, linked
   to the hop it continues from. With the hop it records an envelope: the
   user, the targets that tenant policy permits, the parties permitted to
   continue, and the chain's depth and lifetime limits
   ({{root-establishment}}). It places a handle, an opaque reference to that
   hop, in the ID-JAG as the `identity_continuation_handle` claim
   ({{chain-id}}). The figure calls this hop H0.
2. As in ID-JAG, AgentApp presents the ID-JAG to the gateway's RAS and
   receives an access token. New: the RAS keeps the handle alongside the
   authorization it has just created, so that it can later say which hop that
   access token belongs to ({{ras-processing}}). A RAS that does not implement
   this profile ignores the claim.
3. AgentApp calls the gateway with the access token, as in any OAuth
   deployment.
4. New: the gateway needs a grant for the wiki. It exchanges the access token
   it received at its RAS's token endpoint for an Identity Continuation
   Assertion, the artifact this document defines ({{assertion}}). The RAS
   finds the hop bound to that access token, confirms that the authorization
   is still active, and issues the assertion ({{assertion-issuance}}): a
   short-lived, sender-constrained JWT attesting that hop H0 is active and
   that this gateway may continue it.
5. New: the gateway presents the assertion to the IdP as the `subject_token`
   of a Token Exchange request ({{token-exchange}}), with its own credential
   and a DPoP proof, asking for an ID-JAG for the wiki. The IdP looks up H0,
   checks that the wiki is within the envelope recorded in step 1 (not within
   the scope of the gateway's own access token) and that the gateway is a
   permitted continuer ({{validation}}), resolves Alice's subject identifier
   for the wiki, and issues the ID-JAG. That grant is a new hop, H1, whose
   parent is H0.
6. As in ID-JAG, the gateway presents the ID-JAG to the wiki's RAS and
   receives an access token. If the wiki's RAS does not implement this
   profile, it ignores the handle and the chain ends there. If it does, it
   binds H1 as in step 2; once the gateway calls the wiki API with that token
   as in step 3, steps 4 and 5 can repeat from the wiki's domain.

Three things are new in this flow: a claim in the ID-JAG (step 1), a binding
kept at the RAS (step 2), and the assertion together with the exchange that
consumes it (steps 4 and 5). The base exchanges of steps 1, 2, and 6 and the
call of step 3 are unchanged. Alice's ID Token is presented only to its issuing
IdP, never to the gateway or the wiki, and no party other than the IdP ever
names Alice for the wiki.

The IdP is never told that a RAS redeemed a grant; the assertion of step 4 is
how it learns that the hop was accepted and is still active. That is why it
accepts an assertion about a hop only from that hop's RAS or from a CAI it
trusts to speak for that RAS.

{{example-gateway}} shows these same steps with every artifact spelled out.

# Conventions and Definitions {#terms}

{::boilerplate bcp14-tagged}

This document uses the following terms:

IdP Authorization Server (IdP):
: The authority that authenticates the user, maps the user to each
  pairwise subject, and issues onward grants.

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

Identity Continuation Assertion:
: A short-lived, sender-constrained JWT from a CAI, presented to
  the IdP as a Token Exchange `subject_token` to obtain an onward ID-JAG.

Continuation Assertion Issuer (CAI):
: The role trusted by the IdP to issue Identity Continuation Assertions for a
  tenant. It never resolves the target audience's user subject.

Current actor:
: The workload presenting the assertion to the IdP, named by `act` and
  authenticated to the IdP by its client credential, an `actor_token`, or one
  JWT serving as both. Its canonical actor identity is the (`iss`, `sub`)
  pair that authentication resolves to ({{client-identity}}).

Root actor:
: The actor at the root of a chain: the authenticated OAuth client that
  obtains the first ID-JAG ({{client-identity}}); this is the Client of
  {{I-D.ietf-oauth-identity-assertion-authz-grant}} at the root exchange.
  It authenticates as an OAuth client; an actor token is optional for it as
  for any actor ({{root-actor}}).

Tenant:
: The administrative boundary within which the chain and CAI trust are
  configured; its determination is deployment-defined
  ({{security-trust-model}}).

Trust domain:
: An administrative and authentication boundary within which workloads can
  be directly authenticated, comparable to Workload Identity in Multi System
  Environments (WIMSE) {{I-D.ietf-wimse-arch}}. Its identifier is
  deployment-defined.

Continuation Handle (`identity_continuation_handle`):
: An opaque, unguessable, IdP-generated reference to one hop of a continuation
  chain. It correlates a request with the IdP's state for that hop and confers
  no authority by itself ({{chain-id}}).

Hop:
: One link of a chain: the IdP's record of an ID-JAG it issued, holding an
  immutable reference to its parent hop unless it is the root. Its lineage is
  its path to the root. When a RAS redeems the hop's ID-JAG, it binds the hop
  to its authorization state and the hop becomes ACCEPTED ({{ras-processing}},
  {{hop-activation}}). A hop from which no workload continues is terminal: its
  branch ends there, while sibling branches may continue.

Chain:
: An IdP-held tree of hops under one governing authorization; each hop's
  parent reference gives the tree its shape ({{onward-id-jag}}), and the
  authorization bounds its lifetime ({{lifecycle}}).

Governing authorization:
: The tenant policy decision that permits continuation, together with the
  grant or session anchor resolved from the root subject token, that anchors a
  chain and bounds every continuation under it ({{lifecycle}}).

Root-chain envelope:
: What the IdP evaluates every continuation against. It has two parts: the
  chain identity, the facts of the root exchange, fixed at establishment; and
  the continuation authorization, the onward targets, continuers, and limits
  the IdP recorded at establishment and evaluates under tenant policy as it
  stands, except that targets widen only through a recorded authorization
  basis ({{root-establishment}}).

Authorization basis:
: A reference to tenant policy, recorded in the continuation authorization in
  place of an enumerated target list, that the IdP reads at each continuation
  to decide which targets a chain may reach ({{root-establishment}}).

Continuation-capable:
: Describes an ID-JAG that carries the `identity_continuation_handle` claim
  ({{chain-id}}).

Pairwise subject:
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
  trust per the issuer-trust rule of {{validation}}, and its signature per the
  well-formedness rule.

`aud`:
: REQUIRED. A single string exactly matching the IdP issuer identifier, not
  its token endpoint URL.

`identity_continuation_handle`:
: REQUIRED. The hop being continued ({{chain-id}}).

`act`:
: REQUIRED. The current actor presenting the Token Exchange request, encoded
  as a single-level `act` claim per {{RFC8693}}:

  * `iss` and `sub` are REQUIRED, non-empty strings: the actor's canonical
    actor identity ({{client-identity}}), the issuer of the actor's
    credential and the actor's identifier there, its `client_id` for an
    {{RFC7523}} credential.
  * The IdP compares both with the canonical actor identity of the
    authenticated client and with any `actor_token` ({{client-identity}}).
  * Additional members MAY carry further identity attributes but are
    non-authoritative and MUST NOT affect identity, authorization, lineage,
    or issuance. A recipient MUST ignore members it does not understand,
    and `exp`, `nbf`, `aud`, `scope`, `cnf`, and nested `act` MUST NOT be
    present.

`cnf`:
: REQUIRED. A confirmation claim {{RFC7800}} binding the assertion to a key
  the current actor proves. `cnf` MUST contain exactly one confirmation
  method: `jkt`, the JWK SHA-256 thumbprint {{RFC7638}} of the DPoP key
  {{RFC9449}} ({{security-pop}}).

`iat`, `exp`:
: REQUIRED. `exp` MUST follow `iat`. The assertion is short-lived: `exp - iat`
  SHOULD NOT exceed 300 seconds, and the IdP rejects a lifetime longer than the
  maximum it accepts, which SHOULD be no less than 300 seconds so that a CAI
  using the recommended bound interoperates ({{validation}}).

`jti`:
: REQUIRED. A replay-detection identifier that MUST be unique per `iss`
  during the assertion validity window and MUST contain at least 128 bits
  of entropy.

The assertion is a subject token whose subject the IdP resolves from the
referenced hop, not an {{RFC7523}} JWT-profile assertion. It MUST NOT
contain:

* a top-level `sub`, `auth_time`, `acr`, `amr`, or `sid` claim (these come
  from the envelope), or a top-level `nbf` claim, whose {{RFC7519}}
  semantics would add a validity condition this profile does not define; or
* the Token Exchange request parameters `audience`, `resource`, `scope`,
  `authorization_details`, or `requested_token_type` (these are supplied by
  the request).

The assertion's `aud` identifies the IdP, not the requested target.

Other top-level claims MAY appear but MUST be ignored for validation,
authorization, and issuance.

# Continuation Handles (`identity_continuation_handle`) {#chain-id}

An `identity_continuation_handle` is an opaque, non-bearer reference to one
IdP-held hop of a chain. The IdP mints a fresh handle for each hop and carries
it in that hop's ID-JAG; continuing from a hop produces a child hop with its
own handle, recorded against the hop it continued from. A handle is non-secret
but security-sensitive correlation state: it confers no authority by itself,
yet a handle together with a CAI trust path and the actor's credential is a
larger compromise than the actor's credential alone ({{security-topology}});
{{handle-propagation}} limits where it travels.

The following rules apply:

1. When it establishes or continues a chain ({{root-establishment}}), the IdP
   MUST embed a fresh `identity_continuation_handle` claim in the issued
   ID-JAG, for that root or child hop, and MUST NOT reuse a handle across
   hops.

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

# Multi-Hop Cross-Domain Access {#access}

This section specifies the processing requirements for the flow introduced in
{{protocol-overview}}.

## Establishing a Chain {#root-establishment}

### Root Exchange Request {#root-request}

The root exchange presents a normal subject token, such as an ID Token,
refresh token, or SAML assertion:

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded

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

The root exchange and its ID-JAG conform to the base ID-JAG profile
({{I-D.ietf-oauth-identity-assertion-authz-grant}}) except where this document
extends it to issue a continuation-capable ID-JAG. On the root exchange,
`actor_token` is OPTIONAL.

### Chain Establishment {#chain-establishment}

The IdP, not the client, establishes a chain. The IdP MUST establish one when
the governing authorization for a root exchange permits continuation and the
root subject token resolves to a lifecycle anchor ({{lifecycle-anchors}}). When
the subject token resolves no anchor, the IdP issues the ID-JAG without a
handle, as under the base profile. No request parameter asks for it, and
advertised support ({{metadata-idp}}) signals capability, not authority. To
establish a chain, the IdP MUST include the root handle in the ID-JAG. Absent
the continuation authorization, the IdP MUST NOT establish a chain or include an
`identity_continuation_handle`.

The lifecycle anchor is the user's active IdP session or, for a durable
chain, a refresh token's OAuth grant ({{lifecycle-anchors}}).

This document places no proof-of-possession requirement on the root exchange
beyond the one an optional `actor_token` brings with it ({{root-actor}}). The
root client's obligations are those of the base profile, and sender constraint
becomes a requirement for an actor that continues ({{client-identity}}).

For every hop it creates, root or child, the IdP records the RAS audience
placed in the ID-JAG and any other issuers it trusts to attest that RAS's hops,
whether from tenant configuration or the RAS's nomination ({{metadata-ras}}).

Establishment is at-least-once. Retrying a lost response MAY create a second
chain, and the limits of {{lifecycle-limits}} apply across every chain rooted
in one governing authorization.

### Root Actor {#root-actor}

The root actor is the authenticated OAuth client; its identity rests entirely
on the mapping in {{client-identity}}. The base profile's recommendation to
use a confidential client applies to a root exchange that establishes a chain.

An optional `actor_token` MUST be valid, accepted for continuation, and
sender-constrained to a key the client proves on the exchange with a DPoP
proof {{RFC9449}}. It MUST identify that client and designate the IdP where
applicable. The IdP records the root actor, and that key, only after this
validation.

### Root-Chain Envelope {#root-envelope}

Tenant policy at the IdP determines whether the governing authorization
permits continuation and, if so, populates the root-chain envelope:

| Part | Dimension | Source | At each continuation |
|---|---|---|---|
| Chain identity | The authenticated user and authentication context (`auth_time`, `acr`, `amr`) | root authentication | fixed |
| Chain identity | The root actor and, where the client proved one on the root exchange, its key | root exchange | fixed |
| Chain identity | The governing authorization's anchor and the chain's expiry ({{lifecycle}}) | the root subject token's anchor | fixed |
| Continuation authorization | Onward targets, either enumerated as permitted audiences with their resources, scopes, and authorization details {{RFC9396}}, or recorded as an authorization basis | tenant policy | enumerated: narrows only; basis: as policy stands |
| Continuation authorization | The actors or trust domains permitted to continue, and the basis for that permission | tenant policy | as policy stands |
| Continuation authorization | Any maximum actor-chain depth and the fan-out, rate, or hop-count limits | tenant policy | as policy stands |

Token claims cannot supply these values. The chain identity is fixed: no later
policy or request changes the user, the authentication context, the root
actor, or the anchor a chain is bound to. The continuation authorization is
the tenant's to change (as in ID-JAG, the IdP rather than the user authorizes
each grant). How a change reaches a running chain depends on the form the IdP
recorded.

A policy change that narrows a chain, a target or continuer withdrawn or a
limit tightened, takes effect at the next continuation, whichever form the
envelope takes (the envelope-containment rule of {{validation}}). Continuers
and limits are tenant controls, read as policy stands in either direction.

Target authority widens only through an authorization basis. An envelope that
enumerates its targets MUST NOT admit a target the tenant adds after
establishment ({{example-dynamic}}). A basis-referenced envelope admits
whatever the basis currently reads, such as a service's classification or a
group's membership ({{example-gateway-root}}).

What the root request asked for is not a ceiling either: its audience and
scope bound only the root ID-JAG.

## Continuation-Aware RAS Processing {#ras-processing}

A continuation-aware RAS implements this extension and advertises the
continuation grant profile ({{metadata-ras}}). A RAS that does not implement
it processes an ordinary ID-JAG and ignores the handle. An ID-JAG that such a
RAS accepts cannot become a continuation source.

On accepting a continuation-capable ID-JAG, a continuation-aware RAS MUST:

1. accept the ID-JAG per {{I-D.ietf-oauth-identity-assertion-authz-grant}}.
That processing validates the grant, authenticates the presenting client,
verifies the sender constraint of an ID-JAG that carries `cnf`, applies local
authorization policy, and issues an access token. The access token is
sender-constrained to the confirmed key when the ID-JAG carries `cnf`, as
every onward ID-JAG does ({{client-identity}}); otherwise, as for a root
ID-JAG ({{root-establishment}}), the RAS's own policy decides; and
2. bind `identity_continuation_handle`, the ID-JAG's issuer and tenant, and any
confirmed key to the authorization state it establishes, and record whether
continuation is permitted under the RAS's own policy.

Three rules govern binding the handle:

* The RAS MUST bind the handle and issue the access token as one outcome: no
  access token without its binding, and no binding without a token.
* Repeated redemption of one ID-JAG, identified by its validated issuer and
  handle within the authenticated tenant binding, MUST bind to the same hop
  authorization record, so a retry cannot create multiple records for one
  grant. A matching handle under another issuer is a different grant, not a
  retry.
* The RAS exposes the binding, its record linking the handle to authorization
  state, only within its trust domain; the handle itself travels in the access
  token or another carrier as {{handle-propagation}} describes.

## Hop Activation {#hop-activation}

A hop has two states, PENDING and ACCEPTED, conceptual states rather than
values carried on the wire.

| State | Where it lives | Meaning |
|---|---|---|
| PENDING | IdP | the IdP issued the ID-JAG and has not yet received an attestation of acceptance |
| ACCEPTED | RAS authorization state | the RAS redeemed the grant, authorized it, and bound the handle |

A CAI attests a hop only once it is ACCEPTED. A PENDING hop yields no
assertion and reaches no continuation exchange. A fresh assertion from a
trusted CAI lets the IdP continue from the hop for one request. The hop is
continuable when:

* a CAI trusted for the accepting RAS has freshly attested it as ACCEPTED
  and still active,
* no ancestor is revoked, and
* the current actor is the one the attestation names.

The issuer-trust, chain-state, current-actor, and freshness rules of
{{validation}} establish those facts.

The IdP learns of acceptance only through attestation ({{protocol-overview}}).
The accepting RAS attests its own hops first-hand, and any other trusted CAI
confirms acceptance and activity by the RAS's authorization semantics before
attesting ({{assertion-issuance}}). Absent CAI compromise
({{security-trust-model}}), no trusted CAI attests an issued-but-rejected
ID-JAG, so continuation fails closed.

Acceptance gates continuation but does not bound downstream authority: the IdP
evaluates later targets against the root envelope ({{root-establishment}}),
and local RAS authorization neither narrows nor widens it. Continuation
propagates authorization provenance, not the accepting RAS's authorization
vocabulary: a scope at one audience says nothing about a scope at another
({{rationale-boundary}}).

## Intra-Domain Handle Propagation {#handle-propagation}

When the accepting RAS holds the CAI role, it reads the handle directly from
its authorization state and needs no carrier. Otherwise, a carrier derived
from the RAS binding conveys the handle within the trust domain
({{ras-processing}}) and is accepted only within that domain
({{assertion-issuance}}).

One rule applies to direct reads and every carrier. Each call arrives with a
credential or context that identifies exactly one RAS-bound authorization. The
CAI MUST use the handle bound to that authorization, whether it reads RAS
state directly or receives a carrier derived from it. A requester chooses
which credential to present, but cannot supply or override its handle
separately. A session or subject alone is not enough to select the
authorization: doing so could attach another user's handle to the call.

What identifies the authorization depends on where the call lands:

* For an ingress call, the access token, after the resource verifies its
  proof of possession where the token is sender-constrained.
* For a downstream call, a carrier forwarded from that ingress.
* For a scheduled run, the task named by an authenticated actor, resolved to
  the task authorization for which that actor is designated
  ({{security-envelope}}).

The CAI's acceptance check ({{assertion-preconditions}}) covers staleness for
every carrier.

A Resource Server has no obligations under this document. A carrier SHOULD NOT
expose the handle to a party with no role in continuation. Deployments keep
this security-sensitive correlation state ({{chain-id}}) out of logs, traces,
and responses.

## Assertion Issuance {#assertion-issuance}

The CAI issues the Identity Continuation Assertion that a workload presents to
continue a chain across a boundary. The CAI MUST issue only for an actor it is
authoritative to associate with the RAS-accepted authorization, typically one
in the RAS's trust domain; actor authentication is deployment-specific. It
MUST set the assertion's `aud` to the IdP recorded in the hop's RAS binding
({{ras-processing}}), and it MUST NOT accept an IdP audience supplied by the
requester.

The CAI attests three facts about its own domain:

* The RAS accepted the hop.
* The hop is still active and continuable by the RAS's own authorization
  semantics.
* The authenticated actor proving a key is the party handling the request
  that hop authorized.

Whether that actor may continue, and to what, is the IdP's decision under the
envelope ({{root-establishment}}, {{validation}}).

### Request {#assertion-token-exchange}

A CAI that is an OAuth authorization server, including a RAS acting as its own
CAI, MAY issue assertions from its token endpoint using Token Exchange
{{RFC8693}} as profiled in this and the following subsections. Such a RAS
SHOULD support this method of issuance, so that a workload in its domain has
one request to implement. Issuance by other means remains deployment-specific
({{handle-propagation}}).

The requesting party is the current actor ({{terms}}), acting as an OAuth
client of the CAI; this document calls it the client. The client makes a Token
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

The client MUST authenticate to the token endpoint ({{RFC6749}}, Section 2.3),
and the request MUST include a DPoP proof {{RFC9449}} of the client's own key;
the CAI binds the assertion to that key in `cnf` and does not compare it with
any key the `subject_token` is bound to. For example:

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

* an access token issued by a RAS whose hops the CAI attests, unexpired, and
  valid for a protected resource that the authenticated client operates; or
* a Transaction Token valid for the CAI's trust domain under
  {{I-D.ietf-oauth-transaction-tokens}}, Section 12.2, and carrying the hop's
  handle as chain context ({{handle-propagation}}). The Transaction Token is
  the carrier.

With an access token, the client is a resource server exchanging a token it
received, the scenario of the example in {{RFC8693}}, Section 2.3. A RAS acting
as CAI resolves its own token; a separate CAI resolves it through introspection
{{RFC7662}} or, for a self-contained token, by validating it.

### Preconditions {#assertion-preconditions}

Either `subject_token` type ({{assertion-token-exchange}}) supplies the facts
below. The CAI MUST authenticate the actor and issue only after establishing
these facts:

1. The handle came through an authenticated, confidential,
   integrity-protected chain path or equivalent authenticated state.

2. The current actor controls the key placed in `cnf`.

3. `act` names that actor and, if offline attenuation reached the actor, the
   attenuated credential it received is valid ({{decision-rule}}).

4. The actor is bound to the current transaction, and the handle matches that
   transaction's RAS-bound state ({{hop-activation}}). The handle is input to
   verify, never authority in itself ({{chain-id}}).

5. Evidence that is authoritative by the RAS's own authorization semantics,
   whatever the carrier, confirms that the authorization remains active and
   that its binding still records continuation as permitted. That evidence is
   either a recheck of RAS authorization state or, where the RAS's
   authorization is a self-contained short-lived token, that token's validity.

The subject token's integrity protection and the authenticated request
establish fact 1; the DPoP proof, fact 2; client authentication, fact 3; and
the bound handle and the RAS's acceptance evidence, facts 4 and 5.

A live recheck SHOULD be used where the tenant requires withdrawal of a hop's
authorization to stop fresh assertions before the RAS's token would expire.
With self-contained evidence, the CAI stops issuing when that token expires,
as for any OAuth access token. An assertion already issued remains usable
until its own `exp`, so the withdrawal tail is the token's remaining lifetime
plus the assertion lifetime ({{security-topology}}). A CAI that caps the
assertion's `exp` at the evidence's expiry removes the second term.

A domain may add its own conditions for issuing, for example limiting which of
its workloads may obtain assertions, but such conditions narrow issuance only.
Target or purpose hints can narrow CAI issuance but MUST NOT control the IdP's
target decision, and propagated context MUST NOT override the envelope.

### Successful Response {#assertion-response}

A successful response is a Token Exchange response ({{RFC8693}}, Section
2.2.1) in which `access_token` carries the Identity Continuation Assertion,
`issued_token_type` is
`urn:ietf:params:oauth:token-type:identity-continuation`, `token_type` is
`N_A` (not applicable), and `expires_in` reflects
the assertion's lifetime. This document adds one parameter:

`audience`:
: REQUIRED. The issuer identifier of the IdP to which the client presents the
  assertion, equal to the assertion's `aud`. Because the request carries no
  `audience`, this is how the client learns where the assertion goes.

The client obtains that IdP's `token_endpoint` from its authorization server
metadata ({{RFC8414}}), retrieved with the `oauth-authorization-server`
well-known URI suffix under the issuer identifier. Before using it, the client
confirms that the returned `issuer` exactly matches `audience`. Where the IdP
publishes no metadata, the client uses configuration bound to that issuer
identifier ({{metadata}}). The client SHOULD present the assertion only to an
IdP it is configured to trust; the `audience` parameter tells it where, not
whether.

The response MUST NOT include a `refresh_token`, which would let a client
obtain further assertions without presenting a token or passing the
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
  "audience": "https://idp.example/",
  "expires_in": 120
}
~~~

### Error Response {#assertion-error-response}

On failure, the CAI returns an error response according to {{RFC6749}},
Section 5.2, and {{RFC8693}}, Section 2.2.2. This document specifies the
following error mappings:

* `invalid_request` when the `subject_token` is invalid or unacceptable under
  policy, including when it is unknown, expired, revoked, not valid for a
  resource the client operates, has no bound handle, or names an authorization
  whose binding does not record continuation as permitted, or when the request
  includes a parameter this document prohibits ({{assertion-token-exchange}});
* `unauthorized_client` when the client is not permitted to use this grant
  type; and
* `invalid_dpop_proof` ({{RFC9449}}) for a failed proof.

## Token Exchange {#token-exchange}

An Identity Continuation Assertion is the `subject_token` of an OAuth 2.0
Token Exchange request {{RFC8693}}. The root exchange and a continuation
exchange use the same Token Exchange framework: a continuation exchange
substitutes an Identity Continuation Assertion for the root credential and
adds the actor authentication and DPoP proof described below.

Before a chain can continue to a target, the current actor needs a client
registration or resolvable client identity at that target's RAS
({{onward-id-jag}}).

### Request {#request}

The root exchange is shown in {{root-establishment}}. A continuation exchange
presents an Identity Continuation Assertion, a DPoP proof of the `cnf` key,
and the actor's authentication ({{client-identity}}). The following example
presents a client assertion and a separate `actor_token`, though one JWT can
serve as both:

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

A client whose client assertion is itself a sender-constrained credential that
resolves to the canonical actor identity omits the actor token
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
&client_assertion=<sender-constrained JWT identifying the actor>
~~~

The Token Exchange request, never the assertion, supplies the requested
`audience`, `resource`, `scope`, `requested_token_type`, and any
`authorization_details` {{RFC9396}} ({{assertion-claims}}). A request can
carry multiple `resource` indicators {{RFC8707}}, which the IdP treats as an
order-independent set. The envelope-containment rule ({{validation}}) applies
to `authorization_details` and to scope alike.

### Presenter Authentication {#client-identity}

The current actor MUST authenticate as an OAuth client with a credential that
resolves to its canonical actor identity. The canonical actor identity is the
(`iss`, `sub`) pair the IdP holds for that client. Its `iss` is the actor's
identity authority, and issuer pairing ({{validation}},
{{security-trust-model}}) is defined against that authority, not against
whichever parameter carried the credential. For a client authenticated with a
credential the IdP itself issued or registered directly, such as a client secret
or a mutual-TLS certificate, the identity authority is the IdP's own issuer
identifier and `sub` is the `client_id`.

The IdP MUST derive that pair from its registration of the client or its
configuration of the client's credential issuer, not from the client
authentication method, and MUST NOT accept a self-asserted mapping.

This mapping applies to every exchange. On a root exchange, client
authentication alone identifies the root actor. A root exchange carries no
unconditional proof-of-possession requirement ({{root-establishment}}). On a
continuation exchange ({{request}}), the IdP MUST also match the canonical
actor identity to the assertion's `act` and, where an `actor_token` is
present, to that token.

A separate `actor_token` is OPTIONAL. A deployment presents one when the
client credential does not itself identify the actor and prove its key, or
when it wants independent evidence of the actor.

An `actor_token` MUST NOT be a bearer token: for a JWT the IdP verifies its
`cnf` confirmation (`jkt` in this version), and for an opaque token it obtains
equivalent confirmation from authoritative metadata such as introspection
{{RFC7662}}. Its key is one the actor proves in the request.

A sender-constrained JWT MAY serve as both client assertion and `actor_token`
when it satisfies both profiles. For {{RFC7523}}, its `sub` is the
`client_id`, the canonical actor identity is then the assertion's issuer and
that `client_id`, and the IdP MUST authorize its issuer for that client
({{rationale-client-id}}).

The comparison runs between actor identities, never between a raw OAuth
client identifier and an `act` value:

~~~
authenticated OAuth client
        |  authoritative mapping (IdP registration)
        v
canonical actor identity (iss, sub)
        ^                        ^
        |  equal                 |  equal
   actor_token (if present)     act
~~~

The IdP MUST compare the actor `iss` and `sub` as case-sensitive strings with
no transformation or canonicalization ({{RFC7519}}): the assertion's `act`,
and the identity in any `actor_token`, are each compared with the canonical
actor identity, and identities in different tenants never compare equal.

The actor MUST prove possession of the key in `cnf`; for the `jkt` method, that
proof is a DPoP proof {{RFC9449}}. The IdP MUST bind the onward ID-JAG to a key
the actor proves in the request. DPoP is the only confirmation method this
version defines, so a target validates the onward ID-JAG's confirmation with the
DPoP mechanics it already implements ({{RFC9449}}); a mutual-TLS method
{{RFC8705}} is an open question ({{open-items}}).

A request carries one DPoP proof, so that key is the assertion's `cnf` key,
any `actor_token` is bound to it as well, and the actor's credential, the
assertion, and the onward ID-JAG share one key ({{rationale-client-id}}). Key
rotation takes effect when the actor obtains a new assertion and actor token
bound to the new key.

### Request Validation {#validation}

For a continuation exchange, the IdP MUST reject the request unless every rule
below holds. Their order is not significant, though one rule's input may come
from another's resolution.

1. **Request parameters.**
   * exactly one each of `grant_type`, `subject_token`, `subject_token_type`,
     `requested_token_type`, and `audience`; zero or one `actor_token`, with
     `actor_token_type` present exactly when `actor_token` is ({{RFC8693}},
     Section 2.1);
   * zero or more `resource`, and at most one each of `scope` and
     `authorization_details`, all OPTIONAL and, when present, evaluated by
     the envelope-containment rule; and
   * `grant_type` is `urn:ietf:params:oauth:grant-type:token-exchange`,
     `subject_token_type` is
     `urn:ietf:params:oauth:token-type:identity-continuation`, and
     `requested_token_type` is `urn:ietf:params:oauth:token-type:id-jag`;

2. **Assertion well-formedness.**
   * the assertion is a JWT whose JOSE `typ` header is
     `oauth-identity-continuation+jwt`;
   * it carries exactly one value for each claim required by
     {{assertion-claims}} and none of the claims that section forbids;
   * `iss`, `aud`, `identity_continuation_handle`, and `jti` are non-empty
     strings, `act` and `cnf` are JSON objects with `cnf` naming exactly one
     confirmation method, and `iat` and `exp` are NumericDate numbers;
   * the signature validates under an acceptable algorithm ({{security-alg}},
     {{RFC8725}}) with the issuer's resolved signing keys ({{metadata}});
   * `aud` exactly matches the IdP's issuer identifier; and
   * the JOSE header checks of {{security-alg}} pass;

3. **Issuer trust.**
   * the assertion `iss` is either the accepting RAS itself, identified by the
     ID-JAG `aud` recorded for the hop as a string or one-element array;
   * or another issuer the IdP trusts to attest that RAS's hops, recorded from
     tenant configuration or the RAS's nomination ({{metadata-ras}}); and
   * in either case, that issuer is trusted for the chain's tenant, recorded at
     establishment, and authorized to pair, for that tenant, with the actor's
     identity authority ({{client-identity}}), whether an `actor_token` or the
     IdP's mapping of the client credential supplied that authority;

4. **Chain state.**
   * the handle identifies a hop the IdP issued, on an active chain, that the
     assertion attests as accepted ({{hop-activation}});
   * neither the presented hop nor any ancestor is revoked; and
   * the actor lineage that results from merging consecutive same-actor
     entries, as the onward `act` will ({{onward-id-jag}}), is within its
     depth bound, which counts lineage entries, not hops;

5. **Current actor and binding.**
   * `act` is present, conforms to the schema of {{assertion-claims}}, and
     identifies the current actor, the canonical actor identity of the
     authenticated client ({{client-identity}});
   * where an `actor_token` is present, `actor_token_type` names a token type
     the IdP supports, and the token has a trusted issuer for the actor's
     domain and tenant, is valid for that type, is accepted, designates the
     IdP where applicable, identifies the same actor, and is sender-constrained
     to a key the actor proves in the request ({{client-identity}});
   * the request proves possession of the `cnf` key with a matching DPoP
     proof ({{client-identity}}, {{RFC9449}});
   * the actor is permitted by the chain's continuation authorization
     ({{root-establishment}}) to continue from the presented hop; and
   * the IdP can resolve, for the requested `audience`, both the pairwise
     subject and the actor's client identifier ({{onward-id-jag}});

6. **Freshness and replay.**
   * `iat` is within permitted future clock skew (which SHOULD NOT exceed 60
     seconds), `exp` follows `iat`, the assertion is unexpired, and its
     lifetime does not exceed the maximum the IdP accepts
     ({{assertion-claims}}); and
   * `jti` is not yet reserved for the assertion issuer or, where the IdP
     offers idempotent retry ({{validation-replay}}), is RESERVED or ISSUED
     under a fingerprint matching this request; any other reserved `jti` is
     rejected;

7. **Envelope containment.** The effective authorization the IdP would grant,
   after applying any default scope and policy to the requested audience,
   resource, scopes, and authorization details, is within the root-chain
   envelope, that is:
   * consistent with the chain identity, the fixed facts of the root exchange;
   * within the continuation authorization as recorded at establishment,
     evaluated under tenant policy as it currently applies
     ({{root-establishment}}); and
   * within current IdP actor policy.

   The issued ID-JAG carries the `scope`, `resource`, and
   `authorization_details` values that express that authorization, and the IdP
   rejects a request whose effective authority it cannot establish within the
   envelope. Because {{RFC9396}} defines no generic comparison, containment of
   authorization details uses the comparison rules of each detail type, and the
   IdP rejects a detail type whose rules it does not implement.

### Successful Response {#success-response}

The Token Exchange response follows the base ID-JAG profile: the ID-JAG is
returned in `access_token`, with `token_type` `N_A` (not applicable;
{{RFC8693}}, Section 2.2.1). The response MUST NOT include a `refresh_token`:
a renewable credential would let the workload obtain further grants without
fresh CAI attestation, or root a new chain through the refresh-token anchor,
outside the hop's revocation dependencies.

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
claim ({{chain-id}}), a claim inside `access_token` and not a separate Token
Exchange response parameter; the accepting RAS binds it ({{ras-processing}}),
and the CAI reaches it through RAS state or intra-domain context
({{handle-propagation}}).

There is no chain-expiry response parameter: chain lifetime is authoritative
at the IdP ({{lifecycle}}), and a deployment needing advance warning conveys
it through task or authorization state, an optional ID-JAG claim, or a
management API.

On success, the IdP records a PENDING child ({{hop-activation}}) of the
presented hop and issues an ID-JAG carrying the resolved target `sub` and a
fresh handle. An idempotent retry (the freshness rule; {{validation-replay}})
instead returns the previously issued grant unchanged, creating no new hop or
handle.

### Onward ID-JAG Construction {#onward-id-jag}

The onward ID-JAG conforms to the base ID-JAG profile
({{I-D.ietf-oauth-identity-assertion-authz-grant}}) except where this document
extends it: its `sub` is the IdP-issued pairwise subject for the target
audience, and `aud_sub` remains available under the base profile where the
target's native subject namespace differs.

Where the envelope records `auth_time`, `acr`, or `amr`, the IdP MUST include
them in the onward ID-JAG unchanged. Continuation MUST NOT extend or
strengthen the authentication context, for example by raising `acr` or adding
`amr` beyond the user's root authentication.

The IdP constructs `act` as follows:

* It places the authenticated current actor atop the presented hop's lineage;
  it never copies lineage from the assertion, and siblings do not contribute.
* A hop's parent reference is immutable, and the IdP MUST derive lineage only
  by walking parent references to the root; it keeps no chain-wide actor
  history, so concurrent sibling continuations are independent branches.
* Consecutive identical actors merge into one entry, though the hop record
  remains; policy MAY limit disclosed depth, narrowing what a target sees
  without changing the depth bound the IdP enforces (the chain-state rule of
  {{validation}}).
* A RAS MUST NOT read the absence of a further nested `act` as proof that no
  earlier actor exists, because policy may narrow the disclosed lineage: `act`
  is disclosed lineage, not the authoritative history, which only the IdP's
  hop records hold.

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

The onward ID-JAG's `client_id` is the current actor's identifier at the target
RAS, which the IdP resolves from its registration of the actor's client
identities per target; a target for which the actor has none fails with
`invalid_target` ({{error-response}}).

### Error Response and Recovery {#error-response}

On failure, the IdP returns an error response ({{RFC6749}}, Section 5.2;
{{RFC8693}}, Section 2.2.2):

* The IdP MUST return `invalid_continuation` ({{iana}}) only when the handle is
permanently unusable: unknown, on an expired or ended chain, on a revoked hop
or ancestor, or on a chain whose continuation authorization the tenant has
withdrawn ({{lifecycle-ending}}). * The IdP SHOULD use `invalid_request` for a
malformed, inconsistent, or unacceptable token, including a lifetime above the
maximum the IdP accepts, `invalid_dpop_proof` for a DPoP failure,
`unauthorized_client` for an actor that current tenant policy does not permit
to continue from the presented hop, which leaves the chain continuable by other
actors, `invalid_grant` when the presented hop cannot be continued further
under the chain's depth, fan-out, or hop-count limits ({{lifecycle-limits}}),
and `invalid_target`, `invalid_scope`, or `invalid_authorization_details` for a
request outside the envelope or for a target at which the IdP can resolve no
subject or client identity for the actor.

Recovery from `invalid_continuation` requires establishing a new chain, and
succeeds only where the governing authorization still permits continuation: a
session-anchored chain re-roots by re-authenticating the user, a
grant-anchored chain re-roots from its still-valid grant without the user, and
a handle disabled by withdrawn continuation authorization cannot re-root at
all.

The other errors leave the chain still continuable, so the client abandons
only the current request or, for a hop at its depth bound, further
continuation from that hop.

## Replay Reservation and Retry {#validation-replay}

A consumed assertion is not equivalent to a fresh one. With live acceptance
evidence, the CAI refuses a fresh assertion once the hop's authorization has
lapsed ({{assertion-preconditions}}); with self-contained evidence, issuance
continues until the RAS token expires, and single-use still keeps each consumed
assertion from authorizing more than its one grant within that tail
({{security-pop}}).

The IdP MUST issue at most one grant per assertion, including under concurrent
presentations: it reserves the assertion's (`iss`, `jti`) once validation
succeeds and before it issues the grant, and MUST retain the reservation through
`exp` plus the maximum permitted clock skew. Uniqueness is keyed on (`iss`,
`jti`), since partitioning by tenant alone would let two assertion issuers in
one tenant collide on a reused `jti`. Without idempotent retry this needs only
the set of (`iss`, `jti`) values presented within that window.

A request that fails validation leaves the assertion unreserved and usable
within its window.

A second presentation of a reserved assertion MUST be rejected unless the IdP
offers idempotent retry. An IdP MAY offer it by binding the reservation to a
fingerprint of the request first authorized and recording the reservation as
RESERVED, ISSUED, or FAILED (distinct from the hop states of
{{hop-activation}}). Such an IdP MUST return the previously issued grant for a
presentation matching the fingerprint and MUST reject one that does not. The
fingerprint MUST cover:

* `audience` as an exact string;
* the `resource` values as an order-independent set;
* `scope` as an order-independent set;
* the exact `authorization_details` JSON after form decoding (a different
  serialization is a different request);
* the actor's `iss` and `sub`;
* the confirmation key in `cnf` (its `jkt` thumbprint in this version); and
* a SHA-256 hash of the exact `subject_token` after form decoding, which binds
  the fingerprint to the specific assertion and its handle.

A reservation that does not reach ISSUED before `exp` becomes FAILED, which is
final and requires a fresh assertion.

After a lost response, a client MAY retry the same assertion where the IdP
offers retry, or obtain a fresh assertion. A fresh assertion may create an
equivalent grant and sibling hop but no additional authority. Application
idempotency remains out of scope. Realization guidance is in
{{implementation}}; whether idempotent retry should be mandatory is an open
question ({{open-items}}).

# Chain Lifetime and Revocation {#lifecycle}

A chain anchored to the user's IdP session is the core case: it lives while
the session does. A chain anchored to a refresh token's grant is a durable
chain that outlives the session, as an unattended agent needs
({{example-background}}). The anchors, ending rules, and limits below apply
to both forms; an implementation of session-bounded continuation alone needs
the shared rules but not the grant-specific discussion or the background
example.

A chain is continuable only while active at the IdP. Each cross-boundary hop
is a fresh policy check, and revoking a hop stops its subtree at the next
continuation, fail-closed; an offline-attenuated token, by contrast, stays
usable for its lifetime without contacting an authority. Revocation does not
invalidate an already-issued ID-JAG, so the revocation window is the ID-JAG's
remaining redemption window plus the lifetime of the access token a late
redemption obtains; any refresh token the RAS issues, which the base profile
recommends against, extends it further.

Three independent lifetimes govern a continuation: the ID-JAG's short
redemption window; the access-token lifetime the accepting RAS sets, which
this profile does not constrain; and the IdP-held continuation chain.
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
only for resolution and MUST NOT enter assertions or chain context. Rotation
of a refresh token does not affect the grant anchor.

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
bounds only new
continuations; an ID-JAG already issued remains redeemable for its own
lifetime, since redemption is not a continuation.

The IdP ends a chain when it observes the withdrawal, whether through a
policy event or at the next continuation attempt, and an ended chain stays
ended: restoring the policy that withdrew permission does not revive it, its
handles remain permanently unusable ({{error-response}}), and a new chain is
needed.

The IdP has these duties over chain lifetime:

* it MUST bound chain lifetime by the governing authorization;
* it MUST support administrative revocation of an entire chain and MAY
  revoke an individual hop's subtree; and
* it MUST reject continuation on a revoked, expired, or ended chain.

How an IdP surfaces chains to users and administrators for review and
revocation is deployment-specific; {{GRANT-MGMT}} describes OAuth grant
management for that purpose.

## Limits {#lifecycle-limits}

Revocation of the governing authorization applies to every chain rooted in
it, and the actor-chain depth bound is enforced per branch. Fan-out, rate, or
hop-count limits configured for a governing authorization apply across every
chain rooted in it, so sibling chains share one budget; a retried
establishment ({{root-establishment}}) MUST NOT evade them.

# Authorization Server Metadata {#metadata}

## IdP Authorization Server Metadata {#metadata-idp}

An IdP that supports this profile SHOULD signal it in its authorization server
metadata {{RFC8414}} with the following parameter:

`identity_continuation_supported`:
: OPTIONAL. Boolean, default `false`, indicating that the IdP accepts the
  `urn:ietf:params:oauth:token-type:identity-continuation` subject token type
  and issues continuation-capable ID-JAGs. Such an ID-JAG is still the
  `urn:ietf:params:oauth:token-type:id-jag` type; an IdP that sets this flag
  also lists that type in `identity_chaining_requested_token_types_supported`
  ({{I-D.ietf-oauth-identity-assertion-authz-grant}}). This flag adds only the
  continuation capability.

## Resource Authorization Server Metadata {#metadata-ras}

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
hops; the issuer-trust rule of {{validation}} accepts it directly, subject to
the IdP's issuer trust ({{security-trust-model}}). It MAY advertise
additional Continuation Assertion Issuers it authorizes to attest those hops,
so the IdP can discover them rather than be configured out of band:

`identity_continuation_issuers`:
: OPTIONAL. A JSON array of additional CAI issuer identifiers, each a
  `StringOrURI` {{RFC7519}}, that this Resource Authorization Server
  authorizes to attest hops it accepts; an empty array authorizes no
  additional CAIs. Values are compared with the assertion `iss` as exact,
  case-sensitive strings, and duplicates are ignored.

  The advertisement is a nomination only: the IdP MUST establish each
  issuer's identity and signing keys independently, and the advertisement
  alone MUST NOT establish key trust or override the IdP's tenant
  issuer-pairing policy. Because the IdP evaluates issuer trust and keys
  against its current trusted issuer and key state, removing an issuer or
  revoking its keys de-authorizes it for existing chains.

When a CAI's issuer identifier is that of an OAuth authorization server, the
IdP obtains its signing keys from the `jwks_uri` in that server's metadata
({{RFC8414}}); a CAI without such a `jwks_uri`, like any other CAI, uses
authenticated configuration. A RAS nomination MUST NOT by itself trigger
that retrieval or authorize the issuer; the IdP applies its own issuer
policy first.

The IdP MUST refresh remotely obtained keys under a bounded cache policy, so
a key removed from the JWK Set stops validating once the refresh takes
effect.

# Implementation Considerations {#implementation}

This section is non-normative. It describes ways an IdP can realize this
document's requirements; conformance depends only on the normative sections.

Continuation is per authorization context, not per call. A workload continues
once to obtain an ID-JAG for a target on behalf of one user under one chain,
then reuses the access token it redeems there while that token remains valid
and covers the requested access. Calls for another user, tenant, chain, or key
need their own continuation; reusing a token across chains would attach later
calls to the earlier chain's handle and lineage.

The first call in a context therefore costs the assertion exchange, the
continuation exchange, and the redemption, and later calls in the same
context cost nothing more while the token lasts.

Renewal is another continuation with a fresh assertion, which succeeds only
if the CAI's preconditions still hold ({{assertion-preconditions}}); an
expired access token does not by itself entitle the workload to another.

The online model has an operational price. Every continuation depends on the
IdP being reachable, and a separate CAI depends on the RAS's acceptance
evidence as well. A call path that cannot tolerate either dependency is a case
for offline attenuation only where {{decision-rule}} already allows it, that
is, where the subject and the trusted issuer stay stable across the boundary;
availability alone does not remove the need for the IdP to resolve the pairwise
subject.

The IdP retains hop records for the chain's lifetime and prunes expired or
revoked hop state. It keeps each assertion's (`iss`, `jti`) reservation for as
long as the assertion could still be presented, in state consistent enough
that concurrent presentations yield one grant; an IdP that offers idempotent
retry also keeps the fingerprint and result so that a retry recovers it
({{validation-replay}}).

Because the actor-chain depth bound counts merged lineage entries, a workload
that repeatedly continues as itself never trips it; the fan-out, rate, and
hop-count limits of {{lifecycle-limits}} bound that growth instead.

Failure paths worth testing before deployment:

* Concurrent redemption of one ID-JAG.
* A crash between issuing an ID-JAG and recording its hop.
* A policy change between two continuations.
* A credential from the wrong tenant.
* Local revocation after an assertion has been issued.

An IdP can defer materializing chain state until the first continuation,
provided the handle still resolves to the same root and envelope; deferral
does not relax the replay rules of {{validation-replay}}. Whether
the hop tree could be replaced by self-verifying handles is an open question
({{open-items}}), not a realization this document describes.

## Deployment Topologies {#deployment-topologies}

The two topologies differ only in where the CAI role sits and how it reaches
the hop's state:

| Topology | CAI role held by | CAI handle source | Fits when |
|---|---|---|---|
| Co-located | the accepting RAS | read from RAS state | one operator runs the domain |
| Separate | a separate CAI the IdP trusts for the RAS | a carrier inside the domain | the RAS is shared infrastructure, the gateway is only an RS, or keys and audit need isolation |

Co-located is the baseline: it needs no CAI configuration at the IdP beyond
the RAS's own issuer trust and no self-entry in
`identity_continuation_issuers` ({{metadata-ras}}). The RAS still advertises
the continuation profile ({{metadata}}), and the IdP still holds the RAS's
issuer, key, tenant, and issuer-pairing trust ({{security-trust-model}}).

Both topologies produce the same Identity Continuation Assertion and apply
the same CAI requirements; they differ only in how the CAI reaches the
accepted hop's state. Either CAI can issue the assertion from its token
endpoint ({{assertion-token-exchange}}): a co-located RAS takes the access
token it issued as the `subject_token`, and a separate CAI takes the token
that carries the handle. Both topologies make the same three exchanges per
continuation, the assertion exchange, the continuation exchange, and the
redemption; co-location removes configuration and trust surface, not calls,
while a separate CAI adds the carrier and the state lookup behind it.

A carrier serves either topology. A co-located RAS may place the handle in
its own access token as a key into its state, as the gateway example does
({{example-gateway}}); a separate CAI needs a carrier to receive the handle at
all. Three carriers are common:

* A Transaction Token {{I-D.ietf-oauth-transaction-tokens}} can carry the
  handle as request context ({{rationale-txn}}).
* A signed JWT access token {{RFC9068}} issued by the accepting RAS can carry
  the handle as a claim. The claim records issuance-time state and therefore
  cannot reflect a later revocation; the CAI's acceptance check
  ({{assertion-preconditions}}) covers that.
* For an opaque access token issued by the accepting RAS, its introspection
  response {{RFC7662}} can carry the handle as a member, generated when
  introspection occurs and present only when `active` is `true`.

The replay reservation ({{validation-replay}}) needs an atomic first-writer
decision so that only one concurrent request reaches ISSUED, and the IdP
retains and expires it by the same clock it uses to evaluate `exp`. An IdP
that offers idempotent retry also holds the fingerprint and result in state
consistent enough that a concurrent request under a matching fingerprint waits
for or retries that result; one that does not offer retry rejects the second
presentation and needs no more than the reservation itself.

A RAS can make the handle binding and token issuance of {{ras-processing}}
one outcome with a local transaction, or with a compensating action that
revokes a token whose binding did not commit.

The CAI accounts for retries separately from fan-out and keeps audit records of
its issuance and limit enforcement. The IdP performs end-to-end audit
correlation across a chain, while each RAS logs only its local subject.

An IdP can derive handles from an internal delegation identifier using a
keyed one-way function, provided the derived handles still satisfy rules 1
and 2 of {{chain-id}} and remain unlinkable.

# Security Considerations {#security}

This profile assumes TLS, a correct IdP subject map and root-chain envelope,
and the OAuth guidance of {{RFC9700}}. It principally addresses these
adversaries:

* an on-path attacker replaying or presenting a captured assertion
  ({{security-pop}});
* a compromised intermediate workload broadening authority, continuing the
  wrong user's chain, or raising authentication context ({{security-envelope}},
  {{security-actor-chain}});
* a compromised CAI or actor identity authority, in either deployment topology
  ({{security-trust-model}}, {{security-topology}},
  {{security-actor-issuers}});
* a party influencing the client-to-actor mapping, which is the sole
  authenticator of an actor that presents no `actor_token`
  ({{client-identity}});

* token, type, or algorithm confusion ({{security-alg}});
* a malicious Resource Server or audience attempting cross-domain correlation,
  or metadata that discloses deployment structure ({{privacy}},
  {{security-metadata}}); and
* a faulty carrier or RAS state lookup ({{handle-propagation}},
  {{security-topology}}).

## Sender Constraint and Proof of Possession {#security-pop}

A continuation assertion names the actor the IdP will treat as the chain's
current holder. As a bearer token it would let any party that captured it, in
transit, from a log, or from a compromised intermediary, continue the chain as
that actor. The assertion MUST NOT be accepted as a bearer token {{RFC7800}};
every exchange requires live proof of possession of the `cnf` key, a DPoP proof
{{RFC9449}} for the method this document defines ({{client-identity}}). A
captured assertion is therefore useless without the private key, and because the
onward ID-JAG is bound to the same proven key ({{client-identity}}) and a
continuation-aware RAS binds its access token to that key ({{ras-processing}}),
possession is demonstrated continuously from the first continuation on, not once
at issuance. A terminal RAS runs the base profile and may issue a bearer token;
the chain ends there.

The root hop is different. The root ID-JAG carries no `cnf`, and its RAS
sender-constrains the access token by its own policy, so a root hop may issue a
bearer token. A party that captures such a token can call the workload and so
induce that honest workload's continuation under the workload's own key; the
workload's proof of possession does not prevent this, because the workload is
the legitimate presenter. Because acceptance does not bound downstream authority
({{hop-activation}}), a captured root token reaches every target the envelope
permits, not only the resource the token was issued for. That is the base
profile's
bearer-token exposure at the ingress, not an exposure this profile creates, and
a RAS that sender-constrains its tokens removes it. This profile adds no proof
requirement at the root because continuation rests on the continuing actor's
key, the RAS binding, and the CAI's attestation, not on the root client's key.

At assertion issuance the client proves its own key, which the CAI binds to
the new assertion; it need not prove possession of any key bound to the
incoming subject token ({{assertion-token-exchange}}).

Replay of a captured assertion is confined to the IdP continuation exchange
and requires the actor's key. The freshness rule bounds the window, and
single-use ({{validation-replay}}) confines a consumed assertion to the one
grant it first obtained: without it, an actor whose RAS-local authorization had
lapsed, and whom the CAI would therefore refuse a fresh assertion, could keep
continuing from a consumed one, to any target the envelope permits, until it
expired. Idempotent recovery after a lost response is a client convenience and
optional ({{validation-replay}}).

## Envelope Enforcement and Offline Attenuation {#security-envelope}

The envelope bounds every target and authority. The CAI validates any offline
attenuation segment ({{assertion-issuance}}); the IdP enforces only the
envelope. Because the assertion is target-agnostic, a permitted actor may
select any target within that ceiling.

An envelope that records an authorization basis instead of listing targets
({{root-establishment}}) admits every target the basis permits at request time;
how broad that is depends on the policy the basis references. A gateway that
chooses its upstream at request time is the intended case and also a
confused-deputy surface, since a compromised or misdirected workload can steer
continuation to any target the basis admits.

The IdP's per-target evaluation, the tenant policy that forms the basis, and
the per-authorization fan-out limits bound the damage; a deployment whose
targets are known at establishment gains more protection by enumerating them,
since an enumerated envelope never gains a target ({{root-establishment}}).

Reclassifying a service changes the authority of every open chain whose basis
reads that classification ({{root-establishment}}), so a basis should name a
class the tenant manages for this purpose, such as services marked eligible
for agent continuation, rather than an ambient classification.

Continuers and limits are read as policy stands ({{root-establishment}}), so
adding a continuer admits a new actor into every running chain that policy
governs. A tenant manages that list with the same care as a basis.

Wrong-handle association can continue the wrong user's bounded chain. The
RAS-bound state establishes the authoritative association between the request
and the handle, whether the CAI reads it directly or through a carrier derived
from that state ({{handle-propagation}}); the CAI rejects substitution
({{assertion-preconditions}}).

Because the CAI issues only for an actor it is authoritative to associate with
the accepted authorization ({{assertion-issuance}}), a party that merely holds a
handle cannot bypass the RAS-acceptance path.

A CAI MUST derive a scheduled continuation from durable RAS task
authorization, not from a scheduler-held handle, which would become a durable
bearer-like credential outside the per-call key proof and RAS binding that
gate every other use. The scheduler holds only a task identifier; each
authenticated run re-derives the handle from active task state and still
requires an assertion from a trusted CAI.

Downstream resources may gate access on authentication strength (`acr`) or
methods (`amr`); if continuation could raise those claims, an actor could reach
a step-up-gated resource the user never authenticated strongly enough for.
Authentication context therefore comes only from the root envelope, copied
unchanged into onward ID-JAGs ({{onward-id-jag}}).

## Trust in Actor Identity Authorities {#security-actor-issuers}

The actor's identity authority, the `iss` of its canonical actor identity
({{client-identity}}), vouches for the current actor to the IdP, whether
through an `actor_token` it issued or through the IdP's mapping of the client
credential. A rogue or over-scoped authority is an impersonation vector: a
party controlling one could name an actor in another domain or tenant and
continue that actor's chains. The current-actor and issuer-trust rules of
{{validation}} require an authority trusted for the actor's own domain and
tenant and paired with the CAI, and an accompanying CAI assertion does not
relax that: CAI attestation of the hop and authentication of the actor are
independent checks ({{security-trust-model}}); neither substitutes for the
other.

## Conjunctive Trust and Issuer Pairing {#security-trust-model}

A continuation requires all of these, and no one of them suffices alone:

* a CAI the IdP trusts for the presented hop's accepting Resource
  Authorization Server, which attests the chain-to-actor transition (the
  issuer-trust rule of {{validation}});
* the actor's identity authority, trusted for the current actor's domain and
  tenant, which vouches for the actor through an `actor_token` or the mapping
  of its client credential (the current-actor rule of {{validation}});
* live proof of possession of the confirmed key (the current-actor rule of
  {{validation}}); and
* the IdP's own envelope and current-actor policy (the envelope-containment
  rule of {{validation}}).

The IdP MUST authorize CAI and actor identity authority pairings per tenant;
separate trust in each is insufficient. Tenant determination MUST derive from
authenticated material, not requester-supplied input. The IdP MUST scope CAI
trust by issuer, keys, tenant, and the RAS it attests for.

The IdP MAY learn additional issuers from the RAS's advertised
`identity_continuation_issuers` ({{metadata}}). That advertisement is a
nomination only and establishes no issuer or key trust ({{metadata-ras}});
absent it, additional mappings are configured out of band.

## Topology and Trust {#security-topology}

Accepting the RAS's own identifier under the issuer-trust rule establishes no
issuer, key, tenant, or issuer-pairing trust. Separating the CAI isolates keys
and components but creates no protocol-level quorum: the IdP still sees one
signed attestation ({{deployment-topologies}}).

A workload that obtains the assertion by exchanging the access token or
Transaction Token it holds for the call ({{assertion-token-exchange}})
presents nothing it did not already hold; what it gains is the CAI's
attestation, gated by policy and the acceptance check.

A compromised RAS can fabricate acceptance state in either topology, since a
separate CAI reads that state as authoritative; a compromised separate CAI can
additionally attest a hop the RAS refused. In both topologies, RAS-local
authorization revocation after issuance, which the IdP cannot observe, leaves
the assertion valid for its remaining lifetime; a separate CAI adds any delay
in RAS state reaching it, and where the RAS's acceptance evidence is a
self-contained token ({{assertion-preconditions}}) fresh issuance continues
for that token's remaining lifetime, on top of the assertion's own lifetime,
so a tenant that needs faster withdrawal configures a live recheck. The root
envelope still bounds the result.

## Actor Chain Integrity {#security-actor-chain}

The `act` lineage records who has acted in the delegation. A compromised actor
could try to forge it, to hide its own identity, impersonate a more privileged
prior actor, or fabricate a delegation that never happened. This profile
denies that by construction: an assertion names only the current actor, and
the IdP derives the onward lineage from its own hop records
({{onward-id-jag}}). The IdP MUST reject any mismatch between the current
actor and the assertion's `act`.

Because lineage derives from IdP-held state rather than assertion input, a
party cannot rewrite history it does not control; offline-attenuation
segments, which the IdP does not observe, do not enter lineage. Lineage is
also disclosed, not exhaustive: policy may narrow it ({{onward-id-jag}}), so a
rule such as "deny if a given actor ever participated" cannot be enforced from
`act` alone.

## Token, Type, and Algorithm Confusion {#security-alg}

An attacker may try to pass one token type off as another, downgrade the
signature algorithm, or steer verification to a key it controls. The IdP
applies these verification rules:

* The IdP MUST verify `typ`, reject `alg=none` and symmetric algorithms, and
  allowlist asymmetric algorithms.
* The IdP MUST select keys from trusted issuer configuration; `kid` MAY select
  among them.
* The IdP MUST NOT trust assertion `jku`, `x5u`, embedded `jwk`, or other
  supplied key material.

## Metadata Disclosure {#security-metadata}

Advertising `identity_continuation_issuers` ({{metadata-ras}}) in publicly
readable authorization server metadata reveals which additional CAIs a
Resource Authorization Server authorizes to attest its hops, and can thereby
disclose federation topology, tenant relationships, and deployment structure.
A deployment whose CAI relationships are sensitive SHOULD omit the
advertisement and convey the nomination out of band or through
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
  present only when `active` is `true`; {{handle-propagation}} constrains its
  exposure within the issuing server's trust domain

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

## Actor Identity and Target `client_id` {#rationale-client-id}

`act` and `client_id` answer different questions. `act` names the actor doing
the work: the workload the IdP authenticated as its OAuth client on the
continuation exchange, recorded in the lineage. `client_id` names the OAuth
client that may redeem the grant at the target RAS. Both appear because the
artifact this profile produces is an ID-JAG, which the base profile defines as
a grant a registered client presents; a target RAS authenticates that client
and applies its policy by `client_id`, and need not understand this
continuation profile. So before continuing to a target, the current actor
needs a client identity resolvable at that target's RAS ({{token-exchange}}),
and the IdP places it in the onward ID-JAG.

That is compatibility with ID-JAG, not the profile's model of delegation: a
workload with a strong identity and key but no registration at a target
cannot receive a grant for it because nothing there could redeem the grant,
not because it is any less the actor. The two identifiers often coincide, as
when a gateway registers everywhere under one name, but the profile keeps
them distinct so that lineage records who acted while the grant records who
may redeem.

The IdP's mapping from client to canonical actor identity does not depend on
the client authentication method: a client that authenticates with an
{{RFC7523}} client assertion may have a workload identity in another namespace
as its canonical actor identity. A sender-constrained JWT from an issuer the
tenant trusts is the natural form of a workload credential and, when it
satisfies both profiles, serves as both client assertion and `actor_token`
({{client-identity}}). The shared key

across the actor's credential, the assertion, and the onward ID-JAG is a
consequence of a single proof method, not a property continuation requires:
continuation requires continuity of the actor, and a future confirmation
method could bind these artifacts to different proven keys ({{open-items}}).

The `may_act` claim of {{RFC8693}} does not remove the need for this profile.
It lets an issuer state, inside a token the target already trusts, which party
may later act for the token's subject: an authorization to act, made in
advance. Continuation needs the opposite: a party the previous token never
named must obtain a new token for an audience whose pairwise subject only the
IdP can produce. `may_act` could constrain who may continue; it cannot mint
the subject, so the IdP exchange remains.

## Why Not a Transaction Token {#rationale-txn}

A Transaction Token {{I-D.ietf-oauth-transaction-tokens}} carries request
context within one trust domain. The assertion crosses from that domain to
the IdP, is single-use, and carries neither the target subject nor general
request context. It may be derived from Transaction Token context, but is not
a Transaction Token profile.

## Why Not a Cross-Domain Propagation Token {#rationale-propagation}

The choice follows {{decision-rule}}: a pairwise-subject boundary can be
crossed only by the IdP, which the target trusts to name the user, and IdP
exchange permits current-state and envelope checks at every hop.

Direct propagation instead fits deployments with a global subject, shared
issuer trust, and no need for mid-chain IdP revocation, such as a single
SPIFFE-style trust domain (one workload-identity namespace with no
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

The signed assertion lets the CAI attest the authenticated actor, key,
accepted hop, and any intra-domain policy checks that the IdP cannot observe:
acceptance is a state of the RAS, and the transaction binding is domain-local,
so the assertion is the IdP's only evidence of either.

The IdP still authenticates each current actor and checks its authorization
({{client-identity}}, {{validation}}); what the assertion removes is not that
step but any need for the IdP to reach into another domain's state. The
assertion does not authorize target or scope ({{assertion-issuance}}). Where
domain-local attestation is unnecessary, a recipient-bound direct grant
remains a possible simplification, at the cost of the acceptance evidence and
transaction checks that only a party in that domain can provide.

## Why Asymmetric Signing Only {#rationale-alg}

This profile requires asymmetric signing and forbids encryption and nested
signing ({{names}}), tighter than RFC 8725 {{RFC8725}}, which also permits
verified symmetric algorithms. The restriction is deliberate: asymmetric
verification avoids distributing a shared secret across domains and the
key-confusion risk a symmetric key between the CAI and IdP would create
({{security-alg}}); TLS on every hop and the assertion's minimal contents make
encryption unnecessary; and a single compact signed form removes an
interoperability choice between issuers and verifiers.

## Boundary of the Profile {#rationale-boundary}

Continuation serves the Resource Authorization Servers that trust the common
IdP. A target outside that circle fails in one of two ways: where the IdP
holds no pairwise subject for it or no authorization basis covers it, the
exchange fails with `invalid_target` (the current-actor and
envelope-containment rules of {{validation}}); where the IdP could issue an
ID-JAG regardless, the target rejects it, since it does not trust the issuer.
That is the profile's edge, not a deployment error.

A separate identity-chaining profile can cross it under a bilateral
agreement; for example, a workload can present its Transaction Token to its
own domain's authorization server under
{{I-D.fletcher-transaction-token-chaining-profile}} for a minimized grant to
the partner. The Transaction Token and the handle stay in the domain.

The profile's other boundary is semantic. It establishes who is continuing
what:

* the user;
* the actor;
* the lineage; and
* the accepted authorization the request descends from.

It does not establish why. Whether a requested action belongs to the work the
user or tenant sanctioned is a policy question the IdP answers at each
continuation, with the envelope and any authorization details as its inputs;
nothing in the chain itself carries purpose, and an implementation that reads
the envelope as a complete agent authorization model is mistaken.

Acceptance at a RAS is not a ceiling for later targets ({{hop-activation}}),
because a scope granted at one audience says nothing about a scope at
another. A deployment that
wants the work itself to narrow downstream authority expresses that in the
governing authorization, not in RAS scopes.

## The Test for a Requirement {#rationale-invariants}

What must be true for the IdP to issue the next ID-JAG safely is a short list,
and a requirement that serves none of its items is deployment hardening rather
than part of the profile:

1. the hop being continued was legitimately accepted;
2. a party trusted for that hop says the current actor now holds it;
3. the IdP can authenticate that actor;
4. the actor proves possession of the key bound to this continuation;
5. the IdP independently authorizes the actor, the target, and the requested
   authority; and
6. the chain is within its governing authorization and lifecycle.

The profile's requirements follow from these. Acceptance evidence by the RAS's
own semantics serves item 1 and the CAI's attestation item 2; the canonical
actor identity serves item 3; sender constraint of the assertion and the onward
ID-JAG serves item 4; the envelope under current policy serves item 5, so that
the effective authority at any continuation is the recorded continuation
authorization evaluated under current policy, never more; and the anchor serves
item 6. Single-use of an assertion serves item 1 across time: a consumed
assertion must not keep authorizing after the RAS's acceptance evidence would no
longer support a fresh one ({{validation-replay}}). Where this document offers a
choice, such as the form of acceptance evidence or a separate actor token, the
choice is between mechanisms that satisfy the same item.

# Examples {#examples}

This non-normative appendix illustrates three deployment shapes: a tool
gateway that selects its upstream at request time ({{example-gateway}}),
a SaaS chain across three domains with separate CAIs ({{example}}), and an
unattended background agent ({{example-background}}). The gateway example
comes first because it is the baseline deployment: one domain runs RAS and
CAI together, and the handle travels in the access token.

Message sequences are vertical lifelines with time flowing downward. Each
example identifies its deployment topology before describing the flow.

* Tags mark where payload and state blocks travel: "On the wire" crosses a
  trust boundary, "Intra-domain context" stays within one trust domain, and
  "Server-side state" is never transmitted.
* Continuation handles are numbered H0, H1, and so on, one per hop.
* JWTs are shown as decoded payloads with JOSE headers and signatures
  omitted; client authentication is omitted except where shown; proof of
  possession uses DPoP. The gateway example shows the floor, with a root
  client that never proves a key; the SaaS example shows a root RAS that
  sender-constrains its access token by its own policy
  ({{root-establishment}}). From the first continuation on, every RAS binds
  its token to the proven key ({{ras-processing}}).


## Gateway Example (Co-located RAS and CAI) {#example-gateway}

An agent runtime, AgentApp, calls a tool gateway on behalf of Alice. The
gateway decides at request time which upstream API a tool call needs, here a
wiki, so AgentApp knows the gateway's audience but not the eventual upstream,
and the gateway holds no assertion of Alice's identity addressed to it. This
example shows the gateway obtaining an audience-specific ID-JAG for the wiki
without ever seeing Alice's credential. It is the baseline deployment of this
profile and the one a gateway vendor implements.

Topology: co-located. GatewayRAS is the Resource Authorization Server (RAS)
that accepts the ID-JAG and also the Continuation Assertion Issuer (CAI) for
the hops it accepts, so no separate CAI is involved. It carries the
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

The exchanges below presuppose the following configuration. Every item is an
ordinary OAuth or ID-JAG registration except the three the profile adds: the
IdP's continuation policy, its trust in GatewayRAS as an assertion issuer, and
GatewayRAS's advertisement of the continuation profile.

| Party | Provisioned before the flow | Specified in |
|---|---|---|
| IdP | `agent-app` and `tool-gateway` registered as confidential OAuth clients in `tenant-123`; `https://gateway.example/` authorized as the issuer of `tool-gateway`'s credential; GatewayRAS's issuer `https://ras.gateway.example/` and its signing keys trusted for the hops it accepts; GatewayRAS and `https://gateway.example/` authorized as a CAI and actor identity authority pair for `tenant-123`, since trust in each alone is not enough; tenant policy granting agents read access to productivity tools and permitting `tool-gateway` to continue; `identity_continuation_supported` advertised; `tool-gateway`'s client identifier at WikiRAS recorded for the onward ID-JAG's `client_id` | {{client-identity}}, {{security-trust-model}}, {{root-establishment}}, {{metadata-idp}} |
| GatewayRAS | the continuation grant profile advertised; the IdP trusted as ID-JAG issuer; `tool-gateway` registered as an OAuth client of GatewayRAS and associated with the resource `https://gateway.example/` it operates, which is what lets it obtain assertions for hops bound to access tokens presented to that resource | {{metadata-ras}}, {{assertion-preconditions}} |
| `tool-gateway` | one key pair, used for its credential's `cnf`, its DPoP proofs, and the assertion's `cnf`; a credential issued at `https://gateway.example/` | {{client-identity}} |
| WikiRAS | `tool-gateway` registered as a client, as the base profile requires of any ID-JAG presenter; the IdP trusted as ID-JAG issuer | base profile |

The credential that `tool-gateway` presents to the IdP as its
`client_assertion` ({{example-gateway-continue}}) is, in this example, a JWT
issued by the gateway domain and bound to the gateway's key; it resolves to the
canonical actor identity, so no separate `actor_token` is needed. Any
sender-constrained credential the IdP accepts under {{client-identity}} would
serve; this choice is illustrative. It is addressed
to the IdP alone: the earlier exchange at GatewayRAS
({{example-gateway-ica}}) uses a separate client assertion addressed to
GatewayRAS, as {{RFC7523}} requires.

~~~ json
{
  "iss": "https://gateway.example/",
  "sub": "tool-gateway",
  "aud": "https://idp.example/",
  "cnf": {
    "jkt": "base64url-tool-gateway-key-thumbprint"
  },
  "iat": 1710000230,
  "exp": 1710000290,
  "jti": "tg-cred-01"
}
~~~

Its `iss` and `sub` are the values the assertion's `act` and the onward
ID-JAG's lineage carry ({{assertion-claims}}).

### Root Exchange {#example-gateway-root}

AgentApp exchanges Alice's ID Token, whose `sid` anchors the chain to her IdP
session ({{root-establishment}}), for an ID-JAG addressed to GatewayRAS, the
one audience it knows:

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

Tenant policy permits continuation under the governing authorization, so the
IdP establishes a chain and embeds H0 ({{root-establishment}}). Because the
upstreams are unknown at root time, the envelope records an authorization
basis, the tenant's policy for agent access to its productivity tools, rather
than enumerated targets, and that policy permits `tool-gateway` to continue.

The envelope records that basis by reference rather than enumerating targets,
so nothing about the targets is frozen. The IdP applies the tenant's policy as
it stands at each continuation: which services the tenant classifies as
productivity tools, what access it allows agents to them, and whether
`tool-gateway` may continue. A wiki the tenant adds to that class after this
exchange is therefore admitted in {{example-gateway-continue}} without a new
chain, and one it removes is refused at the next continuation.

What is fixed is the root exchange: Alice, her authentication context,
`agent-app` as root actor, and the session that anchors the chain
({{root-establishment}}). {{security-envelope}} names the exposure live
policy creates and how to scope a basis.

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

AgentApp redeems the ID-JAG at GatewayRAS with the jwt-bearer grant, exactly
as for any ID-JAG, and nothing in this profile asks it to sender-constrain the
exchange; GatewayRAS issues a bearer access token here, its own policy choice
({{ras-processing}}):

~~~
POST /token HTTP/1.1
Host: ras.gateway.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
&assertion=<the ID-JAG above>
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<agent-app JWT>
~~~

GatewayRAS advertises the continuation grant profile ({{metadata-ras}}) and so
recognizes the handle. It binds H0, the issuing IdP, and the tenant it
associates with that IdP to the authorization state behind the access token,
in the same outcome that issues the token ({{ras-processing}}); the ID-JAG
carried no `cnf`, so there is no key to record.

Server-side state at GatewayRAS:

~~~ json
{
  "identity_continuation_handle": "Qm7zXu2VtL9pKe4RaW1nHc",
  "status": "ACCEPTED",
  "continuation_permitted": true,
  "authorization_state": "gw-authz-7a1e",
  "idp": "https://idp.example/",
  "tenant": "tenant-123",
  "client_id": "agent-app",
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
  "iat": 1710000210,
  "exp": 1710000810,
  "jti": "at-gw-0001"
}
~~~

AgentApp calls the gateway with this token. It chooses which token to present,
and with it which authorization, but cannot alter the handle inside the token.
A RAS that sender-constrains its tokens closes the bearer-ingress exposure of
{{security-pop}}; {{example}} shows that variant.

### ToolGateway Obtains the Assertion {#example-gateway-ica}

ToolGateway validates the access token as any resource server would.
Resolving the tool call, it selects the wiki as the upstream. To
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

The DPoP proof on this request proves the `tool-gateway` key, which GatewayRAS
places in the assertion's `cnf`; had the access token been sender-constrained,
its own `cnf` would not be matched against this proof
({{assertion-token-exchange}}).

GatewayRAS confirms the token is its own, unexpired, and addressed to
`https://gateway.example/`, the resource `tool-gateway` is registered to
operate; reads H0 from it and rechecks that the binding is still active;
confirms that the binding records continuation as permitted; and issues the
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
  "jti": "n3Vt7Jq2Xd5PbR9wLc1eKs"
}
~~~

### ToolGateway Continues to WikiRAS {#example-gateway-continue}

ToolGateway resolves the token endpoint of `https://idp.example/`, the
`audience` it was given, from that IdP's metadata and presents the assertion
there as the `subject_token` of a
continuation exchange, with its client credential and a DPoP proof of the same
key, requesting an ID-JAG for WikiRAS. `tool-gateway` is a registered client
of the IdP whose sender-constrained credential resolves to its canonical actor
identity, so it presents no separate `actor_token` ({{client-identity}}):

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
&client_assertion=<sender-constrained tool-gateway credential>
~~~

The IdP validates the exchange ({{validation}}):

* Issuer trust: the assertion is signed by GatewayRAS, the RAS recorded for
  H0's hop, which the issuer-trust rule accepts directly.
* Hop status: H0 names an accepted hop on an active chain.
* act and cnf match: the `act` claim names `tool-gateway`, the authenticated
  client, and the DPoP proof matches `cnf`.
* Basis containment: the wiki target falls within the recorded basis, the
  tenant's policy for agent access to productivity tools, which is how a
  target nobody enumerated at root time, even one classified as a
  productivity tool only after the chain was established, is admitted;
  `wiki.read` is evaluated against that policy, not against the root
  `tools.invoke` scope.

The IdP resolves Alice's wiki subject and issues the ID-JAG with H1 and
`tool-gateway` atop `agent-app` in the lineage.

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

A target outside the envelope fails with `invalid_target`; the chain stays
continuable and only that tool call fails ({{error-response}}).
{{example-dynamic}} shows the same failure for an enumerated envelope.

### WikiRAS Redeems an Ordinary ID-JAG {#example-gateway-terminal}

ToolGateway redeems the ID-JAG at WikiRAS with the jwt-bearer grant and a
DPoP proof of the same key. WikiRAS implements nothing from this profile: it
validates the ID-JAG as the base profile requires, ignores the handle, and
issues an access token without binding H1 ({{ras-processing}}). ToolGateway
calls WikiAPI as Alice's wiki subject and returns the result to AgentApp.

Further calls to WikiAPI for Alice reuse this access token while it remains
valid ({{implementation}}); each further upstream a tool call needs repeats
the exchange of {{example-gateway-ica}} and creates a sibling hop under H0
(H2, and so on).

### What a Gateway Implements {#example-gateway-checklist}

The gateway domain adds two things to an ordinary OAuth deployment:

* GatewayRAS binds the handle when it redeems a continuation-capable ID-JAG,
  atomically with issuing the access token and once per grant, records whether
  continuation is permitted, advertises the continuation grant profile
  alongside the base grants, places the handle in the access token, and issues
  assertions from its token endpoint by Token Exchange with `aud` taken from
  the binding and no refresh token ({{ras-processing}}, {{assertion-issuance}},
  {{metadata-ras}}).
* `tool-gateway` exchanges the access token it received for an assertion, then
  presents that assertion with its client credential and a DPoP proof to the
  IdP for the next ID-JAG, presenting each assertion once
  ({{validation-replay}}).

Both presuppose registrations that ID-JAG already requires: `tool-gateway` is
an OAuth client of the IdP, holding the key its credential and DPoP proofs use,
and a client known to WikiRAS, which the onward ID-JAG names as `client_id`
({{token-exchange}}).

Outside the domain, AgentApp and WikiRAS change nothing: AgentApp performs a
base ID-JAG exchange, proves no key anywhere, and never shares Alice's
credential, and WikiRAS runs the base profile unmodified. The
IdP carries the rest: it
establishes the chain and evaluates each continuation
({{root-establishment}}, {{token-exchange}}).

## SaaS Chain Example (Separate CAI and Transaction Token Carrier) {#example}

A user's request crosses three SaaS domains: ExpenseApp calls ExpenseAPI,
whose workload calls TravelAPI, whose workload calls BookingAPI. Each domain
has its own Resource Authorization Server, and all trust one enterprise IdP
at `https://idp.example/`. Compared with the gateway example, this one uses
a separate CAI and a Transaction Token carrier; {{example-differences}}
lists what differs.

Topology: separate CAI with a Transaction Token carrier.

* Expense domain (`expenses.example`): client `expense-app`; ExpenseRAS,
  Expense TTS, and Expense CAI at `ras.`, `tts.`, and `cai.expenses.example`;
  and the workload `expense-service` behind ExpenseAPI.
* Travel domain (`travel.example`): TravelRAS, Travel TTS, and Travel CAI,
  and the workload `travel-service` behind TravelAPI.
* Booking domain (`booking.example`): BookingRAS in front of BookingAPI. It
  runs the base ID-JAG profile and is terminal in this chain.

The user has a pairwise subject at each RAS, which only the IdP can map. H0 is
bound at ExpenseRAS, H1 at TravelRAS; H2 reaches BookingRAS, which never binds
it.

Hop 0 roots the chain at ExpenseRAS and carries H0 to the workload:

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

Hop 1 continues to TravelRAS; hop 2 repeats it from Travel to Booking:

~~~
ExpenseService    Expense CAI     IdP       TravelRAS  Travel TTS/API
       |               |           |            |             |
       |-exchange TT-->|           |            |             |
       |<--assertion---|           |            |             |
       |--assertion, actor, DPoP-->|            |             |
       |<--------ID-JAG H1---------|            |             |
       |-------jwt-bearer: ID-JAG + DPoP------->| bind H1     |
       |<------------------AT2------------------|             |
       |-------------call TravelAPI: AT2 + DPoP-------------->|
       |               |           |            | TT with H1 to workload
~~~

### Root Exchange (H0 at ExpenseRAS) {#example-first-hop}

ExpenseApp exchanges the user's ID Token, whose `sid` anchors the chain to the
IdP session, for an ID-JAG addressed to ExpenseRAS:


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

Enterprise policy permits continuation to Expense, Travel,
and Booking by designated workloads, so the IdP records an envelope that
enumerates the targets ({{root-establishment}}); the gateway example shows the
other form, an authorization basis with no enumerated targets.

Server-side state:

~~~
(https://ras.expenses.example/, https://api.expenses.example/)
    permitted scopes: expenses.read

(https://ras.travel.example/, https://api.travel.example/)
    permitted scopes: trips.read

(https://ras.booking.example/, https://api.booking.example/)
    permitted scopes: stays.book
~~~

The IdP creates the root hop H0 and embeds it in the ID-JAG ({{chain-id}}).

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

  "iat": 1710000005,
  "exp": 1710000305,
  "jti": "idjag-expense-01"
}
~~~

### ExpenseRAS Binds H0; Expense TTS Carries It {#example-context}

ExpenseApp redeems the ID-JAG at ExpenseRAS for an access token, AT1, exactly
as for any ID-JAG. ExpenseRAS recognizes the continuation profile and binds
H0 to the authorization state behind AT1 in the same outcome that issues it
({{ras-processing}}). The handle stays in this record; AT1 does not carry it.

Server-side state at ExpenseRAS:

~~~ json
{
  "identity_continuation_handle": "kW4uJ8pTe2NxA6rQvD1zYs",
  "status": "ACCEPTED",
  "continuation_permitted": true,
  "authorization_state": "at1-authz-2f9c",
  "client_id": "expense-app",
  "bound_at": 1710000010
}
~~~

ExpenseApp calls ExpenseAPI with AT1. The Expense TTS resolves AT1 against that
record over its own-domain interface with ExpenseRAS, derives H0, and issues a
Transaction Token for `expense-service`, the workload that completes the
request ({{handle-propagation}}). Neither ExpenseApp nor `expense-service`
supplies H0.

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

The Transaction Token stays inside `expenses.example`. The
`tctx.identity_continuation` encoding is illustrative; the profile standardizes
no carrier schema.

### ExpenseService Obtains the Assertion {#example-ica}

`expense-service` presents the Transaction Token as the `subject_token` at
Expense CAI's token endpoint, authenticating as a client and proving its own
key ({{assertion-token-exchange}}):

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

Expense CAI verifies the Transaction Token for its domain, reads H0 from it,
rechecks with ExpenseRAS that the binding is still active and records
continuation as permitted, and issues the assertion. The IdP
accepts Expense CAI's assertions for hops ExpenseRAS accepts because it trusts
Expense CAI for that RAS, through `identity_continuation_issuers` or tenant
configuration, and independently trusts its issuer, keys, tenant, and actor
identity authority pairing ({{metadata}}, {{security-trust-model}}). The
response's `audience`, `https://idp.example/`, tells `expense-service` where
to present it.

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

`expense-service` presents the assertion to the IdP with its credential and a
DPoP proof of the same key, requesting an ID-JAG for TravelRAS. That one
sender-constrained credential serves as both client assertion and
`actor_token` under the dual-use rule ({{client-identity}}):

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
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<the same expense-service credential>
~~~

The IdP validates the exchange ({{validation}}): Expense CAI is trusted for
ExpenseRAS, the RAS recorded for H0; H0 names an accepted hop on an active
chain; `act` names `expense-service`, the authenticated client, and the DPoP
proof matches `cnf`; and TravelRAS, TravelAPI, and `trips.read` match the
envelope's Travel entry. The IdP never calls ExpenseRAS; the assertion is its
evidence of acceptance ({{hop-activation}}). It resolves the user's Travel
subject, creates H1 as a child of H0, and places `expense-service` atop the
root actor `expense-app` in the lineage ({{onward-id-jag}}).

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

`expense-service` redeems the ID-JAG at TravelRAS for AT2, and TravelRAS binds
H1 exactly as ExpenseRAS bound H0. The Travel TTS derives H1 into a Transaction
Token for `travel-service`; only the handle differs from the Expense token:

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

`travel-service` obtains an assertion for H1 from Travel CAI and exchanges it
for an ID-JAG with `audience=https://ras.booking.example/`,
`resource=https://api.booking.example/`, and `scope=stays.book`, all within the
envelope's Booking entry. It presents its own sender-constrained credential as
both client assertion and `actor_token`, as `expense-service` did
({{client-identity}}). The IdP creates H2 under H1 and extends the lineage:

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

BookingRAS is terminal: it redeems the ID-JAG under the base profile, ignores
H2, issues AT3, and binds nothing ({{ras-processing}}). `travel-service` calls
BookingAPI with AT3. A target outside the trust circle cannot be reached this
way ({{rationale-boundary}}).

### What Differs from the Gateway Example {#example-differences}

* The CAI is a separate service in each continuing domain, which the IdP
  trusts for that domain's RAS by configuration or nomination; a co-located
  RAS needs no such record, though the IdP trusts its issuer, keys, tenant,
  and issuer pairing in the same way.
* The handle never enters an access token; a Transaction Token Service derives
  it from the RAS binding for each request, and the workload presents that
  token to obtain the assertion.
* The envelope enumerates its targets, so each continuation is
  matched against an entry rather than evaluated against a policy basis.
* Two continuing domains produce a lineage three entries deep, two
  continuation actors above the root actor, before the terminal hop.
* ExpenseRAS chooses to sender-constrain the root access token, so ExpenseApp
  presents DPoP proofs at the RAS and on its API call; that is RAS policy
  under the base profile, and the gateway example shows the floor without it.
  The continuation-aware RASes downstream bind their access tokens to the
  proven key because their ID-JAGs carry `cnf`, which {{ras-processing}}
  requires.

## Background Agent Example (Scheduled Continuation) {#example-background}

Alice sets up a daily calendar briefing and is absent at every run. Compared
with the SaaS chain example, this one anchors the chain to a grant rather
than a session; {{example-background-differences}} lists what differs. It
also shows a target that tenant policy still excludes being refused.

Topology: separate CAI with a Transaction Token carrier.

* Platform domain (`platform.example`): workload `briefing-agent`; PlatformRAS,
  Platform TTS, and Platform CAI in front of TaskAPI; and the Scheduler, an
  internal component that holds only the task identifier and triggers each
  run.
* Calendar domain (`calendar.example`): CalendarRAS in front of CalendarAPI.
  It is terminal in every run.
* Mail domain (`mail.example`): MailRAS in front of MailAPI, reached only in
  {{example-dynamic}}; likewise terminal.

H0 is bound at PlatformRAS to the task authorization and outlives Alice's
session; each run receives a fresh child of H0 for its terminal target.

### Setup: Anchoring the Chain to a Grant {#example-background-setup}

Alice authorizes "summarize my calendar every morning." Because the task must
outlive her session, `briefing-agent` presents a refresh token from a grant
that permits continuation as the root exchange's subject token, so the chain
is anchored to that grant rather than to her session ({{root-establishment}},
{{lifecycle}}). The root ID-JAG targets PlatformRAS, and the envelope records
both that target and the Calendar target the task needs.

Server-side state (root envelope excerpt):

~~~
(https://ras.platform.example/, https://api.platform.example/tasks)
    permitted scopes: task.manage

(https://ras.calendar.example/, https://api.calendar.example/)
    permitted scopes: calendar.read
~~~

The exchange and RAS binding follow the pattern of {{example-first-hop}} and
{{example-context}}. PlatformRAS binds H0 to a durable task authorization that
it keys by its own task identifier; the record holds no bearer credential.

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

The Scheduler never receives H0 or any user, chain, or bearer credential;
`task-123` identifies a row in PlatformRAS's own state and means nothing
outside the platform.

### Each Run: Deriving H0 from Task State {#example-background-run}

Each run begins with no user present, so H0 comes from state the platform
holds:

~~~
 Scheduler   BriefingAgent      Platform TTS
     |              |                 |
     |---trigger--->|                 | task-123
     |              |-task-123+proof->|
     |              |                 | verify proof + task; derive H0
     |              |<-fresh TT(H0)---|
~~~

The task identifier is not a secret and does not authorize a run. The
Scheduler's trigger authenticates and carries only `task-123`; BriefingAgent
authenticates to the Platform TTS and proves possession of its key; and the
TTS, after confirming that `task-123` is active and that BriefingAgent is its
designated actor, derives H0 into a fresh Transaction Token
({{handle-propagation}}). Neither the Scheduler nor BriefingAgent selects H0.

### Each Run: Continuing to CalendarRAS {#example-background-continue}

With H0 in hand, the run makes the three exchanges of any continuation, the
assertion, the continuation, and the redemption:

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

BriefingAgent exchanges the Transaction Token at Platform CAI's token
endpoint ({{assertion-token-exchange}}) and presents the assertion to the
IdP the response's `audience` names, with its actor credential and a DPoP
proof. Before issuing, Platform CAI authenticates `briefing-agent`, verifies
its key and transaction, and rechecks that PlatformRAS's H0 authorization
remains active.

The assertion and onward ID-JAG have the shapes shown in {{example-ica}} and
{{example-chained}}. CalendarRAS is terminal and issues the access token
without binding the child hop. Each run's child is a sibling, not a
descendant, of the previous run's child.

### A Dynamic Target {#example-dynamic}

Suppose the platform later extends the briefing to include unread mail,
which requires `https://api.mail.example/` behind `https://ras.mail.example/`,
a target nobody named when Alice created the task. The envelope enumerates the
Platform and Calendar targets, so a run's continuation exchange presenting H0
for that audience fails, and adding Mail to tenant policy afterward does not
change that: an enumerated envelope never gains a target
({{root-establishment}}).

~~~
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "error": "invalid_target"
}
~~~

For a deployment that expects dynamic targets, the envelope instead records
an authorization basis, for example the tenant's policy granting the
briefing agent read access to productivity services, and the IdP evaluates
each dynamic target against that policy as it stands at continuation time
(the envelope-containment rule of {{validation}}).

The same exchange then succeeds only if read access to the mail service is
within that policy and it permits `briefing-agent` to reach it; a request
for `mail.send`, which that policy still excludes, fails with
`invalid_scope`. A target-specific failure leaves the chain continuable for
other authorized targets.

### What Differs from the SaaS Chain Example {#example-background-differences}

* The root subject token is a refresh token, so the chain anchors to a grant
  and survives logout ({{lifecycle}}).
* H0 is bound to a durable task authorization that PlatformRAS keys by task
  identifier; the Scheduler holds only that identifier, which is not a secret
  and authorizes nothing.
* Every run derives H0 afresh from active task state and obtains a new
  assertion. Stealing the task record exposes H0 but not the agent key, and
  continuation still needs a fresh assertion while the authorization is
  active.
* Each run creates a sibling child of H0 rather than a descendant of the
  previous run's child.

This pattern requires a user-present setup event to root the chain. Where no
such event exists, for example an administratively mandated agent acting for
users who never authorized it, there is no delegation to continue and this
profile does not apply; such deployments need a differently rooted
authorization, such as administrative policy at the IdP, which is out of scope
for this document.

# Open Items for Working Group Discussion {#open-items}

This non-normative appendix lists unresolved design questions.

\[\[ To be removed before publication as an RFC ]]

1. **Signed assertion versus a recipient-bound direct profile.**
   Could the IdP bind a continuation credential to an intended actor, actor
   class, trust domain, or key and accept it with client authentication,
   sender-constrained `actor_token`, and live key proof? Are the CAI's actor/key
   attestation and domain-local gate worth the added trust configuration
   ({{rationale-grant-type}})?

2. **Mutual-TLS confirmation.** Should this profile define a mutual-TLS
   confirmation method (`x5t#S256`, {{RFC8705}}) alongside `jkt`, and should
   it do so together with ID-JAG ({{client-identity}})? With a second proof
   method available, may the `actor_token` and the onward ID-JAG be bound to
   a different proven key than the assertion?

3. **A client establishment parameter.** Should a client be able to require
   or suppress chain establishment, or negotiate lifetime, depth, or
   permitted continuers ({{root-establishment}})?

4. **Authorization-basis representation.** This document defines an
   authorization basis as a policy reference ({{terms}}) but not its
   representation. Should the envelope expose a testable representation of
   the authorization ceiling, for example:

   ~~~ json
   { "targets": [ { "audience": "https://ras.travel.example/",
       "resource": "https://api.travel.example/",
       "scope": ["trips.read"] } ] }
   ~~~

   Dynamic ceilings might instead use an authorization detail {{RFC9396}} or
   a policy reference.

5. **Document factoring.** Should durable chains ({{lifecycle}},
   {{example-background}}) become a separate profile, leaving this document
   with session-bounded continuation, or does one document with a stoppable
   core serve implementers better?

6. **Stateless hop commitments.** Could a handle be a self-verifying
   commitment carrying the root, parent, RAS, and actor, so that the IdP
   keeps only root, revocation, and reservation state? The proposal must show
   how the IdP recovers a hop's ancestry when the parent record no longer
   exists, since lineage is derived by walking parent references and an
   ancestor's revocation must be detectable ({{onward-id-jag}},
   {{lifecycle}}); how it does so within the handle's 256-character bound
   ({{chain-id}}) without embedding commitments recursively; and what a
   revocation list costs in subtree revocation and unlinkability
   ({{privacy}}).

7. **Mandatory idempotent retry.** {{validation-replay}} requires single-use
   and makes idempotent retry optional; an earlier draft required both. Should
   retry be mandatory, so that a client can rely on recovering a lost response
   at any IdP, at the cost of fingerprint state consistent enough to serve a
   concurrent retry, or is a rejected second presentation followed by a fresh
   assertion an acceptable recovery path ({{implementation}})?

8. **CAI discovery.** Should RAS metadata advertise that the RAS issues
   assertions from its token endpoint, and how does a workload discover a
   separate CAI's endpoint ({{assertion-token-exchange}}, {{metadata-ras}})?

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

* Fresh-eyes review fixes: qualified the single-use rationale for
  self-contained acceptance evidence; rule 4 now tests the presented hop's
  own revocation; chain establishment requires a resolvable anchor and
  otherwise issues a plain ID-JAG; error codes for a non-permitted actor, an
  exhausted depth bound, and prohibited parameters; identity authority
  defined for clients registered directly at the IdP; reservation timing;
  Security Considerations aligned with CAI authority, terminal RAS bearer
  tokens, bearer-ingress reach, and continuer widening; B.2 shows the
  dual-use credential; gateway checklist and provisioning table completed.
* Rewrote Section 5 in plain language with OAuth RFC structure: Establishing a
  Chain gains Root Exchange Request, Chain Establishment, Root Actor, and
  Root-Chain Envelope subsections; rationale moved to Design Rationale and
  Security Considerations; no requirement changed.
* Reduced the Protocol Overview's closing paragraph
 to the one insight the
  walk-through needs, that the IdP learns of acceptance only through the
  assertion; the hop state names now first appear where they are defined.
* Made the gateway example show the floor
: a root client that proves no key,
  a bearer access token at GatewayRAS, and a continuation exchange carrying
  the client credential alone; the SaaS example keeps sender-constrained
  access tokens as the hardened variant.
* Replaced trust-domain membership
 with CAI authority over the actor-to-hop
  association; made the 300-second assertion lifetime a recommended upper
  bound with the IdP's own maximum governing; tied chain lifetime to the
  anchor rather than to the session as such; stated the cost of single-use
  without retry; added the six-item test for a requirement to Design
  Rationale.
* Defined the CAI's acceptance evidence
 by the RAS's own authorization
  semantics, so a RAS whose authorization is a self-contained short-lived
  token satisfies the issuance precondition by that token's validity; a live
  recheck is recommended where fresh issuance must stop before the token
  expires.

* Made the `actor_token` optional on a continuation exchange: the actor
  authenticates as an OAuth client with a credential that resolves to its
  canonical actor identity, presenting an `actor_token` when that credential
  does not identify the actor or as independent evidence; stated that the
  shared key across credential, assertion, and onward ID-JAG is a consequence
  of a single proof method, not a continuation property; defined issuer
  pairing against the actor's identity authority.

* Removed the proof-of-possession requirement from the root exchange
: a root
  client keeps only the base profile's obligations, and sender constraint is
  required of a continuing actor; an optional root `actor_token` still needs
  a proof of the key it is bound to. A RAS binds its access token to the
  confirmed key when the ID-JAG carries `cnf` and otherwise by its own
  policy.

* Kept single-use of an assertion required
 and made idempotent retry
  optional, with the fingerprint rules applying to an IdP that offers retry;
  explained why a consumed assertion is not equivalent to a fresh one; opened
  a question on mandatory retry.

* Defined the canonical actor identity and made it the sole comparand for
  `act` and `actor_token`; reworded the CAI's role as attesting the facts the
  IdP's evaluation requires; characterized the handle as security-sensitive
  correlation state.
* Named the two parts of the root-chain envelope, chain identity and
  continuation authorization, defined the authorization basis as a policy
  reference, and specified that policy narrows a running chain in either
  form but widens its targets only through a recorded basis; stated in Hop
  Activation why RAS acceptance does not attenuate downstream authority.

* Gathered durable-chain material into Chain Lifetime and Revocation, with
  anchors, ending, and limits as subsections and a lead that separates the
  shared rules from the grant-specific discussion; moved the anchor list and
  the cross-chain limits there from Establishing a Chain; opened two
  questions on document factoring and stateless hop commitments.
* Added a Design Rationale subsection separating actor identity (`act`)
  from the target's `client_id` and explaining why `may_act` does not
  suffice; restated the confirmation claim as a proof-of-possession property
  with DPoP as the one method this document defines.
* Narrowed the CAI contract to attested facts: the RAS accepted the hop,
  RAS state still shows it continuable, and the authenticated actor proving a
  key is the party handling that request; dropped the "authorized under CAI
  policy" precondition, leaving a domain free to narrow issuance, and moved
  every decision about who may continue and to what to the IdP.
* Opened the Introduction with the three core properties the protocol rests
  on and stated what continuation does not prove; named co-located as the
  baseline deployment and a separate CAI as advanced, with the same exchange
  count in both; said that the handle correlates state and confers nothing
  and that `act` is disclosed lineage, not authoritative history.
* Aligned the envelope with ID-JAG's policy model: the facts of the root
  exchange are fixed, tenant policy for targets, continuers, and limits
  applies as it stands at each continuation in either direction, and
  "consent" wording became tenant policy throughout; dropped the
  narrow-but-not-broaden rule and the consent-scope open question.
* Applied a whole-draft review: stated in the Introduction that acceptance
  does not make the accepted token's scopes the ceiling for later targets;
  qualified the compatibility claims to root clients with DPoP and a
  resolvable anchor, terminal RASes, and the parties that continue; added a
  worked authorization-basis case to the gateway example; added a
  provisioning table and
  the gateway's dual-use credential to that example; added an operational
  paragraph on availability, state retention, limits, and failure paths.
* Applied a gateway-vendor review: example hop records show the
  continuation-permitted flag the binding requires; the `act` claim states
  where its `iss` and `sub` come from; carriers are described for either
  topology; the gateway checklist names the registrations it presupposes; the
  redemption dedupe rule names its key; continuation is stated to be per
  authorization context, not per call.
* Rewrote the Protocol Overview as a numbered walk-through of the gateway
  deployment, with the figure keyed to the steps, and reduced the Multi-Hop
  Overview to a map from those steps to the sections that specify them.
* Stated the design goal in the Introduction: an opt-in extension that
  requires no change to existing ID-JAG clients or RASes, keeps chain state at
  the IdP, and lets an ID-JAG ecosystem add multi-hop access at gateways.
* Cited validation rules by name, tabulated the envelope, removed the
  duplicate signals table, merged three short Security subsections into
  their neighbors, trimmed the IdP metadata entry, and reorganized the
  background example by hop.
* Trimmed the vocabulary: removed the binding, authenticated context, and
  intra-domain carrier entries and the presenting-actor alias, unified on
  "pairwise subject", narrowed continuation-capable to ID-JAGs, and made
  continuable a plain adjective rather than a third hop state.
* Named the CAI as the subject of the carrier and scheduled-task rules, noted
  the confused-deputy surface of an envelope that records a basis rather
  than targets, and showed client
  authentication in every example request.
* Prohibited a refresh token in the continuation response, required envelope
  containment of the effective authorization after defaults, forbade a
  top-level `nbf` in the assertion, made authentication-context preservation
  unconditional, and corrected the signed-assertion rationale.
* Ordered the protocol section chronologically: establishing a chain, RAS
  binding and hop activation, handle propagation, assertion issuance, the
  continuation exchange, and replay reservation.
* Added a Protocol Overview to the Introduction, regrouped the Overview by
  role, moved Chain Lifetime and Revocation after the protocol section, moved
  replay reservation to the end of that section, and ordered assertion
  issuance as request, preconditions, response.
* Reorganized the SaaS chain example by hop, trimmed restated specification
  text from it, and moved the trust-boundary note to Design Rationale.
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
  when a hop is continuable and what continuation-capable means without
  circularity, and moved the
  per-target client registration prerequisite to the Token Exchange
  introduction.
* Annotated the overview diagram with this profile's additions and the
  terminal hop.
* Made the accepting RAS a CAI for its own hops without configuration; metadata
  now lists only additional CAIs. Added a scope statement to the Overview and a
  non-normative Deployment Topologies subsection holding the carrier
  realizations, added a Topology and Trust security subsection, and switched
  the gateway example to the co-located topology.
* Removed the prohibition on carrying the handle in an access token, along
  with the carrier requirements that restated the CAI preconditions. One rule
  remains: the handle surfaced for a call is the one the RAS bound to the
  authorization that the call's credential or context identifies, read from RAS
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
