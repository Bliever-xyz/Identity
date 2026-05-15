# Onboarding — Developer Implementation Reference

> **Audience** Developers who want to analyse, extend, debug, write tests,
> or build an indexer/other service on top of this module.

---

## Table of Contents

1. [Repository Structure](#1-repository-structure)
2. [Dependencies](#2-dependencies)
3. [Module Breakdown](#3-module-breakdown)
   - [lib/binding/schema.ts](#31-libbindingschema-ts)
   - [lib/binding/message.ts](#32-libbindingmessage-ts)
   - [lib/binding/validator.ts](#33-libbindingvalidator-ts)
   - [lib/nostr/verification.ts](#34-libnostrverification-ts)
   - [lib/evm/client.ts](#35-libevmclient-ts)
   - [lib/evm/verification.ts](#36-libevmverification-ts)
4. [API Routes](#4-api-routes)
   - [POST /api/onboarding](#41-post-apionboarding)
   - [POST /api/bind-identity](#42-post-apibind-identity)
   - [POST /api/verify-identity](#43-post-apiverify-identity)
5. [Validation Pipeline — Step by Step](#5-validation-pipeline--step-by-step)
6. [Error Reason Reference](#6-error-reason-reference)
7. [Extending the System](#7-extending-the-system)
8. [Indexer Integration Contract](#8-indexer-integration-contract)
9. [Common Debugging Scenarios](#9-common-debugging-scenarios)

---

## 1. Repository Structure

```
lib/
├── binding/
│   ├── schema.ts       ← All types, constants, response shapes
│   ├── message.ts      ← Deterministic EVM consent message builder
│   └── validator.ts    ← Core validation logic (pure + async)
├── nostr/
│   └── verification.ts ← Nostr Schnorr signature verification
└── evm/
    ├── client.ts       ← Viem public client (Base chain)
    └── verification.ts ← EVM ERC-1271 signature verification

app/api/
├── onboarding/
│   └── route.ts        ← POST /api/onboarding
├── bind-identity/
│   └── route.ts        ← POST /api/bind-identity
└── verify-identity/
    └── route.ts        ← POST /api/verify-identity
```

**Dependency graph (no cycles):**

```
route.ts
  └── validator.ts
        ├── schema.ts
        ├── message.ts
        ├── nostr/verification.ts  ← nostr-tools/pure
        └── evm/verification.ts
              └── evm/client.ts    ← viem
```

---

## 2. Dependencies

### Runtime (server)

| Package | Version | Used For |
|---------|---------|----------|
| `nostr-tools` | ≥ 2.x | `verifyEvent` (Schnorr signature + event id check) |
| `viem` | ≥ 2.x | `verifyMessage` (ERC-1271), `getAddress` (EIP-55) |
| `next` | ≥ 14.x | `NextRequest`, `NextResponse` route handlers |

### Sub-path imports

```ts
// ✅ Correct — no browser/DOM dependencies, safe in Node/Edge
import { verifyEvent } from "nostr-tools/pure";

// ❌ Wrong — pulls in browser-specific code
import { verifyEvent } from "nostr-tools";
```

---

## 3. Module Breakdown

### 3.1 `lib/binding/schema.ts`

Single source of truth. Import from here; never hardcode constants elsewhere.

**Key exports:**

```ts
// Protocol constants
ONBOARDING_VERSION        = "1.0"
NOSTR_BINDING_KIND        = 30078
NOSTR_BINDING_D_TAG       = "identity-binding"
BINDING_TIMESTAMP_WINDOW_SEC = 300
APP_ID                    = "bliever-v1"
BASE_CHAIN_ID             = process.env.NEXT_PUBLIC_BASE_CHAIN_ID ?? "84532"

// Core payload types
interface BindIdentityPayload   // → /api/bind-identity and /api/onboarding
interface VerifyIdentityPayload // → /api/verify-identity
interface NostrBindingEvent     // Nostr event structure
interface NostrBindingClaim     // JSON inside event.content

// Response types
type BindingResponse           // BindingSuccessResponse | BindingErrorResponse
type OnboardingResponse        // OnboardingSuccessResponse | BindingErrorResponse
type VerifyResponse            // VerifySuccessResponse | VerifyErrorResponse

// Error codes
type BindingErrorReason        // exhaustive union — see section 6
```

---

### 3.2 `lib/binding/message.ts`

```ts
function buildEVMConsentMessage(
  npub: string,
  bindingId: string,
  timestamp: number,
): string
```

Produces a newline-joined string. **This exact string must be used on the
client** when calling `signMessage` via the CDP hook.

```
bliever-v1 Identity Binding v1.0

Nostr pubkey: <npub>
Binding ID:   <bindingId>
Timestamp:    <timestamp>
Chain ID:     <BASE_CHAIN_ID>

By signing, I confirm this Smart Account consents to be linked
with the above Nostr identity.
```

**⚠️ Breaking change rule:** Any edit to this template requires bumping
`ONBOARDING_VERSION`. The validator must then support both the old and new
version during a transition period.

---

### 3.3 `lib/binding/validator.ts`

#### Pure helpers (synchronous, no side effects)

```ts
// Returns true if timestamp is within ±BINDING_TIMESTAMP_WINDOW_SEC of now
function checkTimestamp(timestamp: number): boolean

// Validates Kind, d-tag, binding tag, evm tag
function checkNostrEventStructure(event: NostrBindingEvent): StructureResult

// Parses event.content JSON, validates all required fields are present
function parseBindingClaim(content: string): ParseResult
```

#### Main validator (async)

```ts
async function validateBindingPayload(
  payload: BindIdentityPayload,
): Promise<ValidationResult>
```

**Returns:**
```ts
// Success
{ valid: true, normalizedEvmAddress: string, claim: NostrBindingClaim }

// Failure
{ valid: false, reason: BindingErrorReason }
```

**Execution order (fail-fast):**

| Step | Operation | Type | Cost |
|------|-----------|------|------|
| 1 | `checkTimestamp` | local | O(1) |
| 2 | `checkNostrEventStructure` | local | O(tags) |
| 3 | pubkey equality | local | O(1) |
| 4 | `verifyNostrEvent` | CPU (secp256k1) | ~1ms |
| 5 | `parseBindingClaim` | local | O(content) |
| 6 | EVM address normalisation | local | O(1) |
| 7 | EVM address match | local | O(1) |
| 8 | bindingId match | local | O(1) |
| 9 | `verifyEVMSignature` | **RPC call** | ~50-200ms |

Steps 1–8 are all local. Only step 9 hits the network.

---

### 3.4 `lib/nostr/verification.ts`

```ts
async function verifyNostrEvent(event: NostrBindingEvent): Promise<boolean>
```

Wraps `nostr-tools/pure` `verifyEvent`. Returns `false` on any error
(malformed event, wrong field types, invalid signature). Never throws.

**What `verifyEvent` checks internally:**
1. Serialises the event as `[0, pubkey, created_at, kind, tags, content]`
2. Computes `SHA-256` of the serialised bytes
3. Checks `id === sha256result` (event integrity)
4. Verifies Schnorr signature `sig` over `id` using `pubkey`

---

### 3.5 `lib/evm/client.ts`

```ts
export const basePublicClient: PublicClient  // viem
```

Singleton. Chain selected via `NEXT_PUBLIC_BASE_CHAIN_ID` env var:
- `"8453"` → Base Mainnet (`viem/chains` `base`)
- anything else → Base Sepolia (`viem/chains` `baseSepolia`)

RPC URL priority:
1. `BASE_RPC_URL` env var
2. `https://mainnet.base.org` or `https://sepolia.base.org`

**Note:** The client is module-level. In Next.js serverless functions, each
cold start creates a new client instance. This is fine — no persistent WebSocket
connections are maintained.

---

### 3.6 `lib/evm/verification.ts`

```ts
function safeNormalizeAddress(address: string): string | null

async function verifyEVMSignature(
  message: string,
  signature: HexString,
  expectedAddress: string,
): Promise<boolean>
```

**`safeNormalizeAddress`:** Calls `viem.getAddress` (EIP-55 checksum). Returns
`null` for invalid addresses instead of throwing.

**`verifyEVMSignature`:** Calls `viem.verifyMessage` with the public client.

viem's `verifyMessage` behaviour:

```
if (address is EOA):
  → ecrecover(hash, sig)
  → compare with expectedAddress

if (address is smart contract):
  → eth_call: isValidSignature(hash, sig)  ← Base RPC call
  → check return value === ERC1271_MAGIC_VALUE (0x1626ba7e)
```

CDP ERC-4337 Smart Accounts are contracts, so the on-chain path is always taken.
This is why `BASE_RPC_URL` must be set to a reliable RPC in production.

---

## 4. API Routes

All routes are Next.js App Router handlers (`route.ts`). They accept `POST`
only and return `application/json`.

### 4.1 `POST /api/onboarding`

**File:** `app/api/onboarding/route.ts`

**Request body:** `OnboardingPayload` (= `BindIdentityPayload`)

```jsonc
{
  "npub":         "a1b2c3...",          // 64-char hex Nostr public key
  "evmAddress":   "0xAbCd...",          // CDP Smart Account address
  "nostrEvent": {
    "id":         "event-sha256-hex",
    "pubkey":     "a1b2c3...",
    "kind":       30078,
    "created_at": 1714567890,
    "tags": [
      ["d",       "identity-binding"],
      ["binding", "uuid-v4"],
      ["evm",     "0xAbCd..."]
    ],
    "content":    "{\"evmAddress\":\"0xAbCd...\",\"timestamp\":1714567890,\"bindingId\":\"uuid-v4\",\"version\":\"1.0\",\"appId\":\"bliever-v1\"}",
    "sig":        "schnorr-sig-hex"
  },
  "evmSignature": "0x...",             // EIP-191 signature from Smart Account
  "bindingId":    "uuid-v4",
  "timestamp":    1714567890
}
```

**Success response (200):**

```jsonc
{
  "success":            true,
  "onboardingComplete": true,
  "bindingId":          "uuid-v4",
  "npub":               "a1b2c3...",
  "evmAddress":         "0xAbCd...",   // EIP-55 checksummed
  "verifiedAt":         1714567890
}
```

**Error response (400/500):**

```jsonc
{ "success": false, "reason": "timestamp_expired" }
```

---

### 4.2 `POST /api/bind-identity`

**File:** `app/api/bind-identity/route.ts`

Identical request/response shape to `/api/onboarding`. The difference is
semantic: this endpoint is used for re-linking (key rotation, wallet migration),
not initial signup.

**Success response (200):**

```jsonc
{
  "success":    true,
  "bindingId":  "uuid-v4",
  "npub":       "a1b2c3...",
  "evmAddress": "0xAbCd...",
  "verifiedAt": 1714567890
}
```

---

### 4.3 `POST /api/verify-identity`

**File:** `app/api/verify-identity/route.ts`

**Request body:** `VerifyIdentityPayload`

```jsonc
{
  "npub":       "a1b2c3...",
  "evmAddress": "0xAbCd...",
  "nostrEvent": { /* same structure as above */ }
}
```

Note: `evmSignature`, `bindingId`, and `timestamp` are **not** required here.

**Success response (200):**

```jsonc
{
  "valid":      true,
  "npub":       "a1b2c3...",
  "evmAddress": "0xAbCd..."
}
```

**Error response (400):**

```jsonc
{ "valid": false, "reason": "invalid_nostr_signature" }
```

---

## 5. Validation Pipeline — Step by Step

This is a walkthrough of `validateBindingPayload()` with the exact code path
for each step.

```
Input: BindIdentityPayload {
  npub, evmAddress, nostrEvent, evmSignature, bindingId, timestamp
}
```

**Step 1 — Timestamp freshness**
```ts
const now = Math.floor(Date.now() / 1000);
Math.abs(now - timestamp) <= 300
// fail → "timestamp_expired"
```

**Step 2 — Event structure**
```ts
event.kind === 30078                           // fail → "wrong_event_kind"
event.tags.find(t => t[0]==="d")?.[1] === "identity-binding"  // fail → "wrong_d_tag"
event.tags.some(t => t[0]==="binding")         // fail → "missing_binding_tag"
event.tags.some(t => t[0]==="evm")             // fail → "missing_evm_tag"
```

**Step 3 — Pubkey match**
```ts
nostrEvent.pubkey === npub
// fail → "nostr_pubkey_mismatch"
```

**Step 4 — Schnorr signature (nostr-tools)**
```ts
verifyEvent(nostrEvent)  // checks: id = sha256(serialised), sig valid for pubkey
// fail → "invalid_nostr_signature"
```

**Step 5 — Parse claim JSON**
```ts
JSON.parse(nostrEvent.content)
// fail → "invalid_content_json"
// missing fields → "invalid_claim_fields"
```

**Step 6 — EVM address normalisation**
```ts
getAddress(evmAddress)     // EIP-55 checksum
getAddress(claim.evmAddress)
// fail → "evm_address_invalid"
```

**Step 7 — EVM address consistency**
```ts
normalizedClaim === normalizedExpected
// fail → "evm_address_mismatch"
```

**Step 8 — BindingId cross-field**
```ts
claim.bindingId === bindingId
// fail → "binding_id_mismatch"
```

**Step 9 — ERC-1271 on-chain (viem)**
```ts
verifyMessage({
  client: basePublicClient,          // triggers isValidSignature eth_call
  address: normalizedExpected,
  message: buildEVMConsentMessage(npub, bindingId, timestamp),
  signature: evmSignature,
})
// fail → "invalid_evm_signature"
```

**Success**
```ts
return { valid: true, normalizedEvmAddress, claim }
```

---

## 6. Error Reason Reference

| Reason | Step | Cause | Client Action |
|--------|------|-------|---------------|
| `invalid_payload` | 0 | Missing required field / invalid JSON | Fix request construction |
| `timestamp_expired` | 1 | Binding submitted > 5 min after creation | Retry with fresh payload |
| `wrong_event_kind` | 2 | `event.kind !== 30078` | Fix event construction |
| `wrong_d_tag` | 2 | `d` tag ≠ `"identity-binding"` | Fix event tags |
| `missing_binding_tag` | 2 | No `binding` tag in event | Add `["binding", bindingId]` tag |
| `missing_evm_tag` | 2 | No `evm` tag in event | Add `["evm", evmAddress]` tag |
| `nostr_pubkey_mismatch` | 3 | `event.pubkey` ≠ `npub` | Use correct pubkey |
| `invalid_nostr_signature` | 4 | Schnorr sig invalid or id mismatch | Re-sign event with correct nsec |
| `invalid_content_json` | 5 | `event.content` not valid JSON | Fix content serialisation |
| `invalid_claim_fields` | 5 | Missing field in claim object | Include all required claim fields |
| `evm_address_invalid` | 6 | Not a valid Ethereum address | Use checksummed address |
| `evm_address_mismatch` | 7 | Address in claim ≠ address in payload | Use same address in both |
| `binding_id_mismatch` | 8 | `claim.bindingId` ≠ `bindingId` param | Use same UUID in both |
| `invalid_evm_signature` | 9 | ERC-1271 call returned false | Re-sign consent message |
| `internal_error` | — | Unhandled exception | Retry / report bug |

---

## 7. Extending the System

### Adding a new validation rule

Add a new function to `lib/binding/validator.ts` following the existing pattern:

```ts
// Pure, synchronous, returns a typed result
export function checkNewRule(
  payload: BindIdentityPayload,
): { ok: true } | { ok: false; reason: BindingErrorReason } {
  // ...
  return { ok: true };
}
```

Then insert the call into `validateBindingPayload` **before** the ERC-1271 call
(step 9) to keep it cheap.

Add the new reason code to `BindingErrorReason` in `schema.ts`.

### Supporting a new Nostr event kind

1. Add a new constant in `schema.ts`:
   ```ts
   export const NOSTR_BACKUP_KIND = 30078; // different d-tag
   ```
2. Add a new validator in `validator.ts`.
3. Add a new route if the payload shape differs.

### Supporting multiple chains

1. Extend `lib/evm/client.ts` to export multiple clients:
   ```ts
   export const optimismPublicClient = createPublicClient({...});
   ```
2. Accept a `chainId` field in `BindIdentityPayload`.
3. Route to the correct client in `verifyEVMSignature`.
4. Add chain ID to the consent message (`buildEVMConsentMessage`).

### Bumping `ONBOARDING_VERSION`

1. Update `ONBOARDING_VERSION` in `schema.ts`.
2. Update `buildEVMConsentMessage` in `message.ts`.
3. Add a version check in `parseBindingClaim` to accept old versions during
   a migration window.
4. Update the client-side consent message builder to match.

---

## 8. Indexer Integration Contract

The indexer should:

1. **Call** `POST /api/onboarding` or `POST /api/bind-identity` with the full payload.
2. **On 200 response:** Store the binding in the database.
3. **On 400 response:** Do not store. Log the `reason` for debugging.
4. **On 500 response:** Retry with exponential backoff.

**Minimum schema to store (from success response):**

```sql
CREATE TABLE identity_bindings (
  npub             VARCHAR(64)  NOT NULL,   -- hex Nostr public key
  evm_address      VARCHAR(42)  NOT NULL,   -- EIP-55 checksummed
  binding_id       UUID         NOT NULL,
  nostr_event_id   VARCHAR(64)  NOT NULL,   -- for relay cross-reference
  verified_at      BIGINT       NOT NULL,   -- unix seconds from response
  is_active        BOOLEAN      NOT NULL DEFAULT TRUE,
  PRIMARY KEY (npub),
  UNIQUE (evm_address)
);
```

**Soft re-link:** When a new binding succeeds for an existing `npub`, set
`is_active = FALSE` on the old row and insert a new one (preserve history).

**Relay indexing:** After a successful `/api/onboarding`, the client publishes
the Kind 30078 event to Nostr relays. The indexer can also subscribe to relays
and run `POST /api/verify-identity` on incoming Kind 30078 events to build the
mapping without the user explicitly calling the backend.

---

## 9. Common Debugging Scenarios

### `invalid_evm_signature` despite correct signing

1. **Message mismatch** — The most common cause. Ensure the client uses the
   exact same `buildEVMConsentMessage` implementation (newlines, field order,
   spacing).
   ```ts
   // Server: lib/binding/message.ts
   // Client: copy this function exactly; do not inline the template
   ```

2. **Wrong address type** — CDP may return a proxy address different from the
   Smart Account address. Ensure `evmAddress` is the Smart Account address,
   not the passkey/EOA owner address.

3. **RPC issues** — If `BASE_RPC_URL` is unreachable, `verifyMessage` throws
   and returns `false`. Check RPC connectivity.

### `timestamp_expired` immediately after signing

The client timestamp and server clock may differ. The 300-second window is
generous; if you're seeing immediate expiry, check:
- Client is using `Math.floor(Date.now() / 1000)` (not milliseconds).
- Server NTP sync is working.

### `invalid_nostr_signature`

1. Verify the client uses `finalizeEvent` from `nostr-tools` (not a custom
   implementation).
2. Ensure the `pubkey` field in the event matches the `npub` being submitted.
3. The `content` field must be JSON-stringified **before** finalizing the event.
   Finalizing hashes the content string as-is.

### ERC-1271 call fails in development

Base Sepolia RPC may rate-limit. Set `BASE_RPC_URL` in `.env.local`:
```
BASE_RPC_URL=https://base-sepolia.g.alchemy.com/v2/<YOUR_KEY>
```
