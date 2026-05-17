---
title: "feat: Andamio Open Badges 3.0 Issuer deployment"
type: feat
status: blocked
date: 2026-05-16
deepened: 2026-05-16
reviewed: 2026-05-16
origin: spike/ (imported from orch ob3-prototype, 2026-05-16 — committed source of truth)
---

# feat: Andamio Open Badges 3.0 Issuer deployment

## ⏸ Resume Here (paused 2026-05-16 — read this first)

Plan is written, deepened (1 confidence pass), and document-reviewed (6 personas). The review
corrected a false premise (1EdTech "0/0" was never run — see Overview) and added Phase 0.
**5 strategic decisions are pending James's call** before this plan is implementable. Decisions
1–4 were about to be asked via a 4-question prompt; James paused to resume next session.
Recommendations are mine; pick up by deciding these, then update the plan + re-review.

| # | Decision | Options | My recommendation |
|---|----------|---------|-------------------|
| 1 | **Sequencing** | (a) Phase 0 hard gate, demos use labeled synthetic; (b) forcing function — build ahead; (c) Phase 0 parallel with Units 1–2 | **(a)** unless a funding/partnership demo deadline exists |
| 2 | **Issuer identity / Issue #8** | (a) attestation-host framing + resolve #8 narrow Q before Unit 3; (b) proceed T1 as-is, defer #8; (c) scope v1 to Andamio-controlled courses | **(a)** — defensible regardless of #8 outcome, honest to narrative |
| 3 | **Revocation** | (a) minimal StatusList2021 in v1; (b) reframe goal (no consumer-visible revocation v1); (c) keep as-is deferred | **(a)** — only signal mainstream verifiers honor; ~80/20 |
| 4 | **DID-doc location** | (a) CI-generated from KMS, served static by nginx; (b) live-from-service (current text); (c) decide in Phase 0 | **(a)** — reverses the deepening choice; drift-impossible + survives signing outage |
| 5 | **Deploy topology** | not a strategic fork | Apply default: separate Cloud Run service `credential-badges-issuer`, LB path-routing, ops-repo Terraform delta as named pre-Unit-4 prerequisite |

Full analysis of each is in `## Open Questions → Phase 0` and the Risks table. The other
document-review findings (P0/P1) are already auto-applied into the body; what remains is
only these 5 judgment calls. **Do not proceed to `ce:work` until 1–5 are decided and the
Phase 0 evidence gate is closed.**

## Overview

Promote the validated `ob3-prototype` spike (now committed at `spike/`) into a production
Open Badges 3.0 issuer, fully backed by Andamio as the cryptographic issuer. This turns
on-chain Andamio Credentials (Cardano native assets indexed by Andamioscan) into signed,
independently verifiable OB 3.0 / W3C VC 2.0 credentials.

**Validation status (corrected — read this first).** The credential mapping and OB3 *shape*
are well-developed but **not externally conformance-proven**. Per `spike/validation-results.md`:
local structural checks (14/14), JSON-LD context expansion, and a **VC-JWT** signature
round-trip PASS — on a **synthetic** credential, using the spike's own `did:key` resolver.
**1EdTech OB 3.0 conformance: NOT RUN** (membership-gated). The **Data Integrity
`eddsa-rdfc-2022`** path this plan ships was *not* the path locally validated (JWT was);
its only check is a self-loopback. And **no real `credential_claim` has ever been minted**
on preprod (spike Q1) — the validated sample is synthetic. Therefore this plan is **not**
"promote a proven artifact"; it is "harden a promising spike and obtain the conformance +
real-credential evidence it does not yet have." A **Phase 0 evidence gate** (below) must
close before Units 3–6 lock non-re-litigable decisions.

The work is: (0) obtain 1EdTech membership, run a **DI-signed** sample through the real
conformance kit, and produce ≥1 real preprod `credential_claim`; (1) replace the throwaway
`did:key` with a real Andamio `did:web` identity backed by managed keys; (2) stand up a
deployed assembly+signing service that deterministically re-derives credentials from
on-chain truth; (3) ship a public verification surface — preserving the static host's
allowlist/CI guarantees.

## Problem Frame

Today an Andamio Credential exists only as an on-chain Cardano native asset. No signed,
portable OB3 document is produced anywhere in production. The spike proved the full pipeline
works but cut deliberate corners (`spike/CORNERS-CUT.md`): a throwaway Ed25519 key persisted
to disk, `issuer.id` rewritten to a `did:key` that doesn't match the issuer name, no deployed
service, no public verification URL, no revocation read model. A credential is only "fully
backed by Andamio" when a strict third-party verifier can resolve `issuer.id` → a DID document
→ a verification method controlled by Andamio, and the document was signed by a key Andamio
actually custodies. That is the gap this plan closes.

## Requirements Trace

