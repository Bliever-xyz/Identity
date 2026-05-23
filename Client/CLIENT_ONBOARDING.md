# Client Onboarding Architecture — Theoretical & System Design

> **Audience** Anyone who wants to understand *what* the client-side onboarding module
> does, *why* it is designed this way, and *how* its parts relate — without needing to
> read TypeScript code.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Core Philosophy](#2-core-philosophy)
3. [Identity Model — Two Layers, One Link](#3-identity-model--two-layers-one-link)
4. [High-Level Architecture](#4-high-level-architecture)
5. [The Dual-Signature Handshake — Client Perspective](#5-the-dual-signature-handshake--client-perspective)
6. [Key Security & Privacy Model](#6-key-security--privacy-model)
7. [nsec Storage — The Two Strategies](#7-nsec-storage--the-two-strategies)
8. [Data Flow Diagrams](#8-data-flow-diagrams)
9. [Module Responsibility Map](#9-module-responsibility-map)
10. [Design Decisions & Trade-offs](#10-design-decisions--trade-offs)
11. [Re-link & Key Rotation](#11-re-link--key-rotation)
12. [Environment Configuration](#12-environment-configuration)

---

## 1. Problem Statement

The application gives every user two independent cryptographic identities that must
be provably linked to each other — without a trusted third party, without custodying
any key server-side, and without asking the user to manage seed phrases.

| Identity Layer | Technology | UX |
|---|---|---|
| **Social** | Nostr secp256k1 keypair | Invisible — generated automatically |
| **Value** | CDP ERC-4337 Smart Account on Base | Email / SMS / Google / Apple login |
| **Link** | Dual-signature binding proof | Two prompts: biometric + wallet consent |

The client module is responsible for the full left side of this operation: generating
keys, encrypting them, building both signatures, and submitting the proof to the server.

---

## 2. Core Philosophy

```
Generate locally → Encrypt locally → Sign locally → Submit proof → Publish socially
```

Every cryptographic secret stays on the device. The server only ever sees:
- Public keys (npub, evmAddress)
- Signatures (Schnorr + ERC-1271)
- The binding coordinates (bindingId, timestamp)

No private key material, no seed phrase, no raw key bytes ever leave the browser.

**Capability injection over hard coupling.** The module does not import any CDP SDK,
React hook, or signing library directly. It receives signing as a function parameter
(`SignMessageFn`). This means the module can be called from any framework, tested with
a mock signer, and upgraded to a new CDP SDK version without changing a single line here.

**Client-only execution.** `flow.ts`, `nsec-storage.ts`, and `relay.ts` rely on
browser APIs (IndexedDB, Web Crypto, WebAuthn, WebSocket) and carry a `"use client"`
directive. In Next.js, this prevents the server bundler from touching them. In other
environments, always import these modules inside browser-only code paths — never in
a module that executes on the server.

---

## 3. Identity Model — Two Layers, One Link

### 3.1 Social Layer — Nostr Keypair

The Nostr keypair is a raw secp256k1 key pair generated entirely in the browser
using `crypto.getRandomValues`. It is the user's permanent, portable social identity.

| Property | Detail |
|---|---|
| Curve | secp256k1 (same as Bitcoin / Ethereum) |
| Signature scheme | Schnorr (BIP-340) |
| Public key (`npub`) | 32-byte hex string — safe to share |
| Private key (`nsec`) | 32-byte Uint8Array — **never leaves the device** |
| Generation | `nostr-tools generateSecretKey()` |
| Storage | AES-256-GCM encrypted in IndexedDB |

### 3.2 Value Layer — CDP Smart Account

The Smart Account is created and authenticated by Coinbase Developer Platform.
The client module does not create this account — it receives the already-authenticated
`evmAddress` as input from the CDP SDK.

| Property | Detail |
|---|---|
| Standard | ERC-4337 Account Abstraction |
| Chain | Base Mainnet or Base Sepolia |
| Auth | Email OTP / SMS / Google / Apple — via `@coinbase/cdp-react` |
| Signing | ERC-1271 `isValidSignature` (on-chain) |
| Key custody | Passkey in device secure enclave — CDP never holds the key |

### 3.3 Why Two Independent Keys?

An alternative would derive the Nostr key from the wallet seed. This is rejected:

- **Compartmentalisation:** A compromised CDP session must not expose the social identity.
- **Portability:** Users can migrate their Nostr identity to a new wallet.
- **Nostr philosophy:** The protocol expects keys to be self-generated. Deriving from a
  custody service contradicts the Nostr model.

---

## 4. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CONSUMER (App / Next.js page)                      │
│                                                                             │
│   CDP SDK                          runOnboarding({                          │
│   ┌──────────────┐                   evmAddress,  ← from CDP auth           │
│   │ useSignMessage│────signMessage──▶  signMessage, ← CDP hook injected     │
│   │ evmAddress   │                   relayUrls,                             │
│   └──────────────┘                   storageStrategy                        │
│                                    })                                       │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                    ┌──────────────────▼──────────────────────┐
                    │          lib/onboarding/flow.ts          │
                    │  (Orchestrator — composes all modules)   │
                    └──┬──────┬────────┬──────────┬───────────┘
                       │      │        │          │
            ┌──────────▼──┐ ┌─▼──────┐ │    ┌────▼──────────┐
            │ nostr/keys  │ │nostr/  │ │    │ crypto/nsec-  │
            │ generateKey │ │event   │ │    │ storage       │
            │ pair()      │ │build+  │ │    │ persistNsec() │
            └─────────────┘ │sign()  │ │    └───────────────┘
                            └────────┘ │
                    ┌──────────────────▼──┐    ┌─────────────────┐
                    │ binding/message.ts   │    │ nostr/relay.ts  │
                    │ buildEVMConsent      │    │ publishEvent    │
                    │ Message()            │    │ ToRelays()      │
                    └──────────────────────┘    └─────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │    api/client.ts         │
                    │    submitOnboarding()    │
                    │    POST /api/onboarding  │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │      BACKEND SERVER      │
                    │  validateBinding         │
                    │  Payload() — 9 steps     │
                    │  ERC-1271 on Base RPC    │
                    └─────────────────────────┘
```

---

## 5. The Dual-Signature Handshake — Client Perspective

The client must produce **two independent cryptographic proofs** that reference the
same `bindingId` (UUID v4) and `timestamp`. This is the binding glue.

### Proof A — Nostr Claim (Schnorr signature)

Before any hashing, `buildAndSignBindingEvent` runs two local guards:

- **npub/nsec consistency:** `getPublicKey(nsec)` is derived and compared to the
  provided `npub`. A mismatch throws immediately, preventing a valid-looking Schnorr
  signature that the server would reject as `nostr_pubkey_mismatch`.
- **evmAddress format:** The address is tested against `/^0x[0-9a-fA-F]{40}$/`. A
  malformed address throws immediately, avoiding an `evm_address_invalid` server
  rejection and an unnecessary ERC-1271 RPC call.

The client then builds a Kind 30078 Nostr event whose JSON `content` contains:

```jsonc
{
  "evmAddress":  "0xAbCd...",   // the Smart Account
  "timestamp":   1714567890,   // unix seconds
  "bindingId":   "uuid-v4",    // same UUID as below
  "version":     "1.0",
  "appId":       "bliever-v1"
}
```

The event also carries three required tags:
- `["d", "identity-binding"]` — namespaces it within Kind 30078
- `["binding", "<bindingId>"]` — relay-queryable index
- `["evm", "<evmAddress>"]` — relay-queryable index

`nostr-tools finalizeEvent` then computes:
1. `id = SHA-256(canonical serialisation)` — event integrity hash
2. `sig = Schnorr(id, nsec)` — secp256k1 / BIP-340 signature

### Proof B — EVM Consent (ERC-1271 signature)

The client builds this exact plain-text message:

```
bliever-v1 Identity Binding v1.0

Nostr pubkey: <npub>
Binding ID:   <uuid>
Timestamp:    <unix>
Chain ID:     84532

By signing, I confirm this Smart Account consents to be linked
with the above Nostr identity.
```

The `signMessage` function (injected from the CDP SDK) sends this to the Smart
Account's ERC-1271 signing path, which prompts the user for biometric / passkey
confirmation and returns a `0x`-prefixed EIP-191 signature.

### Why Both Must Reference the Same bindingId

Without `bindingId` in both proofs, an attacker could splice an old Nostr event with
a new EVM signature (or vice versa). The server cross-checks that both signatures
reference an identical UUID before accepting the binding.

---

## 6. Key Security & Privacy Model

### What the Client Produces vs. What Leaves the Device

| Data | Produced by | Leaves device? | How |
|---|---|---|---|
| `nsec` (raw 32 bytes) | `generateSecretKey()` | **Never** | Encrypted in IDB |
| `npub` (hex public key) | Derived from nsec | Yes | Submitted to server |
| Nostr Schnorr signature | `finalizeEvent(nsec)` | Yes | In binding payload |
| EVM consent signature | CDP Smart Account | Yes | In binding payload |
| AES key (PRF strategy) | Authenticator PRF | **Never** | Exists in RAM only, not stored |
| AES key (WebCrypto) | `generateKey()` | **Never** (non-extractable) | Stored as CryptoKey in IDB |

### Threat Mitigations

| Threat | Client-side mitigation |
|---|---|
| **XSS steals nsec** | nsec never in JS-accessible plaintext after key generation frame |
| **IDB profile exfiltrated** | PRF: ciphertext useless without authenticator; WebCrypto: non-extractable key |
| **User cancels CDP prompt** | nsec is persisted BEFORE signing — key is safe even on abort |
| **Replay attack** | 5-minute timestamp window on the server; bindingId is single-use |
| **Clock skew (mobile)** | Timestamp sourced from `Date.now()` — server has 300s tolerance |
| **Key mismatch (npub/nsec)** | `buildAndSignBindingEvent` derives the public key from nsec and compares it to the provided npub before signing — throws locally on mismatch |
| **Corrupted nsec bytes** | `npubFromNsec` validates `nsec.length === 32` before passing to the cryptographic layer |
| **Malformed evmAddress** | `buildAndSignBindingEvent` tests the address format before building the event |
| **Relay failure** | Best-effort publish; server confirmation is the source of truth |

---

## 7. nsec Storage — The Two Strategies

### Strategy 1: PRF (WebAuthn PRF Extension) — Preferred

```
Device Authenticator
      │
      │  PRF input: fixed label "bliever-v1:nsec-encryption-key-v1"
      │  PRF output: 32 bytes (deterministic per credential + origin)
      │
      ▼
AES-256-GCM key  ──encrypt──▶  ciphertext
  (RAM only,                        │
   never stored)                    ▼
                              IndexedDB: { ciphertext, iv, credentialId }
```

**Recovery:** The authenticator re-derives the exact same 32 bytes on every
assertion. The AES key is never persisted.

**Security property:** Decryption requires both the IDB ciphertext AND the
authenticator. Exfiltrating the IDB profile alone is not sufficient.

### Strategy 2: WebCrypto AES-GCM — Fallback

```
crypto.subtle.generateKey(AES-GCM-256, extractable=false)
      │
      │  CryptoKey (non-extractable — raw bytes inaccessible from JS)
      │
      ▼
Encrypt nsec ──▶  ciphertext
      │
      ▼
IndexedDB: { ciphertext, iv, cryptoKey (structured clone) }
```

**Security property:** The raw key bytes never appear in JavaScript. However,
the key and ciphertext coexist in the same IDB profile, which is a weaker
boundary than PRF.

**When to use:** When the user's browser or authenticator does not support the
WebAuthn PRF extension. Detect this by catching the error from the PRF attempt.

---

## 8. Data Flow Diagrams

### New User — runOnboarding

```
Consumer
   │
   │── evmAddress, signMessage ──▶ runOnboarding()
   │                                      │
   │                               ① generateNostrKeypair()
   │                               │   nsec (32 bytes) + npub (hex)
   │                               │
   │                               ② persistNsec()
   │                               │   strategy: prf | webcrypto
   │                               │   ──▶ IndexedDB
   │                               │   ◀── storageHandle
   │                               │
   │                               ③ generateBindingId() + timestamp
   │                               │
   │                               ④ buildAndSignBindingEvent()
   │                               │   Kind 30078 + Schnorr sig
   │                               │
   │                               ⑤ buildEVMConsentMessage()
   │                               │   + signMessage()  [CDP prompt]
   │                               │   ◀── evmSignature (0x…)
   │                               │
   │                               ⑥ submitOnboarding()
   │                               │   POST /api/onboarding
   │                               │   ──▶ { npub, evmAddress, nostrEvent,
   │                               │         evmSignature, bindingId, timestamp }
   │                               │   ◀── { success, bindingId, verifiedAt }
   │                               │
   │                               ⑦ publishEventToRelays()  [best-effort]
   │                               │   Kind 30078 ──▶ wss://relay1, wss://relay2
   │                               │
   │◀── OnboardingResult ──────────┘
   │    { npub, evmAddress, bindingId, verifiedAt, nostrEvent, storageHandle }
```

### Returning User — Session Recovery

```
Consumer
   │
   │── handle ──▶ recoverNsec()
   │               │
   │               ├── IDB: load encrypted bundle
   │               ├── PRF: assert credential → re-derive AES key → decrypt
   │               │   OR
   │               └── WebCrypto: load CryptoKey from IDB → decrypt
   │◀── nsec (Uint8Array[32])
   │
   │  Use nsec with nostr-tools for signing social events.
   │  Zeroize or discard the Uint8Array when done.
```

### Re-link — runBindIdentity

```
Consumer
   │
   │── npub, nsec, new evmAddress, signMessage ──▶ runBindIdentity()
   │                                                     │
   │                                              ① verify npubFromNsec(nsec) === npub
   │                                              ② generateBindingId() + timestamp
   │                                              ③ buildAndSignBindingEvent()
   │                                              ④ signMessage() → evmSignature
   │                                              ⑤ submitBindIdentity()
   │                                                 POST /api/bind-identity
   │                                              ⑥ publishEventToRelays()
   │◀── BindIdentityResult
```

---

## 9. Module Responsibility Map

```
lib/
├── binding/
│   ├── schema.ts        Constants + all types. Import from here; never hardcode.
│   └── message.ts       Deterministic EVM consent message builder.
│                        ⚠️  MUST be byte-identical to the server copy.
│
├── nostr/
│   ├── keys.ts          Keypair generation (generateNostrKeypair).
│   │                    Binary encoding utils (toBase64, fromBase64Url, …).
│   ├── event.ts         Kind 30078 event builder + finalizeEvent signing.
│   └── relay.ts         Best-effort parallel event publication to relays.
│
├── crypto/
│   └── nsec-storage.ts  Encrypted nsec persistence to IndexedDB.
│                        PRF (WebAuthn) + WebCrypto AES-GCM strategies.
│
├── api/
│   └── client.ts        Typed fetch wrappers for all three backend endpoints.
│                        OnboardingApiError carries machine-readable `reason`.
│
└── onboarding/
    └── flow.ts          Top-level orchestrator. The ONLY file consumers import.
                         Exports: runOnboarding, runBindIdentity, recoverNsec,
                                  hasStoredNsec, clearStoredNsec, verifyIdentity.
```

**Dependency rule:** Every arrow in the graph below points strictly downward.
No module imports from a module above it. `flow.ts` is the only file that
knows about all other modules.

```
flow.ts
  ├── nostr/keys.ts
  ├── nostr/event.ts
  ├── nostr/relay.ts
  ├── crypto/nsec-storage.ts
  │     └── nostr/keys.ts     (encoding utils)
  ├── binding/message.ts
  │     └── binding/schema.ts
  └── api/client.ts
        └── binding/schema.ts
```

---

## 10. Design Decisions & Trade-offs

### Decision: nsec Persisted Before Signing

In `runOnboarding`, the nsec is encrypted and written to IndexedDB **before**
either signature prompt is shown to the user.

**Why:** If the user dismisses the CDP wallet prompt, closes the tab, or the
browser crashes mid-flow, the Nostr key is already safe in storage. The user
can be guided to retry the signature steps without losing their identity.

**Trade-off:** A user who completes step 2 (IDB write) but abandons the flow
will have an orphaned nsec in storage. `hasStoredNsec()` lets the application
detect this state. Two recovery paths are available:

- **Resume:** call `recoverNsec(handle)` to retrieve the existing nsec, then
  pass it to `runBindIdentity` to complete the signing steps with the same
  Nostr identity — preserving the user's npub.
- **Reset:** call `clearStoredNsec()` before calling `runOnboarding` again to
  generate a fresh identity from scratch.

The application decides which path to offer based on product requirements.

### Decision: signMessage is Injected, Not Imported

The flow module accepts `signMessage: (msg: string) => Promise<HexString>` as
a parameter rather than importing a specific CDP hook or SDK function.

**Why:** This makes the module framework-agnostic and independently testable.
A React consumer passes `useSignMessage()` from `@coinbase/cdp-hooks`. A test
passes a mock that returns a fixed hex string. Neither change requires modifying
`flow.ts`.

### Decision: Relay Publication is Best-Effort with Transient-Only Retry

After the server confirms the binding, `publishEventToRelays` is called but its
result does not gate `OnboardingResult`. A single timeout budget covers the
entire attempt per relay — both the connect and the publish — so a relay that
accepts the connection but then goes silent cannot hang its slot. Each relay is
retried up to 2 times with linear backoff, but **only when the error is
transient** (timeout, network drop, WebSocket close). Permanent relay rejections
such as `BAD EVENT`, `blocked`, or `duplicate` fail immediately without retrying
— they cannot succeed on re-attempt and would waste the backoff delay. A
zero-success result logs a warning but does not throw.

**Why:** The source of truth for the binding is the server's confirmation (and
the indexer's storage of the response). Relays are the discovery layer; their
availability should not be a hard dependency of onboarding. The retry layer absorbs
transient failures — common on mobile connections — without exposing retry complexity
to the consumer. Skipping retries on permanent errors avoids unnecessary UX delay
when a relay has definitively rejected the event.

**Trade-off:** A user whose event reaches zero relays after all retries will be
undiscoverable via relay queries until re-publication is attempted. The `nostrEvent`
field in `OnboardingResult` enables the consumer to re-publish at any time using
`publishEventToRelays(result.nostrEvent, relayUrls)`.

### Decision: bindingId Validated Before Signing

The orchestrator generates the `bindingId` once and passes it to both the Nostr
event builder and the EVM consent message builder. If either building step fails,
the same bindingId is not reused for a retry — `generateBindingId()` is called
again, producing a fresh UUID.

**Why:** Reusing a bindingId across multiple submissions (even failed ones) could
allow an attacker to observe the UUID and attempt to front-run the binding.
Generating a fresh UUID per attempt keeps each attempt atomic.

### Decision: npub/nsec Consistency Check in runBindIdentity

`runBindIdentity` derives `npubFromNsec(nsec)` and compares it to the provided
`npub` before doing any signing or network work.

**Why:** A mismatch between the provided nsec and npub would produce a valid-looking
Schnorr signature that the server would reject at step 3 (`nostr_pubkey_mismatch`),
wasting the ERC-1271 RPC call. The local guard makes the failure immediate, cheap,
and clear.

---

## 11. Re-link & Key Rotation

Kind 30078 is a Nostr **addressable (replaceable) event**. Its `d` tag
(`"identity-binding"`) is the address key. Publishing a new Kind 30078 event with
the same `d` tag replaces the previous one on relays.

### Key Rotation Scenario

```
User rotates Nostr key:
  1. Generate new nsec/npub (outside this module or via a fresh runOnboarding).
  2. Recover existing nsec via recoverNsec() if needed for the old npub's final posts.
  3. Call runBindIdentity({ npub: newNpub, nsec: newNsec, evmAddress: same }).
  4. The server verifies the new proof. The indexer updates the npub ↔ evmAddress mapping.
  5. The new Kind 30078 event replaces the old one on relays.
```

### Wallet Migration Scenario

```
User migrates to a new Smart Account:
  1. Recover nsec from storage.
  2. Call runBindIdentity({ npub: same, nsec: same, evmAddress: newAddress }).
  3. The new EVM consent signature is from the new Smart Account.
  4. The server verifies. Indexer updates evmAddress for this npub.
```

---

## 12. Environment Configuration

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `NEXT_PUBLIC_BASE_CHAIN_ID` | No | `"84532"` | `"8453"` for mainnet, `"84532"` for Sepolia. Must match server. |
| `NEXT_PUBLIC_API_BASE_URL` | No | `""` (same-origin) | Override when API lives on a different origin. |
| `NEXT_PUBLIC_CDP_PROJECT_ID` | Yes | — | CDP project identifier (used by CDP SDK, not this module directly). |
| `NEXT_PUBLIC_DEFAULT_RELAYS` | No | — | Comma-separated relay WSS URLs. Consumer passes these to `relayUrls`. |

> **Sync requirement:** `NEXT_PUBLIC_BASE_CHAIN_ID` must be identical on client and
> server. The value is embedded in the EVM consent message by `buildEVMConsentMessage`.
> A mismatch causes `invalid_evm_signature` on every submission because the message
> the client signed differs from the message the server reconstructs.