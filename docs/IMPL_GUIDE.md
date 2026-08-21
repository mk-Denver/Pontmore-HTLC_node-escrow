# Pontmore HTLC Escrow — Implementation Guide

A step-by-step guide to building a Nostr-coordinated Lightning hold-invoice escrow service. This guide covers the `lightning_hold_invoice` subtype only. For other subtypes (`custodial_escrow`, `cashu_escrow`), see the PIP repository.

This implementation conforms to PIP-02 (swap state machine) and PIP-03 (dispute policy). Descriptor format is defined by PIP-01 — this guide uses descriptors but does not restate PIP-01.

---

## 1. Environment setup

### Dependencies

- Node.js ≥ 20 or Rust ≥ 1.75
- A running Lightning node (LND ≥ 0.17 or Core Lightning ≥ 23.11) with hold-invoice support
- Access to Nostr relays (at least one writable relay)

### Lightning node requirements

- Create hold invoices via `holdinvoice` (LND) or `invoice` with `--dev-hodl` / `hodl-invoice` plugin (CLN).
- Subscribe to HTLC arrival events: `SubscribeInvoices` gRPC stream (LND) or `htlc_accepted` hook (CLN).
- Call `SettleInvoice` with the preimage (LND) or `invoice` notification settle (CLN).
- Outbound liquidity for payouts on RELEASE and REFUND.

---

## 2. Nostr identity and relay layer

### 2.1 Keypair management

Generate or load a Nostr keypair for the arbiter. The nsec is used for:

- Signing the k30361 escrow descriptor (per PIP-01; this guide assumes the descriptor exists but does not define it).
- Signing swap state events (per PIP-02).
- NIP-44 shared-key derivation for private DMs.
- Gift Wrap encapsulation for private execution payloads.

Never expose nsec in plaintext on disk.

### 2.2 Relay connectivity

Maintain persistent WebSocket connections to at least one writable relay. Subscribe to:

- **Swap agreement events** — tagged with the arbiter's npub.
- **Swap state events** — public lifecycle transitions from participants.
- **Gift Wrap events** — sealed private payloads addressed to the arbiter's npub.

### 2.3 NIP-44 encryption

All private payloads are encrypted using NIP-44 (XChaCha20-Poly1305 with ECDH shared secret from sender's nsec and recipient's npub).

```
conversation_key = secp256k1.ecdh(sender_seckey, recipient_pubkey, compressed=true)
nonce = CSPRNG(24 bytes)
ciphertext = xchacha20_poly1305_encrypt(
    key=SHA256(conversation_key),
    nonce=nonce,
    plaintext=payload_bytes,
    aad=b""
)
envelope = base64(nonce + ciphertext)
```

### 2.4 Gift Wrap (private execution lane)

Private execution payloads — BOLT 11 invoice strings, settlement instructions, escrow secrets — travel through the Gift Wrap lane. These are NIP-44 encrypted messages sealed inside a wrapper event and published to relays.

Public swap state events carry only lifecycle transitions. The private lane carries what the public events must not expose.

---

## 3. Swap agreement protocol

### 3.1 Agreement structure

The escrow agreement is a signed Nostr event that references the arbiter's k30361 descriptor:

```json
{
  "kind": <swap_kind>,
  "tags": [
    ["p", "<arbiter_npub>"],
    ["p", "<buyer_npub>"],
    ["p", "<seller_npub>"],
    ["e", "<k30361_event_id>", "<relay_hint>", "descriptor"],
    ["amount", "100000000", "msat"],
    ["expires", "1724000000"]
  ],
  "content": "{\"escrow_id\":\"...\",\"conditions\":{\"release\":\"delivery_confirmed\",\"refund\":\"timeout_or_dispute\"}}",
  "created_at": 1724000000
}
```

The `e` tag binds the swap to a specific escrow configuration. The `amount` tag declares the escrow value in millisatoshis.

### 3.2 Agreement validation (arbiter side)

On receiving an agreement event, the arbiter:

