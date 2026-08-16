# The costheta Federation Protocol

### An open proposal · v0.3 (draft) · August 2026

**Author:** Andrei Nemirschi · the costheta project · Constanța, Romania · federation@costheta.dev

**Status of this document.** This is a draft proposal, published for discussion. It describes a protocol that today runs on one detector, under one operator. Nothing in this document should be read as a description of a deployed multi-node network. Open questions are marked as such. Comments and prospective node operators: federation@costheta.dev

---

## 1. Abstract

costheta turns cosmic-ray muon arrival times into commit-and-reveal decisions with cryptographically signed certificates. This proposal specifies how multiple independently operated detector nodes attest each decision, so that the validity of a certificate derives from protocol structure rather than from trust in any single operator, including the protocol's author.

## 2. Motivation and trust model

A certificate produced by one operator proves provenance and integrity, but its binding to a real physical event rests on that operator's honesty. The purpose of federation is to replace this residual trust with structure: a fully attested certificate requires agreement between nodes that are operationally, geographically, and personally independent.

The design follows a long lineage: Turing's oracle machines (1939), computation coupled to a non-computable source; Rabin's randomness beacons (1983); and federated public infrastructure such as drand.

**What the protocol guarantees:** that the committed request preceded the attested physical event; that the certificate has not been altered; that no single node, including the coordinating one, could have produced a fully attested certificate alone; and that when the network has more nodes than the quorum requires, the selection of attesting nodes is derived from an event that did not yet exist when the request was committed, over a published snapshot of the node registry, and is recomputable by anyone from public data.

**What it does not guarantee:** that any individual node's hardware is honest (addressed statistically, §8); secrecy of outcomes (all pulses are public); suitability for cryptographic key material (explicitly out of scope, §9); that a selected node will complete its attestation (a node can refuse to reveal; refusal cannot change an outcome, only prevent one, and is publicly attributed, §7.1 and §9).

## 3. Terminology

- **Node**: a muon detector plus its host computer, run by one operator, holding its own signing keys.
- **Operator**: the person with physical custody of a node. One operator may run several detectors; they constitute one node (one trust domain).
- **Trust domain**: the set of hardware under a single operator's control.
- **Pulse**: one protocol event: a detected coincidence, its timestamp, and derived randomness, chained to the previous pulse by hash.
- **Attestation**: a node's signed statement that it observed (or co-witnessed within the agreement window) a given pulse.
- **Hub**: the coordinating service (currently costheta.dev). The hub assembles certificates but cannot forge attestations.

## 4. Certificate format

A certificate is a JSON object, canonically serialized, carrying an array of per-node attestations:

```
{
  "version": "0.3",
  "commitment": "sha256:…",
  "commit_time": "…Z",
  "registry_hash": "sha256:…",
  "selection": { "pulse_index": 4416, "counter": 0 },
  "pulse": {
    "index": 4417,
    "t_dev_ms": 336021,
    "adc": 365,
    "pressure_hpa": 1014.2,
    "prev_hash": "sha256:…"
  },
  "outcome": 1,
  "attestations": [
    {
      "node_id": "constanta-1",
      "window": ["…Z", "…Z"],
      "sig_ed25519": "…",
      "sig_mldsa": "…"
    }
  ]
}
```

Each attestation is dual-signed (Ed25519 per RFC 8032; ML-DSA per FIPS 204). Outcome derivation is public and specified in §5 and §7. With one attesting node the base construction is `H(commitment ‖ pulse)` reduced to the option range; with several, all quorum pulses are included (§7). The pulse chain (`prev_hash`) is append-only and published; retroactive modification of any pulse invalidates all subsequent ones.

`registry_hash` is the hash of the published node-registry snapshot in effect at commit time. `selection` identifies the selection pulse and retry counter of §7.1, so that quorum selection is recomputable from the certificate alone. With N = 1 both fields are present and carry the degenerate values (the single-node registry, selection counter 0).

### 4.1 Commitment construction

A commitment MUST be constructed as:

    commitment = SHA-256( salt ‖ options_canonical )

