# RAS-binding model rewrite — architecture decisions (authoritative, rev 2)

Base: commit 9faf537. Branch: ras-binding-model.
Rev 2 folds in the nine review corrections. Approved architecture; this doc is
implementation-authoritative.

## The one-line model
RAS acceptance is the activation and provenance point for each hop. The hop
reference travels IN the ID-JAG; the accepting RAS binds it to authorization
state; it propagates LOCALLY within the accepting domain; the current-domain
actor obtains the NEXT ID-JAG before crossing the boundary. No raw handle
header, no response parameter.

## Cross-boundary artifacts (ONLY these)
1. Identity Continuation Assertion — addressed to the IdP.
2. The resulting ID-JAG — addressed to the next RAS, carries the next hop.
3. The resulting access token — addressed to the next API.
Transaction Tokens stay intra-domain (audience semantics).

## Citation-safety rule
Do NOT renumber chain-id rules (1-8) or validation rules (1-16); repurpose in
place. Every `{{...}}, rule N` must still resolve to sensible content.

## Hop state machine (correction 4) — NORMATIVE
1. IdP creates a PENDING hop when it issues the ID-JAG carrying the handle.
2. The target RAS ATOMICALLY binds the hop to authorization state ONLY if
   ID-JAG validation, client auth, sender-constraint, local policy, and
   access-token issuance all succeed (ACCEPTED state). See {{ras-processing}}.
3. The RAS's mapped Chain Authority issues an assertion ONLY from ACCEPTED,
   currently-active authorization state.
4. A valid assertion from that mapped CA IS the evidence of RAS acceptance the
   IdP relies on (CONTINUABLE). There is NO RAS-to-IdP callback.
5. Trust note (answers the earlier critique): "RAS-acceptance provenance" is
   therefore exactly trust in the mapped CA. It gives an honest-domain audit
   invariant, not an IdP-verifiable cryptographic proof of RAS acceptance. State
   this honestly in Security; do not overclaim.
6. A handle copied from an issued-but-REJECTED ID-JAG is unusable: no mapped CA
   will attest a hop that never reached ACCEPTED state.
Validation rule 7 MUST distinguish "known PENDING hop" from
"ACCEPTED/CONTINUABLE hop," or state that a valid mapped-CA assertion is what
satisfies the CONTINUABLE condition.

## chain-id rules — repurpose in place
- Rule 1: IdP MUST embed a fresh handle as the `identity_continuation_handle`
  CLAIM of each continuation-capable ID-JAG (root + each continuation hop); no
  reuse. [DONE on branch]
- Rule 2: entropy/format; drop "HTTP field value" clause. [DONE]
- Rule 3: handle appears as an ID-JAG claim and in intra-domain chain context;
  crosses a boundary ONLY inside an ID-JAG or Identity Continuation Assertion.
  [DONE]
- Rule 4 (REPURPOSED): handle MUST NOT appear in an OAuth ACCESS TOKEN or in a
  Resource Server's EXTERNAL authorization claims; per-hop/per-domain freshness
  keeps pairwise-unlinkability. [DONE — but reconcile with TT visibility, below]
- Rule 6: RAS BINDS the handle to authorization state; RAS/RS/CA MUST NOT modify
  the value. [DONE]
- Rules 5,7,8: keep. [DONE]

## Transaction Token visibility (correction 1) — RESOLVED, recommended model
Adopt: the handle MAY be visible to AUTHORIZED transaction-chain workloads that
consume the Transaction Token (including API implementations), subject to
non-authority, confidentiality, and log-suppression rules. It MUST NOT appear in
an OAuth access token or in a Resource Server's external authorization claims.
=> Rewrite rule 4 / privacy to say "never in an access token or external
authorization claim," NOT "the API never sees it." Remove the flat "API never
sees it" language from spec and draft.

