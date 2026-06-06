---
title: "Identity Continuation Assertion for OAuth 2.0 Token Exchange"
abbrev: "Identity Continuation Assertion"
category: std

docname: draft-mcguinness-oauth-id-continuation-assertion-00
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
  arch: "https://datatracker.ietf.org/wg/oauth/about/"
  github: "mcguinness/draft-mcguinness-oauth-id-continuation-assertion"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-id-continuation-assertion/draft-mcguinness-oauth-id-continuation-assertion.html"

author:
 -
    fullname: "Karl McGuinness"
    organization: "ConductorOne"
    email: "karl.mcguinness@conductorone.com"

normative:
  RFC6749:
  RFC6755:
  RFC7519:
  RFC7638:
  RFC7662:
  RFC7800:
  RFC8693:
  RFC8705:
  RFC9449:
  I-D.ietf-oauth-identity-chaining:
  I-D.ietf-oauth-identity-assertion-authz-grant:

informative:
  RFC8417:
  RFC8707:
  RFC8725:
  RFC9700:
  I-D.ietf-oauth-transaction-tokens:
  I-D.ietf-wimse-arch:
  OIDC.FrontChannelLogout:
    title: "OpenID Connect Front-Channel Logout 1.0"
    target: "https://openid.net/specs/openid-connect-frontchannel-1_0.html"
    date: false
    author:
      - org: "OpenID Foundation"
  I-D.niyikiza-oauth-attenuating-agent-tokens:
  # NOTE: draft-hingnikar-cecchetti-wimse-waffles is not yet on the IETF
  # Datatracker; this reference (title, URL) is unverified. Replace with the
  # authoritative citation once it is published.
  WAFFLES:
    title: "Waffles: Stacked Delegated JWTs (work in progress)"
    target: "https://datatracker.ietf.org/doc/draft-hingnikar-cecchetti-wimse-waffles/"
    date: false
    seriesinfo:
      Internet-Draft: draft-hingnikar-cecchetti-wimse-waffles

...

--- abstract

This document profiles OAuth 2.0 Token Exchange to define the Identity
Continuation Assertion: a short-lived, sender-constrained JWT that carries
verifiable evidence of a delegation chain into a Token Exchange request. A
Continuation Authorization Server can then issue an onward authorization grant
without subjecting the user to a new interactive authentication. The primary
use case is same-Identity-Provider (same-IdP) SaaS-to-SaaS chaining, in which
several Resource Authorization Servers trust a single enterprise IdP and each
names the user under its own audience-local (pairwise) subject identifier.
Because only the IdP holds the map from a root delegation to each audience's
subject, the IdP remains the sole issuer of the onward Identity Assertion JWT
Authorization Grant (ID-JAG) at every cross-boundary hop. The assertion is the
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
second application, which in turn must call a third. Each application exposes
an API protected by its own Resource Authorization Server (RAS), and all of
these Resource Authorization Servers trust the same enterprise IdP for single
sign-on (SSO), for delegation continuity, and for resolving the user's
identity into an audience-local subject identifier.

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
through a new interactive flow merely because TravelSaaS must call BookingSaaS
to complete the original request. The IdP remains the trust anchor and the
sole issuer of onward authorization grants. Each onward grant is
audience-scoped and carries the subject identifier appropriate for its target
RAS.

## Why a New Input Is Needed {#motivation}

The first hop is an ordinary identity assertion exchange: ExpenseApp holds a
credential for the authenticated user (for example, an ID Token or refresh
token) and exchanges it at the IdP for an ID-JAG, exactly as in
{{I-D.ietf-oauth-identity-assertion-authz-grant}}. Every subsequent hop is
different. By the time TravelSaaS must call BookingSaaS, the user is no longer
present and TravelSaaS holds no end-user credential it could present to the
IdP; the only thing that crossed the ExpenseSaaS-to-TravelSaaS boundary is
chain context. TravelSaaS therefore cannot perform a normal identity assertion
exchange to obtain the next grant, even though the same IdP could mint it.

This document defines the missing input. An Identity Continuation Assertion is a
short-lived, sender-constrained, verifiable statement about the in-flight
delegation chain that a service presents to the IdP in place of an end-user
credential, so that the IdP can resolve the next audience's subject and issue
the next onward grant. It is the evidence; the ID-JAG that comes back is the
authority.

## Core Principle {#core-principle}

The IdP is the only party that can name the user to each RAS, because each RAS
resolves the user under its own (pairwise) subject identifier and only the IdP
holds the map from the root delegation to those per-audience subjects. That is
why the IdP remains the sole issuer at every cross-boundary hop: no downstream
party and no offline authority can mint a correct onward `sub`.

The Identity Continuation Assertion is a Token Exchange subject token used to
obtain an onward ID-JAG. The Identity Assertion JWT
Authorization Grant (ID-JAG) {{I-D.ietf-oauth-identity-assertion-authz-grant}}
is the output profile of this document.

This document defines the evidence format, Token Exchange parameters,
validation rules, and IANA registrations needed for that exchange. It does not
define a new access token format, does not allow a Resource Server to consume
the Identity Continuation Assertion directly, and does not allow a Chain
Authority to name the user for the target audience.

## Relationship to Other Specifications

