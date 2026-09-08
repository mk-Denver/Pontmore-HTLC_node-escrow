# Pontmore — HTLC Node Escrow

**Pontmore-compatible, Nostr-coordinated, node-controlled Lightning hold-invoice escrow.**

This implementation coordinates value exchange between a Pontmore `customer` (buyer) and `agent` (seller) through an escrow node. The escrow node creates the hold invoice, retains its preimage, monitors the incoming HTLC, and executes release or refund. Neither trading party can unilaterally settle or cancel the escrow invoice.

This repository implements PIP-01 `custodial_escrow` using a Lightning hold invoice as the operator's funding lock. The operator controls the invoice claim and outgoing payout, so advertising the service as participant-controlled `lightning_hold_invoice` would understate custody. Its public event log follows [PIP-02](https://github.com/pontmore/protocol/blob/main/PIP-02-swap-state-machine.md), and its dispute and timeout policy follows [PIP-03](https://github.com/pontmore/protocol/blob/main/PIP-03-dispute-policy.md). The canonical protocol repository is the source of truth when this implementation guide conflicts with a PIP.

### Pontmore role mapping

| Pontmore role | UI alias in this repository | Responsibility |
|---------------|-----------------------------|----------------|
| `agent` | seller | Provides the traded claim and a payout invoice |
| `customer` | buyer | Requests the swap and funds the escrow hold invoice |
| escrow operator | node/arbiter | Controls the hold invoice and executes PIP-03 resolutions |

The public protocol fields and `actor_role` values always use the canonical Pontmore names. UI text may use buyer and seller for readability.

---

## Pontmore Protocol Stack

```
┌──────────────────────────────────────────────┐
│ PIP-01                                       │
│ "What escrow configuration is this?"         │
│ Escrow descriptor (kind 30361)               │
└────────────────────┬─────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────┐
│ PIP-02                                       │
│ "What is the append-only swap history?"      │
│ Request, transition, evidence, dispute, note │
│ and snapshot events                          │
└────────────────────┬─────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────┐
│ PIP-03                                       │
│ "What happens when things go wrong?"         │
│ Dispute & timeout policy                     │
└────────────────────┬─────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────┐
│ Referenced Service Schema (OpenAPI/AsyncAPI)  │
│ "How do I operate this subtype?"              │
│ Endpoints, auth, payloads, errors and         │
│ hold-invoice authorization rules              │
└────────────────────┬─────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────┐
│ Lightning / Nostr implementation              │
└──────────────────────────────────────────────┘
```

---

## Architecture

```
                 CUSTOMER                    AGENT
                      │                         ▲
                      │ pays hold invoice       │ payout invoice
                      ▼                         │
                    ESCROW NODE ────────────────┘
                 (owns preimage and LND RPC)
                                 │
                   ┌─────────────┴─────────────┐
                   │                           │
              NORMAL PATH                  DISPUTE PATH
              (no arbiter)                (arbiter activated)
                   │                           │
         ┌─────────┴─────────┐                 │
         │                   │                 │
      node releases        node refunds     dispute raised
 → PAYOUT_PENDING         → REFUNDED             │
    → SETTLED
         │                   │                 ▼
         ▼                   ▼             ARBITER
       done                done               │
                                         reviews evidence
                                              │
                                         publishes signed
                                         resolution transition
                                              │
                                    ┌─────────┴─────────┐
                                    │                   │
                                 RELEASE              REFUND
                                    │                   │
                                    ▼                   ▼
                         PAYOUT_PENDING → SETTLED     REFUNDED
```

### Layer separation

| Layer      | Role                                    | Active in normal path | Active in dispute |
|------------|-----------------------------------------|-----------------------|-------------------|
| Lightning  | Incoming customer HTLC and agent payout | yes                   | yes               |
| Nostr      | Canonical PIP-02 public event log        | yes                   | yes               |
| Arbiter    | PIP-03 dispute resolution                | **no**                | **yes**           |

The escrow node generates the hold invoice, retains the preimage `p`, and has exclusive LND permission to settle or cancel it. The customer and agent never receive `p`. An authorized resolution is therefore enforceable against both trading parties, although the node itself remains trusted and operationally controls settlement.

---

### Key design decisions

1. **Node-controlled escrow.** The customer pays a hold invoice created by the escrow node. The agent separately provides a payout invoice. The node alone can settle or cancel the incoming HTLC.

2. **Executable arbitration.** An authorized solver selects release or refund. The escrow service validates that authorization and performs the corresponding LND operations instead of merely publishing advice.

3. **`p` stays inside the escrow service.** The node generates and encrypts the hold-invoice preimage. Solvers authorize an outcome but do not receive `p` or direct Lightning credentials.

4. **Least-privilege separation.** Solvers sign release/refund directives. Only the escrow execution service can call LND. Solver keys cannot move funds directly, and Lightning credentials cannot create policy decisions.

5. **Dispute window fits within the actual HTLC lifetime.** A Lightning hold-invoice HTLC cannot wait indefinitely. Request expiry, invoice expiry, final CLTV delta, accepted HTLC expiry height, dispute deadline, and safety buffer are distinct values defined by the service schema. Seconds and block deltas must not be treated as interchangeable.