## Sections REMOVED
- {#response-param} response parameter — delete; xrefs -> rule 1 / {{ras-processing}}.
- {#context-binding} HTTP handle header — delete; replace with
  {#transaction-token-context}. xrefs -> {{transaction-token-context}}.
- `identity_continuation_exp` RESPONSE PARAMETER — delete (see lifetimes below).

## Lifetimes (correction 5) — do NOT conflate
Three distinct lifetimes, stated separately:
- ID-JAG `exp`: redemption lifetime of that grant JWT (short).
- RAS authorization state: local authorization lifetime + revocation.
- IdP chain state: AUTHORITATIVE continuation lifetime + revocation (may be much
  longer than any ID-JAG `exp`).
Removing the advisory response param is fine, but chain expiry is NOT derived
from ID-JAG `exp`. Advisory advance-warning of chain expiry, if needed, goes in:
authenticated task state, an OPTIONAL ID-JAG claim, or a management API. Keep
chain lifetime/revocation authoritative at the IdP ({{lifecycle}}).

## Sections ADDED
- {#ras-processing} "Continuation-Aware Resource Authorization Server" (NORMATIVE):
  validate ID-JAG; authenticate client; verify sender constraint; apply local
  policy; issue access token; ATOMICALLY bind handle->authorization state on
  success; record continuation-allowed; expose association only to own-domain
  TTS/CA privately; handle never enters an access token or external authz claim.
  Non-normative internal-record example.
- {#transaction-token-context} "Transaction Token Chain Context":
  `tctx.identity_continuation` = {iss, tenant, handle}; requester MUST NOT supply
  or override via request_details; TT stays intra-domain; workloads propagate
  unchanged; visibility per correction-1 model.
- CA issuance requirements (correction 6) — NORMATIVE, in {{assertion-issuance}}
  or {{transaction-token-context}}: a TT is NOT a workload credential and may be
  replayed within its lifetime, so the CA MUST also:
    * authenticate the requesting workload independently;
    * verify proof of possession;
    * bind that actor to the transaction;
    * reject caller-supplied handle substitution;
    * recheck the underlying RAS authorization is still active;
    * enforce POLICY-BOUNDED issuance per (txn, actor) with rate/fan-out limits
      and audit records (universal single-use is too strict; legitimate fan-out
      needs several assertions);
    * a caller target/purpose hint MAY limit issuance but MUST NOT become
      authoritative over the IdP's target decision.
- Discovery (correction 7) — TWO capabilities, do not overload one boolean:
    * IdP capability: accepts continuation assertions and issues
      continuation-capable ID-JAGs (retain/define an IdP metadata signal).
    * RAS capability: recognizes the continuation ID-JAG grant profile and binds
      the claim — advertise a continuation grant-profile identifier in the RAS's
      `authorization_grant_profiles_supported`.
- {#hop-authority-map} (fold into {{root-establishment}}): IdP records, per hop,
  the target RAS and the Chain Authorities authorized to attest continuation from
  it; accepts continuation from a hop only from an authorized CA of that RAS's
  domain.
- {#task-provenance} durable task-authorization provenance (correction 2):
  the background/scheduled model MUST root at an explicit PlatformRAS/TaskRAS:
    BriefingAgent obtains PlatformRAS ID-JAG containing H0
      -> PlatformRAS accepts it
      -> PlatformRAS creates durable task authorization bound to H0
      -> scheduler later supplies only task_id (never a raw handle)
      -> Platform TTS derives H0 from authenticated task state
      -> BriefingAgent continues H0 to CalendarRAS (sibling hop per run)
  Without this root the example falls back to a trusted sidecar handle (the very
  thing being removed). NOTE: this is the "TaskRAS" my critique flagged as a
  coupling smell; under the approved paradigm it is REQUIRED, so make it explicit.

## RAS acceptance is a GATE, not a ceiling (correction 8) — state explicitly
"RAS acceptance is a continuation gate, not an authorization ceiling. The IdP
continues to evaluate downstream targets solely against the root-chain envelope
and current actor policy." Cross-domain scope vocabularies are not generally
comparable; RAS-derived narrowing, if ever added, needs signed constraints and
an explicit intersection model (future item). The envelope remains the SOLE
authorization ceiling.

## Chain Authority ownership (NEW rule)
The CA that attests a continuation belongs to the domain whose RAS accepted the
PARENT hop. CA issues only from ACCEPTED RAS/TTS state (per state machine).

## Validation ({{validation}}) changes
- Onward/output ID-JAG now MUST CARRY `identity_continuation_handle` for the new
  hop (invert the current "MUST NOT carry" line).
- Rule 7: distinguish PENDING vs CONTINUABLE (state machine above).
- Add: assertion `iss` (CA) MUST be an authorized CA for the presented hop's
  target-RAS domain ({{hop-authority-map}}).
- `sid`/exclusions unchanged.

## Example source-side actor sequence (correction 3) — RECORD EXACTLY
Same-IdP:
  ExpenseRAS accepts H0
  ExpenseService (NOT TravelService) obtains Travel ID-JAG(H1)
  ExpenseService redeems it at TravelRAS and calls TravelAPI
  TravelRAS/TTS binds H1
  TravelService obtains Booking ID-JAG(H2)
Rule: the CURRENT-domain actor continues, before the boundary crossing. If an
implementer keeps "pass H0 to TravelService, then TravelService continues," the
undefined cross-domain handoff returns. Actor lineage in Travel ID-JAG:
travel is NOT yet present; act = expense-service acting-for expense-app.
Booking lineage: travel-service acting-for expense-service acting-for expense-app.
Gateway and Background examples follow the same rule (per proposal sketches).

## INVARIANT SWEEP (correction 9) — REQUIRED global pass, not section-local
The following obsolete invariants appear across abstract, Relationship section,
sender-constraint rationale, examples, and Design Rationale. Remove/replace ALL:
- "the onward ID-JAG is unchanged" — now carries the handle claim.
- "the RAS is continuation-unaware" — now continuation-aware (binds).
- "the RAS never sees the handle" — now sees its own hop's handle in the ID-JAG.
- "this document is not an ID-JAG profile" — it NOW profiles ID-JAG issuance and
  RAS processing.
- "ordinary ID-JAG processing is preserved" — RAS processing is extended.
REPLACE (not invert) the "Why Not a Profile of ID-JAG" rationale section: the
document now explicitly profiles ID-JAG. Add a final grep-driven sweep for these
phrases before declaring the rewrite done.

## What STAYS (already-correct decisions — keep)
- Opaque, random, non-bearer per-hop handle; dedicated (not `jti` or stable
  `dlg_id`); handle in the ID-JAG; fresh child / immutable parent; branch-specific
  lineage; source/current-domain continuation before crossing; TTs intra-domain;
  per-hop target-RAS -> authorized-CA mapping; no raw handle header; no response
  parameter; RAS-private state (not an RS-consumed access-token claim); scheduler
  holds a task id, not a handle.
- Root-chain envelope as SOLE ceiling; sender-constrained assertions; actor
  tokens + live DPoP; IdP target/scope/subject/policy evaluation; chain lifetime
  and revocation at the IdP; offline attenuation intra-domain; stable delegation
  correlation as a separate future feature.

## Terminology adds
- Transaction Token Service (TTS): derives the handle from ACCEPTED authorization
  state for the local transaction context.
- Continuation-capable ID-JAG: an ID-JAG carrying `identity_continuation_handle`.
- Hop states: PENDING, ACCEPTED, CONTINUABLE (revoked/expired terminal).