1. Verifies all participant signatures (buyer + seller).
2. Checks that the arbiter's npub is tagged.
3. Resolves the referenced k30361 descriptor and verifies its signature.
4. Confirms `escrow_type` is `lightning_hold_invoice`.
5. Confirms the amount is within operating limits.
6. Confirms `dispute_rules.policy` is `pip03`.
7. Checks the agreement has not expired.
8. Assigns a monotonic escrow ID and stores the agreement.

---

## 4. Lightning hold-invoice integration

### 4.1 Hold invoice creation

When the agreement is validated:

1. Generate `p ← CSPRNG(32 bytes)` — the payment preimage.
2. Compute `H = SHA256(p)`.
3. Create a **hold invoice** with `hash = H` and the agreed amount.

**LND (gRPC):**
```
lncli addholdinvoice --hash=<hex(H)> --amt=<amount_msat>m --memo="Pontmore escrow <id>"
```

**CLN:**
```
lightning-cli invoice <amount_msat>msat "Pontmore escrow <id>" "Pontmore escrow <id>" <expiry_seconds>
# requires hold-invoice plugin or hodl mode
```

4. Store the mapping: `escrow_id → (payment_hash, preimage, bolt11_invoice, amount_msat, state)`.
5. Deliver the BOLT 11 invoice string to the buyer via **Gift Wrap** (not as a public Nostr event).

The preimage `p` never leaves the arbiter's process. It is only used in the `settle` RPC call.

### 4.2 Hold vs standard invoices

Hold invoices are required for this implementation. A standard invoice auto-settles on HTLC arrival, giving the arbiter no custody control. With a hold invoice:

- The buyer's HTLC arrives at the arbiter's node.
- The arbiter calls `settle(preimage)` to claim the HTLC.
- Until `settle` is called, the arbiter can `cancel` the invoice to refund the buyer.
- After `settle`, the funds are custodied. The arbiter controls when and how to release or refund.

### 4.3 HTLC monitoring

**LND:** Subscribe to `SubscribeInvoices` gRPC stream. Watch for `state = ACCEPTED` (HTLC arrived, awaiting settle).

```proto
service Lightning {
    rpc SubscribeInvoices (InvoiceSubscription) returns (stream Invoice);
}
```

**CLN:** Register an `htlc_accepted` hook plugin. Receive a JSON-RPC notification when an HTLC matches a hold invoice.

On HTLC arrival:
1. Match the `payment_hash` to an escrow ID.
2. Call `settle(preimage)` to claim the HTLC.
3. Transition swap state to `FUNDED` — publish a signed Nostr event for the state transition.
4. Begin escrow decision evaluation.

### 4.4 Fund disbursement

After the decision is made, the arbiter routes funds out of its Lightning node:

- **RELEASE (to seller):** Pay a Lightning invoice provided by the seller, or use keysend to the seller's node.
- **REFUND (to buyer):** Pay a Lightning invoice provided by the buyer, or use keysend.

The escrow secret `e` is delivered as cryptographic proof. Actual fund movement requires the arbiter to execute the Lightning payment.

---

## 5. Swap state machine (PIP-02)

### 5.1 States

```
PENDING   →  agreement received, hold invoice not yet created
INVOICED  →  hold invoice created and delivered, awaiting payment
FUNDED    →  HTLC settled, arbiter holds funds, awaiting decision
RELEASED  →  escrow secret delivered to seller, payout initiated
REFUNDED  →  escrow secret delivered to buyer, payout initiated
DISPUTED  →  escalated per PIP-03 dispute escalation
EXPIRED   →  agreement expired before funding
CANCELED  →  canceled before funding by agreement
```

### 5.2 State transitions

```
PENDING ──→ INVOICED ──→ FUNDED ──→ RELEASED
                           │
                           ├──→ REFUNDED
                           │
                           └──→ DISPUTED
PENDING ──→ EXPIRED
PENDING ──→ CANCELED
```

Each transition is published as a signed Nostr event. The event `content` carries the new state and metadata. Private payloads (invoice strings, preimages, settlement instructions) never appear in these events.

---

## 6. Escrow decision engine (PIP-03)

### 6.1 Secret generation

Two independent secrets are generated per escrow:

```
p ← CSPRNG(32 bytes)   # Lightning preimage — committed as SHA256(p) in hold invoice
e ← CSPRNG(32 bytes)   # Escrow authorization secret — held until decision
```