6. **Funding cardinality vs custody.** The descriptor's `funding_rules` declare how many participants must fund the escrow (funding confirmation cardinality). This is distinct from custody, spending, authorization, or signature thresholds. `funding_threshold` MUST NOT be interpreted as a spending policy.

7. **Nostr as the coordination layer.** Nostr pubkeys are canonical Pontmore identities. Public lifecycle history uses the immutable PIP-02 event kinds. Private payloads travel through the companion Gift Wrap lane and never override public history.

8. **Implementation behavior belongs to the service schema.** PIP-01 is a compatibility descriptor, not an endpoint or authorization specification. Hold-invoice creation, settlement, cancellation, idempotency, and authentication must be defined by the descriptor's referenced OpenAPI or AsyncAPI schema.

### Required descriptor declaration

The deployment's kind `30361` content declares at least:

```json
{
  "version": 1,
  "escrow_type": "custodial_escrow",
  "networks": ["lightning"],
  "funding_rules": {
    "funding_threshold": 1,
    "participant_count": 1
  },
  "dispute_rules": {
    "policy": "pip03"
  },
  "reference_format": "bolt11_or_custodial_escrow_reference",
  "service": {
    "schema": {
      "type": "openapi",
      "url": "https://escrow.example.com/pontmore-lightning-custodial-v1.openapi.json"
    }
  },
  "updated_at": 1724000000
}
```

The event also uses a stable `d` tag and repeated `network` tags for relay discovery. `content.networks` remains canonical. Deployments must replace the example schema URL with a safe, versioned HTTPS artifact defining this implementation's exact behavior.

---

## Protocol flow

### Phase 1 — Request and acceptance

1. Customer and agent negotiate terms over Nostr or another declared channel.
2. The customer publishes an immutable PIP-02 swap request event (`kind 7300`) containing `version`, `swap_id`, `swap_type`, `agent`, `customer`, `escrow_reference`, `fiat`, `bitcoin`, and `expiry`.
3. The agent accepts or rejects the request with a PIP-02 transition event (`kind 7301`). Each transition declares `swap_id`, `state`, `prev_state`, `actor_role`, `reason`, and `created_at`.
4. Non-public terms and payment instructions travel through versioned Gift Wrap messages correlated by `swap_id`.

### Phase 2 — Funding (node-controlled HTLC)

1. The escrow node generates `p ← CSPRNG(32 bytes)` and stores it encrypted.
2. The node computes `H = SHA256(p)` and creates the hold invoice for the requested amount.
3. The node delivers the BOLT 11 hold invoice to the customer through the private lane.
4. The agent delivers a separate payout invoice to the node through the private lane.
5. The customer validates and pays the escrow invoice. The HTLC becomes pending on the escrow node.
6. After verifying the complete accepted payment and payout instruction, the node publishes `FUNDED` (`kind 7301`) and may publish reference-style evidence (`kind 7302`).

### Phase 3 — Normal resolution (no arbiter)

```
FUNDED
   │
   ├── delivery confirmed ──► node settles incoming HTLC
   │                          and pays agent invoice ──────────► SETTLED
   │
   └── refund condition met ──► node cancels hold invoice ─────► REFUNDED
```

The escrow node controls `settle` vs `cancel`. Normal release requires the authorization declared by the service schema. The node records execution durably and publishes final state only after Lightning reconciliation.

### Phase 4 — Dispute resolution (arbiter activated)

```
FUNDED
   │
 dispute raised (by buyer or seller)
   │
   ▼
DISPUTED
   │
 arbiter reviews evidence
   │
   ▼
arbiter publishes signed PIP-02 transition event (`kind 7301`):
   {
     "swap_id": "...",
     "state": "RELEASE_AUTHORIZED" | "REFUND_AUTHORIZED",
     "prev_state": "DISPUTED",
     "actor_role": "escrow_operator",
     "reason": "...",
     "created_at": ...
   }
   │
   ▼
RELEASE_AUTHORIZED or REFUND_AUTHORIZED
```

The transition signature is provided by the normal Nostr event signature; a signature field is not embedded in `content`. The escrow service verifies that the signer is the assigned, write-authorized solver, records the directive idempotently, and executes it. A release settles the incoming HTLC and pays the agent's invoice; a refund cancels the incoming hold invoice. A `kind 7304` note may carry minimal public context while sensitive evidence remains private.

### Dispute window constraint

```
HTLC CREATED ───────────────────────► CLTV EXPIRY ──► HTLC FAILS
                  │                              │
                  ├── dispute window ────────────┤
                  │   (must fit within this)     │
                  │                              │
            dispute must be raised, reviewed,
            and resolved before CLTV expiry
```

---

## Cryptographic boundaries

| Layer          | Secret | Purpose                          | Exposure                 |
|----------------|--------|----------------------------------|--------------------------|
| Lightning      | `p`    | Incoming hold-invoice settlement | Escrow execution service |
| Solver         | —      | Signed resolution authorization  | Public Nostr relays      |
| Identity       | nsec   | Signing and NIP-44 key agreement | Never exposed            |