This profile builds directly on OAuth 2.0 Token Exchange {{RFC8693}} and JSON
Web Tokens {{RFC7519}}. It extends the cross-domain model of OAuth Identity and
Authorization Chaining Across Domains
{{I-D.ietf-oauth-identity-chaining}} and uses the ID-JAG
{{I-D.ietf-oauth-identity-assertion-authz-grant}} as its onward grant
type. It is intended to compose with, and is positioned relative to, the
Workload Identity in Multi-System Environments (WIMSE) architecture
{{I-D.ietf-wimse-arch}} and offline attenuated delegation tokens such as
Waffles {{WAFFLES}} and Attenuating Authorization Tokens (AAT)
{{I-D.niyikiza-oauth-attenuating-agent-tokens}}.

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
  Identity Continuation Assertion or a `chain_id`.

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
  evidence for a given tenant and chain class and to issue Identity
  Continuation Assertions. The Chain Authority is not the authority for
  resolving the target audience's user subject identifier.

Tenant and chain class:
: A tenant is the administrative boundary (for example, an enterprise customer
  of the IdP) within which a chain is established and trust in a Chain
  Authority is configured. A chain class is a deployment-defined category of
  chains (for example, by application, sensitivity, or policy) that an IdP can
  use to scope which Chain Authorities it trusts and which continuations it
  permits. Both are deployment-defined and otherwise out of scope for this
  document.

Chain Identifier (`chain_id`):
: An opaque, unguessable, IdP-generated identifier for the root delegation
  context; see {{chain-id}}.

Root-chain envelope:
: The set of constraints the IdP records when it establishes a chain at root
  issuance and evaluates every continuation against: the authenticated user,
  the authentication context (`auth_time`, `acr`, `amr`), the audiences and
  resources the chain may target, the maximum scope the chain may request, the
  maximum actor-chain depth, and the chain's expiry. The envelope is derived
  from the user's authentication and consent and from IdP policy; how it is
  determined is out of scope for this document. See {{validation}} and
  {{lifecycle}}.

Audience-local (pairwise) subject:
: The subject identifier under which a particular RAS names the user. Distinct
  Resource Authorization Servers MAY name the same user with different
  identifiers; only the IdP holds the map between them.

# Overview

## Layering {#layering}

This profile occupies one layer of a larger identity stack. Each layer answers
a distinct question and composes with the others:

~~~
Workload identity (WIMSE):       who the calling workload/agent is
Intra-domain delegation:         offline attenuated fan-out within
                                 one trust domain (a Waffles- or
                                 AAT-style stack)
Identity Continuation Assertion: cross-boundary evidence to the IdP
ID-JAG:                          onward grant issued by the IdP
~~~

An Identity Continuation Assertion goes in; an ID-JAG comes out.

## When to Use This Profile Versus Offline Attenuation {#decision-rule}

The deciding question is subject resolution, not cost.

* If the next audience requires a *different* subject identifier (pairwise
  subjects, where only the IdP holds the map), the IdP MUST mint that hop. An
  Identity Continuation Assertion to the IdP is used. Offline tokens cannot
  produce the next audience's `sub`.

* If the *same* subject identifier, or a key-based workload identity, suffices
  down a local fan-out, the parties SHOULD NOT round-trip the IdP. An offline
  attenuated delegation token (Waffles {{WAFFLES}} or AAT
  {{I-D.niyikiza-oauth-attenuating-agent-tokens}}) is used instead, and the IdP
  is reserved for the boundary hops.

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
  "iss": "https://ca.expenses.example/",
  "aud": "https://idp.example/",
  "chain_id": "01JZ8F4J9J8Y3NDK5WQ4P9K7Q2",

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

  "iat": 1710000500,
  "exp": 1710000800,
  "jti": "continuation-assertion-01"
}
~~~

The claims have the following meanings and requirements:

`iss`:
: REQUIRED. The Chain Authority that issued the assertion. The IdP MUST verify
  that this issuer is a trusted Chain Authority for this tenant and chain class
  and that the assertion was signed with a key authorized for that issuer.

`aud`:
: REQUIRED. It MUST identify the Continuation Authorization Server being asked
  to issue the onward grant. The IdP MUST validate `aud` using exact audience
  matching against one of its configured issuer or token endpoint identifiers.

`chain_id`:
: REQUIRED. The authoritative binding to the root delegation and user. The IdP
  resolves the user, and the target-audience subject, from it. See {{chain-id}}.

`act`:
: REQUIRED. The actor lineage, encoded as nested `act` claims per {{RFC8693}}.
  The outermost `act` claim identifies the current actor presenting the Token
  Exchange request, and nested `act` claims identify prior actors. Claims inside
  `act` are identity claims only; non-identity claims such as `exp`, `nbf`,
  `aud`, `scope`, and `cnf` MUST NOT be used inside `act`. When the inbound
  segment was an offline attenuated stack, the Chain Authority SHOULD reflect
  the verified leaf actor as the outermost actor. It MAY retain
  deployment-specific audit evidence for the offline segment separately from
  the RFC 8693 `act` object.

`cnf`:
: REQUIRED. A confirmation claim {{RFC7800}} that binds the assertion to the
  key of the actor that will present it to the IdP. `cnf` MUST contain a key
  confirmation method: `jkt`, the JWK SHA-256 thumbprint {{RFC7638}} of that
  key when Demonstrating Proof of Possession (DPoP) {{RFC9449}} is used, or
  `x5t#S256`, the certificate thumbprint when mutual-TLS client authentication
  is used. The presenting actor demonstrates live possession of the confirmed
  key on the Token Exchange request, and the Chain Authority binds the
  assertion to it at issuance. The proof-of-possession model is specified in
  {{sender-constrained-presentation}}.

