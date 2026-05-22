# Client Onboarding — Developer Implementation Reference

> **Audience** Developers who want to analyse, extend, debug, write integration tests,
> or build on top of the client onboarding module.

---

## Table of Contents

1. [Repository Placement](#1-repository-placement)
2. [Dependencies](#2-dependencies)
3. [Module Breakdown](#3-module-breakdown)
   - [lib/binding/schema.ts](#31-libbindingschema-ts)
   - [lib/binding/message.ts](#32-libbindingmessage-ts)
   - [lib/nostr/keys.ts](#33-libnostrkeys-ts)
   - [lib/nostr/event.ts](#34-libnostrevents-ts)
   - [lib/nostr/relay.ts](#35-libnostrrelay-ts)
   - [lib/crypto/nsec-storage.ts](#36-libcryptonsec-storage-ts)
   - [lib/api/client.ts](#37-libapiclient-ts)
   - [lib/onboarding/flow.ts](#38-libonboardingflow-ts)
4. [Public API Reference](#4-public-api-reference)
5. [Call Sequence — Step by Step](#5-call-sequence--step-by-step)
6. [Error Handling Reference](#6-error-handling-reference)
7. [Extending the System](#7-extending-the-system)
8. [Integration Contract (for App Consumers)](#8-integration-contract-for-app-consumers)
9. [Common Debugging Scenarios](#9-common-debugging-scenarios)

---

## 1. Repository Placement

These files map directly onto the repository structure visible in the project:

```
lib/
├── binding/
│   ├── schema.ts          ← CLIENT copy — mirrors server lib/binding/schema.ts
│   └── message.ts         ← CLIENT copy — mirrors server lib/binding/message.ts
├── nostr/
│   ├── keys.ts            ← NEW (client-only)
│   ├── event.ts           ← NEW (client-only)
│   └── relay.ts           ← NEW (client-only)
├── crypto/
│   └── nsec-storage.ts    ← NEW (client-only)
├── api/
│   └── client.ts          ← NEW (client-only)
└── onboarding/
    └── flow.ts            ← NEW — sole public entry point
```

Server-side modules (`lib/evm/`, server-side `lib/nostr/verification.ts`) are
**not** imported by any client file. The dependency boundary is strict.

---

## 2. Dependencies

### Runtime (browser / Next.js client bundle)

| Package | Used In | Purpose |
|---|---|---|
| `nostr-tools` | `nostr/keys.ts`, `nostr/event.ts`, `nostr/relay.ts` | `generateSecretKey`, `getPublicKey`, `finalizeEvent`, `Relay` |
| (none beyond nostr-tools) | `crypto/nsec-storage.ts` | Uses native Web Crypto + IndexedDB APIs |
| (none) | `api/client.ts` | Uses native `fetch` |

> **Zero external crypto dependencies beyond nostr-tools.** All AES-256-GCM and
> PRF operations use `window.crypto.subtle` (built into every modern browser and
> the Node.js `crypto` global from v19).

> **Client-only constraint.** `flow.ts`, `nsec-storage.ts`, and `relay.ts` all carry a
> `"use client"` directive. In Next.js, this prevents the bundler from including them
> in any server component or server action. In non-Next.js apps, never `import` these
> modules inside a file that runs on Node/SSR — use dynamic import (`const mod = await import("…")`)
> inside a browser-only code path instead.

### Sub-path import note

```ts
// ✅ Correct — full nostr-tools import is fine on the client
import { generateSecretKey, getPublicKey, finalizeEvent } from "nostr-tools";
import { Relay } from "nostr-tools";

// ✅ Also correct — for server-only code
import { verifyEvent } from "nostr-tools/pure";  // no DOM deps
```

---

## 3. Module Breakdown

### 3.1 `lib/binding/schema.ts`

**Role:** Single source of truth for all constants, protocol types, and client-only
types. All other modules import from here; constants are never inlined.

**Key exports:**

```ts
// Protocol constants [SERVER-MIRROR — must stay byte-identical]
ONBOARDING_VERSION     = "1.0"
NOSTR_BINDING_KIND     = 30078
NOSTR_BINDING_D_TAG    = "identity-binding"
APP_ID                 = "bliever-v1"
BASE_CHAIN_ID          = process.env.NEXT_PUBLIC_BASE_CHAIN_ID ?? "84532"

// Shared structural types [SERVER-MIRROR]
interface NostrBindingEvent    // Nostr event shape (matches finalizeEvent output)
interface NostrBindingClaim    // JSON embedded in event.content
type HexString                 // `0x${string}`

// Client-only types
interface NostrKeypair         // { nsec: Uint8Array, npub: string }
type NsecStorageStrategy       // "prf" | "webcrypto"
interface NsecStorageHandle    // Opaque reference to IDB record
type SignMessageFn             // (message: string) => Promise<HexString>

// Flow parameter types
interface OnboardingParams     // Input to runOnboarding()
interface OnboardingResult     // Output of runOnboarding()
interface BindIdentityParams   // Input to runBindIdentity()
interface BindIdentityResult   // Output of runBindIdentity()
```

**Sync requirement:** Any constant tagged `[SERVER-MIRROR]` must be updated in
both `lib/binding/schema.ts` files simultaneously. A version bump requires updating
`ONBOARDING_VERSION` in both copies plus `buildEVMConsentMessage` in both
`message.ts` copies.

---

### 3.2 `lib/binding/message.ts`

**Role:** Builds the plain-text EVM consent message that the CDP Smart Account signs.

```ts
function buildEVMConsentMessage(
  npub: string,
  bindingId: string,
  timestamp: number,
): string
```

**⚠️ IMMUTABILITY CONTRACT**

This file is a byte-for-byte mirror of the server's `lib/binding/message.ts`.
The server calls the same function to reconstruct the message for ERC-1271
verification. Any difference (whitespace, newlines, field order) causes
`invalid_evm_signature` on every submission.

The implementation uses `[...].join("\n")` — **not** a template literal with
indentation — to give deterministic control over every character.

Output format (exact):

```
bliever-v1 Identity Binding v1.0

Nostr pubkey: <npub>
Binding ID:   <bindingId>
Timestamp:    <timestamp>
Chain ID:     <BASE_CHAIN_ID>

By signing, I confirm this Smart Account consents to be linked
with the above Nostr identity.
```

Note the exact spacing: `Nostr pubkey: ` (one space after colon), `Binding ID:   `
(three spaces), `Timestamp:    ` (four spaces), `Chain ID:     ` (five spaces).
This alignment is intentional and must not change.

---

### 3.3 `lib/nostr/keys.ts`

**Role:** Keypair generation + binary encoding utilities.

```ts
// Keypair generation
function generateNostrKeypair(): NostrKeypair
// Returns { nsec: Uint8Array[32], npub: string (64 hex chars) }
// Entropy: crypto.getRandomValues via nostr-tools generateSecretKey()

function npubFromNsec(nsec: Uint8Array): string
// Derives the hex public key from a raw secret key.
// Used after recovering nsec from storage.

// Binary encoding (used by nsec-storage.ts)
function toBase64(bytes: Uint8Array): string
function fromBase64(str: string): Uint8Array
function toBase64Url(bytes: Uint8Array): string   // URL-safe, no padding
function fromBase64Url(str: string): Uint8Array
```

**Implementation note on toBase64:** Uses chunked `String.fromCharCode` instead
of spreading the full array, preventing a stack overflow on large buffers.

---

### 3.4 `lib/nostr/event.ts`

**Role:** Builds and cryptographically finalises the Kind 30078 binding event.

```ts
interface BuildBindingEventParams {
  npub: string;        // for intent documentation; finalizeEvent derives it from nsec
  evmAddress: string;  // EIP-55 checksummed
  bindingId: string;   // UUID v4
  timestamp: number;   // unix seconds
  nsec: Uint8Array;    // 32-byte secret key — used by finalizeEvent
}

function buildAndSignBindingEvent(params: BuildBindingEventParams): NostrBindingEvent
```

**What `finalizeEvent` does internally:**

```
1. template.pubkey = getPublicKey(nsec)
2. Serialize: JSON.stringify([0, pubkey, created_at, kind, tags, content])
3. id  = SHA-256(serialised bytes)
4. sig = schnorrSign(id, nsec)   ← BIP-340 Schnorr
```

**Critical ordering:** `JSON.stringify(claim)` is called BEFORE `finalizeEvent`.
The content string is hashed as-is. Stringifying after finalization would produce
a different hash and an invalid event id.

**Tag structure produced:**

```ts
tags: [
  ["d",       "identity-binding"],  // namespace tag
  ["binding", bindingId],           // relay-queryable index
  ["evm",     evmAddress],          // relay-queryable index
]
```

The server's `checkNostrEventStructure` validates the presence of all three tags.

---

### 3.5 `lib/nostr/relay.ts`

**Role:** Best-effort parallel publication of the signed event to Nostr relays.

```ts
interface RelayPublishOutcome {
  url: string;
  success: boolean;
  error?: string;       // reason if success === false (final attempt)
}

interface PublishResult {
  outcomes: RelayPublishOutcome[];
  atLeastOneSuccess: boolean;
}

async function publishEventToRelays(
  event: NostrBindingEvent,
  relayUrls: string[],
  timeoutMs?: number,    // default 8_000 ms per attempt per relay
  maxRetries?: number,   // default 2 — additional attempts after initial failure
): Promise<PublishResult>
```

**Implementation details:**

- Each relay gets its own `Promise` via `Relay.connect(url)` from nostr-tools.
- A `setTimeout`-backed race enforces the per-relay timeout per attempt independently.
- On failure, each relay is retried up to `maxRetries` times with linear backoff (1 s × attempt number). Only the error from the final failed attempt is recorded in `outcomes`.
- `Promise.allSettled` collects every relay's final outcome (no early exit on failure).
- Each `Relay` connection is closed in a `finally` block — no persistent sockets.
- The function never throws; all errors are captured in `outcomes[n].error`.

**Caller contract:** `flow.ts` calls this after the backend confirms the binding.
A `PublishResult` where `atLeastOneSuccess === false` logs a warning but does NOT
cause `runOnboarding` to reject. The consumer can re-publish using `result.nostrEvent`.

---

### 3.6 `lib/crypto/nsec-storage.ts`

**Role:** AES-256-GCM encrypted persistence of the nsec in IndexedDB.

#### IndexedDB schema

| Property | Value |
|---|---|
| Database name | `"bliever"` |
| Database version | `1` |
| Object store | `"keys"` |
| Record key | `"nsec_bundle"` |
| Record type | `NsecIdbRecord` (internal) |

#### Internal record structure

```ts
interface NsecIdbRecord {
  strategy: "prf" | "webcrypto";
  ciphertext: string;    // base64-encoded AES-GCM ciphertext
  iv: string;            // base64-encoded 12-byte random nonce
  credentialId?: string; // PRF only: base64url WebAuthn rawId
  cryptoKey?: CryptoKey; // WebCrypto only: non-extractable, stored as structured clone
}
```

#### PRF Strategy — Step by Step

**`persistNsec` (PRF):**

```
1. navigator.credentials.create() with prf extension
   → returns PublicKeyCredential
2. Extract prf.results.first (ArrayBuffer, 32 bytes)
3. crypto.subtle.importKey("raw", prfOutput, "AES-GCM", false, ["encrypt","decrypt"])
   → CryptoKey (non-extractable, exists only in this call frame)
4. AES-GCM encrypt(nsec) → { ciphertext, iv }
5. IDB.put({ strategy:"prf", ciphertext, iv, credentialId })
```

**`recoverNsec` (PRF):**

```
1. IDB.get("nsec_bundle") → record
2. navigator.credentials.get() with prf extension, allowCredentials: [record.credentialId]
   → triggers biometric / PIN prompt
3. Extract prf.results.first (same 32 bytes as step 2 of persist)
4. crypto.subtle.importKey(…) → same AES key
5. AES-GCM decrypt(record.ciphertext, record.iv) → nsec (Uint8Array)
```

**PRF salt:** `"bliever-v1:nsec-encryption-key-v1"` (UTF-8 encoded).
This label is fixed. Changing it invalidates all existing PRF-encrypted bundles
because the authenticator produces a different PRF output for a different input.

#### WebCrypto Strategy — Step by Step

**`persistNsec` (WebCrypto):**

```
1. crypto.subtle.generateKey(AES-GCM-256, extractable=false, ["encrypt","decrypt"])
   → CryptoKey (raw bytes inaccessible from JS)
2. AES-GCM encrypt(nsec) → { ciphertext, iv }
3. IDB.put({ strategy:"webcrypto", ciphertext, iv, cryptoKey })
   ← CryptoKey stored as structured clone (IndexedDB native support)
```

**`recoverNsec` (WebCrypto):**

```
1. IDB.get("nsec_bundle") → record (contains CryptoKey via structured clone)
2. AES-GCM decrypt(record.ciphertext, record.iv, record.cryptoKey) → nsec
```

#### Public API

```ts
// Store nsec
async function persistNsec(
  nsec: Uint8Array,
  options: PersistNsecOptions,
): Promise<NsecStorageHandle>

// Recover nsec (triggers biometric prompt if PRF strategy)
async function recoverNsec(
  handle: NsecStorageHandle,
  options?: RecoverNsecOptions,
): Promise<Uint8Array>

// Check without decrypting — safe, no biometric prompt
async function hasStoredNsec(): Promise<boolean>

// Permanent removal — irreversible
async function clearStoredNsec(): Promise<void>
```

---

### 3.7 `lib/api/client.ts`

**Role:** Typed HTTP wrappers for all three backend endpoints.

```ts
// POST /api/onboarding
async function submitOnboarding(payload: BindingPayload): Promise<OnboardingSuccessResponse>

// POST /api/bind-identity
async function submitBindIdentity(payload: BindingPayload): Promise<BindingSuccessResponse>

// POST /api/verify-identity
async function verifyIdentity(payload: VerifyPayload): Promise<VerifySuccessResponse>
```

All three functions:
1. Call `fetch` with `Content-Type: application/json`.
2. On `!response.ok`: throw `OnboardingApiError(reason, status)`.
3. On network failure (fetch throws): throw `OnboardingApiError("internal_error", 0)`.

#### OnboardingApiError

```ts
class OnboardingApiError extends Error {
  reason: BindingErrorReason | "unknown_error";  // machine-readable server code
  status: number;                                 // HTTP status (0 = network failure)
}
```

**Base URL resolution:**

```ts
const BASE_URL = process.env.NEXT_PUBLIC_API_BASE_URL ?? "";
// Empty string = same-origin relative URL (correct for standard Next.js deploy).
// Override for cross-origin deployments.
```

---

### 3.8 `lib/onboarding/flow.ts`

**Role:** Top-level orchestrator. The only file consumers import directly.

#### `runOnboarding(params: OnboardingParams): Promise<OnboardingResult>`

Full execution sequence:

| Step | Code | Network? | Notes |
|---|---|---|---|
| 1 | `generateNostrKeypair()` | No | Synchronous; entropy from `getRandomValues` |
| 2 | `persistNsec(nsec, { strategy })` | No (PRF: biometric) | Must succeed before any signing |
| 3 | `generateBindingId()` + `timestamp` | No | UUID v4 + `Date.now() / 1000` |
| 4 | `buildAndSignBindingEvent(…)` | No | Schnorr signature via nostr-tools |
| 5 | `buildEVMConsentMessage(…)` | No | Deterministic string builder |
| 6 | `signMessage(consentMessage)` | CDP prompt | Returns `0x…` EIP-191 sig |
| 7 | `submitOnboarding(payload)` | Yes | POST /api/onboarding + ERC-1271 RPC |
| 8 | `publishEventToRelays(…)` | Yes (best-effort) | Per-relay retry with linear backoff; failure logs warning, does not throw |

**nsec persistence before signing (step 2 before steps 4–6):**

```ts
// Persist first — key is safe even if user cancels CDP prompt
storageHandle = await persistNsec(nsec, { strategy, rpId });

// Only then sign
const nostrEvent = buildAndSignBindingEvent({ nsec, … });
const evmSignature = await signMessage(consentMessage);
```

If `persistNsec` throws (e.g. PRF not supported, IDB quota exceeded), the
function throws immediately with a descriptive message and no signing occurs.

#### `runBindIdentity(params: BindIdentityParams): Promise<BindIdentityResult>`

Differences from `runOnboarding`:
- No keypair generation (caller provides `nsec`).
- No nsec storage (caller manages it).
- Adds `npubFromNsec(nsec) === npub` guard before any I/O.
- Calls `submitBindIdentity` → `POST /api/bind-identity`.
- Relay publish is fire-and-forget (`.catch()`, not `await`).

#### `generateBindingId()` (internal)

```ts
// Uses crypto.randomUUID() directly — available on all supported targets:
// Chrome 92+, Firefox 95+, Safari 15.4+, Node 19+
// These are the effective minimums because IndexedDB + WebAuthn are required anyway.
function generateBindingId(): string {
  return crypto.randomUUID();
}
```

#### `buildDualSignatureProof(…)` (internal shared helper)

```ts
// Step A: Nostr signature
const nostrEvent = buildAndSignBindingEvent({ npub, evmAddress, bindingId, timestamp, nsec });

// Step B: EVM signature
const consentMessage = buildEVMConsentMessage(npub, bindingId, timestamp);
const evmSignature = await signMessage(consentMessage);

return { nostrEvent, evmSignature };
```

Both steps reference the **same** `bindingId` and `timestamp`. This shared reference
is the binding glue that the server's step 8 (`claim.bindingId === bindingId`) and
the EVM message reconstruction validate.

---

## 4. Public API Reference

```ts
// ── Primary entry points ───────────────────────────────────────────────────
import {
  runOnboarding,       // New user: generate key, sign, submit, publish
  runBindIdentity,     // Re-link: sign with existing key, submit, publish
} from "lib/onboarding/flow";

// ── Session management ─────────────────────────────────────────────────────
import {
  recoverNsec,         // Decrypt nsec from IDB (may trigger biometric)
  hasStoredNsec,       // Check if IDB bundle exists (no biometric)
  clearStoredNsec,     // Permanently delete IDB bundle
} from "lib/onboarding/flow";

// ── Read-only verification ─────────────────────────────────────────────────
import {
  verifyIdentity,      // POST /api/verify-identity — Nostr-only proof check
} from "lib/onboarding/flow";

// ── Key utility ────────────────────────────────────────────────────────────
import {
  npubFromNsec,        // Derive npub from recovered nsec
} from "lib/onboarding/flow";

// ── Error class ────────────────────────────────────────────────────────────
import {
  OnboardingApiError,  // { reason: BindingErrorReason, status: number }
} from "lib/onboarding/flow";
```

---

## 5. Call Sequence — Step by Step

### First-time Onboarding (consumer pseudocode)

```ts
// Consumer receives these from CDP SDK after user authenticates
const evmAddress: string = cdpAccount.address;
const signMessage: SignMessageFn = async (msg) => await cdpSignMessage(msg);

try {
  const result = await runOnboarding({
    evmAddress,
    signMessage,
    relayUrls: ["wss://relay.damus.io", "wss://nos.lol"],
    storageStrategy: "prf",           // try PRF first
    prfRpId: "app.bliever.io",
  });

  // Store the handle in app state / lightweight localStorage
  // (the handle is NOT the key — it is safe to store)
  localStorage.setItem("nsec_handle", JSON.stringify(result.storageHandle));

  // result.npub — use to build the user's Nostr profile
  // result.evmAddress — EIP-55 checksummed, from server
  // result.nostrEvent — already published to relays
} catch (err) {
  if (err instanceof OnboardingApiError) {
    switch (err.reason) {
      case "timestamp_expired":
        // Regenerate payload — clock skew; retry immediately
        break;
      case "rate_limited":
        // Back off 60 seconds
        break;
      default:
        // Show generic error
    }
  } else {
    // nsec persist failed, network down, etc.
  }
}
```

### Returning User — Session Recovery

```ts
const handle: NsecStorageHandle = JSON.parse(
  localStorage.getItem("nsec_handle") ?? "null"
);

if (handle && await hasStoredNsec()) {
  // Triggers biometric if strategy === "prf"
  const nsec = await recoverNsec(handle, { rpId: "app.bliever.io" });
  const npub = npubFromNsec(nsec);

  // Use nsec for signing social events via nostr-tools
  // Zeroize when done: nsec.fill(0)
}
```

### Profile Verification Before Enabling Tip Button

```ts
const result = await verifyIdentity({
  npub: aliceNpub,
  evmAddress: aliceEvmAddress,
  nostrEvent: aliceKind30078Event, // fetched from relays
});

if (result.valid) {
  enableTipButton(result.evmAddress);
}
```

---

## 6. Error Handling Reference

### Errors thrown by `runOnboarding` / `runBindIdentity`

| Error type | When | `reason` / message |
|---|---|---|
| `Error` | `persistNsec` fails (IDB error, PRF unsupported) | Descriptive message, no `reason` |
| `Error` | nsec/npub mismatch (runBindIdentity only) | Descriptive message |
| `OnboardingApiError` | Server returns 400 | See table below |
| `OnboardingApiError` | Server returns 429 | `"rate_limited"`, `status: 429` |
| `OnboardingApiError` | Server returns 500 | `"internal_error"`, `status: 500` |
| `OnboardingApiError` | Network failure | `"internal_error"`, `status: 0` |

### Server reason codes (from `BindingErrorReason`)

| Reason | Cause | Client action |
|---|---|---|
| `invalid_payload` | Missing field in submitted payload | Fix payload construction |
| `timestamp_expired` | `timestamp` is > 5 min old | Regenerate: new `bindingId` + `timestamp` |
| `wrong_event_kind` | `event.kind !== 30078` | Check `buildAndSignBindingEvent` |
| `wrong_d_tag` | `d` tag ≠ `"identity-binding"` | Check event tags |
| `missing_binding_tag` | No `["binding", …]` tag | Check event tags |
| `missing_evm_tag` | No `["evm", …]` tag | Check event tags |
| `nostr_pubkey_mismatch` | `event.pubkey !== npub` | Verify nsec/npub correspondence |
| `invalid_binding_id` | `bindingId` is not UUID v4 | Check `generateBindingId()` output |
| `invalid_nostr_signature` | Schnorr sig invalid | Verify `finalizeEvent` was used |
| `invalid_content_json` | `event.content` not valid JSON | Verify `JSON.stringify(claim)` |
| `invalid_claim_fields` | Missing field or wrong version | Check `NostrBindingClaim` construction |
| `evm_address_invalid` | Not a valid Ethereum address | Use EIP-55 checksummed address |
| `evm_address_mismatch` | Claim address ≠ payload address | Use same address in both |
| `binding_id_mismatch` | `claim.bindingId !== bindingId` | Use same UUID in both |
| `invalid_evm_signature` | ERC-1271 returned false | See debugging section |
| `rate_limited` | > 10 req/min from this IP | Back off 60 s |
| `internal_error` | Unhandled server exception | Retry with exponential backoff |

---

## 7. Extending the System

### Adding a new nsec storage strategy

1. Add the new literal to `NsecStorageStrategy` in `schema.ts`:
   ```ts
   export type NsecStorageStrategy = "prf" | "webcrypto" | "your_strategy";
   ```
2. Add a `NsecIdbRecord` field for any new key material.
3. Implement `yourStrategyCreateAndEncrypt` and `yourStrategyDecrypt` in
   `nsec-storage.ts`.
4. Add a branch to the `persistNsec` / `recoverNsec` switch.

### Adding a new relay publication strategy

Currently `relay.ts` uses `Relay.connect()` + `relay.publish()` from nostr-tools.
To support authenticated relays (NIP-42) or pool-based connections:

1. Create `lib/nostr/relay-pool.ts` implementing the same `PublishResult` interface.
2. Pass a `publishFn` parameter into `flow.ts` to swap implementations.

### Supporting a new consent message version

1. Bump `ONBOARDING_VERSION` in `lib/binding/schema.ts` (both client AND server copies).
2. Update `buildEVMConsentMessage` in `lib/binding/message.ts` (both copies).
3. Update the server validator to accept the old version during the transition window.
4. Update the `NostrBindingClaim.version` type narrowing in `schema.ts`.

### Adding pre-flight timestamp correction (mobile clock skew)

If `timestamp_expired` errors are frequent on mobile devices, add a server-time
endpoint and correct the local clock before generating the payload:

```ts
// In flow.ts, before step 3:
const { now } = await fetch("/api/time").then(r => r.json());
const timestamp = now; // server unix seconds, not local Date.now()
```

---

## 8. Integration Contract (for App Consumers)

### What the consumer MUST provide

| Input | Source | Type |
|---|---|---|
| `evmAddress` | CDP SDK after auth | `string` (EIP-55 checksummed) |
| `signMessage` | CDP `useSignMessage` hook (wrapped) | `(msg: string) => Promise<HexString>` |

### What the consumer MUST NOT do

- Import from `lib/nostr/keys.ts`, `lib/nostr/event.ts`, or `lib/crypto/nsec-storage.ts`
  directly. Use `flow.ts` exports only.
- Store the raw `nsec` (Uint8Array) in React state, localStorage, or cookies.
- Log the `nsec` or any field of `NostrKeypair`.
- Reuse a `bindingId` across multiple submission attempts.

### What the consumer receives

```ts
interface OnboardingResult {
  npub: string;                // 64-char hex — the user's permanent social identity
  evmAddress: string;          // EIP-55 checksummed — from server response
  bindingId: string;           // UUID v4 — for logging / debugging only
  verifiedAt: number;          // unix seconds — server-side confirmation time
  nostrEvent: NostrBindingEvent; // signed Kind 30078 — re-publish or cache
  storageHandle: NsecStorageHandle; // opaque ref — persist in localStorage
}
```

### signMessage wrapper (example for CDP hooks)

```ts
// In your React component / provider (NOT in flow.ts)
import { useSignMessage } from "@coinbase/cdp-hooks";

function useOnboardingSignMessage(): SignMessageFn {
  const { signMessage } = useSignMessage();
  return async (message: string): Promise<HexString> => {
    const result = await signMessage({ message });
    return result.signature as HexString;
  };
}
```

---

## 9. Common Debugging Scenarios

### `invalid_evm_signature` after correct CDP signing

**Most common cause — message mismatch.**

The server reconstructs the EVM consent message in `validateBindingPayload` step 9
using the same `buildEVMConsentMessage` function. If the client version differs in
any character:

1. Confirm both `lib/binding/message.ts` files are byte-identical.
2. Confirm `BASE_CHAIN_ID` resolves to the same value on client and server.
   - Client: `process.env.NEXT_PUBLIC_BASE_CHAIN_ID` (build-time)
   - Server: `process.env.NEXT_PUBLIC_BASE_CHAIN_ID` (runtime)
   Both must be `"8453"` or `"84532"`.
3. Confirm the `signMessage` wrapper is calling the Smart Account's ERC-1271 path,
   not an EOA/passkey signer. The `evmAddress` must be the Smart Account address.

**Diagnostic:** Log `buildEVMConsentMessage(npub, bindingId, timestamp)` on the
client and compare character-by-character with a server-side debug log.

### `timestamp_expired` immediately

1. Confirm `Math.floor(Date.now() / 1000)` — not milliseconds.
2. Mobile devices with disabled NTP may be > 5 minutes off. Fetch server time:
   ```ts
   const { now } = await fetch("/api/time").then(r => r.json());
   ```
3. Confirm `BINDING_TIMESTAMP_WINDOW_SEC` is `300` on both client and server.

### PRF strategy throws "PRF extension did not return output"

The user's authenticator does not support the WebAuthn PRF extension. Common cases:
- Firefox (PRF support experimental as of 2024).
- Older Android/iOS versions.
- Non-passkey FIDO2 security keys.

**Mitigation:**

```ts
try {
  result = await runOnboarding({ …, storageStrategy: "prf" });
} catch (err) {
  if (err.message.includes("PRF extension")) {
    // Retry with webcrypto fallback
    result = await runOnboarding({ …, storageStrategy: "webcrypto" });
  }
}
```

### `recoverNsec` throws "No nsec bundle found in IndexedDB"

Causes:
1. User cleared browser data / IndexedDB.
2. User is on a different device.
3. User never completed onboarding (nsec stored but CDP prompt was dismissed — see below).
4. The IDB database name or store name was changed between versions.

Check `hasStoredNsec()` before calling `recoverNsec`. If it returns `false`,
route the user to re-onboard or restore from an external backup mechanism.

### Orphaned nsec — user abandoned mid-flow

`runOnboarding` writes the nsec to IndexedDB (step 2) before either signature prompt.
If the user dismisses the CDP wallet prompt or closes the tab after step 2 but before
the server confirms the binding, an nsec bundle exists in IDB that is not registered
on the backend.

**Detection:** Call `hasStoredNsec()` at app startup. If it returns `true` but the
user's backend profile shows no confirmed binding, they are in the orphaned state.

**Recovery options (application decides):**

```ts
// Option A — route to retry: recover the existing nsec and re-run the signing steps
//   via runBindIdentity (the key is already stored, no new keypair needed).
const nsec = await recoverNsec(handle, { rpId: "app.bliever.io" });
const npub = npubFromNsec(nsec);
await runBindIdentity({ npub, nsec, evmAddress, signMessage, relayUrls });

// Option B — start fresh: discard the orphaned key and re-onboard
await clearStoredNsec();
await runOnboarding({ evmAddress, signMessage, relayUrls, storageStrategy });
```

Option A is preferable because the user's Nostr identity (npub) is preserved.

### `invalid_nostr_signature`

1. Verify `buildAndSignBindingEvent` calls `finalizeEvent` from nostr-tools (not a
   custom implementation).
2. Verify `content = JSON.stringify(claim)` is called **before** `finalizeEvent`.
3. Verify the `nsec` passed to `buildAndSignBindingEvent` corresponds to the `npub`
   submitted in the payload. Use `npubFromNsec(nsec) === npub` to check.

### Relay publish fails for all relays

Relay publish failure does NOT fail onboarding. Each relay is retried up to 2 times
(with 1 s and 2 s delays) before being marked as failed in `outcomes`. If every relay
still fails after retries, re-publish manually using `result.nostrEvent`:

```ts
import { publishEventToRelays } from "lib/nostr/relay";

// Default: 2 retries, 8 s timeout per attempt
const publishResult = await publishEventToRelays(result.nostrEvent, relayUrls);

// Or increase retries and timeout for very slow connections:
const publishResult = await publishEventToRelays(result.nostrEvent, relayUrls, 12_000, 4);

console.log(publishResult.outcomes);
```

Common causes: relay offline, WebSocket blocked by corporate firewall, relay requires
NIP-42 authentication. Add more relay URLs for resilience.

### Rate limit (429) during development / automated testing

The server's in-memory limiter caps each source IP at 10 req/min. During dev:

- Slow down test runs or increase `RL_MAX_REQUESTS` in the server `bind-identity/route.ts` temporarily.
- The client's `OnboardingApiError` will have `status: 429` and `reason: "rate_limited"`.
- Wait 60 seconds or restart the server (resets in-memory state).