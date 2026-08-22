# Pontmore — HTLC Node Escrow

**Nostr-coordinated P2P Lightning atomic swap with arbiter dispute-resolution fallback.**

Pontmore enables P2P value exchange between buyer and seller using a Lightning hold-invoice HTLC. The arbiter is **only activated when a dispute is raised** — the normal path is a direct atomic swap between the two parties. The arbiter publishes a signed Nostr decision; `p` remains a pure Lightning settlement secret and never leaves the HTLC layer.

This repository implements the `lightning_hold_invoice` escrow subtype defined in [PIP-01](https://github.com/pontmore/pips/blob/main/PIP-01-escrow-descriptor.md). Swap state transitions follow [PIP-02](https://github.com/pontmore/pips/blob/main/PIP-02-swap-state-machine.md). Dispute policy follows [PIP-03](https://github.com/pontmore/pips/blob/main/PIP-03-dispute-policy.md).

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
│ "What public state is the swap in?"          │
│ Swap state machine                           │
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
│ Service Schema (OpenAPI / AsyncAPI)           │
│ "How do I actually interact with it?"         │
│ Endpoints, auth, payloads, error format       │
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
                    BUYER ─────────────── SELLER
                      │                      │
                      │   Lightning HTLC     │
                      │   (hold invoice)     │
                      │                      │
                      └──────────┬───────────┘
                                 │
                   ┌─────────────┴─────────────┐
                   │                           │
              NORMAL PATH                  DISPUTE PATH
              (no arbiter)                (arbiter activated)
                   │                           │
         ┌─────────┴─────────┐                 │
         │                   │                 │
    seller settles       seller cancels    dispute raised
    → SETTLED            → REFUNDED             │
         │                   │                 ▼
         ▼                   ▼             ARBITER
       done                done               │
                                         reviews evidence
                                              │
                                         publishes signed
                                         Nostr decision
                                              │
                                    ┌─────────┴─────────┐
                                    │                   │
                                 RELEASE              REFUND
                                    │                   │
                                    ▼                   ▼
                                 SETTLED             REFUNDED
```

### Layer separation

| Layer      | Role                                    | Active in normal path | Active in dispute |
|------------|-----------------------------------------|-----------------------|-------------------|
| Lightning  | HTLC between buyer and seller           | yes                   | yes               |
| Nostr      | Agreement, state events, decision       | yes                   | yes               |
| Arbiter    | Dispute resolution                      | **no**                | **yes**           |

The arbitter does **not** generate the hold invoice, does **not** hold the preimage `p`, and does **not** custody funds. The HTLC is between buyer and seller directly.

---

### Key design decisions

1. **P2P atomic swap by default.** The normal path is a direct Lightning HTLC between buyer and seller. The seller creates a hold invoice with preimage `p`; the buyer pays it. On delivery confirmation, the seller settles. No arbiter involvement.

2. **Arbiter as dispute-resolution overlay.** The arbiter is only activated when a dispute is raised. Its output is a **signed Nostr decision event** — not a Lightning settlement secret, not a custody action.

3. **`p` stays in the Lightning layer.** The hold-invoice preimage `p` is generated by the seller and used only to settle the HTLC. The arbiter never sees `p`, never holds `p`, and never distributes `p`.

4. **No `e` in the normal path.** An escrow authorization secret is not needed when the swap resolves normally. On dispute, the arbiter publishes a signed decision directly — no separate secret.

5. **Dispute window fits within HTLC CLTV expiry.** A Lightning hold invoice HTLC cannot wait indefinitely. The dispute must be raised, reviewed, and resolved before the HTLC's CLTV timeout. The agreement declares this window explicitly.

6. **Funding cardinality vs custody.** The descriptor's `funding_rules` declare how many participants must fund the escrow (funding confirmation cardinality). This is distinct from custody, spending, authorization, or signature thresholds. `funding_threshold` MUST NOT be interpreted as a spending policy.

7. **Nostr as the coordination layer.** Identities are Nostr keypairs. Swap states are public Nostr events. Private payloads travel through Gift Wrap. Arbiter decisions are signed, publicly-verifiable Nostr events.

---

## Protocol flow

### Phase 1 — Agreement

1. Buyer and seller negotiate terms over Nostr.
2. Both parties sign a swap agreement event containing:
   - Swap ID, parties, amount
   - Conditions for release (delivery confirmed) and refund (timeout)
   - Arbiter pubkey for dispute fallback
   - Dispute window (must close before HTLC CLTV expiry)
3. The signed agreement is published as a Nostr event.

### Phase 2 — Funding (P2P HTLC)

1. **Seller generates** `p ← CSPRNG(32 bytes)` — the Lightning preimage.
2. Seller computes `H = SHA256(p)`.
3. Seller creates a hold invoice with `hash = H` and the agreed amount. The invoice has a CLTV expiry that accommodates the dispute window.
4. Seller delivers the BOLT 11 invoice to the buyer via Gift Wrap.
5. Buyer pays the invoice. The HTLC is now pending on the seller's node.
6. Both parties publish a `FUNDED` state event.

### Phase 3 — Normal resolution (no arbiter)

```
FUNDED
   │
   ├── delivery confirmed ──► seller calls settle(preimage) ──► SETTLED
   │
   └── refund condition met ──► seller calls cancelinvoice ────► REFUNDED
```

The seller controls `settle` vs `cancel`. The signed agreement and public swap state provide accountability.

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
arbiter publishes signed Nostr decision event:
   {
     "swap_id": "...",
     "decision": "RELEASE" | "REFUND",
     "reasoning": "...",
     "timestamp": ...,
     "signature": "<arbiter_schnorr_sig>"
   }
   │
   ▼
CLOSED
```

The arbiter's decision is a publicly-auditable Nostr event. The parties are expected to honor it. A future PTLC/adaptor-signature upgrade will enforce this cryptographically.

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
| Lightning      | `p`    | Hold-invoice settlement          | Seller only, LN protocol |
| Arbiter        | —      | Signed decision events           | Public Nostr relays      |
| Identity       | nsec   | Signing and NIP-44 key agreement | Never exposed            |

`p` is generated by the seller, used only in `settle(preimage)`, and never seen by the arbiter. The arbiter produces signed Nostr events as decisions — no secrets, no custody.

---

## Swap states

```
PENDING    →  agreement signed, hold invoice not yet created
INVOICED   →  hold invoice created and delivered, awaiting HTLC
FUNDED     →  HTLC pending on seller's node, awaiting resolution
SETTLED    →  seller settled HTLC with preimage — normal release
REFUNDED   →  seller cancelled HTLC — normal refund
DISPUTED   →  dispute raised, arbiter reviewing
CLOSED     →  arbiter decision published, swap concluded
EXPIRED    →  agreement expired before funding
```

### State transitions

```
PENDING ──→ INVOICED ──→ FUNDED ──→ SETTLED          (normal path)
                                ├──→ REFUNDED         (normal refund)
                                └──→ DISPUTED         (arbiter activated)
                                          │
                                          ▼
                                       CLOSED
PENDING ──→ EXPIRED
```

Each transition is a signed Nostr event. `SETTLED` and `REFUNDED` on the normal path are signed by the seller. `DISPUTED` is signed by whichever party raises it. `CLOSED` is signed by the arbiter.

---

## Security properties

- **Normal-path atomicity.** The buyer's funds are locked in the hold-invoice HTLC. The seller can only claim them by settling with `p`. The seller's refund path is `cancelinvoice`. Both are Lightning-enforced.
- **Dispute auditability.** Arbiter decisions are signed Nostr events with verifiable Schnorr signatures. Any relay observer can verify the arbiter made a particular decision at a particular time.
- **Non-repudiation.** All agreements, state transitions, and arbiter decisions are signed.
- **Arbiter has no custody.** The arbiter never controls the HTLC, never holds `p`, and cannot unilaterally move funds.

---

## Limitations (MVP)

- **Cooperative enforcement.** On dispute, the seller must honor the arbiter's decision by calling `settle` or `cancel`. A malicious seller can ignore the decision and keep the HTLC unresolved until CLTV expiry refunds them (or settle and steal). Enforcement is social/reputational — not cryptographic.
- **CLTV-bound dispute window.** Dispute resolution must complete before HTLC expiry. Long disputes on short-CLTV channels are at risk.
- **Single arbiter.** No redundancy or threshold-based arbitration.

### Path to cryptographic enforcement (PTLC upgrade)

A future PTLC/adaptor-signature subtype solves the enforcement problem:

- The PTLC requires the arbiter's adaptor signature to settle.
- The seller **cannot** settle unilaterally after a dispute is raised.
- The arbiter provides the adaptor to the winning party.
- This is `settlement_enforcement: cryptographic` — no trust in the seller's cooperation.

The Nostr coordination, swap state machine, dispute policy, and decision delivery layers remain identical.

---

## Project structure

```
pontmore/
├── README.md
├── docs/
│   └── IMPL_GUIDE.md
├── src/
│   ├── nostr/             ← Nostr identity, event signing, NIP-44, Gift Wrap
│   ├── swap/              ← swap agreement, state machine
│   ├── lightning/         ← hold-invoice creation (seller), HTLC monitoring
│   └── arbiter/           ← dispute review, signed decision publishing
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