`iat`, `exp`:
: REQUIRED. The assertion lifetime MUST be short. `exp` MUST be after `iat`,
  and `exp - iat` MUST be no more than 300 seconds; see {{validation}}.

`jti`:
: REQUIRED. A unique identifier used by the IdP for replay detection. The value
  MUST be unique per issuer (`iss`) within the assertion validity window.

The assertion MUST NOT contain a top-level `sub`, `auth_time`, `acr`, or `amr`
claim. The IdP obtains the user and root authentication context from the
authoritative root-chain envelope indexed by `chain_id`. The current workload
is identified by the outermost `act` claim and authenticated by the
`actor_token`. Repeating these values in the assertion would create competing
sources of identity or authentication context without adding authority.

In deployments that already maintain a delegation or mission identifier, the
`chain_id` MAY be derived from or projected from that identifier, provided the
result still satisfies the requirements of {{chain-id}}.

## Claims That Are Deliberately Excluded {#excluded-claims}

The following Token Exchange request values MUST NOT be conveyed by the
Identity Continuation Assertion:

~~~
audience (target)
resource
scope
requested_token_type
~~~

These stay in the Token Exchange request ({{token-exchange}}) so that direct
and chained Token Exchange calls have an identical shape. The requested
`audience`, `resource`, `scope`, and `requested_token_type` always come from
the Token Exchange request, never from the assertion. The assertion's `aud`
claim is different: it identifies the continuation IdP that consumes the
assertion, not the target audience requested in the Token Exchange request.

## Assertion Issuance and Key Binding {#assertion-issuance}

A presenting actor obtains an Identity Continuation Assertion from the Chain
Authority for a given `chain_id` ({{flow}}, step 7). The Chain Authority MUST
bind the assertion to the requesting actor's key by setting the appropriate
`cnf` confirmation method ({{assertion-claims}}) for that key, and MUST do so
only after establishing that the requesting actor controls that key and is the
actor recorded in the outermost `act` claim. The mechanism by which the actor
authenticates to the Chain Authority and proves control of its key is
deployment-specific; it is typically the workload's existing mutually
authenticated credential. An assertion whose `cnf` is not bound to the
requesting actor's key is not presentable because the IdP requires live proof
of possession of the confirmed key ({{sender-constrained-presentation}}).

## Chain-Context Provenance {#context-provenance}

The mechanism used to transport chain context between workloads is
deployment-specific, but its security properties are not. A Chain Authority
MUST issue an Identity Continuation Assertion only after establishing all of
the following:

1. the `chain_id` was received through an authenticated, confidential, and
   integrity-protected channel from a participant authorized to continue that
   chain, or was obtained from equivalent authenticated state held by the Chain
   Authority;

2. the requesting actor is authorized under Chain Authority policy to continue
   the chain;

3. the requesting actor controls the key placed in `cnf`; and

4. the actor lineage placed in `act` was derived from authenticated context or
   from a verifiable delegation artifact whose integrity and delegation rules
   the Chain Authority has validated.

Possession of `chain_id` alone is insufficient to satisfy these requirements.
Values received as propagated context, including actor history or requested
authority, MUST NOT override the root-chain envelope held by the IdP. The
transport MAY carry deployment-specific hints, but the Identity Continuation
Assertion contains only the claims defined in {{assertion-claims}}, and the IdP
applies its own root-chain and current-actor policy during the exchange.

# Chain Identifier (`chain_id`) {#chain-id}

The `chain_id` identifies the root delegation context established when the IdP
issued the initial ID-JAG. It is opaque, unguessable, IdP-generated, and used
by the IdP to correlate a continuation back to that root and to resolve the
correct per-audience subject.

`chain_id` is a non-bearer reference to that delegation context, closer in
spirit to a grant identifier than to a token. It conveys no authority on its
own, and it is never dereferenced into, or presented in place of, a token: it is
not a token handle or reference token. Authority at each hop comes from the
sender-constrained Identity Continuation Assertion and the root-chain envelope,
not from `chain_id`.

This follows an established pattern of non-bearer correlation claims carried in
a JWT. The OpenID Connect `sid` (Session ID) claim {{OIDC.FrontChannelLogout}}
and the `txn` (Transaction Identifier) claim {{RFC8417}} are likewise opaque,
issuer-generated identifiers that index server-side state and convey no
authority of their own. `chain_id` differs chiefly in being confined to the
control plane: unlike `sid`, which a Resource Server can see in an ID Token,
`chain_id` MUST NOT appear in any token a Resource Server consumes (the rules
below keep it out of the data plane; see also {{privacy}}).

The following rules apply:

1. The IdP MUST generate `chain_id` for delegations that are eligible for
   continuation, and MUST return it as a Token Exchange *response* parameter
   ({{response-param}}).

2. `chain_id` MUST contain at least 128 bits of entropy and MUST NOT contain
   user-identifying information.

3. `chain_id` lives in the control plane only: Token Exchange responses,
   propagated chain context, and Identity Continuation Assertions.

