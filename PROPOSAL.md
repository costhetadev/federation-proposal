# The costheta Federation Protocol
### An open proposal · v0.1 (draft) · August 2026
**Author:** Andrei Nemirschi · the costheta project · Constanța, Romania · federation@costheta.dev

**Status of this document.** This is a draft proposal, published for discussion.
It describes a protocol that today runs on one detector, under one operator.
Nothing in this document should be read as a description of a deployed
multi-node network. Open questions are marked as such. Comments and
prospective node operators: federation@costheta.dev

---

## 1. Abstract

costheta turns cosmic-ray muon arrival times into commit-and-reveal decisions
with cryptographically signed certificates. This proposal specifies how
multiple independently operated detector nodes attest each decision, so that
the validity of a certificate derives from protocol structure rather than
from trust in any single operator, including the protocol's author.

## 2. Motivation and trust model

A certificate produced by one operator proves provenance and integrity, but
its binding to a real physical event rests on that operator's honesty. The
purpose of federation is to replace this residual trust with structure:
a fully attested certificate requires agreement between nodes that are
operationally, geographically, and personally independent.

The design follows a long lineage: Turing's oracle machines (1939),
computation coupled to a non-computable source; Rabin's randomness beacons
(1983); and federated public infrastructure such as drand.

**What the protocol guarantees:** that the committed request preceded the
attested physical event; that the certificate has not been altered; that no
single node, including the coordinating one, could have produced a fully
attested certificate alone.

**What it does not guarantee:** that any individual node's hardware is
honest (addressed statistically, §7); secrecy of outcomes (all pulses are
public); suitability for cryptographic key material (explicitly out of
scope, §8).

## 3. Terminology

- **Node**: a muon detector plus its host computer, run by one operator,
  holding its own signing keys.
- **Operator**: the person with physical custody of a node. One operator
  may run several detectors; they constitute one node (one trust domain).
- **Trust domain**: the set of hardware under a single operator's control.
- **Pulse**: one protocol event: a detected coincidence, its timestamp,
  and derived randomness, chained to the previous pulse by hash.
- **Attestation**: a node's signed statement that it observed (or
  co-witnessed within the agreement window) a given pulse.
- **Hub**: the coordinating service (currently costheta.dev). The hub
  assembles certificates but cannot forge attestations.

## 4. Certificate format

A certificate is a JSON object, canonically serialized, carrying an array
of per-node attestations:

```json
{
  "version": "0.1",
  "commitment": "sha256:…",
  "commit_time": "…Z",
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

Each attestation is dual-signed (Ed25519 per RFC 8032; ML-DSA per FIPS 204).
Outcome derivation is public: `outcome = H(commitment ‖ pulse) mod n_options`.
The pulse chain (`prev_hash`) is append-only and published; retroactive
modification of any pulse invalidates all subsequent ones.

## 5. Node lifecycle

**Enrollment.** A node generates its keypairs on-device, at first boot, in
the operator's custody. Private keys never leave the node; the hub receives
public keys only. The hub publishes the node registry (id, public keys,
status, join date).

**Operation.** Nodes initiate all connections (outbound HTTPS only); the
hub never connects inward. A node streams its pulses and signs attestations
for decision requests routed to it.

**Departure.** An operator may leave at any time. Departed nodes remain in
chain history; their past attestations remain verifiable.

## 6. Attestation and agreement

A decision certificate is **fully attested** when at least *M* of the *N*
active nodes have signed attestations whose time windows mutually overlap.

- **Decided (was OQ-1):** the attestation quorum is **M = min(2, N)** as
  the protocol default, permanent across network growth. With N = 1
  (today), certificates carry a single attestation and are labeled
  accordingly; the format is identical, only the array length differs.
  From N ≥ 2, every certificate carries two attestations, the pair drawn
  per OQ-4. Rationale: constant certificate size and latency at any
  network scale; the 1→2 transition is the qualitative trust step, and
  unpredictability of the attesting pair (OQ-4) is what continues to
  grow with N. Higher M is available per request, priced accordingly;
  the attestation count is always visible in the certificate.
- **Open question (OQ-2):** the agreement window. Node clocks drift
  (consumer RTCs); the window must tolerate honest drift while bounding
  the interval in which a dishonest hub could substitute pulses.
  Candidate: windows anchored to hub receipt time with per-node drift
  bounds learned from health beacons.
- **Open question (OQ-3):** revenue sharing. Paid certificates settle
  per decision; the candidate split is an equal share per attesting node
  for half of the net revenue, the remainder to the hub (coordination,
  settlement, verification infrastructure). Principles that are not open:
  payment follows attestation work only; no payment for recruitment, none
  for mere enrollment, no token. Splits are published and auditable
  against the public certificate chain. To be settled with the first
  independent operators, before any multi-node revenue exists.
- **Open question (OQ-4):** quorum selection at larger N. When M < N,
  the choice of which nodes attest must not belong to the hub, or the
  hub regains exactly the discretion federation removes. Candidate: the
  quorum for a decision is derived deterministically from the protocol's
  own randomness (selection seeded by `H(prev_pulse_hash ‖ commitment)`
  over the active node set), making it unpredictable before the preceding
  pulse, publicly recomputable afterwards, and uniform over time (which
  also distributes attestation work, and with it OQ-3 revenue, without
  anyone's hand on the scale). Inert below N ≈ 4, where M = N applies.

## 7. Anomaly handling

Nodes are semi-trusted. The hub (and any observer, from published data)
monitors each node's pulse stream for: rate excursions incompatible with
cosmic-ray flux and its barometric modulation; ADC distributions departing
from the expected energy-deposit spectrum; clock behavior inconsistent
with declared drift. Flagged nodes are excluded from attestation quorums
pending review; exclusions are public.

## 8. Limits and non-goals

Throughput is bounded by physics (about one usable event per several
seconds per detector); the protocol serves decisions, not bulk randomness.
Public randomness is by definition unsuitable for secret key material.
The protocol does not attempt Byzantine agreement over pulse *content*
across nodes observing different particles; attestation binds requests to
*a* physical event under agreed windows, not to a global shared event.

## 9. Status and versioning

v0.1, draft. One detector, one operator, one attestation per certificate.
Changes to this proposal are versioned in the public repository; releases
are archived with a DOI. This version: https://doi.org/10.5281/zenodo.21901354
Implementations should treat all fields not defined here as reserved.

---

*This proposal is licensed under the Creative Commons Attribution 4.0
International License (CC BY 4.0). Contact: federation@costheta.dev*