where `salt` is at least 16 bytes drawn from a cryptographically secure local generator, and `options_canonical` is the caller's option list in a canonical serialization of the caller's choosing. The salt and the option list remain with the caller; the service receives only the digest and the option count.

The salt is REQUIRED. Without it, a commitment over a small option set can be reversed by enumeration: an observer who can guess the candidate options can hash each candidate and identify the committed set. Callers who later need to prove what was committed (dispute resolution, audit) disclose the salt and option list; verification is then a single hash.

The salt protects confidentiality only. It does not, and is not intended to, constrain how many commitments a caller creates; the protocol's resistance to commitment grinding comes from quorum selection (§7.1), not from the commitment format.

### 4.2 Certificate levels

Two certificate levels are defined. The format is identical; the level is determined by the attestation array.

A **single-node certificate** carries one attestation. It allows anyone to verify the signature, the integrity of the record, the binding of the commitment and the derivation of the outcome. Trust in the operator's physical record remains: the certificate does not independently establish that the attested event occurred as recorded.

A **fully attested certificate** carries attestations from a quorum of M nodes (§7) in distinct trust domains, selected under §7.1 and each contributing its own detection to the combined derivation. No single node, including the coordinating one, can produce a fully attested certificate alone.

Certificates state their own level implicitly through the attestation count; verifiers MUST determine the level from the array, not from any label, and MUST check that attesting nodes belong to distinct trust domains as published in the node registry.

## 5. Derivation and bias

Outcome derivation must not introduce structure that the physical source does not have. Known sources of bias, and how each is handled:

**Modulo bias.** Reducing a hash to a range by `mod n` slightly favours low residues whenever `n` does not divide the hash space evenly. With a 256-bit digest the effect is far below any measurable threshold, but implementations MUST use rejection sampling: interpret the digest as an integer, reject and re-derive with an incremented counter if it falls in the non-uniform tail, and only then reduce. The rejection procedure is deterministic and publicly recomputable, so verification is unaffected.

**Multiple outcomes from one pulse.** Deriving several values from the same event (for example ranked winners in a single draw) MUST use domain separation: `H(commitment ‖ pulse ‖ index)` with an explicit index per value. Without it, derived values are correlated.

**Non-uniform inter-arrival times.** Muon arrivals are Poissonian, so raw intervals are exponentially distributed and unsuitable as bits without extraction. Where bits are derived from intervals rather than from the signed pulse digest, an unbiasing step (von Neumann extraction over consecutive interval pairs, or an equivalent extractor) is REQUIRED.

**Detector dead time.** Event processing and logging impose a minimum separation between recorded coincidences, depleting the short-interval region of the distribution. This does not affect the unpredictability of any single event, and is absorbed by hashing, but it means raw timestamp series are not a clean exponential sample.

**Barometric modulation.** The muon flux varies with atmospheric pressure by roughly 0.2 percent per millibar, so the rate is mildly non-stationary over hours and days. Pressure is recorded in every pulse, which makes the correlation auditable rather than hidden. It is also one of the anomaly checks in §8.

**Timestamp resolution.** Device timestamps are recorded in milliseconds. The entropy available per pulse is therefore bounded by clock resolution rather than by physics, and is smaller than the arrival-time distribution alone would suggest. Certificates bind decisions to events; they are not claims about bit yield.

## 6. Node lifecycle

**Enrollment.** A node generates its keypairs on-device, at first boot, in the operator's custody. Private keys never leave the node; the hub receives public keys only. The hub publishes the node registry (id, public keys, status, join date).

**Operation.** Nodes initiate all connections (outbound HTTPS only); the hub never connects inward. A node streams its pulses and signs attestations for decision requests routed to it.

**Departure.** An operator may leave at any time. Departed nodes remain in chain history; their past attestations remain verifiable.

## 7. Attestation and agreement

A decision certificate is **fully attested** when every node in the selected quorum has contributed a pulse of its own within the agreement window, and the outcome is derived from all of them together.