4. `chain_id` MUST NOT appear in any ID-JAG or access token that a Resource
   Server consumes. This preserves the audience-local subject property: a
   Resource Server sees only its own subject, audience, scope, and actor chain,
   and nothing that correlates the user across SaaS boundaries.

5. End-to-end audit correlation is performed at the IdP, which holds `chain_id`
   for every hop. Per-Resource-Server logs use that server's own pairwise
   subject.

6. Resource Authorization Servers, Resource Servers, and Chain Authorities MUST
   NOT modify `chain_id`.

7. `chain_id` alone is not proof of authorization and MUST NOT be treated as a
   bearer credential. The IdP MUST use it only as a lookup handle for root
   chain state, subject resolution, and policy evaluation.

## Stronger Privacy Mode {#chain-id-privacy}

If colluding chain participants must not be able to correlate even "same
delegation", the IdP MAY issue a distinct per-audience chain reference that
maps internally to a single root delegation, instead of a single shared
`chain_id`. See {{privacy}}.

# Chain Lifetime and Revocation {#lifecycle}

A chain is continuable only while the IdP considers it active. Because every
cross-boundary hop is an exchange at the IdP ({{flow}}), the IdP is in the loop
at each hop and can stop a chain at any point.

The IdP MUST bound the continuation lifetime of a `chain_id`. That lifetime
MUST NOT exceed the lifetime of the authentication context and authorization
that established the chain (for example, the user's session, the governing
refresh token, or an explicit maximum chain lifetime set by policy), and the
IdP MUST reject a continuation whose `chain_id` has expired ({{validation}}).
Continuing a chain MUST NOT extend the root authentication context: `auth_time`,
`acr`, and `amr` are fixed at root issuance, and onward grants inherit them
unchanged ({{security-assurance}}).

The IdP MUST be able to revoke a chain (for example, when the user's session is
terminated, the governing refresh token is revoked, or policy otherwise
withdraws the delegation), and MUST stop honoring continuation for a revoked
`chain_id`. Because each onward grant requires a fresh exchange at the IdP,
revocation takes effect at the next hop, with no need to reach or invalidate
chain evidence already held by intermediate services. This is a deliberate
difference from offline attenuated delegation (Waffles or AAT), where a minted
child token remains usable for its lifetime without contacting an authority.
Access tokens already issued by a Resource Authorization Server remain valid
for their own lifetimes and are governed by that server's revocation
mechanisms.

# Token Exchange Profile {#token-exchange}

An Identity Continuation Assertion is used as the `subject_token` of an OAuth
2.0 Token Exchange request {{RFC8693}}. A direct request and a chained request
have the same shape: the client changes only `subject_token` and
`subject_token_type`.

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

The requested `audience`, `resource`, `scope`, and `requested_token_type` are
supplied by the Token Exchange request and never by the assertion. Following
{{I-D.ietf-oauth-identity-assertion-authz-grant}}, `audience` identifies the
target Resource Authorization Server and `resource` {{RFC8707}} identifies the
protected resource for which access is ultimately requested.

## Sender-Constrained Presentation {#sender-constrained-presentation}

The Identity Continuation Assertion is sender-constrained: it is bound by `cnf`
to the key of the actor that presents it ({{assertion-claims}}). This profile
uses proof of possession aligned with the direct ID-JAG request
{{I-D.ietf-oauth-identity-assertion-authz-grant}}:

* The presenting actor MUST demonstrate live possession of the confirmed key on
  the Token Exchange request to the IdP. With DPoP {{RFC9449}}, the actor
  includes a DPoP proof JWT computed with that key, carried in the `DPoP` HTTP
  header (and therefore not in the request body shown above); the IdP MUST
  verify that the DPoP proof key thumbprint equals the assertion's `cnf.jkt`.
  Alternatively, the actor MAY use mutual-TLS client authentication
  {{RFC8705}}, in which case `cnf` carries `x5t#S256`. The IdP MUST verify that
  the presented client certificate thumbprint equals that value. The IdP MUST
  reject a request that does not prove possession of the confirmed key.

* The Token Exchange request MUST include an `actor_token`, which authenticates
  the workload identity of the current actor, the outermost actor in the
  assertion's `act` chain. The `actor_token` MUST itself be sender-constrained
  to the same key confirmed by the assertion's `cnf`. For a JWT actor token,
  the IdP MUST verify a `cnf.jkt` or `cnf.x5t#S256` value that matches both the
  assertion and the live proof. For an opaque actor token, the IdP MUST obtain
  the equivalent confirmation value from authoritative token metadata, such as
  a token introspection response {{RFC7662}}. A bearer actor token MUST NOT be
  accepted. These checks bind the authenticated workload identity, the
  assertion, and the demonstrated key to one actor.

The onward ID-JAG the IdP issues is itself sender-constrained to the same key
through its own `cnf` ({{onward-id-jag}}). The presenting actor proves
possession of that key again when it exchanges the ID-JAG at the target Resource
Authorization Server, exactly as for a directly issued ID-JAG
{{I-D.ietf-oauth-identity-assertion-authz-grant}}.

## The `chain_id` Response Parameter {#response-param}

When the IdP issues an onward grant for a delegation that is eligible for
further continuation, it MUST return `chain_id` as a top-level member of the
Token Exchange response (alongside `access_token`, `issued_token_type`, and so
forth), so that the next hop's Chain Authority can construct the next Identity
Continuation Assertion. `chain_id` MUST NOT be carried as a claim inside the
issued ID-JAG ({{chain-id}}, rule 4).

