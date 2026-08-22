# Pontmore HTLC Escrow — Implementation Guide

A step-by-step guide to building a Nostr-coordinated P2P Lightning atomic swap with arbiter dispute-resolution fallback. This guide covers the `lightning_hold_invoice` subtype only.

**Normal path:** Direct Lightning HTLC between buyer and seller. The arbiter is not involved.
**Dispute path:** Either party raises a dispute. The arbiter reviews evidence and publishes a signed Nostr decision.

This implementation conforms to PIP-02 (swap state machine) and PIP-03 (dispute policy). Descriptor format is defined by PIP-01 — this guide uses descriptors but does not restate PIP-01.

---

## 1. Environment setup

### Dependencies

- Node.js ≥ 20 or Rust ≥ 1.75
- **Seller** must run a Lightning node (LND ≥ 0.17 or Core Lightning ≥ 23.11) with hold-invoice support
- **Buyer** needs a Lightning wallet capable of paying invoices
- **Arbiter** needs only Nostr relay access — no Lightning node required for fund custody
- Access to Nostr relays (at least one writable relay)

### Role requirements

| Role    | Lightning node | Hold-invoice | HTLC monitoring | Nostr keypair |
|---------|---------------|--------------|-----------------|---------------|
| Seller  | required      | must create  | must monitor    | required      |
| Buyer   | wallet only   | n/a          | n/a             | required      |
| Arbiter | none required | n/a          | n/a             | required      |

The arbiter does **not** need a Lightning node. The HTLC is between buyer and seller directly. The arbiter only publishes signed Nostr decisions.

---

## 2. Nostr identity and relay layer

### 2.1 Keypair management

Each participant (buyer, seller, arbiter) generates or loads a Nostr keypair.

**Seller nsec** is used for:
- Signing the k30361 escrow descriptor (if seller is also the escrow operator)
- Signing swap state events (INVOICED, FUNDED, SETTLED, REFUNDED)
- NIP-44 shared-key derivation for delivering the BOLT 11 invoice to the buyer
- Gift Wrap encapsulation for private payloads

**Buyer nsec** is used for:
- Signing swap agreement events
- NIP-44 decryption of the Gift Wrap invoice
- Signing dispute events if needed

**Arbiter nsec** is used for:
- Signing the k30361 escrow descriptor
- Signing dispute decision events (CLOSED)
- NIP-44 decryption of Gift Wrap evidence payloads

Never expose nsec in plaintext on disk.

### 2.2 Relay subscriptions

Each participant maintains persistent WebSocket connections to at least one writable relay.

