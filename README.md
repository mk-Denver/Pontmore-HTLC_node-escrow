# Pontmore — HTLC Node Escrow

**Nostr-coordinated Lightning hold-invoice escrow with arbiter-mediated dispute resolution.**

Pontmore enables trust-minimized value exchange between two parties by combining Nostr for identity, coordination, and encrypted delivery with Lightning Network hold invoices for bitcoin settlement. An arbiter holds and conditionally releases escrow authorization without ever exposing the Lightning payment preimage — keeping the payment layer and the escrow decision layer cryptographically separated.

This repository implements the `lightning_hold_invoice` escrow subtype defined in [PIP-01](https://github.com/pontmore/pips/blob/main/PIP-01-escrow-descriptor.md). Swap state transitions follow [PIP-02](https://github.com/pontmore/pips/blob/main/PIP-02-swap-state-machine.md). Dispute policy follows [PIP-03](https://github.com/pontmore/pips/blob/main/PIP-03-dispute-policy.md).

---

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│                         NOSTR                              │
│              Identity + Coordination                       │
│                                                           │
│  ┌──────────────────┐    ┌──────────────────────────────┐ │
│  │  Escrow Descriptor│    │  Swap State Machine (PIP-02)│ │
│  │  (PIP-01, k30361)│    │  + Gift Wrap private lane   │ │
│  └────────┬─────────┘    └──────────────┬───────────────┘ │
│           │                             │                  │
│  NIP-44 encrypted authorization delivery                  │
└───────────┼─────────────────────────────┼──────────────────┘
            │                             │
       signed escrow                 service schema
       agreement                     (openapi/asyncapi)
            │                             │
    ┌───────┴────────┐     ┌───────┐      │
    │     Buyer      │     │ Seller│      │
    │   (Alice)      │     │ (Bob) │      │
    └───────┬────────┘     └───┬───┘      │
            │                  │          │
            └────────┬─────────┘          │
                     │                    │
                ┌────┴────┐               │
                │ ARBITER │◄──────────────┘
                └────┬────┘
                     │
       ┌─────────────┴──────────────┐
       │                            │
  ┌────┴─────┐              ┌──────┴──────┐
  │Lightning │              │   Escrow    │
  │  Node    │              │   Engine    │
  │          │              │             │
  │ hold     │              │ secret(e)   │
  │ invoice  │              │             │
  │ (preimage│              │ RELEASE /   │
  │  p)      │              │ REFUND      │
  └────┬─────┘              └──────┬──────┘
       │                           │
       │   encrypted authorization │
       └───────────┬───────────────┘
                   │
            Nostr delivery
                   │
              recipient(s)
```

### Key design decisions

1. **Separate Lightning preimage from escrow secret.** The hold-invoice preimage `p` proves and captures payment. A distinct escrow secret `e` controls fund release. The arbiter never exposes `p` outside the Lightning protocol.

2. **Hold invoices, not standard invoices.** A hold invoice (`holdinvoice` / `hodl` invoice) gives the arbiter explicit settle/cancel control over the HTLC. The arbiter calls `settle(preimage)` when the HTLC arrives, then holds the funds until the escrow decision is made.

3. **Nostr as the coordination layer.** Identities are Nostr keypairs. Swap states are public Nostr events. Private payloads — raw invoices, settlement instructions, escrow secrets — travel through the Gift Wrap private lane.

4. **Public/private boundary.** Wallet identifiers, custody backend details, private payment credentials, raw BOLT 11 invoice strings, and settlement secrets are **never** published in public Nostr events.

5. **Arbiter-controlled Lightning node (MVP).** The arbiter operates the node that receives escrowed funds. A future trust-minimized version replaces this with a 2-of-3 multisig or PTLC/adaptor-signature construction.

---

## Protocol flow

### Phase 1 — Agreement

1. Buyer and seller negotiate terms over Nostr.
2. Both parties sign an escrow agreement event containing: escrow ID, parties, amount, conditions for release/refund, arbiter pubkey.
3. The signed agreement is published as a Nostr event.

### Phase 2 — Funding (hold invoice)

1. Arbiter validates the signed agreement.
2. Arbiter generates two independent 256-bit secrets:
   - `p` — the Lightning preimage (SHA256(p) goes into the hold invoice).
   - `e` — the escrow decision secret.
3. Arbiter creates a Lightning hold invoice with `H = SHA256(p)` for the agreed amount.
4. Arbiter delivers the BOLT 11 invoice to the buyer via Gift Wrap (private lane).
5. Buyer pays the invoice. The HTLC arrives at the arbiter's node.
6. Arbiter calls `settle(preimage)` to claim the HTLC. `p` is consumed by the Lightning protocol and never exposed to Nostr.

### Phase 3 — Escrow decision

The arbiter evaluates:

```
Release = Payment confirmed ∧ Delivery confirmed
Refund  = Payment confirmed ∧ Refund condition met
```

### Phase 4 — Authorization delivery

The arbiter constructs an authorization object:

```json
{
  "escrow_id": "abc123",
  "buyer": "npub1...",
  "seller": "npub1...",
  "amount_msat": 100000000,
  "decision": "RELEASE",
  "secret": "e",
  "timestamp": 1724000000,
  "signature": "<arbiter_signature_over_canonical_json>"
}
```

This object is encrypted to the recipient via NIP-44 and delivered through Gift Wrap. The recipient uses the escrow secret `e` to claim or verify the outcome.

---

## Cryptographic separation

| Layer          | Secret | Purpose                          | Exposure                 |
|----------------|--------|----------------------------------|--------------------------|
| Lightning      | `p`    | Hold-invoice payment condition   | LN protocol only         |
| Escrow         | `e`    | Authorization / claim            | Gift Wrap encrypted DMs  |
| Identity       | nsec   | Signing and NIP-44 key agreement | Never exposed            |

The hold-invoice preimage `p` is **never** used as the escrow authorization and **never** appears in public Nostr events. This prevents the Lightning payment secret from leaking through the message layer.

---

## Security properties

- **Payment atomicity.** The buyer's funds are provably locked in the hold invoice HTLC before the escrow decision begins.
- **Hold-invoice control.** The arbiter calls `settle(preimage)` explicitly, giving it custody over exactly the funds it needs to escrow.
- **Authorization confidentiality.** The escrow secret `e` is end-to-end encrypted to each party's Nostr keypair via NIP-44 and Gift Wrap.
- **Non-repudiation.** All agreements and state transitions are Nostr-signed events with verifiable signatures.

---

## Limitations (MVP)

- **Custodial escrow.** The arbiter controls the Lightning node and can unilaterally withhold or steal funds.
- **Single arbiter.** No redundancy or threshold-based arbitration.
- **No on-chain fallback.** Funds are purely Lightning-native with no blockchain enforcement.

### Path to trust-minimization

Replace the arbiter's custodial Lightning node with:

- A **2-of-3 multisig** where any two of {buyer, seller, arbiter} can spend, or
- A **PTLC/adaptor-signature** construction where the arbiter provides a signature adaptor that unlocks only after the decision is published.

The Nostr coordination, agreement signing, and encrypted authorization delivery layers remain unchanged.

---

## Project structure

```
pontmore/
├── README.md
├── docs/
│   └── IMPL_GUIDE.md
├── src/
│   ├── nostr/             ← Nostr identity, event signing, NIP-44, Gift Wrap
│   ├── escrow/            ← escrow agreement, state machine, decision engine
│   ├── lightning/         ← LND/CLN hold-invoice integration, HTLC monitoring
│   └── arbiter/           ← arbiter server, REST endpoints
└── tests/
```

---

## Dependencies

- **Nostr.** [nostr-tools](https://github.com/nbd-wtf/nostr-tools) or [rust-nostr](https://github.com/rust-nostr/nostr) for event creation, signing, NIP-44 encryption, and Gift Wrap.
- **Lightning.** LND (REST/gRPC) or Core Lightning (commando/sparko) with hold-invoice support.
- **Storage.** SQLite or PostgreSQL for escrow state persistence.

---

## License

MIT