Returning `chain_id` as a response parameter rather than as a claim is
deliberate: it keeps `chain_id` in the control plane, where it reaches the
requester that will continue the chain, and out of the data-plane ID-JAG that a
Resource Authorization Server consumes. Only the continuing party needs
`chain_id`; the Resource Authorization Server does not, and carrying it in the
ID-JAG would expose a cross-hop correlation handle to every audience that
receives a token ({{privacy}}).

`chain_id` is delivered to the party that performed the Token Exchange and is
conveyed from there to the workload that continues the chain over a
control-plane channel. For example, it can accompany the request as it crosses
into the next service, or a Chain Authority in the originating trust domain can
hold it. It is never carried in the ID-JAG or in an access token. Therefore,
the target Resource Authorization Server and Resource Server do not receive it
from either token. How `chain_id` is conveyed is deployment-specific, but the
provenance and integrity requirements of {{context-provenance}} apply.

A non-normative response example is:

~~~ json
{
  "access_token": "<id-jag>",
  "issued_token_type": "urn:ietf:params:oauth:token-type:id-jag",
  "token_type": "N_A",
  "expires_in": 300,
  "chain_id": "01JZ8F4J9J8Y3NDK5WQ4P9K7Q2"
}
~~~

# Same-IdP Core Flow {#flow}

The canonical same-IdP SaaS-to-SaaS flow proceeds as follows. Steps 6 through
12 repeat for each additional cross-boundary hop (for example, TravelRAS to
BookingRAS).

~~~
1.  ExpenseApp authenticates the user at the IdP.

2.  ExpenseApp requests an ID-JAG for ExpenseRAS via Token Exchange.

3.  IdP issues:
      ID-JAG: iss=IdP, aud=ExpenseRAS,
              sub=<subject for user at ExpenseRAS>,
              scope, cnf
    and returns chain_id as a Token Exchange response parameter.
    IdP records the root-chain envelope: chain_id -> { user,
        auth_time/acr/amr, permitted audiences/resources, max scope,
        max depth, expiry }.
    (chain_id is not a claim inside the ExpenseRAS ID-JAG.)

4.  ExpenseApp exchanges the ID-JAG at ExpenseRAS for AT1 and
    invokes ExpenseSaaS. ExpenseApp conveys chain_id to ExpenseSaaS
    over an authenticated, confidential, and integrity-protected
    control-plane channel associated with the request. chain_id is
    not carried in the ID-JAG or AT1.

5.  ExpenseSaaS propagates authenticated, confidential, and
    integrity-protected chain context to TravelService:
      chain_id; verified actor lineage.
    (No hop-local subject is propagated across the SaaS boundary by
     default. Local fan-out inside a domain MAY use an offline
     attenuated stack.)

6.  TravelService needs to call TravelAPI behind TravelRAS.

7.  TravelService obtains an Identity Continuation Assertion from the
    Chain Authority for that chain_id.

8.  TravelService presents the assertion to the IdP as subject_token
    and requests:
      requested_token_type = id-jag
      audience = TravelRAS
      resource = TravelAPI
      scope    = trips.read

9.  IdP validates the assertion, root-chain state, actor policy,
    requested audience/resource/scope, and sender constraint.

10. IdP resolves the user subject identifier for TravelRAS.

11. IdP issues a new ID-JAG:
      iss=IdP, aud=TravelRAS,
      sub=<subject for same user at TravelRAS>,
      act=updated actor chain, cnf
    and returns chain_id again as a response parameter for any
    further hop. (No chain_id claim in the TravelRAS ID-JAG.)

12. TravelService exchanges the new ID-JAG at TravelRAS for AT2.
    Repeat 6-12 for BookingRAS.
~~~

# IdP Validation for ID-JAG Output {#validation}

Before issuing an onward ID-JAG, the IdP MUST reject the Token Exchange request
unless all of the following hold:

1. the JOSE `typ` header value is `oauth-identity-continuation+jwt`;

2. the assertion signature validates using a key authorized for the assertion
   issuer, and the JOSE `alg` is an asymmetric signature algorithm on the IdP's
   configured allowlist (the `none` algorithm MUST be rejected; see
   {{security-alg}});

3. the assertion `aud` identifies the IdP;

4. the assertion `iss` is a trusted Chain Authority for this tenant and chain
   class;

5. `chain_id` is known, active, unexpired, and eligible for continuation;

6. the assertion does not contain a top-level `sub`, `auth_time`, `acr`, or
   `amr` claim;

7. the `act` chain is present, identifies the current actor as the outermost
   actor, contains only identity claims, and is acceptable and within max-depth
   policy;

8. the `actor_token` authenticates the current actor, that actor is the
   outermost actor in the `act` chain, and the actor is permitted by IdP policy
   to continue this chain;

9. the request proves possession of the key confirmed by `cnf` (a DPoP proof
   {{RFC9449}} matching `cnf.jkt`, or a client certificate {{RFC8705}} matching
   `cnf.x5t#S256`), and the `actor_token` is sender-constrained to that same
   key ({{sender-constrained-presentation}});

10. `jti` has not been successfully consumed before for the assertion issuer;