**Seller subscribes to:**
- Swap agreement events (tagged with seller's npub)
- Dispute events (tagged with the swap ID)
- Gift Wrap events (addressed to seller's npub)

**Buyer subscribes to:**
- Swap state events (tagged with the swap ID)
- Gift Wrap events (addressed to buyer's npub — contains the BOLT 11 invoice)
- Arbiter decision events (tagged with the swap ID)

**Arbiter subscribes to:**
- Swap agreement events (tagged with arbiter's npub — to track active swaps)
- Dispute events (tagged with arbiter's npub)
- Gift Wrap events (addressed to arbiter's npub — contains evidence payloads)

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

Private payloads — BOLT 11 invoice strings, delivery proofs, dispute evidence — travel through the Gift Wrap lane. These are NIP-44 encrypted messages sealed inside wrapper events.

Public swap state events carry only lifecycle transitions. The private lane carries what the public events must not expose.

---

## 3. Swap agreement protocol

### 3.1 Agreement structure

The swap agreement is a signed Nostr event:

```json
{
  "kind": <swap_kind>,
  "tags": [
    ["p", "<seller_npub>"],
    ["p", "<buyer_npub>"],
    ["p", "<arbiter_npub>"],
    ["e", "<k30361_event_id>", "<relay_hint>", "descriptor"],
    ["amount", "100000000", "msat"],
    ["dispute_window", "3600"],
    ["htlc_expiry", "7200"],
    ["expires", "1724000000"]
  ],
  "content": "{\"swap_id\":\"...\",\"conditions\":{\"release\":\"delivery_confirmed\",\"refund\":\"timeout\"}}",
  "created_at": 1724000000
}
```

Key tags:
- `amount` — swap value in millisatoshis
- `dispute_window` — seconds available for dispute resolution (must be less than `htlc_expiry`)
- `htlc_expiry` — seconds until the hold-invoice HTLC CLTV expires
- `expires` — unix timestamp after which the agreement is void

The `e` tag binds the swap to a specific escrow configuration (the arbiter's k30361 descriptor).

### 3.2 Agreement flow

1. Buyer and seller negotiate terms off-chain or via Nostr DMs.
2. Buyer creates the agreement event, signs it, sends to seller.
3. Seller reviews and co-signs.
4. The doubly-signed agreement is published to relays.
5. Arbiter observes the agreement event and records the swap ID for potential dispute tracking.

---

## 4. P2P Lightning HTLC (seller side)

### 4.1 Hold invoice creation (seller)

The seller creates the hold invoice directly — the arbiter is not involved.

1. Generate `p ← CSPRNG(32 bytes)` — the payment preimage.
2. Compute `H = SHA256(p)`.
3. Create a hold invoice with `hash = H`, the agreed amount, and CLTV expiry matching `htlc_expiry` from the agreement.

**LND (gRPC):**
```
lncli addholdinvoice --hash=<hex(H)> --amt=<amount_msat>m --memo="Pontmore swap <id>" --cltv_expiry=<htlc_expiry_blocks>
```

**CLN:**
```
lightning-cli invoice <amount_msat>msat "Pontmore swap <id>" "Pontmore swap <id>" <expiry_seconds>
```

4. Store the mapping: `swap_id → (payment_hash, preimage, bolt11_invoice, amount_msat, state)`.
5. Deliver the BOLT 11 invoice string to the buyer via Gift Wrap.
6. Publish an `INVOICED` state event.
7. Begin monitoring for HTLC arrival.

The preimage `p` never leaves the seller's process. The arbiter never sees it.

### 4.2 HTLC monitoring (seller)

**LND:** Subscribe to `SubscribeInvoices` gRPC stream. Watch for `state = ACCEPTED`.

**CLN:** Register an `htlc_accepted` hook plugin.

On HTLC arrival:
1. Match the `payment_hash` to a swap ID.
2. Publish a `FUNDED` state event.
3. Begin normal-path evaluation or wait for dispute.

### 4.3 Normal path — settlement (seller)

When the delivery condition is met (delivery confirmed by buyer or external oracle):

1. Call `settle(preimage)` on the hold invoice.
2. Publish a `SETTLED` state event.
3. Swap is complete.

### 4.4 Normal path — refund (seller)

When the refund condition is met (timeout, mutual agreement):

1. Call `cancelinvoice` on the hold invoice.
2. Publish a `REFUNDED` state event.
3. HTLC funds return to buyer. Swap is complete.

### 4.5 Hold vs standard invoices

Hold invoices are required. A standard invoice auto-settles on HTLC arrival, removing the seller's ability to refund. With a hold invoice:
- The buyer's HTLC arrives at the seller's node.
- The seller can `settle(preimage)` to claim or `cancelinvoice` to refund.
- The seller must decide before the HTLC's CLTV expiry — after expiry, the HTLC auto-fails and funds return to the buyer.
- The dispute window must fit within the CLTV window.

---

## 5. Swap state machine (PIP-02)

### 5.1 States

```
PENDING    →  agreement signed, hold invoice not yet created
INVOICED   →  hold invoice created and delivered to buyer
FUNDED     →  HTLC pending on seller's node, awaiting resolution
SETTLED    →  seller settled HTLC — normal release, swap complete
REFUNDED   →  seller cancelled HTLC — normal refund, swap complete
DISPUTED   →  dispute raised, arbiter reviewing evidence
CLOSED     →  arbiter decision published, swap concluded
EXPIRED    →  agreement expired before funding or HTLC CLTV expired
```

### 5.2 State transitions

```
PENDING ──→ INVOICED ──→ FUNDED ──→ SETTLED           (normal, seller signs)
                              ├───→ REFUNDED           (normal, seller signs)
                              └───→ DISPUTED           (either party signs)
                                        │
                                        ▼
                                     CLOSED           (arbiter signs)
PENDING ──→ EXPIRED
FUNDED ──→ EXPIRED                                    (HTLC CLTV expired)
```

Each transition is a signed Nostr event. Signing authority:
- `INVOICED`, `SETTLED`, `REFUNDED` — signed by seller
- `FUNDED` — signed by seller (or both parties)
- `DISPUTED` — signed by the disputing party
- `CLOSED` — signed by arbiter

### 5.3 HTLC expiry fallback

If the HTLC CLTV expires before any resolution:
- Lightning auto-fails the HTLC, funds return to the buyer.
- Publish `EXPIRED` state event.
- This is a terminal state. If a dispute was pending, the arbiter may still publish a decision for reputational purposes, but the HTLC is gone.

---

## 6. Dispute resolution (arbiter side)

### 6.1 Dispute initiation

Either party publishes a `DISPUTED` Nostr event:

```json
{
  "kind": <dispute_kind>,
  "tags": [
    ["e", "<swap_agreement_event_id>", "<relay_hint>"],
    ["p", "<arbiter_npub>"]
  ],
  "content": "{\"swap_id\":\"...\",\"reason\":\"...\"}",
  "created_at": ...
}
```

To prevent abuse, the disputing party SHOULD include a small Lightning payment (keysend) to the arbiter as a dispute fee. The fee deters frivolous disputes and compensates the arbiter for review.

### 6.2 Evidence submission

Both parties submit evidence via Gift Wrap to the arbiter:
- Delivery proofs, screenshots, tracking information
- Signed messages, timestamps
- Any mutually agreed oracle attestations

Evidence is encrypted to the arbiter's npub and delivered through the Gift Wrap lane.

### 6.3 Arbiter review

The arbiter:
1. Receives the `DISPUTED` event.
2. Collects evidence from both parties via Gift Wrap.
3. Evaluates conditions from the swap agreement.
4. Publishes a signed `CLOSED` decision event:

```json
{
  "kind": <decision_kind>,
  "tags": [
    ["e", "<swap_agreement_event_id>", "<relay_hint>"],
    ["e", "<dispute_event_id>", "<relay_hint>"],
    ["p", "<buyer_npub>"],
    ["p", "<seller_npub>"]
  ],
  "content": "{\"swap_id\":\"...\",\"decision\":\"RELEASE\",\"reasoning\":\"...\",\"timestamp\":1724000000}",
  "created_at": 1724000000
}
```

The arbiter signs this event with their nsec. The signature is a Schnorr signature verifiable by any Nostr client.

### 6.4 Decision enforcement (MVP — cooperative)

The seller is expected to honor the decision:
- **RELEASE** → seller calls `settle(preimage)`, publishes `SETTLED`
- **REFUND** → seller calls `cancelinvoice`, publishes `REFUNDED`

If the seller ignores the decision, the buyer has a verifiable signed arbiter event as evidence for off-chain dispute resolution (reputation systems, marketplace bans, legal recourse).

### 6.5 Dispute window enforcement

The arbiter MUST publish the decision before the HTLC CLTV expiry. The arbiter's implementation should:
1. Track `htlc_expiry` from the swap agreement.
2. Set an internal deadline at `htlc_expiry - buffer` (e.g., 10 blocks).
3. If the deadline passes without a decision, publish a `CLOSED` event with `decision: "EXPIRED"`.
4. Alert both parties that the HTLC will auto-fail.

---

## 7. Arbiter server

### 7.1 Core loop

```
while running:
    ensure k30361 descriptor is published and up-to-date
    accept Nostr events (agreements, disputes)
    for each new swap agreement:
        record swap for potential dispute tracking
    for each dispute event:
        notify arbiter operator
        collect evidence from both parties
        evaluate against swap conditions
        if decision reached:
            publish signed CLOSED event
        if deadline approaching without decision:
            publish EXPIRED CLOSED event
```

The arbiter does **not**:
- Generate hold invoices
- Hold preimages
- Monitor HTLC states
- Settle or cancel invoices
- Custody funds

### 7.2 Storage schema

```
swaps:
  id                  TEXT PRIMARY KEY
  buyer_npub          TEXT NOT NULL
  seller_npub         TEXT NOT NULL
  amount_msat         INTEGER NOT NULL
  payment_hash        TEXT                  # SHA256(preimage) — observed from agreement
  htlc_expiry         INTEGER               # seconds from agreement
  dispute_window      INTEGER               # seconds from agreement
  state               TEXT NOT NULL         # PENDING | INVOICED | FUNDED | SETTLED | REFUNDED | DISPUTED | CLOSED | EXPIRED
  agreement_json      TEXT                  # full signed agreement
  created_at          INTEGER NOT NULL
  updated_at          INTEGER NOT NULL

disputes:
  id                  TEXT PRIMARY KEY
  swap_id             TEXT REFERENCES swaps(id)
  raised_by_npub      TEXT NOT NULL
  reason              TEXT
  evidence            TEXT                  # JSON array of evidence references
  decision            TEXT                  # RELEASE | REFUND | EXPIRED
  decision_event_id   TEXT                  # Nostr event ID of CLOSED event
  created_at          INTEGER NOT NULL
  resolved_at         INTEGER
```

### 7.3 REST endpoints (arbiter)

| Method | Path                      | Description                          |
|--------|---------------------------|--------------------------------------|
| GET    | `/swap/:id`               | Get swap state                       |
| POST   | `/swap/:id/dispute`       | Raise a dispute (either party)       |
| POST   | `/swap/:id/evidence`      | Submit evidence (authenticated)      |
| GET    | `/swap/:id/decision`      | Get arbiter decision                 |

---

## 8. Client integration

### 8.1 Buyer flow

1. Load Nostr keypair.
2. Connect to relays. Subscribe to swap state events.
3. Negotiate terms with seller.
4. Sign swap agreement, send to seller for co-signing.
5. Receive BOLT 11 hold invoice via Gift Wrap.
6. Pay the invoice via Lightning.
7. Wait for `FUNDED` state event.
8. Confirm delivery (if goods received).
9. Wait for `SETTLED` state event — swap complete.

If delivery never arrives:
1. Publish `DISPUTED` event before dispute window closes.
2. Submit evidence to arbiter via Gift Wrap.
3. Wait for arbiter `CLOSED` decision.
4. If decision is `REFUND` and seller honors it, HTLC cancels, funds return.

### 8.2 Seller flow

1. Load Nostr keypair. Load Lightning node credentials.
2. Connect to relays.
3. Accept terms from buyer, co-sign agreement.
4. Generate `p`, create hold invoice with CLTV >= dispute_window + buffer.
5. Deliver BOLT 11 invoice to buyer via Gift Wrap. Publish `INVOICED`.
6. Wait for HTLC arrival → publish `FUNDED`.
7. If delivery confirmed: call `settle(preimage)`, publish `SETTLED`.
8. If refund condition: call `cancelinvoice`, publish `REFUNDED`.
9. If dispute raised: stop, do not settle or cancel. Wait for arbiter decision.

### 8.3 Arbiter setup

1. Load nsec (env, vault, or HSM).
2. Connect to relays. Subscribe to agreement and dispute events.
3. No Lightning node required.
4. Run core loop.

---

## 9. Testing

### 9.1 Unit tests

- Swap agreement signing and validation (both parties, expired, missing tags).
- Hold-invoice creation (verify payment_hash matches SHA256(preimage)).
- State machine transitions — reject invalid transitions (e.g., SETTLED → DISPUTED).
- Dispute event signing and validation.
- Arbiter decision event signing and signature verification.
- NIP-44 encryption round-trip.
- Gift Wrap encapsulation and decapsulation.

### 9.2 Integration tests

- **Regtest Lightning network.** Use Polar with two LND nodes (seller + buyer wallets).
- **Local Nostr relay.** nostr-rs-relay in Docker.
- **Normal release path.** Agreement → hold invoice (seller) → HTLC (buyer pays) → FUNDED → seller settles → SETTLED.
- **Normal refund path.** Agreement → hold invoice → HTLC → FUNDED → seller cancels → REFUNDED.
- **Dispute path.** Agreement → hold invoice → HTLC → FUNDED → DISPUTED → arbiter reviews → CLOSED (RELEASE) → seller honors.
- **Dispute + seller defection.** Agreement → HTLC → FUNDED → DISPUTED → CLOSED (REFUND) → seller ignores → HTLC expires → EXPIRED. Verify buyer has signed arbiter event as evidence.
- **HTLC expiry before dispute resolution.** Agreement → hold invoice with short CLTV → HTLC → FUNDED → dispute raised too late → HTLC expires → EXPIRED.

### 9.3 Test harness

A scripted runner that:
1. Spawns regtest Lightning nodes (seller node + buyer wallet).
2. Starts a local Nostr relay.
3. Creates keypairs for buyer, seller, arbiter.
4. Arbiter publishes k30361 descriptor.
5. Executes each scenario and asserts: state transitions, HTLC settle/cancel outcomes, arbiter decision signatures.

---

## 10. Deployment considerations

### 10.1 Security

- **nsec storage.** All keypairs encrypted at rest. Arbiters should use vaulted keys.
- **Seller's preimage storage.** `preimage_p` encrypted at rest with a data encryption key.
- **Lightning node access (seller only).** LND: mTLS + macaroons restricted to `invoice:write`, `invoices:read`. CLN: commando runes with minimal permissions. The arbiter does not need Lightning node access.
- **Dispute fee.** A small Lightning payment (keysend) should accompany dispute events to deter abuse.

### 10.2 Operational

- **CLTV buffer.** Seller MUST set `htlc_expiry` >= `dispute_window` + arbiter review time + safety buffer (e.g., 1 hour).
- **Arbiter availability.** The arbiter must be online and responsive within the dispute window. SLAs should be defined.
- **Monitoring.** Alert on: FUNDED swaps approaching dispute window end without resolution, arbiter decision deadline approaching, relay disconnections.
- **Reputation.** Signed arbiter decisions are permanent public record. Malicious seller behavior (ignoring decisions) and frivolous buyer disputes accumulate reputational cost.

### 10.3 Path to cryptographic enforcement (PTLC upgrade)

The following upgrade makes enforcement cryptographic instead of cooperative:

- Replace `settle(preimage)` with a PTLC that requires the arbiter's adaptor signature.
- The seller cannot settle unilaterally after a dispute is raised — the arbiter's adaptor is required.
- The arbiter provides the adaptor to the winning party.
- This is `settlement_enforcement: cryptographic`.

The Nostr coordination, swap state machine, dispute policy, and decision event format remain unchanged.