`p` is used only in the hold-invoice settle call. `e` is encrypted and delivered in the authorization object.

### 6.2 Decision logic

The arbiter evaluates conditions per PIP-03:

```python
def evaluate_escrow(swap):
    if swap.state != "FUNDED":
        return None
    if condition_met(swap.conditions.release):
        return "RELEASE"
    if condition_met(swap.conditions.refund):
        return "REFUND"
    return None  # still waiting
```

Conditions can include:
- Delivery confirmation (seller publishes proof as a Nostr event or submits via REST).
- Timeout expiry (swap `expires` tag elapsed since FUNDED).
- Arbiter override (manual dispute resolution).

### 6.3 Authorization object

When a decision is reached:

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

The `signature` is a Schnorr signature by the arbiter's nsec over the canonical JSON serialization (excluding the `signature` field itself).

### 6.4 Dispute escalation

When either party raises a dispute per PIP-03:

1. Publish a `DISPUTED` state transition event.
2. Freeze withdrawal until resolution.
3. Arbiter reviews evidence and issues a binding decision.
4. Publish a `RELEASED` or `REFUNDED` state event and deliver the authorization object.

---

## 7. Authorization delivery

### 7.1 Encryption

```python
def encrypt_authorization(auth_object, recipient_pubkey):
    conversation_key = secp256k1.ecdh(arbiter_seckey, recipient_pubkey, compressed=True)
    key = SHA256(conversation_key)
    nonce = CSPRNG(24)
    ciphertext = xchacha20_poly1305_encrypt(
        key=key,
        nonce=nonce,
        plaintext=canonical_json(auth_object),
        aad=b""
    )
    return base64(nonce + ciphertext)
```

### 7.2 Gift Wrap delivery

The encrypted authorization is wrapped in a Gift Wrap event and published to relays:

```
Gift Wrap event
  ├── sealed sender: random one-use pubkey
  ├── recipient: target npub
  └── content: NIP-44 encrypted authorization object
```

The recipient decrypts the outer Gift Wrap with their nsec, then decrypts the inner payload to recover the authorization object.

### 7.3 Delivery policy

| Decision   | Public event     | Gift Wrap target     | Content                           |
|------------|------------------|----------------------|-----------------------------------|
| RELEASE    | `RELEASED`       | seller               | Full authorization + secret `e`   |
| REFUND     | `REFUNDED`       | buyer                | Full authorization + secret `e`   |
| DISPUTED   | `DISPUTED`       | (none)               | No authorization until resolved   |

---

## 8. Arbiter server

### 8.1 Core loop

```
while running:
    accept Nostr events (agreements, proofs, disputes)
    poll Lightning node for HTLC arrivals
    for each HTLC arrived:
        settle(preimage)
        publish FUNDED event
    for each funded swap:
        decision = evaluate_escrow(swap)
        if decision:
            wrap_and_deliver_authorization(swap, decision)
            publish state event
    for each expired pending swap:
        publish EXPIRED event
```

### 8.2 Storage schema

```
swaps:
  id              TEXT PRIMARY KEY
  buyer_npub      TEXT NOT NULL
  seller_npub     TEXT NOT NULL
  amount_msat     INTEGER NOT NULL
  payment_hash    TEXT UNIQUE          # SHA256(preimage)
  preimage_p      BLOB                 # Lightning preimage — encrypted at rest
  secret_e        BLOB                 # Escrow secret — encrypted at rest
  bolt11_invoice  TEXT                 # BOLT 11 string — encrypted at rest
  state           TEXT NOT NULL        # PENDING | INVOICED | FUNDED | RELEASED | REFUNDED | DISPUTED | EXPIRED | CANCELED
  conditions      TEXT                 # JSON: release and refund conditions
  expires_at      INTEGER              # Unix timestamp
  created_at      INTEGER NOT NULL
  updated_at      INTEGER NOT NULL
```

`preimage_p`, `secret_e`, and `bolt11_invoice` should be encrypted at rest using a separate data encryption key derived from the arbiter's operational credentials.

### 8.3 REST endpoints