11. `iat` and `exp` are valid NumericDate values, `iat` is not in the future
    beyond the IdP's permitted clock skew, `exp` is after `iat`, the assertion
    has not expired, and the assertion lifetime is no more than 300 seconds;

12. the requested `audience` is permitted by the root-chain envelope;

13. the requested `resource` is permitted by the root-chain envelope;

14. the requested `scope` is within the root-chain envelope and IdP policy for
    the current actor;

15. the requested output token type is
    `urn:ietf:params:oauth:token-type:id-jag`; and

16. the IdP can resolve the target-audience subject for the requested
    `audience`.

After all other validation succeeds, the IdP MUST atomically check and record
the tuple (`iss`, `jti`) as consumed as part of issuing the onward grant. At
most one concurrent exchange using the same tuple may succeed. An
implementation MUST NOT permanently consume the tuple solely because a request
failed validation before grant issuance. If an implementation reserves a tuple
before completing issuance, it MUST release the reservation after failure or
expire it promptly. The IdP MUST retain a successfully consumed tuple until at
least the assertion's `exp`, allowing for its permitted clock skew.

On success, the IdP resolves the audience-local subject for the requested RAS
and issues the onward ID-JAG with that `sub`. The onward ID-JAG MUST NOT carry
a `chain_id` claim.

If validation fails, the IdP MUST return an OAuth error response as defined by
{{RFC8693}} and {{RFC6749}}. The IdP SHOULD use `invalid_request` for malformed
or internally inconsistent requests, `invalid_target` for a requested
`audience` or `resource` outside the root-chain envelope, and `invalid_scope`
for a requested `scope` outside the root-chain envelope or IdP policy for the
current actor.

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

TravelRAS processes this as an ordinary ID-JAG per
{{I-D.ietf-oauth-identity-assertion-authz-grant}}. It does not need to
understand the Identity Continuation Assertion, and it never sees `chain_id`.
The `client_id` matches the outer `act.sub` by construction.

# Security Considerations {#security}

## Sender Constraint and Proof of Possession

The Identity Continuation Assertion MUST be sender-constrained via `cnf`
{{RFC7800}} and MUST NOT be accepted as a bearer token. The presenting actor
demonstrates possession of the key confirmed by `cnf` (using a DPoP proof
{{RFC9449}} or mutual-TLS client authentication {{RFC8705}}) on the Token
Exchange request, and the IdP MUST verify that the demonstrated key matches
`cnf`, that the `actor_token` is sender-constrained to the same key, and that
the actor authenticated by that token is the outermost actor of the `act` chain
({{sender-constrained-presentation}}, {{validation}}). Binding the assertion,
the demonstrated key, and the authenticated workload identity to a single
actor means a stolen assertion or stolen actor token cannot be replayed by a
different party without the confirmed private key. The assertion is also
short-lived and single-use ({{security-replay}}), bounding the window even
against the key holder.

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

## Monotonic Attenuation and Bounded Depth

Where an offline attenuated delegation stack (Waffles {{WAFFLES}} or AAT
{{I-D.niyikiza-oauth-attenuating-agent-tokens}}) feeds an Identity Continuation
Assertion, authority MUST narrow monotonically along that offline segment,
delegation depth MUST be bounded, and the actor chain SHOULD be
cryptographically parent-hash linked rather than merely descriptive. The IdP
MUST enforce that the requested `audience`, `resource`, and `scope` are within
the root-chain envelope and within IdP policy for the current actor
({{validation}}). This prevents a compromised intermediate actor from
broadening the chain beyond what the root delegation authorized.

## Trust in the Chain Authority

The IdP MUST accept assertions only from Chain Authorities it trusts for the
relevant tenant and chain class ({{validation}}). A compromised Chain
Authority can request continuations within the envelope of chains it is trusted
for. Deployments SHOULD scope Chain Authority trust as narrowly as practical
and SHOULD monitor for anomalous continuation patterns. Because the IdP
enforces the root-chain envelope, a compromised Chain Authority cannot exceed
the original delegation's audiences, resources, or scopes.

## Actor Chain Integrity

The nested `act` claim in {{RFC8693}} records the current actor and prior
actors, but nested prior actors are informational for access-control decisions.
An IdP MUST therefore base authorization decisions on the root-chain state, the
authenticated current actor, and the requested `audience`, `resource`, and
`scope`; it MUST NOT treat unverified nested actor history as independent
authority. When an offline attenuated segment is summarized in an Identity
Continuation Assertion, the Chain Authority MUST verify that segment before
issuing the assertion.

## Token, Type, and Algorithm Confusion {#security-alg}

The IdP MUST verify the JOSE `typ` header value
`oauth-identity-continuation+jwt` and MUST NOT process an Identity Continuation
Assertion as any other kind of JWT, in accordance with {{RFC8725}}. The IdP
MUST reject an assertion whose JOSE `alg` header is `none`, and MUST restrict
accepted algorithms to a configured allowlist of asymmetric signature
algorithms. Symmetric (MAC) algorithms MUST NOT be accepted. The IdP MUST
select the verification key from its trust configuration for the assertion
`iss`, and MUST NOT trust key material or key references carried in the
assertion header to choose that key. General OAuth security guidance in
{{RFC9700}} applies.

# Privacy Considerations {#privacy}