**Quorum size.** The attestation quorum is **M = min(2, N)** as the protocol default, permanent across network growth. With N = 1 (today), certificates carry a single attestation and are labelled accordingly; the format is identical, only the array length differs. From N >= 2, every certificate carries two attestations. Rationale: constant certificate size and latency at any network scale; the 1 to 2 transition is the qualitative trust step, and what continues to grow with N is the unpredictability of which pair is selected. Higher M is available per request, priced accordingly; the attestation count is always visible in the certificate.

**Nodes do not observe the same particle.** Each node detects its own events; there is no shared physical event across sites, and the protocol does not attempt to construct one. Each selected node contributes the next coincidence it detects after the commitment. What the nodes jointly establish is ordering: the commitment was sealed before independent physical events occurred at separate sites, on different hardware, each signed by a different operator. The desynchronisation between detectors is used as a strength, not treated as a defect.

**Combined derivation.** The outcome is derived from all pulses in the quorum, in canonical node-id order:

`outcome = reduce( H(commitment ‖ pulse_1 ‖ pulse_2 ‖ … ‖ pulse_M) )`

with the reduction of §5. No node determines the outcome alone, including the coordinating one.

### 7.1 Timing and agreement rules

**Quorum selection.** When M < N, the choice of which nodes attest must not belong to the hub, or the hub regains exactly the discretion federation removes. It must also not be derivable by the requester at commit time: the seed in earlier drafts, H(prev_pulse_hash ‖ commitment), was computable by the requester before committing, since the previous pulse is already public. A requester could therefore grind commitments until the selection landed on a convenient quorum.

v0.3 derives the selection from an event that does not exist when the commitment is sealed:

    seed = H( "costheta-quorum-v1" ‖ registry_hash ‖ commitment ‖ pulse_s )

where `pulse_s` is the first pulse appended to the public chain with a hub receipt time strictly after the commitment was recorded (the *selection pulse*), and `registry_hash` is the hash of the published node-registry snapshot in effect at commit time. The quorum is the first M distinct nodes produced by expanding H(seed ‖ i) for i = 0, 1, 2, … over that snapshot. Both inputs are published; anyone can recompute the selection from the certificate.

This closes the requester's grinding: the deciding input to the seed does not exist at commit time. It closes the hub's discretion over membership: the eligible set is pinned by a hash fixed before the selection pulse arrives, so nodes cannot be added, removed or reordered after the commitment. The chain's append-only ordering, which is public, determines which pulse is the selection pulse.

What remains is narrower: the operator of the node that happens to produce the selection pulse cannot choose its content (the pulse is a physical detection, chained to its predecessor), but could delay or suppress it, shifting selection to the next pulse. Suppression is visible as a rate anomaly under §8 and is attributable, because the chain shows whose pulse was due. This residual is declared in §9.

**Two-phase commitment between nodes.** Combined derivation would otherwise let a node that reveals last compute the outcome before publishing, and substitute a different contribution if the result is unfavourable. Commit-and-reveal removes substitution: once a node has committed to H(pulse), the only pulse it can validly reveal is that one. It does not remove refusal. A node can always decline to reveal; no protocol can force a message. What the protocol does is make refusal unable to change any outcome, and make it visible and attributable when it happens.

1. Each selected node sends `H(pulse)` to the hub, not the pulse.
2. Once all commitments are in, the hub requests the reveal.
3. Nodes publish their pulses. A pulse that does not match its commitment invalidates the attestation and is recorded publicly.

No node sees another's pulse before binding itself to its own. This is the same commit-and-reveal the protocol sells to its clients, applied between its own participants.

**Timeouts and failure handling.** Commit phase: **30 seconds** from request. This is bounded by physics rather than by network latency: at a single-detector rate near 0.21 events per second, inter-arrival times are exponential and roughly 0.2 percent of intervals exceed 30 seconds, so a shorter window would exclude honest nodes routinely. Reveal phase: **5 seconds** from the hub's request, since no detection is being waited on.

The two phases fail differently, and the difference is load-bearing.

A **commit-phase** expiry carries no information about any outcome: no node has yet bound itself to a pulse, and no pulse has been revealed. The quorum is therefore reselected once, deterministically, by incrementing the counter in the selection seed. If the second attempt also expires, the request fails and is not charged.