| Method | Path                      | Description                          |
|--------|---------------------------|--------------------------------------|
| POST   | `/swap/create`            | Submit a signed swap agreement       |
| GET    | `/swap/:id`               | Get swap state                       |
| POST   | `/swap/:id/proof`         | Submit delivery proof (seller)       |
| POST   | `/swap/:id/dispute`       | Raise a dispute (either party)       |

---

## 9. Client integration

### 9.1 Buyer flow

1. Load Nostr keypair.
2. Connect to relays. Subscribe to swap state events.
3. Negotiate terms with seller.
4. Sign swap agreement referencing the arbiter's k30361 descriptor.
5. Submit agreement to arbiter.
6. Receive BOLT 11 hold invoice via Gift Wrap.
7. Pay the invoice via Lightning.
8. Wait for `FUNDED` state event on Nostr.
9. Wait for outcome: `RELEASED` or `REFUNDED` state event → decrypt Gift Wrap authorization.

### 9.2 Seller flow

1. Load Nostr keypair.
2. Connect to relays.
3. Accept terms from buyer, co-sign agreement.
4. Wait for `FUNDED` state event.
5. Provide delivery proof (Nostr event or REST endpoint).
6. On `RELEASED` state event, decrypt Gift Wrap authorization.
7. Verify authorization signature. Provide Lightning invoice for payout.

### 9.3 Arbiter setup

1. Load nsec (env, vault, or HSM).
2. Connect to relays. Subscribe to agreement and swap events.
3. Connect to Lightning node (LND gRPC or CLN) — verify hold-invoice support.
4. Run the core loop.

---

## 10. Testing

### 10.1 Unit tests

- Swap agreement signing and validation.
- Hold-invoice creation (verify payment_hash matches SHA256(preimage)).
- Hold-invoice settle → HTLC claim.
- State machine transitions — reject invalid transitions.
- Decision engine — correct RELEASE vs REFUND vs timeout.
- NIP-44 encryption round-trip.
- Gift Wrap encapsulation and decapsulation.
- Authorization object signing and verification.

### 10.2 Integration tests

- **Regtest Lightning network.** Use Polar or a local regtest setup with LND nodes that have hold-invoice support enabled.
- **Local Nostr relay.** Use nostr-rs-relay in Docker.
- **Happy path.** Agreement → hold invoice → HTLC arrival → settle → FUNDED → RELEASE → Gift Wrap → payout.
- **Refund path.** Agreement → hold invoice → settle → FUNDED → timeout → REFUND → Gift Wrap → payout.
- **Dispute path.** Agreement → settle → FUNDED → DISPUTED → resolution → RELEASE/REFUND → Gift Wrap.
- **Cancel path.** Hold invoice created but expired → arbiter calls `cancelinvoice` → HTLC refunded to buyer.

### 10.3 Test harness

A scripted runner that:
1. Spawns regtest Lightning nodes with hold-invoice support.
2. Starts a local Nostr relay.
3. Creates keypairs for buyer, seller, arbiter.
4. Arbiter publishes k30361 descriptor.
5. Executes each scenario and asserts state transitions, Gift Wrap delivery, and authorization verification.

---

## 11. Deployment considerations

### 11.1 Security

- **nsec storage.** Encrypted at rest (env variable, vault, HSM). Never in plaintext on disk.
- **Preimage storage.** `preimage_p`, `secret_e`, and `bolt11_invoice` encrypted at rest with a separate data encryption key.
- **Lightning node access.** LND: mTLS + macaroons restricted to `invoice:write`, `invoices:read`, `offchain:write`. CLN: commando runes with minimal permissions.
- **Rate limiting.** Limit swap agreement submissions per npub.

### 11.2 Operational

- **Hold invoice expiry.** Set hold-invoice CLTV expiry to match the swap agreement expiry. If the HTLC hasn't arrived by then, call `cancelinvoice` and transition to `EXPIRED`.
- **Node liquidity.** Arbiter must have sufficient outbound liquidity for payouts. Pre-configure channels to well-connected peers.
- **Fee management.** Deduct routing fees from the escrowed amount or charge a flat service fee.
- **Monitoring.** Alert on: stuck swaps (FUNDED without decision), HTLC expiry approaching without settlement, relay disconnections, Lightning node downtime.