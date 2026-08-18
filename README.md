# ANGX on Holepunch — Technical Feasibility Assessment

Feasibility of the angeliaX (ANGX) core loop — two signed append-only logs, witnessing, bases, and partnering — on the Hypercore / Pear stack, with a scoped build plan for a Raspberry Pi-resident prototype.

| | |
|---|---|
| **Prepared for** | ANGX (angx-system) |
| **Prepared by** | heart-IT |
| **Date** | August 2026 |
| **Status** | Shared for review |

---

## Verdict

**Buildable as described — no structural blockers.** The design's instincts match the stack's grain: append-only signed feeds, offline-first operation, sparse replication, no global search. Notably, the entire scoped prototype is *single-writer* — every feed has exactly one authoring keypair — so it needs none of the stack's multi-writer consensus machinery (Autobase), which removes the largest source of complexity and risk from a build like this.

Five parts of the specification are load-bearing for the prototype but underspecified ([§5](#5-design-gaps-requiring-resolution)); they are design decisions, not implementation details, and should be resolved in writing before the build starts. The Raspberry Pi target is viable with a 64-bit OS and modest hardware discipline ([§7](#7-raspberry-pi-target)). Estimated effort for the scoped core loop: **8–14 engineer-weeks** ([§8](#8-build-plan-and-estimates)).

---

## Contents

- [§1 Scope and sources](#1-scope-and-sources)
- [§2 Concept-to-primitive mapping](#2-concept-to-primitive-mapping)
- [§3 What the stack provides natively](#3-what-the-stack-provides-natively)
- [§4 Why no Autobase is needed](#4-why-no-autobase-is-needed)
- [§5 Design gaps requiring resolution](#5-design-gaps-requiring-resolution)
- [§6 Companion tools: reader and bridge](#6-companion-tools-reader-and-bridge)
- [§7 Raspberry Pi target](#7-raspberry-pi-target)
- [§8 Build plan and estimates](#8-build-plan-and-estimates)
- [§9 Scoping questions](#9-scoping-questions)
- [§10 Risk register](#10-risk-register)
- [§11 Recommended engagement structure](#11-recommended-engagement-structure)

---

## §1 Scope and sources

This assessment covers the prototype scope defined in the inquiry: steward and base keypair generation; registering operational and commons nodes; posting steward signals (`operational` / `failure` / `learning` / `retired`) and witness signals gated on the witness holding a verified node; base initialization and curation into a base's collection; and base partnering (handshakes, mutual signing, partner-chain reachability). The companion tools `angx-reader` and `angx-bridge` are out of build scope but reviewed briefly in [§6](#6-companion-tools-reader-and-bridge) because two of their assumptions touch the core design.

Documents reviewed in full: `angeliax` README, SCHEMA.md, CONSTRAINTS.md, WALKTHROUGH.md, WALKTHROUGH-commons.md; `angx-reader` and `angx-bridge` proposals. Stack claims below are made against the current Hypercore 11-era releases (hypercore 11, corestore 7, hyperswarm 4, current Pear runtime).

## §2 Concept-to-primitive mapping

Every construct in the schema maps onto an existing, maintained primitive. Nothing requires forking the stack or writing novel cryptography.

| Spec concept | Stack primitive | Fit |
|---|---|---|
| One node = one append-only signed feed | One Hypercore per node, derived from the steward's Corestore — the stack's key manager, which derives any number of feed keypairs deterministically from one master key. The Node ID is the core's Ed25519 public key — 256 bits, 64 hex characters, exactly as the schema specifies. | clean |
| Signal / Entry IDs, citable and verifiable | Address every entry as `(feedKey, seq)` — the feed's public key plus the entry's position in that feed — rather than a random ID. This is what makes the schema's own claim — "any client can independently verify a cited Entry ID exists in the referenced base's feed" — literally true, at zero cost. | clean |
| Immutability, signing, tamper-evidence | Native Hypercore behavior: BLAKE2b Merkle tree with Ed25519-signed roots, verified automatically on replication. Constraints 3, 6, and 8 come for free. | native |
| Learning-signal attachments, ≤10 MB, fetched on demand | A Hyperdrive per node (internally a Hyperbee for metadata plus Hyperblobs for content). Sparse replication is the default, so attachments download only when fetched — the exact behavior SCHEMA.md's Content Persistence section asks for. | clean |
| Library (automatic per-keypair collection) | A Hyperbee derived from the keypair, listing own node keys and replicated feed keys, announced on Hyperswarm under its discovery key. This directly resolves the "library address resolution" open question. | clean |
| Querying (type, location, signal text, witness activity…) | Local Hyperbee/Hyperdb indexes built by tailing replicated feeds. A base can additionally publish its index as its own core, letting remote clients query it sparsely without replicating every curated feed first. | clean |
| Base: identity + Collection Log + Partner Log | One base Hypercore (single writer: the base keypair) carrying typed entries per the schema, plus the published index above. | clean |
| Steward co-signature (consent-required curation); dual-signed handshakes | Detached libsodium signatures over a canonical encoding of the entry payload, embedded inside the appended entry. Hypercore itself signs only with the feed key; second signatures are payload-level. A standard, well-understood pattern. | clean |
| Partner handshake exchange | A small typed message channel between the two bases (Protomux, the stack's layer for running typed protocols over one encrypted peer connection): propose → review cited entries → co-sign → each side appends to its own feed independently, as the schema requires. | clean |
| Discovery | One Hyperswarm instance per client; one topic per discovery key (node topics, base topic, library topic). The "node curation discovery" open question resolves itself: curating bases already sit on the node's topic to replicate it. | clean |
| Space keypair ↔ base keypair relationship | The spec's proposed sibling *derivation* does not achieve public verifiability ([G-5](#g-5--consent-required-contact-privacy-is-the-hardest-feature-in-the-spec)). The right primitive is an attestation proof chain (`keet-identity-key`-style): a root key signs a statement vouching for each related key, and anyone can verify those signatures — the same mechanism Keet uses to tie a user's devices to one identity. Maintained ecosystem code, not custom cryptography. | spec correction |
| Multiple base stewards (open question, out of scope) | Autobase, later. Correctly deferred by the spec; nothing in the scoped prototype forecloses it. | deferred |

## §3 What the stack provides natively

- **Append-only enforcement, signatures, Merkle verification** — every replicated block is verified against the feed's signed Merkle root before acceptance. Tampering, insertion, and rewriting are rejected by the protocol, not by application code.
- **Encrypted transport everywhere** — all peer connections run over Noise-handshaked encrypted streams. No plaintext path exists to accidentally use.
- **Offline-first operation** — local writes need no connectivity; replication catches up whenever peers meet. This matches the walkthroughs' intermittent-connectivity settlements without extra work.
- **NAT traversal** — roughly 95% of consumer-network connections are direct; the remainder fall back to relaying. Bases behind home routers need no port forwarding.
- **Sparse replication** — peers fetch only the blocks they read. Attachment-heavy nodes stay cheap for everyone who doesn't open the attachments.

## §4 Why no Autobase is needed

The most important structural observation in this assessment: **every feed in the scoped prototype has exactly one writer.** A steward's node feeds are written only by the steward's keypair. A base's feed is written only by the base keypair. Witness signals live in the *witness's own* feed, not the observed node's. Cross-actor constructs — co-signed curation entries, dual-signed handshakes — are second signatures *inside* a single-writer entry, not second writers. In practice: the base writes the curation entry; the steward's approval travels as a signature carried within it.

Multi-writer coordination (Autobase) is where P2P builds on this stack accumulate most of their complexity, subtle bugs, and testing cost. The ANGX core loop avoids it entirely by design — likely by accident of good instincts, and worth preserving deliberately. The one place multi-writer may eventually matter (several stewards operating one base) is already marked as an open question in the schema and can be added later without reworking the core.

## §5 Design gaps requiring resolution

These gaps sit inside the scoped prototype. Each is resolvable — proposed resolutions follow — but they are *protocol decisions*, not implementation details, and should be agreed in writing before the build starts ([§11](#11-recommended-engagement-structure)).

### G-1 — Witness gating is read-time validation, not write-time enforcement

Nothing can stop a keypair from appending a witness-shaped entry to its own feed — that is the nature of author-signed logs. Concretely: when David posts witness signals about Amara's filter, they go into David's own feed, and no central point exists where his verified status could be checked at the moment of writing. "Requires a verified node" can therefore only mean that every client and base independently evaluates the witness's status and *discards invalid witness signals when reading and indexing*. This is consistent with Constraint 10's own framing ("the claim is cryptographic; the act is not"), but the spec never states it, and it changes what gets built: the heart of the prototype is a deterministic **verification resolver**: a fixed rulebook that maps the feeds a client holds to `verified` / `unverified` / `unknown` and gives the same answer on every device holding the same data.

**Resolution:** specify the resolver as part of the protocol: its inputs, its rules, and its behavior when data is missing. Signals from unverifiable witnesses are retained (append-only) but excluded from witness-gated views and marked in UI.

### G-2 — Witness discoverability is unspecified — the largest missing mechanism

Witness signals live in the witness's feed but conceptually attach to the observed node. In walkthrough terms: David's witness signals about Amara's filter land in David's feed, and nothing in the schema tells mathare-kitchen — which curates Amara's node — that David's feed now contains signals about it. Unless someone who already knows of his feed replicates it, those signals are invisible. Without a discovery mechanism, the two-stream model (steward stream + witness stream) cannot be assembled by anyone.

**Resolution:** witnesses announce on the observed node's discovery topic (which curating bases and the steward already join). Peers on that topic exchange a small typed message — "I hold witness signals for this node; my library key is X" — after which normal replication and indexing take over. Natural to the stack, but it is new protocol surface and needs sign-off.

### G-3 — Verified status is recursive and depends on data availability

Node verified ⇐ curated by a verified base ⇐ holds an *active* handshake ⇐ mutual entries in two feeds, minus unilateral dissolutions and retirements. Evaluating this requires holding the relevant base feeds — and their partners' feeds one hop out. The computation is deterministic *given the data*; the spec must say what a client answers when it doesn't hold the data, and must pin down the first-base / second-base bootstrap special case exactly. Concretely: to decide whether David's node is verified, a client needs the Kampala base's feed (for the `added` entry), that base's Partner Log, and the partner's own feed (to confirm the handshake is mutual and still active). When any of those is missing, the honest answer is "unknown" — a state distinct from "unverified", and one the spec currently has no word for.

**Resolution:** three-valued resolver output (`verified` / `unverified` / `unknown`) with defined replication prerequisites; bootstrap cases encoded as explicit rules with test vectors.

### G-4 — Timestamps are self-reported; "active as of time T" is unverifiable

An append-only feed proves order *within* one feed, never wall-clock time. Nothing stops a steward from writing an entry today that carries last year's date. Any rule phrased as "was verified at the time of posting" is therefore unenforceable as written.

**Resolution:** anchor validity to citations rather than clocks. Happened-*after* is provable: a witness signal referencing a Collection Log entry `(baseKey, seq)` provably came after it. The schema already uses citation-based verification for handshakes (Reviewed Entries) — this extends the spec's own idea. Wall-clock timestamps remain as display metadata only.

### G-5 — Consent-required Contact privacy is the hardest feature in the spec

"Contact visible only to whoever the steward has granted access" on a publicly replicated feed means field-level encryption plus per-grantee key distribution — the Contact field encrypted so that each approved reader, and no one else, can open it. That is a key-management subsystem hiding in one paragraph. Separately, the proposed base-keypair *derivation* ("cryptographically verifiable by any client") does not work as described: Ed25519 has no publicly verifiable child-key derivation without exposing seed material. The goal — provable space↔base linkage — is achieved instead with an attestation proof chain (a `keet-identity-key`-style root that attests both keys, verifiable by anyone).

**Resolution:** build the structural half of consent (steward co-signature on curation) in v1; either defer selective Contact disclosure or price it explicitly as a sealed-box grant subsystem. Replace derivation with attestation in the spec.

### G-6 — Partner-chain queries need acceptance criteria

The proposed default — a query at any base transitively reaches every connected base in one action, the way Fatima's "pythium" query crosses the partner chain in the walkthrough — implies decisions the spec doesn't yet make: how deep traversal goes, how loops are prevented, how results from many bases merge, and what happens when a base in the chain is offline. Two very different builds satisfy "partner-chain reachability": **(a)** traverse partner logs to enumerate reachable bases, then query each reachable base's published index — tractable, recommended for the prototype; **(b)** a live recursive search protocol — a significantly larger lift.

**Resolution:** prototype acceptance criteria pinned to (a); (b) evaluated after the prototype exists.

## §6 Companion tools: reader and bridge

Both are out of build scope, correctly independent of the core, and broadly sound. Two notes affect their proposals:

- **angx-reader's file-watching model doesn't match Hypercore's storage.** The proposal has the indexer watch a `feeds/` directory for new signal files. Hypercore 11 persists blocks in a RocksDB-backed store, not per-signal files — there is no directory of signals to watch. The fix is small and makes reader simpler: subscribe to append events through a client API instead of watching the filesystem. Embedding on-CPU (ONNX MiniLM-class models) is realistic on a Pi for trickle indexing.
- **angx-bridge's double-injection question mostly answers itself.** Re-injecting an already-known signed block into a Hypercore is a natural no-op — blocks are verified against the feed's Merkle root and duplicates are simply already present. Two internet-connected catchers injecting the same update is safe. LoRa-class bandwidth is also a comfortable fit for 120-character signals (attachments already wait for real connectivity by design).

## §7 Raspberry Pi target

**Viable, with conditions.** The stack runs on Node.js LTS (or Bare) on 64-bit ARM Linux. The conditions:

- **64-bit OS is mandatory** — the stack's native components (cryptography, UDP transport, storage) ship precompiled binaries only for 64-bit ARM Linux (`linux-arm64`), not for 32-bit ARM. Raspberry Pi OS 64-bit on a Pi 4 or Pi 5 with 4 GB+ RAM is the reference target.
- **SSD or high-endurance storage** — the RocksDB write pattern will wear out cheap SD cards. A small USB SSD, or at minimum a high-endurance card, should be part of the reference hardware note alongside the UPS the schema already recommends.
- **Headless daemon, not a desktop app** — the base runs as a systemd-managed process. This also insulates the base from desktop-runtime churn.
- **Week-one hardware spike** — prebuild coverage for `linux-arm64` is expected but must be verified on real hardware before the timeline is committed, not assumed. This spike is scheduled inside M0 ([§8](#8-build-plan-and-estimates)).

The resource math is comfortable: signals are ~200 bytes, so tens of thousands of entries amount to megabytes; attachments replicate sparsely; a base curating dozens to a few hundred nodes means that many swarm topics on one instance, which is within normal operating range for the stack.

## §8 Build plan and estimates

Estimates assume a CLI/daemon-grade prototype — full protocol, real replication, integration tests, on-hardware validation — with the walkthroughs' tabbed GUI client as a separately priced follow-on ([§9](#9-scoping-questions), Q-1). Estimates are engineer-weeks for one senior engineer; two engineers compress calendar time roughly 40%. M0 is delivered inside Package 1 ([§11](#11-recommended-engagement-structure)); M1–M6 form Package 2.

| Milestone | Content | Estimate | Price |
|---|---|---|---|
| M0 | Wire schema and canonical encodings, keypair/identity layout, Corestore structure, **Raspberry Pi hardware spike** (§7) | 1 – 1.5 wk | in Package 1 |
| M1 | Node registration (both logs), steward signals, Hyperdrive attachments | 1 – 1.5 wk | $3,000 |
| M2 | Library, replication, Hyperbee indexes, local queries | 1.5 – 2 wk | $4,500 |
| M3 | Witnessing: signals, discovery protocol (G-2), verification resolver (G-1, G-3) | 2 – 3 wk | $7,000 |
| M4 | Base initialization (threshold detection, bootstrap cases), Collection Log including consent co-sign path | 2 wk | $5,000 |
| M5 | Partnering: handshake protocol, dual signing, dissolution and retirement, partner-chain reachability (G-6a) | 2 – 3 wk | $6,500 |
| M6 | Pi deployment hardening, two-Pi partnering soak test, operator documentation | 1 – 1.5 wk | $3,500 |
| | **Total, scoped core loop** (Package 1 $4,500 + Package 2 $29,500) | **10.5 – 14.5 wk** | **$34,000** |

The milestone sums give 10.5–14.5 weeks at the full field-grade acceptance bar. The realistic floor is **8–10 weeks** if Package 1's design stage lands the simplest viable witness-discovery design (G-2), the CLI acceptance bar stays lean, and partner-chain reachability stays at enumerate-and-query (G-6a) — hence the **8–14 engineer-week** range this assessment quotes overall. The prices above are fixed either way: the effort range is the builder's risk, not the client's.

Payment terms: each milestone is invoiced only on delivery and acceptance — nothing is paid in advance — and the next milestone starts once the previous invoice is settled. The engagement can stop after any milestone, with everything delivered to date the client's to keep. Prices assume the scope assumptions above (CLI/daemon-grade, G-6a reachability, structural consent per G-5); if design-stage sign-off changes a later milestone's scope, that milestone's price is revised and agreed before it starts.

Calendar translation: roughly **2–4 months** for one senior engineer, or about 2–2.5 months with two at the top of the range. A minimal desktop GUI client over the same daemon is a separately scoped follow-on, adding an estimated **4–6 weeks**. The design-resolution work for G-1 through G-6 adds **0.5–1 week** up front and de-risks everything after it; [§11](#11-recommended-engagement-structure) bundles it with M0 as the first fixed-price package.

## §9 Scoping questions

Answers to these determine the final quote and timeline:

1. **Q-1 — UI expectations.** The walkthroughs describe a tabbed GUI client; the prototype scope list does not mention UI. Is the prototype CLI/daemon-grade (recommended first), or is a minimal GUI client part of this engagement?
2. **Q-2 — Authority over open questions.** May the build team resolve the spec's open questions (and gaps G-1…G-6) during Package 1's design stage, subject to your written sign-off? Several are protocol decisions that will be load-bearing for every future implementation.
3. **Q-3 — Partner-chain acceptance criteria.** Does enumerate-and-query-each-reachable-base (G-6, option a) satisfy "partner-chain reachability" for the prototype?
4. **Q-4 — Consent scope.** Is structural consent (co-signed curation) sufficient for v1, with selective Contact disclosure deferred or priced separately (G-5)?
5. **Q-5 — Field deployment.** How many physical bases (Pis) should the build actually stand up and soak-test — the two-device minimum in M6, or a larger pilot?

## §10 Risk register

| Risk | Severity | Mitigation |
|---|---|---|
| Witness-discovery protocol (G-2) is novel surface; a weak design fragments the witness stream | high | Resolve in Package 1's design stage with your sign-off; build on the topic-announcement pattern the stack already uses; integration-test with partitioned peers. |
| Verified-status semantics under missing data (G-3) drift between implementations | medium | Specify the resolver as a pure function with published test vectors; three-valued output. |
| Self-reported timestamps enable backdating (G-4) | medium | Citation-anchored ordering for anything protocol-relevant; clocks demoted to display metadata. Accepted residual risk consistent with Constraint 10's reputational model. |
| `linux-arm64` prebuild gaps force local native builds on Pi | low | M0 hardware spike verifies the full dependency set on target hardware in week one, before the timeline is committed. |
| SD-card wear under RocksDB write load kills field bases | low | SSD / high-endurance storage in the reference hardware note; soak test in M6 measures write amplification. |
| Scope creep from the spec's open-questions list during the build | medium | Package 1's design stage closes G-1…G-6 in writing; other open questions explicitly parked; milestone acceptance criteria fixed at kickoff. |
| Desktop-runtime churn in the Pear platform affects client packaging | low | Bases run as headless daemons independent of desktop packaging; GUI client decisions deferred to Package 3. |

## §11 Recommended engagement structure

1. **Package 1 — design + foundations** (fixed price; design resolution + milestone M0). Two parts, in order: *(a)* close G-1 through G-6 and questions Q-1…Q-5 as short written protocol decisions with your sign-off, delivered as a schema addendum both sides build against; *(b)* implement M0 against those signed-off decisions — wire schema and code generation, keypair/identity layout, Corestore structure — validated on reference Raspberry Pi hardware. Ends with running code and confirmed dates for the remaining milestones; their prices are already fixed in §8 and change only if design sign-off changes a milestone's scope, agreed before that milestone starts.
2. **Package 2 — core-loop build** (M1–M6; $29,500 across six milestones, each invoiced on delivery). Each milestone lands as reviewable, tested code; M6 ends with two physical bases partnered and soak-tested on target hardware.
3. **Package 3 — optional follow-ons.** Minimal GUI client (~4–6 weeks); selective Contact disclosure (G-5); federated live search (G-6b); multi-steward bases via Autobase; reader/bridge integration — each scoped and priced after the core proves out.

---

*Prepared against the angx-system specification set (angeliax, angx-reader, angx-bridge) as of August 2026. Stack claims verified against Hypercore 11-era releases. This document is a feasibility assessment, not a binding quote; commercial terms follow scope confirmation ([§9](#9-scoping-questions)).*