A **reveal-phase** failure is different: by the time reveals begin, pulses become public as they are published, and a node that has seen the others' contributions can compute what the outcome would be. If failure at this stage triggered reselection, a node could abort selectively, discarding outcomes it disliked and re-rolling the decision. Earlier drafts reselected on any expiry; v0.3 does not. A reveal-phase failure terminates the request: it fails, it is not charged, it is never re-rolled. The failure is recorded against the withholding node and published. A withholding node can therefore deny a specific answer, at public cost to itself, but can never obtain a different one.

The protocol deliberately does not degrade to a smaller quorum on any timeout: doing so would let an adversarial node force single-attestation mode by remaining silent.

Every timeout is recorded against the node that caused it and published. Occasional misses indicate network trouble; a high rate, and especially a rate that correlates with request activity, is an anomaly under §8.

These figures are first-iteration parameters derived from one detector's measured rate. They are expected to be recalibrated once data from multiple nodes exists.

**Open question OQ-2: the agreement window.** Node clocks drift, and the window must tolerate honest drift while bounding the interval in which a dishonest hub could substitute pulses. Candidate: windows anchored to hub receipt time with per-node drift bounds learned from health beacons.

**Open question OQ-3: revenue sharing.** Paid certificates settle per decision; the candidate split is an equal share per attesting node for half of the net revenue, the remainder to the hub (coordination, settlement, verification infrastructure). Principles that are not open: payment follows attestation work only; no payment for recruitment, none for mere enrollment, no token. Splits are published and auditable against the public certificate chain. To be settled with the first independent operators, before any multi-node revenue exists.

## 8. Anomaly handling

Nodes are semi-trusted. The hub (and any observer, from published data) monitors each node's pulse stream for: rate excursions incompatible with cosmic-ray flux and its barometric modulation; ADC distributions departing from the expected energy-deposit spectrum; clock behavior inconsistent with declared drift. Flagged nodes are excluded from attestation quorums pending review; exclusions are public.

## 9. Limits and non-goals

Throughput is bounded by physics (about one usable event per several seconds per detector); the protocol serves decisions, not bulk randomness. Public randomness is by definition unsuitable for secret key material. The protocol does not attempt Byzantine agreement over pulse *content* across nodes observing different particles; attestation binds requests to *a* physical event under agreed windows, not to a global shared event.

**Withholding.** A selected node can refuse to reveal its contribution. The protocol cannot prevent this; it ensures instead that refusal is outcome-neutral (a reveal-phase failure terminates the request rather than re-rolling it, §7.1), publicly attributed, and cumulatively disqualifying under §8. Availability of any single decision is therefore not guaranteed; integrity of every completed decision is.

**Settlement dependency.** Paid decisions settle through a third-party x402 facilitator, which verifies the client's payment authorisation and submits the on-chain transfer. This is an operational dependency outside the protocol: it affects whether a payment can be completed, not whether a certificate can be verified. Certificates remain independently verifiable from the published keys regardless of how settlement occurred. The facilitator is configurable and expected to change as the ecosystem matures.

## 10. Status and versioning

v0.3, draft. One detector, one operator, one attestation per certificate. Changes from v0.2: quorum selection is reseeded from the first pulse following the commitment over a pinned registry snapshot, closing commitment grinding (§7.1); reveal-phase failures terminate the request instead of reselecting, closing selective abort, and withholding is declared as a visible, outcome-neutral limit (§7.1, §9); commitment construction is normative and salted (§4.1); certificate levels are named and defined (§4.2); certificate format 0.3 adds `registry_hash` and `selection` fields; timing and agreement rules are collected under §7.1. Changes to this proposal are versioned in the public repository; releases are archived with a DOI. This version: DOI assigned at release · All versions: https://doi.org/10.5281/zenodo.21901353 · Repository: https://github.com/costhetadev/federation-proposal · Implementations should treat all fields not defined here as reserved.

---

*This proposal is licensed under the Creative Commons Attribution 4.0 International License (CC BY 4.0). Contact: federation@costheta.dev*
