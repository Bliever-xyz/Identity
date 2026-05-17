# Onboarding Architecture — Theoretical & System Design

> **Audience** Anyone who wants to understand *what* this system does, *why* it
> is designed this way, and *how* the parts relate — without needing to read code.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Core Philosophy](#2-core-philosophy)
3. [Identity Model — The Two-Layer Approach](#3-identity-model--the-two-layer-approach)
4. [High-Level Architecture](#4-high-level-architecture)
5. [The Dual-Signature Binding Protocol](#5-the-dual-signature-binding-protocol)
6. [Security Model & Threat Mitigation](#6-security-model--threat-mitigation)
7. [API Surface Overview](#7-api-surface-overview)
8. [Data Flow Diagrams](#8-data-flow-diagrams)
9. [Design Decisions & Trade-offs](#9-design-decisions--trade-offs)
10. [Environment Configuration](#10-environment-configuration)

---

## 1. Problem Statement

Building a social application on Base Chain with Nostr protocol requires solving
a fundamental tension:

| Goal | Challenge |
|------|-----------|
| **Decentralised social identity** | Nostr keys are raw cryptographic material — hard for mainstream users |
| **Onchain value transfer (tips, payments)** | Requires a blockchain wallet — even harder for mainstream users |
| **Web2-grade onboarding UX** | Email / SMS / OAuth — users must not see seed phrases or passkeys |
| **Self-sovereign ownership** | No third party should custody user keys |

The system solves this by giving every user **two independent identities** that
are cryptographically linked at the application layer — but each managed by the
best tool for its purpose.

---

## 2. Core Philosophy

```
Social Layer  →  Pure Nostr Protocol          (decentralised, portable, censorship-resistant)
Value Layer   →  Base Chain / CDP Smart Wallet (programmable money, gasless UX)
Linking       →  Dual-Signature Binding       (cryptographic proof, no trusted third party)
UX Goal       →  Web2 onboarding UX           →  Web3 ownership
```

**Key principle:** The two identities are **independent** (neither derived from
the other) but **linked** by mutual cryptographic consent. Either can be updated
or rotated without destroying the other.

---

## 3. Identity Model — The Two-Layer Approach

### 3.1 Social Identity — Nostr Keypair

| Property | Value |
|----------|-------|
| Algorithm | secp256k1 (same curve as Bitcoin/Ethereum) |
| Signature scheme | Schnorr (BIP-340) |
| Public key | 32-byte hex string (`npub`) |
| Private key | 32-byte secret (`nsec`) — **never leaves the device** |
| Generation | `crypto.getRandomValues()` client-side |
| Storage | AES-GCM encrypted in IndexedDB (PRF or WebCrypto key) |
| Protocol role | Signs all social actions: posts, DMs, follows, binding events |

The Nostr keypair is the user's **permanent decentralised identity**. It is
portable across all Nostr clients and relays.

### 3.2 Value Identity — CDP ERC-4337 Smart Account

| Property | Value |
|----------|-------|
| Standard | ERC-4337 Account Abstraction |
| Chain | Base (mainnet) / Base Sepolia (dev) |
| Authentication | Email OTP / SMS / Google / Apple OAuth — via Coinbase CDP |
| Key custody | Passkey in device secure enclave — **CDP never holds the key** |
| Signature scheme | ERC-1271 `isValidSignature` (on-chain verification) |
| Gasless UX | CDP Paymaster sponsors gas for new users |
| SDK | `@coinbase/cdp-react` / `@coinbase/cdp-hooks` |

The Smart Account is the user's **onchain wallet**. No seed phrase is ever
shown. The user authenticates with an email OTP or social login they already know.

### 3.3 Why Two Independent Keys?

An alternative approach would derive the Nostr key from the wallet key (or
vice versa). This is explicitly rejected because:

- **Key separation:** A compromised CDP session should not expose the social
  identity, and vice versa.
- **Portability:** The user can migrate their Nostr identity to a different
  wallet without losing their social graph.
- **Nostr philosophy:** The protocol expects keys to be self-generated and
  self-custodied. Deriving from a custody service contradicts this.

---

## 4. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CLIENT (Browser)                              │
│                                                                         │
│  ┌──────────────────┐         ┌──────────────────────────────────────┐  │
│  │  CDP Embedded    │         │       Local Crypto Module            │  │
│  │  Wallet SDK      │         │                                      │  │
│  │                  │         │  generateSecretKey()                 │  │
│  │  Email/SMS/OAuth │         │  → nsec (Uint8Array[32])             │  │
│  │  → Smart Account │         │  → npub (hex string)                 │  │
│  │    on Base       │         │  PRF/WebCrypto → encrypt nsec        │  │
│  └────────┬─────────┘         └──────────────┬───────────────────────┘  │
│           │ evmAddress                        │ npub                     │
│           └──────────────────┬────────────────┘                         │
│                              │                                           │
│                    ┌─────────▼──────────┐                               │
│                    │  Binding Module     │                               │
│                    │                     │                               │
│                    │  Sign Kind 30078    │ ← nsec signs                 │
│                    │  Sign EVM consent   │ ← Smart Account signs        │
│                    └─────────┬──────────┘                               │
└──────────────────────────────┼──────────────────────────────────────────┘
                               │ BindIdentityPayload
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        BACKEND (This codebase)                          │
│                                                                         │
│  POST /api/onboarding        POST /api/bind-identity                    │
│  POST /api/verify-identity                                              │
│                    │                                                    │
│                    ▼                                                    │
│         ┌──────────────────┐                                           │
│         │  validateBinding  │                                           │
│         │  Payload()        │                                           │
│         │                   │                                           │
│         │  1. Timestamp     │                                           │
│         │  2. Event struct  │                                           │
│         │  3. Pubkey match  │                                           │
│         │  4. Schnorr sig   │ (nostr-tools/pure)                       │
│         │  5. Claim JSON    │                                           │
│         │  6. EVM address   │                                           │
│         │  7. BindingId     │                                           │
│         │  8. ERC-1271 sig  │ (viem → Base RPC)                        │
│         └────────┬──────────┘                                           │
└──────────────────┼──────────────────────────────────────────────────────┘
                   │ BindingSuccessResponse
                   ▼
         ┌─────────────────┐          ┌─────────────────┐
         │  Indexer        │          │  Nostr Relays   │
         │  (separate svc) │          │  (Kind 30078    │
         │  stores mapping │          │   published by  │
         │  npub ↔ evm     │          │   client)       │
         └─────────────────┘          └─────────────────┘
```

---

## 5. The Dual-Signature Binding Protocol

### Why Two Signatures?

A single signature (Nostr key claiming an EVM address) is a **unidirectional
claim** — it proves the Nostr key *asserts* ownership of the EVM address, but
does not prove the EVM account *consents* to the link.

**Attack with one-sided binding:**
> Mallory publishes a Kind 30078 event claiming Bob's Smart Account address.
> Mallory now receives all tips intended for Bob.

**Solution:** Require both identities to sign a shared payload, making
impersonation impossible unless the attacker controls *both* private keys
simultaneously.

### The Three-Step Handshake

```
Step 1 — Nostr signs a claim about the EVM address
──────────────────────────────────────────────────
Kind 30078 event, content:
{
  "evmAddress":  "0xABC...",   ← the Smart Account
  "timestamp":   1714567890,
  "bindingId":   "uuid-v4",    ← unique per binding attempt
  "version":     "1.0",
  "appId":       "bliever-v1"
}
Signed by: nsec (Schnorr signature)

Step 2 — EVM Smart Account consents to the link
────────────────────────────────────────────────
EIP-191 personal_sign message:
  "bliever-v1 Identity Binding v1.0
   Nostr pubkey: <npub>
   Binding ID:   <uuid>
   Timestamp:    <unix>
   Chain ID:     84532
   By signing, I confirm this Smart Account consents..."
Signed by: CDP Smart Account (ERC-1271)

Step 3 — Backend verifies both, returns acknowledgement
────────────────────────────────────────────────────────
Server checks:
  ✓ Nostr Schnorr signature valid
  ✓ EVM ERC-1271 signature valid
  ✓ Both reference same bindingId and timestamp
  ✓ Timestamp within 5-minute window
Returns: { success: true, bindingId, npub, evmAddress, verifiedAt }
```

### Binding Uniqueness via `bindingId`

The `bindingId` (UUID v4) is generated fresh for each binding attempt. It
appears in both signatures, cryptographically tying them together into an
atomic proof. Without the same `bindingId` in both signatures, the server
rejects the binding — preventing signature splicing attacks.

### Updatability

Kind 30078 is a **replaceable event** in Nostr (addressable by `d` tag).
If the user rotates their Nostr key or migrates to a new Smart Account, they
publish a new Kind 30078 with the same `d` tag (`"identity-binding"`). The
latest event replaces the old one on relays. The backend indexer should track
only the most recent binding.

---

## 6. Security Model & Threat Mitigation

### Threat Matrix

| Attack | Mitigation |
|--------|-----------|
| **XSS steals session token** | Session token never used as an encryption key; nsec encrypted with WebAuthn PRF (device-bound key) |
| **Replay attack** | 5-minute timestamp window + unique `bindingId` |
| **Identity hijacking** | Dual-signature: attacker needs both nsec AND CDP Smart Account access simultaneously |
| **Address impersonation** | EVM consent signature requires passkey/OAuth interaction — cannot be automated |
| **Relay spam / fake bindings** | Backend ignores events not backed by a valid ERC-1271 signature |
| **RPC provider outage** | Three-tier fallback transport (primary → secondary → public) ensures ERC-1271 calls survive single-provider failures |
| **CDP session compromise** | nsec encrypted with WebAuthn PRF, not derived from CDP session |
| **Server-side key exposure** | Server never handles private keys; only public keys and signatures travel to backend |
| **Device clock drift** | The 5-minute window tolerates typical NTP drift, but devices with manually-set clocks that are off by more than 5 minutes will fail onboarding. The client should fetch server time before generating the payload if clock accuracy cannot be assumed (common on mobile). |

### Key Security Properties

| Property | How Achieved |
|----------|-------------|
| nsec never leaves device | Client-side generation + PRF/WebCrypto encryption |
| EVM key never transmitted | CDP manages it in secure enclave; only signatures travel |
| No custodial key storage | Backend stores only public mappings (npub ↔ evmAddress) |
| Cryptographic consent | Both keys must sign; neither party can link without the other's consent |
| Replay protection | Timestamp window + unique bindingId |
| Forward secrecy | New PRF evaluation per session → new encryption key for nsec |

---

## 7. API Surface Overview

| Endpoint | Method | Purpose | When Used |
|----------|--------|---------|-----------|
| `/api/onboarding` | POST | Full onboarding — validate new user's dual-signature proof | New user registration |
| `/api/bind-identity` | POST | Re-link — validate a new binding for existing users | Key rotation / wallet migration |
| `/api/verify-identity` | POST | Read-only Nostr-side proof check | Profile loading, before enabling SocialFi |

### Why Three Endpoints?

- **Separation of concerns:** Onboarding and re-linking are write operations
  (indexer will store the result). Verification is read-only (no storage needed).
- **Different signature requirements:** Verify only checks the Nostr proof (fast,
  no RPC call). Bind/Onboard check both proofs (includes ERC-1271 RPC call).
- **Optimised for use cases:** Profile loads happen frequently; they should not
  pay the cost of an on-chain ERC-1271 call every time.

---

## 8. Data Flow Diagrams

### New User Onboarding

```
Browser                       CDP                    Backend              Base RPC
   │                            │                        │                    │
   │──── Email / OAuth ─────────▶                        │                    │
   │◀─── Smart Account created ─│                        │                    │
   │                            │                        │                    │
   │  generateSecretKey()        │                        │                    │
   │  → npub, nsec               │                        │                    │
   │  PRF encrypt nsec           │                        │                    │
   │  store in IndexedDB         │                        │                    │
   │                            │                        │                    │
   │  sign Kind 30078 (nsec)     │                        │                    │
   │  sign EVM consent (CDP)     │                        │                    │
   │                            │                        │                    │
   │─────────────────── POST /api/onboarding ───────────▶│                    │
   │                            │                        │──── verifyEvent()  │
   │                            │                        │  (local, no RPC)   │
   │                            │                        │                    │
   │                            │                        │── isValidSignature ▶
   │                            │                        │◀─ true/false ───── │
   │                            │                        │                    │
   │◀─────────────── { success: true, bindingId } ───────│                    │
   │                            │                        │                    │
   │  publish Kind 30078         │                        │                    │
   │  → Nostr Relays             │                        │                    │
```

### Profile Load / Tip Verification

```
Browser                     Nostr Relay              Backend
   │                             │                       │
   │── fetch Kind 30078 event ──▶│                       │
   │◀─ { id, pubkey, sig, ... } ─│                       │
   │                             │                       │
   │────── POST /api/verify-identity ─────────────────▶  │
   │                             │                       │── verifyEvent() (local)
   │                             │                       │── parseBindingClaim()
   │                             │                       │── checkAddresses()
   │◀──────── { valid: true, evmAddress } ───────────────│
   │                             │                       │
   │  enable Tip button          │                       │
   │  show wallet balance        │                       │
```

---

## 9. Design Decisions & Trade-offs

### Decision: Backend is stateless (no DB)

The backend does not persist any data. It only validates cryptographic proofs
and returns structured responses. The indexer is a separate service.

**Why:** Clean separation of concerns. The validation logic can be deployed
independently, scaled horizontally, and reasoned about without database state.

**Trade-off:** Callers (indexer) must handle idempotency — submitting the same
binding twice will get two `200` responses, both valid.

### Decision: Fail-fast validation order

Validation steps run in order of cheapness (no-network first, network last).
The ERC-1271 RPC call is always last.

**Why:** Malformed or tampered payloads are caught locally (microseconds)
before burning an RPC call. This protects against DoS via cheap invalid requests.

### Decision: Separate `verify-identity` from `bind-identity`

Verification does NOT check the EVM consent signature (no RPC call).

**Why:** Profile verification is called frequently (every profile view). A full
ERC-1271 RPC call on every view would be expensive. The Nostr Schnorr signature
alone is sufficient to confirm the social identity claim for display purposes.
The EVM signature is verified once at bind time; subsequent checks trust the
relay-stored event.

### Decision: `buildEVMConsentMessage` is in its own file

The message builder is isolated because it is a **shared contract** between
the server (this codebase) and the client. Any developer changing this file
must understand they are making a breaking change.

### Decision: Timestamp appears in three places (intentional redundancy)

`timestamp` appears in the top-level payload, in `NostrBindingClaim` (inside
`event.content`), and is reflected in the EVM consent message that the Smart
Account signs. This is not an oversight — it is a tamper-detection mechanism.

Each layer is cryptographically independent: the Schnorr signature covers the
content (including `claim.timestamp`), and the ERC-1271 signature covers the
consent message (including the `timestamp` parameter). An attacker who modifies
the timestamp in one layer will break a signature in another. Collapsing these
fields would remove that cross-layer guarantee.

### Decision: `nostr-tools/pure` (not `nostr-tools`)

The `/pure` sub-path export has no DOM dependencies, making it safe in Node.js
and Edge runtimes. The main `nostr-tools` export pulls in browser-specific code.

---

## 10. Environment Configuration

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `NEXT_PUBLIC_BASE_CHAIN_ID` | No | `84532` | `"8453"` for mainnet, `"84532"` for Sepolia. **Any other value causes a startup error** (validated in `lib/evm/client.ts`). |
| `BASE_RPC_URL` | Recommended | — | Primary Alchemy RPC URL. If absent, the transport chain starts at `BASE_RPC_FALLBACK_URL`. |
| `BASE_RPC_FALLBACK_URL` | Optional | — | Secondary RPC (backup Alchemy key, QuickNode, etc.). Tried if `BASE_RPC_URL` is unreachable. |
| `NEXT_PUBLIC_CDP_PROJECT_ID` | Yes (client) | — | CDP project identifier |
| `NEXT_PUBLIC_CDP_APP_NAME` | Yes (client) | — | CDP application name |
| `NEXT_PUBLIC_DEFAULT_RELAYS` | No (client) | damus/nos.lol | Comma-separated Nostr relay WSS URLs |

> **Production note:** Always set `BASE_RPC_URL` (and optionally `BASE_RPC_FALLBACK_URL`)
> to private RPC providers (Alchemy, QuickNode, etc.). The public Base endpoints are
> rate-limited and unsuitable for ERC-1271 verification at scale. The fallback transport
> chain means a single provider outage no longer causes all EVM signature verifications
> to fail; viem automatically advances to the next reachable transport.