`p` is generated and encrypted by the escrow node, used only for incoming HTLC settlement, and never disclosed to either party or a solver. The escrow node controls the funds operationally and must be treated as a trusted service boundary.

---

## Swap states

```
REQUESTED           → immutable kind 7300 request exists
ACCEPTED            → agent accepted the request
INVOICED            → hold invoice delivered privately
FUNDED              → complete HTLC payment accepted by the escrow node
DISPUTED            → kind 7303 dispute opened; arbiter reviewing
RELEASE_AUTHORIZED  → arbiter or normal policy authorized release
REFUND_AUTHORIZED   → arbiter or timeout policy authorized refund
PAYOUT_PENDING      → incoming HTLC settled; outgoing agent payout unresolved
SETTLED             → incoming settlement and agent payout verified
REFUNDED            → Lightning cancellation or expiry verified
EXPIRED             → request expired before funding
```

### State transitions

```
REQUESTED ──→ ACCEPTED ──→ INVOICED ──→ FUNDED
   │                                      ├──→ RELEASE_AUTHORIZED ──→ PAYOUT_PENDING ──→ SETTLED
   ▼                                      ├──→ REFUND_AUTHORIZED  ──→ REFUNDED
EXPIRED                                   └──→ DISPUTED
                                                    ├──→ RELEASE_AUTHORIZED ──→ PAYOUT_PENDING ──→ SETTLED
                                                    └──→ REFUND_AUTHORIZED  ──→ REFUNDED
```

Every state change is an append-only `kind 7301` event with a coherent `prev_state`. A dispute is opened with `kind 7303` and reflected by a transition to `DISPUTED`. Immutable history is authoritative over the optional replaceable snapshot (`kind 30362`). State names and Lightning operations are implementation-specific and must also appear in the referenced service schema; PIP-02 defines the event grammar, not this subtype's state vocabulary.

---

## Security properties

- **Party-resistant settlement.** Neither customer nor agent can settle or cancel the escrow invoice because neither has the preimage or node credentials.
- **Executable dispute outcome.** An authorized solver's directive is executed by the node that controls the hold invoice.
- **Dispute auditability.** Arbiter decisions are signed Nostr events with verifiable Schnorr signatures. Any relay observer can verify the arbiter made a particular decision at a particular time.
- **Authenticated history.** Swap requests, transitions, evidence references, disputes, notes, and snapshots are signed Nostr events. This authenticates the publisher but does not prove that every published business claim is true.
- **Solver/executor separation.** A solver does not hold `p` or LND credentials. The escrow node executes only authenticated, policy-valid directives.

---

## Limitations (MVP)

- **Trusted escrow node.** Enforcement is cryptographic against the trading parties, not against the operator. A compromised node can settle early, cancel incorrectly, withhold payout, or become unavailable.
- **CLTV-bound dispute window.** Dispute resolution must complete before HTLC expiry. Long disputes on short-CLTV channels are at risk.
- **Single arbiter.** No redundancy or threshold-based arbitration.
- **Two-leg release is not atomic.** Settling the incoming HTLC and paying the agent invoice are separate operations. The node must persist intent, retry payout safely, and expose `PAYOUT_PENDING` rather than claiming success prematurely.
- **Liquidity lock.** A held payment consumes HTLC slots and route liquidity, so this subtype is unsuitable for long-running delivery or arbitration windows.
- **Implementation schema required.** This repository does not become interoperable merely by using PIP events; deployments must publish and validate the exact service schema referenced by their kind `30361` descriptor.

### Path to cryptographic enforcement (PTLC upgrade)

A future PTLC/adaptor-signature subtype could address the enforcement problem if the complete construction is supported and independently reviewed:

- The PTLC requires the arbiter's adaptor signature to settle.
- The seller **cannot** settle unilaterally after a dispute is raised.
- The arbiter provides the adaptor to the winning party.
- This is `settlement_enforcement: cryptographic` — no trust in the seller's cooperation.

The PIP-02 event grammar and PIP-03 policy boundary can remain compatible, but the new subtype must advertise its own custody, authorization, timeout, and settlement behavior through PIP-01 and its referenced service schema.

---

## Project structure

```
pontmore/
├── README.md
├── docs/
│   └── IMPL_GUIDE.md
├── src/
│   ├── nostr/             ← Nostr identity, event signing, NIP-44, Gift Wrap
│   ├── swap/              ← PIP-02 request and append-only state history
│   ├── lightning/         ← node-owned hold invoice, payout and reconciliation
│   └── arbiter/           ← solver authorization and executable resolutions
└── tests/
```

---

## Dependencies

- **Nostr.** [nostr-tools](https://github.com/nbd-wtf/nostr-tools) or [rust-nostr](https://github.com/rust-nostr/nostr) for event creation, signing, NIP-44 encryption, and Gift Wrap.
- **Lightning.** LND (REST/gRPC) or Core Lightning (commando/sparko) with hold-invoice support.
- **Storage.** SQLite or PostgreSQL for swap and dispute state persistence.

---

## License

MIT