- **R1** — Andamio is the cryptographic OB3 issuer: real `did:web:credentials.andamio.io`,
  managed signing key, production issuer Profile, 1EdTech conformance preserved. (orig req 1; Issues #3, #5)
- **R2** — Assembly service maps Andamioscan on-chain data → signed OB3 `AchievementCredential`
  pinning the hosted context/issuer/badge image, populating `onChainAnchor` /
  `onChainAttestation`; lives in this repo as a deployed service. (orig req 2)
- **R3** — Holder binding: a defined `credentialSubject.id` strategy. (orig req 3)
- **R4** — Public verification surface: unauthenticated per-credential URL, server-side
  re-derivation each view, explicit states, on-chain revocation read model reconciled with
  OB3 status-list mechanism. (orig req 4; spike Q9)
- **R5** — `badge_id` ↔ Andamio Credential (`<course_id>.<slt_hash>`) mapping ownership and
  registry location, with a forward-compatible 1-credential-list data model. (orig req 5; Issue #11)
- **R6** — Issuer sovereignty: ship **T1** (single Andamio DID), design the **T2** per-org-DID
  path, defer **T3** org self-sovereign. Preserve the existing static host allowlist/CI
  guarantees throughout. (Issues #3, #4, #6, #7)

## Scope Boundaries

- **Not** building T2 per-org issuer DIDs or T3 org self-sovereignty — only designing the T1→T2 path.
- **Not** building OB3 Bitstring Status List revocation infrastructure — only the on-chain
  read model + a documented designed path (spike Phase 4).
- **Not** changing the on-chain protocol, mint policies, or the `credential_claim` transaction.
- **Not** rebuilding the auth'd self-only holder dashboard at
  `andamio-app-v2/src/app/(app)/credentials/page.tsx` — this is the public verifiable surface;
  backend-sharing is explicitly decided against (different trust + auth model).
- **Not** re-deriving the inner/outer Blake2b state hashes in the service (spike `mapping.md`
  notes TS/Go parity is incomplete) — the service trusts Andamioscan's indexed hashes as the
  on-chain source of truth, same as the spike.

### Deferred to Separate Tasks

- T2 per-org issuer DIDs + provisioning runbook: Issues #4, #6 (designed here, built later).
- Multi-issuer ↔ PQ3 cross-issuer-prereq reconciliation: Issue #7 (spec note here, full work later).
- Per-course on-chain policy key-custody research: Issue #8 (informs T2/T3; does not block T1).
- Recipient-chosen `credentialSubject.id` (email-hash / wallet / per-recipient DID): spike Q2 —
  v1 ships the pseudonymous URN default; richer binding is a later iteration tied to the claim flow.
- Bitstring Status List revocation: spike Phase 4.

## Context & Research

### Relevant Code and Patterns

- `spike/src/mapper.ts` — locked Andamio→OB3 transform; field-by-field spec in `spike/mapping.md`.
- `spike/src/sign-di.ts` — Data Integrity `eddsa-rdfc-2022` signer; this is the production
  signing path (1EdTech-validated). `spike/src/sign.ts` (JWT) is the spike baseline only.
- `spike/src/path-b.ts` — programmatic builder: alias + policy id → full OB3 doc from chain.
- `spike/src/verify.ts` + `spike/src/plutus.ts` — on-chain datum-membership check (the
  revocation/anchor read model foundation).
- `spike/src/keys.ts` — throwaway key generation; the thing being **replaced** by managed keys.
- Existing static host: `Dockerfile`, `nginx/default.conf`, `scripts/ci/check-allowlist.sh`,
  `.dockerignore`, `.github/workflows/{ci,deploy}.yml` — allowlist + tag-triggered WIF deploy
  to Cloud Run (GCP project `andamio-credentials`). Pattern to preserve, not break.

### Institutional Learnings

- `spike/CORNERS-CUT.md` — the 8 cut corners; this plan hardens corners 1, 2, 4, 5, 6, 7.
- `spike/open-questions.md` — 14 open questions; Q3/Q4 resolved, Q2/Q5/Q8/Q9/Q10 addressed here.
- `spike/prerequisite-chaining.md` — PQ1–PQ6 ratified defaults; PQ3 interacts with R6/Issue #7.
- Issues #3–#8, #11 — the issuer-identity decision graph; #3 is the ratified anchor (T1 chosen).

### External References

- W3C VC Data Model 2.0; Open Badges 3.0 (context `v3p0/context-3.0.3.json`).
- `did:web` method spec — DID doc served over HTTPS (this repo already serves HTTPS JSON-LD).
- Data Integrity `eddsa-rdfc-2022` cryptosuite — the canonicalized-RDF Ed25519 proof that
  passed 1EdTech in the spike (preferred over VC-JWT for JSON-LD-native verifier acceptance).
- GCP KMS (asymmetric Ed25519 sign) — managed key custody in the existing `andamio-credentials` project.

## Key Technical Decisions

- **Language: keep TypeScript.** The DI signing implementation is TS
  (`@digitalbazaar/data-integrity` + `eddsa-rdfc-2022`); the spike's "port to Go" note assumed
  the service would live in `andamio-api`, which changed (service lives here). Keeping TS
  avoids re-introducing canonicalization/signing risk. Caveat: this is *not* yet justified by
  an external conformance pass (see Phase 0) — it is justified by reusing the most-developed
  implementation; Phase 0 must confirm the TS DI path actually passes 1EdTech before this is
  treated as settled.
- **Signing: Data Integrity `eddsa-rdfc-2022`, not VC-JWT.** JSON-LD-native; the spike's
  real-recipient samples use it. **Not** 1EdTech-validated yet (the spike's only validated
  signature path was VC-JWT; DI has a self-loopback check only — `spike/CORNERS-CUT.md` #7).
  Phase 0 gates this. JWT retained as a fallback until DI conformance is observed, not assumed.
- **Issuer identity: T1 — single `did:web:credentials.andamio.io`.** DID doc at
  `/.well-known/did.json`; per-course differentiation via Profile name/logo only. Ratified in Issue #3.
- **Key custody: GCP Cloud KMS `EC_SIGN_ED25519`, HSM protection level.** Confirmed feasible
  (GA 2024): Cloud KMS Ed25519 is PureEdDSA over **raw** message bytes (the `data` field, never
  `digest`) — exactly what `eddsa-rdfc-2022` needs, since the cryptosuite canonicalizes+hashes
  *before* calling the signer. Private key never leaves KMS. **Do not** use protection level
  `EXTERNAL` (Ed25519 unconfirmed there). **Do not** enable KMS automatic rotation (it would
  silently desync the published DID-doc public key from the signing version) — rotation is
  managed explicitly: new KMS key version → new versioned DID-doc fragment → new credentials
  use the new fragment. Replaces `spike/src/keys.ts` disk key (corner 1).
- **Signer integration is a first-class seam, not a hack.** `new DataIntegrityProof({ signer,
  cryptosuite: eddsaRdfc2022 })` accepts a custom signer `{ id, algorithm:'Ed25519',
  async sign({data}) }`. The KMS adapter MUST return the **raw 64-byte** Ed25519 signature
  unmodified and **assert `length === 64`** before passing it to `DataIntegrityProof` (Cloud
  KMS already returns raw; any DER/base64/hex transcode corrupts every proof). `signer.id` =
  the fully-qualified versioned DID URL (`did:web:credentials.andamio.io#key-YYYY-MM`),
  byte-identical to the DID-doc fragment, forever. Verification method published as
  `Multikey` / `publicKeyMultibase`, listed in **both** `verificationMethod` and
  `assertionMethod`. Old verification methods are **retained in the DID doc** while any
  credential signed under them must still verify (non-expiring badges ⇒ keys published
  effectively forever); removing a verification method is reserved for the compromise
  kill-switch, which intentionally invalidates credentials signed under it.
- **Signing is HARD-gated on a positive on-chain anchor check (security-critical).** The KMS
  signing call MUST be unreachable unless a single shared datum read proves: (a) the
  caller-supplied `policyId` is a key in the recipient's current global-state datum credential
  map, (b) the on-chain hash for that policy is **byte-equal** to the caller-supplied
  `sltHash`, and (c) the on-chain alias bytes match the resolved `recipient`. Failing any →
  `not-found` / `revoked-signal` with **zero** KMS operations. The gate read and the signed
  assertion consume one datum read (no TOCTOU). The issuer DID is a fixed server-side constant;
  it is never derived from request input and has no fallback key/DID path. Rationale: without
  this, the endpoint is an open Andamio-signing oracle (`spike/src/path-b.ts` resists
  "policy absent" but does **not** assert the caller `sltHash` equals the on-chain hash;
  `spike/src/verify.ts` is a post-hoc CLI harness, not a precondition).
- **DID document is served by the dynamic service, derived live from the KMS public key.**
  `did:web:credentials.andamio.io` resolves to `/.well-known/did.json`; the dynamic service
  owns that route and builds the document from `KMS getPublicKey` of the active signing key
  version(s). Rationale: makes DID-doc/signing-key drift **structurally impossible** (same
  philosophy as no-persistence) and removes a static file an attacker could swap. Tradeoff:
  did:web resolution availability is coupled to the service; mitigated by serving did.json
  with an edge/CDN cache (independent of Andamioscan, cheap, no KMS sign) and a documented
  freshness/rotation cadence. This **supersedes** the earlier "did.json on the static host"
  framing — the static host keeps `/context`, `/issuer` Profile, `/badges` only.
- **Deterministic reproducibility, not mere re-derivation — scoped honestly.** Every request
  rebuilds from Andamioscan state via the locked mapper; pinning `validFrom` **and**
  `proof.created` to the claim-tx `block_time` removes the *obvious* time-varying fields. But
  byte-stability is **only achievable within a single build + key epoch** and is **not yet
  demonstrated**: RDFC-2022 canonicalizes expanded RDF, so (i) a context-version change
  (Andamio context is still `v0`, "shape may change"), (ii) any Andamioscan field/ordering
  change in the mutable course content the body embeds, or (iii) a key rotation (re-derived
  old credentials sign under the *current* fragment, since nothing is persisted) each silently
  break byte-identity for already-distributed copies. **Open dependency, not resolved:** the
  spike does not have a per-recipient claim-tx `block_time` source (it uses policy-mint time;
  Q8 punted; no real claim exists — Q1). Phase 0 must either empirically demonstrate
  byte-stability (same on-chain input, two independent builds + a simulated rotation → assert
  byte-identity) or the plan drops the byte-stability contract and **reconsiders the rejected
  "durable proof + derived body" alternative**. Until then this is a gating prerequisite of
  Unit 4 and the KMS-SPOF mitigation that depends on it is provisional.
- **Production document loader is closed and build-time-pinned.** No arbitrary network fetch
  at request time during canonicalization: the W3C VC 2.0 / OB3 / Andamio contexts and the
  `did:web` resolver are bundled or fetched only from the pinned same-origin host with
  integrity checks; any unlisted URL is rejected (no network fallthrough). Rationale: a
  per-request server-side JSON-LD signer with a permissive loader (spike CORNERS-CUT #3/#5/#7)
  is an SSRF + availability dependency on the signing path — not acceptable as "TBD".
- **`credentialSubject.id`: v1 = pseudonymous URN** `urn:andamio:{network}:recipient:{studentStateAsset}`
  (spike default — on-chain-derivable, privacy-preserving). Recipient-chosen binding deferred (Q2).
- **Badge registry: in this repo.** `badge_id` is the `<course_id>.<slt_hash>` URN itself
  (1:1 Andamio Credential → OB3 Achievement). Data model stores a credential **list** but v1
  only ever emits length-1 lists (forward-compat for future multi-credential badges; no
  multi-credential logic/UX built).

## Open Questions

### Resolved During Planning

- Issuer sovereignty tier → T1 now, T2 designed, T3 deferred (user-confirmed; Issue #3).
  **Caveat:** T1's *correctness* (a single Andamio DID as `issuer` for every course) depends
  on Issue #8 (per-course mint-policy custody), which Phase 0 must resolve — see below.
- Service home → new deployed service in this repo (user-confirmed).
- Pull scope → spike committed as source of truth here; internal strategy gitignored (done).
- Signing approach → Data Integrity `eddsa-rdfc-2022` *intended* (JSON-LD-native), JWT as
  fallback. **Not** conformance-validated yet — see Phase 0.
- Language → TypeScript (reuse the most-developed implementation; not conformance-justified yet).
- KMS feasibility → confirmed by external research: Cloud KMS `EC_SIGN_ED25519` (HSM) is
  PureEdDSA over raw bytes, injectable as a custom `DataIntegrityProof` signer returning raw
  64-byte sigs. This is the one foundational item that *is* solidly established.

### Phase 0 — Must Resolve Before Units 3–6 Lock (was wrongly "Resolved")

These were incorrectly marked resolved; the spike evidence does not support them. They are
**gating** — Units 3–6 freeze the mapper, language, signing path, and identity model on them:

- **1EdTech conformance NOT RUN.** `spike/validation-results.md:13` — membership-gated, never
  executed; what passed was a *local* check on a *synthetic* credential via the spike's own
  resolver, on the **VC-JWT** path (not the DI path this ships). Obtain membership, run a
  **DI-signed** sample through the real kit. Until then "this is not greenfield" is false.
- **No real `credential_claim` exists** on preprod (spike Q1). Produce ≥1 real claimed
  credential before the mapper/identity model is frozen against synthetic fixtures.
- **`validFrom`/`proof.created` source unproven.** The spike uses *policy-mint* time, not
  per-recipient *claim-tx* `block_time` (Q8 punted). Byte-stable determinism — and the
  KMS-SPOF mitigation built on it — is unestablished until this source is real and shown
  stable across fetches/builds/rotation. If unrecoverable, choose a documented deterministic
  fallback or reconsider "durable proof + derived body".
- **Issue #8 (mint-policy custody)** is a T1-*correctness* gate, not just a T2/T3 input: if
  course owners (not Andamio) control the mint policy, a single Andamio `issuer` DID asserts
  authority it does not hold. Resolve the narrow question ("does Andamio control the policy
  for v1 courses?") before Unit 3 freezes issuer semantics.
- **DID-doc location** (live-from-service vs CI-generated-static) — a genuine tradeoff
  surfaced in review (drift vs cache-coherence vs availability coupling); pending decision.

### Deferred to Implementation

- Exact GCP KMS key resource layout and IAM binding — must use a **dedicated sign-only
  service identity, distinct from the deploy WIF identity** (a CI compromise must not be able
  to sign); Cloud Audit Logs for CryptoKeyVersion use enabled and retained. Resource layout
  depends on `andamio-credentials` infra (Terraform in the private ops repo).
- Domain routing: `/.well-known/did.json` and `/credentials/...` both served by the dynamic
  service; path-routing vs subdomain vs which artifact owns the apex — confirm against Cloud
  Run + domain-mapping behavior at deploy time. The static host keeps `/context`, `/issuer`,
  `/badges`.
- Andamioscan endpoint/field shape **and a freshness/confirmation-depth signal** for the live
  read path — confirm against the running `andamio-api` proxy; needed to distinguish
  indexer-lag from true revocation (see Unit 5). Note the plan currently says both
  "Andamioscan via andamio-api proxy" and `Blockfrost.fromEnv` — reconcile which is the
  production read path (the gate should arguably read chain *directly*, not via a single
  indexed oracle, since it authorizes signing).
- **Deployment topology (blocking before Unit 4):** the existing `.github/workflows/deploy.yml`
  is concretely bound to Cloud Run service `credential-badges`, a single `v*.*.*` tag trigger,
  and a WIF identity ref-constrained in private ops Terraform. "Two independently tag-deployed
  artifacts" is not free: needs distinct service names, an apex/path routing decision (the
  static host and the service both answer `credentials.andamio.io`; Unit 2 *freezes*
  `issuer.url` against that apex), and ops-repo Terraform deltas (second WIF/SA, domain
  mapping). Promote from "confirm at deploy time" to a named pre-Unit-4 prerequisite.
- `credentialSubject.id` derivation is ambiguous between the two spike impls
  (`mapper.ts` uses `studentStateAsset`; `path-b.ts` uses bare `alias`). The gated single
  datum read yields `alias`, not `studentStateAsset` — specify exactly which value and field,
  and reconcile any extra lookup against the "one datum read, no TOCTOU" invariant.
- The signed body embeds `accessToken.userTokenHolder` (current wallet address) — this both
  de-anonymizes the pseudonymous subject and **breaks byte-stability across a wallet
  transfer**. Default decision: **exclude `userTokenHolder` from the signed document** unless
  a written rationale justifies keeping it.

## Output Structure

    credential-badges/
      issuer/
        profile.jsonld            # MODIFIED — v0 minimal → production Profile (id=did:web)
      service/                    # NEW — promoted issuer service (TypeScript)
        src/                      # graduated from spike/src (mapper, sign-di, path-b, verify, plutus)
                                  # + KMS signer adapter, did.json generator, closed doc loader
        Dockerfile                # service container (separate from static-host Dockerfile)
                                  # serves /.well-known/did.json (live from KMS) + /credentials/...
      docs/
        plans/                    # this plan
        runbooks/
          issuer-provisioning.md  # NEW — T1 runbook, key-compromise response, T2 path (Issue #6)
        verifier-guidance.md      # NEW — verification, on-chain revocation, indexer-lag caveat
      CODEOWNERS                  # NEW — review gate on trust-critical issuer/ + did.json generator
      spike/                      # imported source of truth (CI-ignored, not served)
      .github/workflows/
        deploy.yml                # MODIFIED — add service build/deploy lane (tag-gated)
        ci.yml                    # MODIFIED — service tests + DID/profile smoke + key-pin invariant

## High-Level Technical Design

> *This illustrates the intended approach and is directional guidance for review, not
> implementation specification. The implementing agent should treat it as context, not code
> to reproduce.*

```
Verifier / consumer
        │  GET credentials.andamio.io/credentials/{network}/{policyId}/{sltHash}/{recipient}
        ▼
┌──────────────────────────────────┐     ┌─────────────────────────────┐
│  Issuer Service (Cloud Run, TS)  │     │ Static Host (nginx, existing)│
│  serves /.well-known/did.json    │     │ allowlisted, immutable cache │
│   (built live from KMS pubkey —  │ pins│                             │
│    drift impossible)             │◄────┤ /context/v0.jsonld          │
│  serves /credentials/...         │     │ /issuer  (prod Profile, NEW)│
│                                  │     │ /badges/{badge_id}.svg      │
│  1. validate path params         │     └─────────────────────────────┘
│     (network = trust enum)       │
│  2. ONE datum read (Andamioscan  │
│     via andamio-api proxy)       │
│  3. HARD GATE: policy∈datum ∧    │
│     onchain-hash == sltHash ∧    │
│     alias == recipient ?         │
│        no ─► not-found /         │
│             revoked-signal       │
│             (ZERO KMS calls)     │
│        yes ▼                     │
│  4. map → OB3 (mapper.ts),       │
│     validFrom/proof.created =    │
│     claim-tx block_time          │
│  5. sign via KMS adapter ────────┼───► GCP KMS EC_SIGN_ED25519 (HSM)
│     (raw 64-byte, DI eddsa-      │      key never leaves KMS
│      rdfc-2022)                  │
│  6. emit byte-stable doc + state │
└──────────────────────────────────┘

State machine per request (gate decides BEFORE any KMS op):
  not-found        ── policy/alias absent in datum, OR sltHash != on-chain hash
  revoked-signal   ── claim tx exists but pair now absent (burn/transfer);
                       MUST be distinguished from indexer-lag (freshness check)
  anchored+signed  ── gate passed AND KMS sign OK (deterministic, byte-stable)
  anchored+unsigned── gate passed, KMS unavailable: degraded, on-chain truth only
```

## Implementation Units

- [ ] **Unit 1: Managed issuer key + allowlist/CI foundation + provisioning/compromise runbook**

**Goal:** Stand up the KMS-held signing key with correct custody, and reconcile the static
host's allowlist/CI **before** any `service/` directory exists, so Phases 2–3 are not red-CI
by construction. No static `did.json` is created — the DID document is served live by the
service (Unit 4).

**Requirements:** R1, R6

**Dependencies:** None (hard prerequisite for Units 3–6).

**Files:**
- Modify: `scripts/ci/check-allowlist.sh` (add `service/`, `spike/`, `docs/` to IGNORED — they
  must never be served by the static host), `.dockerignore`, static-host `Dockerfile`
- Create: `CODEOWNERS` (review gate on `issuer/profile.jsonld` and the service's did.json
  generator + signer — trust-critical; the generic tag-gate is *not* sufficient review here)
- Create: `docs/runbooks/issuer-provisioning.md` — T1 key creation/custody; **explicit
  key-compromise response section distinct from rotation**; T2 path designed (Issue #6)
- Test: extend `.github/workflows/ci.yml` allowlist assertions

**Approach:**
- KMS key: purpose `ASYMMETRIC_SIGN`, algorithm `EC_SIGN_ED25519`, protection level **HSM**,
  in the `andamio-credentials` project. **Automatic rotation disabled.** A **dedicated
  sign-only service identity, separate from the deploy WIF identity** (CI compromise must not
  be able to sign). Cloud Audit Logs for CryptoKeyVersion use enabled + retained. Infra
  (Terraform/IAM) lives in the private ops repo — runbook captures repeatable steps + rollback.
- Runbook **rotation** path: new KMS key version → new versioned DID-doc fragment
  (`#key-YYYY-MM`) → new credentials sign under the new fragment; old fragments retained while
  any credential signed under them must still verify (coordinate retention with credential
  lifetime — non-expiring badges ⇒ keys published effectively forever).
- Runbook **compromise** path (separate, deliberately destructive): remove the compromised
  verification method from the DID doc **entirely** (intentionally breaks all credentials
  signed under it), revoke the sign IAM binding, disable/destroy the key version. State that
  re-issuance is cheap *because* of the no-persistence design — turn the no-DB property into
  the compromise mitigation. The verifier-facing consequence is documented in Unit 5.
- Allowlist reconciliation lands here, with single ownership, so the moment Unit 3 creates
  `service/` the static-host CI is already correct (it currently fails closed on any unlisted
  top-level path — good, but must be pre-reconciled).

**Patterns to follow:** allowlist lockstep across `Dockerfile` / `.dockerignore` /
`check-allowlist.sh` (Issue #4 pattern); existing IGNORED-prefix handling for `.git`/`.github`.

**Test scenarios:**
- Edge case: allowlist CI passes with `service/`, `spike/`, `docs/` IGNORED, and still fails
  on a genuinely unlisted top-level path (guarantee not weakened).
- Edge case: a file placed under `service/` does not become servable by the static host.
- Test expectation: KMS/IAM/runbook content has no behavioral tests in this repo (infra in
  ops repo) — covered by the Unit 4 deploy-time key-pin invariant.

**Verification:** Allowlist CI green with the new IGNORED prefixes; runbook documents
rotation **and** a distinct compromise kill-switch; KMS key exists with HSM + no-auto-rotation
+ dedicated sign identity + audit logging.

- [ ] **Unit 2: Production issuer Profile**

**Goal:** Upgrade `issuer/profile.jsonld` from v0 minimal to a production OB3 Profile whose
`id` is the `did:web` and `url` is the Profile, consistent with Unit 1.

**Requirements:** R1

**Dependencies:** Unit 1 (DID id must exist to reference).

**Files:**
- Modify: `issuer/profile.jsonld`
- Test: `.github/workflows/ci.yml` (existing `/issuer` smoke; extend to assert `id` = did:web)

**Approach:** Set `id` to `did:web:credentials.andamio.io`, keep `url` →
`https://credentials.andamio.io/issuer`, real Andamio name/description/logo. Remove the
spike disclaimer. Profile stays mutable (non-immutable cache) by existing design, **but
`issuer.id` is a frozen field within the otherwise-mutable Profile** — a static-host
redeploy may change display fields, never `id`. CODEOWNERS (Unit 1) gates this file.

**Patterns to follow:** existing `issuer/profile.jsonld` structure and `nginx` `/issuer`
exact-match serving.

**Test scenarios:**
- Happy path: `GET /issuer` → 200 `application/ld+json`, `type` includes `Profile`,
  `id` = the did:web, `url` = the Profile URL.
- Edge case: `issuer.id` here matches `issuer.id` embedded in freshly assembled credentials
  AND the service's served did.json `id` (no drift between Profile, credentials, DID doc).
- Edge case: CI fails if a Profile change alters `issuer.id` (frozen-field invariant).

**Verification:** Verifier dereferences both `issuer.id` and `issuer.url` with no warnings.

- [ ] **Unit 3: Graduate spike → production assembly/signing library**

**Goal:** Move the mapper + DI signer + chain reader from `spike/src/` into `service/src/`
as a production module that signs via KMS instead of a disk key, with the anchor gate as a
blocking precondition.

**Requirements:** R2, R3, R5

**Dependencies:** Unit 1 (KMS key + DID); **Phase 0 gates** (1EdTech DI conformance, real
claim, `block_time` source, Issue #8) — Unit 3 freezes the mapper, so it must not start
until Phase 0 closes.

**Concrete controls (from security review — must be specified, not left to the implementer):**
- Anchor gate (a) policy ∈ the recipient's global-state datum credential-map key set,
  (b) on-chain hash **byte-equal** caller `sltHash`, (c) datum alias hex **byte-equal**
  `asciiToHex(recipient)`; if the datum alias contains any non-printable-ASCII byte the gate
  returns `not-found` unconditionally (no encoding-ambiguity resolution).
- The gate **reuses the single in-memory datum object**, not `spike/src/verify.ts`'s
  independent `bf.getTxOutput` fetch (that fetch would reintroduce the forbidden TOCTOU).
- **No in-process or edge caching of the datum read on the gate→sign path** — every signing
  request performs a fresh datum read; caching is permitted only on the non-signing
  verification view.
- Exclude `accessToken.userTokenHolder` (wallet address) from the signed document.
- `signer.id`/issuer DID is a fixed server-side constant; delete the `spike/src/path-b.ts`
  hardcoded `did:key` fallback. Pin `validFrom`/`proof.created` to claim-tx `block_time`.

**Files:**
- Create: `service/src/` — graduated `mapper.ts`, `sign-di.ts`, `path-b.ts`, `verify.ts`,
  `plutus.ts`; a **KMS signer adapter** replacing `keys.ts`; an **anchor-gate** module; a
  **closed document loader** + **`did:web` resolver** replacing the spike's permissive loader
- Create: `service/package.json`, `service/tsconfig.json`
- Test: `service/src/__tests__/` (mapper fidelity, KMS adapter, anchor-gate, loader)

**Approach:**
- Keep `mapper.ts` byte-faithful to `spike/mapping.md` — it is the locked spec; do not
  "improve" it. 1:1 Andamio Credential → OB3 Achievement; `badge_id` = the three-part URN.
  Data model carries a credential **list**, v1 emits length-1 only.
- **KMS signer adapter:** implement `{ id, algorithm:'Ed25519', async sign({data}) }` and pass
  to `new DataIntegrityProof({ signer, cryptosuite: eddsaRdfc2022 })`. Canonicalization +
  hashing stay inside the library (the adapter is a pure byte→byte leaf). Call Cloud KMS with
  the message in the **`data`** field (never `digest`); return the signature as a raw
  `Uint8Array` **unmodified** — assert `length === 64`; no DER/base64/hex transcode. `signer.id`
  = `did:web:credentials.andamio.io#key-YYYY-MM`, the fixed server-side issuer DID + active
  versioned fragment — **never** request-derived, **no** `did:key` fallback (delete the
  `spike/src/path-b.ts` hardcoded `did:key` default).
- **Anchor gate (security-critical):** one shared datum read; signing is unreachable unless
  (a) caller `policyId` ∈ recipient global-state datum map, (b) on-chain hash **byte-equal**
  caller `sltHash`, (c) on-chain alias == resolved `recipient`. The spike's `verify.ts`
  equality logic becomes a **blocking precondition**, not a post-hoc CLI. Gate failure ⇒
  `not-found`/`revoked-signal`, zero KMS calls. No TOCTOU: the gate and the signed assertion
  use the same datum read.
- **Closed document loader + `did:web` resolver:** build-time-pinned context set (W3C VC 2.0
  / OB3 / Andamio `v0`), no arbitrary network fetch at request time, unlisted URL rejected
  (no fallthrough); real `did:web` resolver replacing spike `didKeyResolve` (CORNERS-CUT
  #3/#5/#7 — owned here, not "TBD").
- `credentialSubject.id` = pseudonymous URN default (carry spike behavior).

**Execution note:** Characterization-first — before refactoring, assert the graduated mapper
reproduces an existing 1EdTech-passing sample (`spike/samples/*-real.jsonld`) byte-for-byte
under the same inputs (with `validFrom`/`proof.created` pinned). Lock behavior, then swap the
key backend and add the gate.

**Patterns to follow:** `spike/src/mapper.ts`, `spike/src/sign-di.ts`, `spike/src/path-b.ts`,
`spike/src/verify.ts`, `spike/src/plutus.ts`.

**Test scenarios:**
- Happy path: known on-chain inputs → OB3 doc structurally identical to the validated spike
  sample; KMS-signed credential verifies against the live DID doc.
- Happy path: KMS adapter returns exactly 64 bytes; an independent Ed25519 verifier validates
  it against the KMS `getPublicKey` of the same key version.
- **Error path (forgery — highest priority):** well-formed `policyId`/`sltHash`/`recipient`
  that is NOT in the on-chain datum, OR `sltHash` ≠ on-chain hash for a real policy →
  `not-found`/`not-anchored`, **the signer is asserted never invoked**, no signed doc emitted.
- Edge case: credential list of length 1 emits a single Achievement (multi-length unused in v1).
- Edge case: document loader rejects an unlisted `@context`/DID URL (no network fallthrough).
- Error path: KMS unavailable → signing fails cleanly; no unsigned doc emitted as signed.
- Integration: assemble → gate → KMS sign → independent DI verification round-trips green.

**Verification:** Graduated module reproduces 1EdTech 0/0 on existing samples signed by the
real KMS did:web key; the signer is provably unreachable for non-anchored input.

- [ ] **Unit 4: Deployed service — credential endpoint + live DID document**

**Goal:** Wrap Unit 3 in a deployed Cloud Run service that serves both
`/.well-known/did.json` (built live from the KMS public key) and the credential endpoint,
emitting deterministically reproducible, KMS-signed credentials.

**Requirements:** R1, R2, R4

**Dependencies:** Unit 3.

**Files:**
- Create: `service/src/server.ts` (HTTP entry: `/credentials/...` + `/.well-known/did.json`)
- Create: `service/Dockerfile`
- Modify: `.github/workflows/deploy.yml` (tag-gated service deploy lane, WIF-constrained),
  `.github/workflows/ci.yml`
- Test: `service/src/__tests__/server.test.ts`

**Approach:**
- `GET /.well-known/did.json` → built live from `KMS getPublicKey` of the active key
  version(s), published as `Multikey`/`publicKeyMultibase` in `verificationMethod` **and**
  `assertionMethod`. The public key the verifier sees and the key the service signs with come
  from the same KMS source ⇒ **drift impossible by construction**. Served with an edge/CDN
  cache (cheap, no Andamioscan, no KMS sign) and a documented freshness/rotation cadence;
  did:web resolution availability is consciously coupled to this service (tradeoff accepted —
  see Alternatives).
- `GET /credentials/{network}/{policyId}/{sltHash}/{recipient}`: **strict path-param
  validation before any chain call** — `network` is a **closed trust-affecting enum**
  (`mainnet`|`preprod`; it selects the upstream/credentials via `Blockfrost.fromEnv`),
  `policyId`/`sltHash` strict hex-length, `recipient`/alias constrained before
  `asciiToHex`/upstream-path interpolation (SSRF/path-injection). Then: one datum read →
  Unit 3 anchor gate → on pass, map with **`validFrom` AND `proof.created` pinned to the
  claim-tx `block_time`** (deterministic; byte-stable across fetches) → KMS sign → return
  JSON-LD. **No persistence** — re-derivation is byte-identical, so third parties can cache/
  re-share/offline-verify safely.
- **Abuse/cost control (load-bearing, since no-persistence removes the natural cache):**
  the anchor gate runs before any KMS op (no KMS spend on forgery attempts); Cloud Run
  `max-instances` cost ceiling; an edge rate limit; a documented Andamioscan/Blockfrost read
  budget; upstream-quota-exhausted ⇒ explicit error state, never a fake/unsigned-as-signed doc.
- Separate Cloud Run service; same `andamio-credentials` project + WIF/tag-gate discipline.
  Static host deploy lane unchanged.

**Patterns to follow:** existing `.github/workflows/deploy.yml` WIF + `refs/tags/v*`
constraint; `spike/src/path-b.ts` build flow.

**Test scenarios:**
- Happy path: valid on-chain credential → 200, signed OB3 doc, verifies independently.
- Happy path: two requests for the same logical credential return **byte-identical** docs
  (deterministic — `validFrom`/`proof.created` from block_time, not request time).
- Happy path: `GET /.well-known/did.json` `verificationMethod` byte-equals `KMS getPublicKey`
  of the active key version (deploy-time invariant — replaces a committed-file key-pin).
- Error path: well-formed but non-anchored input → `not-found`, **no KMS sign call**, no doc.
- Error path: `network` not in the enum / malformed hex params → 400, **no chain call**.
- Error path: Andamioscan unreachable → 5xx explicit error state; Andamioscan stale (lag) is
  handled in Unit 5, never silently surfaced as `revoked`.
- Integration: deployed `*.run.app` URL serves a doc a third-party DI verifier accepts; the
  served did.json resolves `did:web:credentials.andamio.io`.

**Verification:** Public URL returns a 1EdTech-conformant, byte-stable credential; served
did.json provably matches the KMS signing key; non-anchored input incurs zero KMS cost.

- [ ] **Unit 5: Public verification surface + on-chain revocation read model**

**Goal:** A human-facing, unauthenticated verification view with explicit states and a
defined on-chain revocation signal, reconciled with the OB3 status-list mechanism.

**Requirements:** R4

**Dependencies:** Unit 4.

**Files:**
- Create: `service/src/verify-view.ts` (server-rendered status page; reuse `spike/src/render.ts` approach)
- Create: `docs/verifier-guidance.md`
- Test: `service/src/__tests__/verify-view.test.ts`

**Approach:**
- States: `anchored+signature-valid`, `anchored+signature-unavailable` (degraded — still
  shows on-chain truth), `not-found`, `revoked-signal` (claim tx exists but the
  `(policyId, typed_hash)` pair is absent from the recipient's current global-state datum —
  burn/transfer, via `spike/src/verify.ts` + `plutus.ts`).
- **Indexer-lag safety (must-fix):** `revoked-signal` MUST be distinguishable from
  Andamioscan staleness. "Absent because burned" and "absent because the indexer lags/partial-
  indexed" look identical at the datum layer; surfacing a held, valid credential as `revoked`
  is *worse* than an outage (silent incorrectness vs honest error). Use a freshness /
  confirmation-depth signal on the Andamioscan response (or a fallback direct chain read)
  before declaring `revoked-signal`; if freshness is indeterminate, return a distinct
  `indeterminate` state, never `revoked`.
- Reconcile with OB3: v1 emits **no** `credentialStatus`. `docs/verifier-guidance.md` states
  the normative rule — "the on-chain asset is authoritative; anchor presence (Andamioscan-
  mediated, may lag chain by N) confirms the credential is still held" — and documents two
  things explicitly: (1) the issuer's anchor/revocation signal is *mediated by an indexer*,
  with the direct Blockfrost/cexplorer evidence URLs for independent confirmation; (2)
  **key-compromise containment**: removal of a verification method from the DID doc
  intentionally invalidates credentials signed under it, and verifiers should treat
  anchor-presence as the authoritative liveness signal, not the signature alone. Bitstring
  Status List is the designed-but-deferred robust mechanism (spike Phase 4).

**Patterns to follow:** `spike/src/verify.ts`, `spike/src/plutus.ts`, `spike/src/render.ts`.

**Test scenarios:**
- Happy path: valid held credential → `anchored+signature-valid`.
- Edge case: claim tx exists, asset no longer in datum, indexer fresh → `revoked-signal`.
- Edge case: pair absent but Andamioscan freshness indeterminate/lagging → `indeterminate`,
  **never** `revoked-signal` (held valid credential must not show as revoked on lag).
- Edge case: unknown credential → `not-found` (no signed doc emitted).
- Error path: signing backend down but chain readable → `anchored+signature-unavailable`,
  on-chain facts still shown.
- Integration: revocation read uses the same datum-membership logic as the assembly anchor
  gate (one source of truth, no divergence).

**Verification:** Each state reachable and correct; verifier-guidance documents the
on-chain-authoritative rule and the deferred status-list path.

- [ ] **Unit 6: T2/PQ3 designed-not-built + spec hygiene**

**Goal:** Capture the deferred T2/PQ3 design so the chosen T1 architecture stays coherent,
and document the shipped system. (Allowlist/CI reconciliation moved to Unit 1 as a hard
prerequisite — it is not cleanup.)

**Requirements:** R5, R6

**Dependencies:** Units 1–5.

**Files:**
- Modify: `docs/runbooks/issuer-provisioning.md` (extend with T2 `issuers/<org>` path scheme
  + per-org key custody design — Issues #4, #6, designed not built)
- Create: `docs/badge-registry.md` (badge_id = `<course_id>.<slt_hash>` URN; SVG-primary;
  presentation-only invariant from Issue #11; length-1-list rule)
- Modify: `README.md` ("What's here" — service vs static host, badge_id convention,
  trust-critical files)

**Approach:**
- Document the T1→T2 path (per-org DID slug convention, key-custody fork) and the Issue #7
  PQ3 cross-issuer-prereq reconciliation as a written spec note — decision-coupled,
  deliberately deferred, so the deferral is coherent rather than ad hoc.
- README states which files are trust-critical (CODEOWNERS-gated) and the static/dynamic split.

**Test scenarios:**
- Test expectation: none — design/docs content; the allowlist CI invariants live in Unit 1
  and the key-pin invariant in Unit 4.

**Verification:** README + runbook + badge-registry accurately describe the shipped
architecture, the trust boundary, and the deferred T2/PQ3 path.

## System-Wide Impact

- **Interaction graph:** New dynamic service reads Andamioscan via the existing `andamio-api`
  proxy, calls GCP KMS to sign, and serves both `/credentials/...` and `/.well-known/did.json`
  (built live from KMS). Static host gains only the production `/issuer` Profile. The auth'd
  holder dashboard in `andamio-app-v2` is intentionally untouched and not backend-shared.
- **Error propagation:** Chain-read or signing failure must degrade to an explicit state
  (`not-found` / `signature-unavailable`), never a silently invalid or fake-signed document.
- **State lifecycle risks:** No signed documents persisted — eliminates stale-cache drift by
  construction. Key rotation handled via multi-method DID doc; pre-rotation credentials must
  still verify (test in Unit 3).
- **API surface parity:** The pseudonymous `credentialSubject.id` and 1:1 badge model must
  match `spike/mapping.md` exactly so spike-validated samples remain conformant.
- **Integration coverage:** Unit 4/5 cross-layer tests against a deployed URL + an independent
  DI verifier are the real proof; unit tests on the mapper alone are insufficient.
- **Unchanged invariants:** Static host allowlist, tag-gated WIF deploy, immutable context
  cache, on-chain protocol, and the badge-image-is-presentation-only rule are explicitly
  preserved.
- **Threat model (explicit):** The issuer's trust boundary includes (1) the GCP KMS key +
  its sign-only IAM (a single Andamio root key signs all credentials — compromise mints
  unlimited back-dated valid-forever credentials; mitigated by the DID-doc-removal
  kill-switch in Unit 1/5, dedicated sign identity ≠ deploy identity, audit logging);
  (2) the dynamic service code that serves did.json + gates signing (supply-chain: a
  malicious commit could add an attacker verification method or weaken the gate — mitigated
  by CODEOWNERS on trust-critical paths + the Unit 4 deploy-time key-pin invariant; the
  generic tag-gate is necessary but **not sufficient** review for these); (3) the unauth
  public endpoint (signing-oracle risk — neutralized only by the Unit 3/4 hard anchor gate).
  Andamioscan is an upstream trust dependency: the surface proves "the indexer says this
  pair is in the datum," not chain directly — stated in verifier-guidance with independent
  evidence URLs.

## Risks & Dependencies

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| **Signing-oracle: endpoint signs well-formed-but-not-on-chain input** | Med | **Critical** | Unit 3/4 hard anchor gate (policy∈datum ∧ hash byte-equal ∧ alias match) is a blocking precondition with zero KMS ops on failure; test asserts signer never invoked on non-anchored input |
| **KMS key compromise → unlimited valid-forever forged credentials** | Low | **Critical** | Unit 1 runbook compromise kill-switch (remove VM from DID doc entirely); dedicated sign identity ≠ deploy WIF identity; KMS audit logging; verifier-guidance says anchor-presence is authoritative, not signature alone |
| **did:web supply-chain: malicious commit swaps verification method / weakens gate** | Low | **High** | CODEOWNERS on trust-critical paths; Unit 4 deploy-time invariant (served did.json key == KMS getPublicKey); service-built did.json removes a swappable static file; tag-gate is necessary-not-sufficient |
| **Andamioscan lag → held valid credential shown as `revoked-signal`** | Med | High | Unit 5 freshness/confirmation-depth check; `indeterminate` state instead of `revoked` when lag is possible; distinct from "Andamioscan down" (honest 5xx) |
| **Unauth per-request KMS-sign cost/DoS (no-persistence removes natural cache)** | Med | Med | Gate-before-sign (no KMS spend on forgeries); Cloud Run max-instances ceiling; edge rate limit; documented upstream read budget |
| **Runtime context/DID fetch on signing path = SSRF + availability dependency** | Med | High | Unit 3 closed build-time-pinned document loader + real did:web resolver; unlisted URL rejected, no network fallthrough |
| KMS signing changes canonicalized bytes vs spike disk key, breaking 1EdTech 0/0 | Low | High | De-risked: Cloud KMS `EC_SIGN_ED25519` is PureEdDSA over raw bytes (canonicalize+hash stay in the cryptosuite); adapter returns raw 64-byte; Unit 3 characterization test + 1EdTech re-run after swap |
| DID-doc/KMS key drift across artifacts | Low→0 | High | Eliminated by construction: service builds did.json live from KMS getPublicKey (not a separately-deployed static file) |
| KMS/service outage makes historical credentials unverifiable | Low | Med | Deterministic byte-stable docs ⇒ already-fetched credentials verify offline; only never-fetched ones affected during outage; did.json edge-cached independent of KMS |
| Live Andamioscan field/endpoint shape differs from spike fixtures | Med | Med | Wire against running `andamio-api` proxy, not fixtures; explicit error state on mismatch |
| `did:web` resolution edge cases in strict verifiers | Low | High | Unit 4 invariant + 1EdTech re-run is the gate; Multikey in both verificationMethod and assertionMethod |
| T2/PQ3 deferral creates later spec contradiction | Low | Med | Unit 6 writes the designed path now so the deferral is coherent |
| Infra (KMS/IAM/Terraform) lives in private ops repo | — | Med | Runbook captures repeatable steps + rollback; this repo ships served artifacts + plumbing only |

## Phased Delivery

### Phase 0 — Evidence gate (no code lock until these close)
Obtain 1EdTech membership and run a **DI-signed** sample through the real conformance kit;
produce ≥1 real preprod `credential_claim`; establish a reproducible per-recipient claim-tx
`block_time` source (or a documented deterministic fallback) and demonstrate byte-stability
across two builds + a simulated rotation; resolve Issue #8 (mint-policy custody) for v1
courses; decide the DID-doc location and deployment topology. Until Phase 0 closes, the
mapper/language/signing/identity decisions in Units 3–6 are provisional, not ratified.

### Phase 1 — Cryptographic + CI foundation (Units 1–2)
KMS key (HSM, no-auto-rotation, dedicated sign identity, audit logging), allowlist/CI
reconciliation (hard prerequisite — `service/` must not red-CI the static host), runbook
with rotation **and** compromise kill-switch, CODEOWNERS, production Profile. The DID
document is *not* a static file — it ships with the service in Phase 2. Gate: allowlist CI
green; runbook reviewed.

### Phase 2 — Production assembly service + live DID doc (Units 3–4)
Graduate the validated spike to a deployed service serving both the credential endpoint and
`/.well-known/did.json` (live from KMS). Gates: (a) characterization test reproduces a
1EdTech-passing sample byte-for-byte; (b) signer provably unreachable for non-anchored input;
(c) public `*.run.app` serves a 1EdTech-conformant, byte-stable credential and served
did.json byte-matches the KMS key.

### Phase 3 — Verification surface + revocation read model (Unit 5)
Public per-credential view, explicit states, on-chain revocation signal, verifier guidance.

### Phase 4 — Hygiene + deferred-path design (Unit 6)
Allowlist/CI reconciliation, T2/PQ3 designed-not-built, README/runbook/badge-registry.

## Documentation / Operational Notes

- `docs/runbooks/issuer-provisioning.md` — T1 key creation/custody; **rotation** (additive,
  non-destructive) **and a distinct compromise kill-switch** (destructive VM removal); audit
  logging; sign-identity ≠ deploy-identity; T2 designed path.
- `docs/verifier-guidance.md` — how a third party verifies; the on-chain-authoritative
  (indexer-mediated, may lag) revocation rule + independent evidence URLs; compromise-
  containment consequence; deferred status-list path.
- `docs/badge-registry.md` — badge_id convention + invariants (Issue #11).
- `CODEOWNERS` — `issuer/profile.jsonld` + the service's did.json generator/signer/gate are
  trust-critical; require second-party review (tag-gate alone is insufficient).
- Rollback: static host and service are independently tag-deployed. **Caveat:** because
  did.json is service-built from live KMS, key drift is impossible — but a service rollback
  reverts the gate/signer; the Unit 4 deploy-time key-pin invariant runs every deploy.
  Rotation never invalidates prior credentials; compromise intentionally does (re-issue is
  cheap by design).
- Update Issues #3 (resolved: T1), #5 (did:web served by service), and link this plan from
  #4/#6/#7/#11.

## Alternative Approaches Considered

- **Port to Go in `andamio-api`** (spike's original note) — rejected: re-introduces signing
  canonicalization risk against a passing conformance result; splits issuer identity from
  issuance across repos. Superseded by the user's "service in this repo" decision.
- **VC-JWT signing** — rejected: DI `eddsa-rdfc-2022` already validated 0/0 at 1EdTech and is
  JSON-LD-native; JWT retained only as the spike baseline.
- **Persist whole signed credentials** — rejected: introduces drift between document and
  chain state.
- **Durable proof + derived body** (sign once at claim time, store only the proof/its anchor,
  re-derive the body) — considered, not adopted for v1: it would make the proof KMS-
  independent for reads (mitigating the KMS-as-availability-SPOF and per-request-cost
  arguments), but it reintroduces a persistence store and a sign-at-claim coupling to the
  on-chain `credential_claim` flow (out of scope; protocol-side). The chosen path —
  **deterministic byte-stable re-derivation** — captures most of the benefit without storage:
  a third party who fetched a credential once can verify it offline forever (so historical
  verifiability does not actually depend on KMS uptime for already-distributed credentials),
  and the anchor gate makes per-request KMS cost an attacker-can't-trigger non-issue. Revisit
  durable-proof if sign-at-claim lands protocol-side or KMS sign latency/quota proves binding.
- **KMS as a per-request availability SPOF** — accepted with mitigation, not designed away:
  a KMS/service outage degrades *newly requested, never-before-fetched* credentials to
  `anchored+signature-unavailable`; previously distributed (byte-stable) credentials remain
  independently verifiable; did.json is edge-cached independent of KMS. The alternative
  (durable proof) was weighed above.
- **T2 per-org DIDs in v1** — rejected for v1 (Issue #3): pulls provisioning (#6) + key
  custody (#8) into scope without a forcing function; designed-not-built instead.

## Sources & References

- **Origin (committed source of truth):** `spike/` (imported from orch `ob3-prototype`, 2026-05-16)
- Mapping spec: `spike/mapping.md`; corners: `spike/CORNERS-CUT.md`; open Qs: `spike/open-questions.md`
- Prereq chaining / PQ defaults: `spike/prerequisite-chaining.md`
- Validated samples: `spike/samples/*-real.jsonld`; validation: `spike/validation-results.md`
- Issuer-identity decision graph: Issues #3 (anchor), #4, #5, #6, #7, #8, #11
- External: W3C VC Data Model 2.0; Open Badges 3.0 `v3p0/context-3.0.3.json`; `did:web`;
  W3C Data Integrity EdDSA Cryptosuites v1.0 (`eddsa-rdfc-2022`)
- GCP Cloud KMS — Key algorithms (`EC_SIGN_ED25519`), AsymmetricSign REST ref (`data` vs
  `digest`), release notes (Ed25519 GA 2024-04-15)
- `@digitalbazaar/data-integrity` + `eddsa-rdfc-2022-cryptosuite` + `ed25519-multikey`
  (custom signer seam, raw-64-byte proofValue encoding, Multikey/publicKeyMultibase)
- Microsoft Entra Verified ID — signing-key rotation vs credential-lifetime coordination;
  did:webvh valid-keys (multi-key assertionMethod retention)
- Deepening pass 2026-05-16: best-practices-researcher (KMS feasibility),
  architecture-strategist (drift/determinism), security-sentinel (signing-oracle/compromise)