The `chain_id` is the primary cross-hop correlation handle, and the rules in
{{chain-id}} are designed to keep it out of the data plane. Because `chain_id`
MUST NOT appear in any ID-JAG or access token consumed by a Resource Server,
and because each RAS sees only its own pairwise subject, no Resource Server can
correlate the user across SaaS boundaries using tokens it receives. End-to-end
correlation is possible only at the IdP, which is the trust anchor and already
holds the full map.

`chain_id` MUST NOT contain user-identifying information and MUST carry at least
128 bits of entropy ({{chain-id}}, rule 2), so that possession of a `chain_id`
does not reveal the user and cannot be guessed.

Where even "same delegation" correlation among colluding chain participants is a
concern, the IdP MAY adopt the stronger privacy mode of {{chain-id-privacy}},
issuing a distinct per-audience chain reference that maps internally to one root
delegation. This prevents participants at different audiences from linking their
observations to a single shared identifier.

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

Reference:
: This document, {{names}}

## Media Type Registration

IANA is requested to register the following media type in the "Media Types"
registry, corresponding to the JOSE `typ` header value
`oauth-identity-continuation+jwt`.

Type name:
: application

Subtype name:
: oauth-identity-continuation+jwt

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: 8bit; an Identity Continuation Assertion is a JWT {{RFC7519}}, which is a
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
: Karl McGuinness (karl.mcguinness@conductorone.com)

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
: chain_id

Claim Description:
: Chain Identifier: an opaque, IdP-generated identifier for the root delegation
  context, used to correlate a continuation to its root and to resolve the
  per-audience subject.

Change Controller:
: IETF

Specification Document(s):
: This document, {{chain-id}}

## OAuth Parameters Registration

IANA is requested to register the following value in the "OAuth Parameters"
registry established by {{RFC6749}}. The parameter is used in the OAuth 2.0
Token Exchange {{RFC8693}} response.

Name:
: chain_id

Parameter Usage Location:
: token response

Change Controller:
: IETF

Reference:
: This document, {{response-param}}

Note: The token type URI `urn:ietf:params:oauth:token-type:id-jag` referenced by
this document is registered by
{{I-D.ietf-oauth-identity-assertion-authz-grant}} and is not registered here.

--- back

# Design Rationale {#rationale}

This appendix is non-normative. It explains why the Identity Continuation
Assertion is defined as a distinct Token Exchange subject token, rather than as
a profile of an existing artifact, and why a cross-boundary hop cannot be served
by propagating an existing token. The recurring answer is per-audience subject
resolution: only the IdP can name the user to the next audience ({{motivation}},
{{core-principle}}).

## Why Not a Profile of ID-JAG {#rationale-idjag}

An ID-JAG {{I-D.ietf-oauth-identity-assertion-authz-grant}} and an Identity
Continuation Assertion sit on opposite sides of the same Token Exchange and play
opposite roles, so neither can be a profile of the other:

* An ID-JAG is the **output**: an authorization grant. An Identity Continuation
  Assertion is the **input**: evidence presented as a `subject_token`.

* An ID-JAG's `aud` is the target Resource Authorization Server; an Identity
  Continuation Assertion's `aud` is the IdP. The two are validated by different
  parties under different rules.

* An ID-JAG MUST carry the user's resolved per-audience subject; an Identity
  Continuation Assertion MUST NOT carry a top-level `sub`
  ({{assertion-claims}}). Profiling ID-JAG would force a `sub` to mean
  precisely what this profile forbids.

* `chain_id` MUST NOT appear in any ID-JAG ({{chain-id}}, rule 4), yet it is the
  defining claim of an Identity Continuation Assertion. "An ID-JAG carrying
  `chain_id`" is self-contradictory.

An Identity Continuation Assertion is also not a grant: presenting it to a
Resource Authorization Server is meaningless. It is the input that *produces* a
chained ID-JAG, not a kind of ID-JAG.

## Why Not a Profile of a Transaction Token {#rationale-txn}

Transaction Tokens {{I-D.ietf-oauth-transaction-tokens}} are the closest
neighbor (a short-lived signed JWT carrying delegation context, often issued by
a service that could also act as the Chain Authority), but they sit at a
different layer ({{layering}}). A Transaction Token is an *intra-domain* object:
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
use offline propagation between hops wherever it is sound: that is the Waffles
and AAT layer, used when the subject does not change ({{decision-rule}}). It
cannot serve the boundary where the subject changes, for four reasons:

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

# Worked Example (Same-IdP, Two Hops) {#example}

This appendix is non-normative. It walks the canonical flow of {{flow}}
end-to-end for a single user: ExpenseApp invokes ExpenseSaaS, and ExpenseSaaS
calls TravelSaaS, whose TravelService must call TravelAPI to finish the
request. All parties trust one enterprise IdP at `https://idp.example/`.
Proof of possession uses DPoP. JWTs are shown as decoded payloads; JOSE headers
and signatures are omitted. The values are consistent with the examples in
{{assertion-claims}}, {{response-param}}, and {{onward-id-jag}}.

The participants and values used throughout:

* User: the human, authenticated once at the IdP.
* IdP: `https://idp.example/`.
* ExpenseApp: `expense-app`, the user-facing application.
* ExpenseSaaS: `https://expenses.example/`.
* ExpenseRAS: `https://ras.expenses.example/`.
* ExpenseAPI: `https://api.expenses.example/`.
* TravelSaaS: `https://travel.example/`.
* TravelService: `travel-service`, the workload calling TravelAPI.
* TravelRAS: `https://ras.travel.example/`.
* TravelAPI: `https://api.travel.example/`.
* Chain Authority: `https://ca.expenses.example/`.
* `chain_id`: `01JZ8F4J9J8Y3NDK5WQ4P9K7Q2`.
* Subjects: `expense-local-subject` at ExpenseRAS and
  `travel-local-subject` at TravelRAS.

At a glance, the message flow is:

~~~
User authenticates once at the IdP.

ExpenseApp -> IdP: (1) exchange ID Token
IdP -> ExpenseApp: (2) ID-JAG(ExpenseRAS) + chain_id
ExpenseApp -> ExpenseRAS: (3) ID-JAG
ExpenseRAS -> ExpenseApp: (4) AT1
ExpenseApp -> ExpenseSaaS: (5) API request + chain_id

ExpenseSaaS -> TravelSaaS: protected request + chain context
TravelSaaS -> TravelService: verified chain context

TravelService -> CA: (6) request assertion
CA -> TravelService: (7) Identity Continuation Assertion
TravelService -> IdP: (8) assertion exchange + DPoP
IdP -> TravelService: (9) ID-JAG(TravelRAS) + chain_id
TravelService -> TravelRAS: (10) ID-JAG
TravelRAS -> TravelService: (11) AT2
TravelService -> TravelAPI: (12) API call
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
subject_token=<id_token>
subject_token_type=urn:ietf:params:oauth:token-type:id_token
actor_token=<sender-constrained expense-app credential>
actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

The IdP authenticates the user from the ID Token, verifies that ExpenseApp's
actor credential is constrained to the DPoP key, records the root-chain
envelope keyed by a freshly generated `chain_id`, and responds:

~~~ json
{
  "access_token": "<ID-JAG for ExpenseRAS>",
  "issued_token_type": "urn:ietf:params:oauth:token-type:id-jag",
  "token_type": "N_A",
  "expires_in": 300,
  "chain_id": "01JZ8F4J9J8Y3NDK5WQ4P9K7Q2"
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
  "cnf": { "jkt": "base64url-expense-app-key-thumbprint" },
  "iat": 1710000005,
  "exp": 1710000305,
  "jti": "idjag-expense-01"
}
~~~

ExpenseApp exchanges this ID-JAG at ExpenseRAS for an access token (AT1) and
invokes ExpenseSaaS, exactly as for any ID-JAG
{{I-D.ietf-oauth-identity-assertion-authz-grant}} (not shown). ExpenseApp
conveys `chain_id` to ExpenseSaaS in authenticated, confidential, and
integrity-protected control-plane context associated with that API request.
The `chain_id` does not appear in the ID-JAG or AT1.

## Crossing the Boundary into TravelSaaS {#example-context}

To complete the request, ExpenseSaaS calls TravelSaaS, propagating the chain
context to TravelService over an authenticated, confidential, and
integrity-protected channel:

~~~
chain_id   = 01JZ8F4J9J8Y3NDK5WQ4P9K7Q2
actor chain = expense-app (ExpenseSaaS)
~~~

The user's ExpenseRAS-local subject (`expense-local-subject`) is not propagated
across the boundary. The IdP retains the authoritative authentication context
and scope envelope; those values are not supplied by TravelService or repeated
in the Identity Continuation Assertion.

## Obtaining the Identity Continuation Assertion {#example-ica}

TravelService needs to call TravelAPI behind TravelRAS. It requests an Identity
Continuation Assertion from the Chain Authority for this `chain_id`, proving
control of its key so the Chain Authority can bind `cnf` to it
({{assertion-issuance}}). The Chain Authority returns the assertion shown in
{{assertion-claims}}: `iss` is the Chain Authority, `aud` is the IdP,
`chain_id` is the value above, the outermost `act` is `travel-service`, and
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
and the actor token's key confirmation, `travel-service` is the outermost
actor, the `chain_id` is active and within the root-chain envelope, and the
requested `audience`, `resource`, and `scope` are permitted. It then resolves
the user's TravelRAS-local subject and returns:

~~~ json
{
  "access_token": "<ID-JAG for TravelRAS>",
  "issued_token_type": "urn:ietf:params:oauth:token-type:id-jag",
  "token_type": "N_A",
  "expires_in": 300,
  "chain_id": "01JZ8F4J9J8Y3NDK5WQ4P9K7Q2"
}
~~~

The decoded ID-JAG for TravelRAS is the one shown in {{onward-id-jag}}: same
user, but `sub` is now `travel-local-subject`, and it carries no `chain_id`.
The `chain_id` is returned again as a response parameter so a further hop to
BookingRAS can repeat this step.

## Using the TravelRAS ID-JAG {#example-use}

TravelService exchanges the TravelRAS ID-JAG at TravelRAS for an access token
(AT2), presenting a fresh DPoP proof with the same `travel-service` key, and
calls TravelAPI. TravelRAS processes the ID-JAG as an ordinary ID-JAG and
never sees the Identity Continuation Assertion or the `chain_id`.

# Acknowledgments
{:numbered="false"}

The author thanks the authors of OAuth Identity and Authorization Chaining
Across Domains, the Identity Assertion JWT Authorization Grant, Waffles, and
Attenuating Authorization Tokens, on whose work this profile builds.
