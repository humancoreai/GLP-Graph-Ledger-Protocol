# Protocol Specification v0.3 – Deterministic Wire Profile

**Status:** Normative  
**Date:** 2026-01-23  
**Base:** Protocol Specification v0.2 (Draft)  
**Purpose:** Eliminate all wire-level ambiguities for guaranteed interoperability

---

## Preamble

**This document is a NORMATIVE PATCH to v0.2.**

v0.3 introduces NO new features, NO semantic changes, NO architectural modifications.

**Goal:**
> Two independent implementations, given identical semantic input, MUST produce byte-identical Events, Nodes, hashes, and signatures.

**Scope:**
- Wire encoding determinism
- Hash canonicalization
- Floating-point policy
- Signal computation normalization

**Non-scope:**
- Application-layer decisions (stability formulas, UI, storage)
- Optional features (graph_distance, custom policies)

---

## 1. Content Canonicalization (NORMATIVE)

**Replaces:** v0.2 implicit content handling

### 1.1 content_bytes Formation

**All Node content MUST be reduced to canonical bytes before hashing.**

#### 1.1.1 Text Content

```
Input: UTF-8 string
Canonical form:
  1. Apply Unicode Normalization Form C (NFC) per UAX #15
  2. Encode as UTF-8 bytes (no BOM)
  3. Result = content_bytes

Example:
  Input:  "café" (U+0063 U+0061 U+0066 U+00E9)
  NFC:    "café" (U+0063 U+0061 U+0066 U+00E9) [unchanged]
  UTF-8:  0x63 0x61 0x66 0xC3 0xA9
  
  Input:  "café" (U+0063 U+0061 U+0066 U+0065 U+0301) [decomposed]
  NFC:    "café" (U+0063 U+0061 U+0066 U+00E9) [composed]
  UTF-8:  0x63 0x61 0x66 0xC3 0xA9
  
  → Both produce identical content_bytes
```

**Normalization MUST occur:**
- Before computing `content_id`
- Before insertion into CBOR payload
- MUST NOT occur on already-hashed content (preserves immutability)

**Invalid UTF-8:**
- Sequences containing invalid UTF-8 MUST be rejected
- Clients MUST NOT attempt repair/guessing

---

#### 1.1.2 Binary Content

```
Input: Raw bytes (images, video, audio, binary data)
Canonical form:
  content_bytes = input (unchanged)

No normalization applied.
Hash of binary content is direct SHA-256(bytes).
```

---

#### 1.1.3 URI References

```
Input: URI pointing to external content
Canonical form:
  1. Parse URI per RFC 3986
  2. Apply URI normalization per RFC 3986 Section 6.2.2:
     - Scheme and host to lowercase
     - Decode unreserved characters (%41 → A)
     - Remove default port (:80 for http, :443 for https)
     - Remove empty path segments (a//b → a/b)
  3. Encode normalized URI as UTF-8
  4. Result = content_bytes

Example:
  Input:  "HTTP://Example.COM:80/foo/../bar"
  Normal: "http://example.com/bar"
  UTF-8:  0x68 0x74 0x74 0x70 ...
```

**When URI is stored:**
- Node stores normalized URI string
- Content is NOT fetched during hashing
- `content_id` = hash of URI, not of fetched content

---

#### 1.1.4 Structured Data (Future)

```
Reserved for v0.4+:
  JSON, CBOR, Protobuf embedded as content
  
v0.3 stance:
  Treat as binary (no schema-aware normalization)
```

---

### 1.2 content_id Calculation (NORMATIVE UPDATE)

**Replaces:** v0.2 Section 2.1.1

```
content_id = SHA-256(content_bytes)

where content_bytes is formed per Section 1.1
```

**Properties:**
- Deterministic (same semantic content → same hash)
- Collision-resistant (SHA-256 security assumptions)
- Content-addressed (hash IS the identifier)

---

### 1.3 assertion_id Calculation (NORMATIVE UPDATE)

```
assertion_id = SHA-256(CBOR_canonical({
    "content_id": bytes,      # 32-byte SHA-256 hash
    "creator": bytes,         # 32-byte Ed25519 public key
    "timestamp": uint,        # Unix timestamp (seconds since epoch)
    "immutable_metadata": {
        "schema_version": text,     # semver string
        "content_type": text        # enum value as string
    }
}))
```

**CBOR encoding rules:**
- Keys MUST be sorted lexicographically (per RFC 8949 Section 4.2.1)
- Map MUST use definite-length encoding
- All integers MUST use smallest encoding (no leading zeros)
- Timestamp MUST be encoded as CBOR uint (major type 0)

**Example (informative):**

```
{
  "content_id": h'A3F2...',  # 32 bytes
  "content_type": "text",
  "creator": h'B7E1...',     # 32 bytes  
  "immutable_metadata": {
    "content_type": "text",
    "schema_version": "0.3.0"
  },
  "timestamp": 1705881600
}

CBOR canonical hex: A4 64 ... (deterministic)
assertion_id: SHA-256(above) = C8A3...
```

---

## 2. Unicode Normalization Policy (NORMATIVE)

### 2.1 Normalization Form

**All text content MUST use Unicode Normalization Form C (NFC).**

**References:**
- Unicode Standard Annex #15 (UAX #15)
- https://www.unicode.org/reports/tr15/

**Rationale:**
- NFC is canonical composed form (most compact)
- Most widely supported across platforms
- Stable across Unicode versions (compatibility guaranteed)

---

### 2.2 Application Points

**NFC MUST be applied:**

| Location | When |
|----------|------|
| Node text content | Before computing `content_id` |
| Edge context field | Before CBOR encoding |
| Event metadata strings | Before computing `event_id` |
| CostPolicy organization field | Before signing |

**NFC MUST NOT be applied:**

| Location | Why Not |
|----------|---------|
| Already-hashed content | Hash is immutable identifier |
| Binary content | No concept of Unicode |
| Cryptographic keys | Raw bytes, not text |

---

### 2.3 Implementation Requirements

**Clients MUST:**
- Use UAX #15 compliant library (e.g., ICU, Python `unicodedata`)
- Validate NFC output (some libraries have bugs)
- Test with decomposed input (ensure normalization works)

**Test Vector:**

```
Input:  "é" (U+00E9, single codepoint)
NFC:    "é" (U+00E9, unchanged)

Input:  "é" (U+0065 U+0301, decomposed)
NFC:    "é" (U+00E9, composed)

Both MUST produce content_id: SHA-256(0xC3 0xA9)
```

---

## 3. Floating-Point Policy (NORMATIVE)

**Replaces:** v0.2 Section 2.3.1 ambiguity on floats

### 3.1 Policy Statement

**Floating-point values are PROHIBITED in all signed Event payloads.**

**Rationale:**
- IEEE-754 has edge cases (NaN, −0, subnormals)
- CBOR float encoding has two valid forms (float16, float32, float64)
- No use case in current protocol requires floats in Events

---

### 3.2 Where Floats Are Allowed

**Floats MAY be used:**

| Context | Status | Example |
|---------|--------|---------|
| Application layer (stability scores) | Allowed | `Stability(n) = 45.3` |
| Client-side visualization | Allowed | Graph coordinates |
| Signal computation results | Allowed | `overlap = 0.73` |

**Floats MUST NOT appear:**

| Context | Status |
|---------|--------|
| Event payloads | Prohibited |
| Node content (unless as opaque binary) | Prohibited |
| Edge strength field | Prohibited (use rational or fixed-point) |

---

### 3.3 Alternative Encodings

**For values requiring precision in Events:**

#### Fixed-Point Decimals
```
strength = 0.75
→ Encode as: {numerator: 3, denominator: 4}
→ CBOR: {"numerator": 3, "denominator": 4}
```

#### Scaled Integers
```
strength = 0.75
→ Encode as: 7500 (scaled by 10000)
→ CBOR: 7500
→ Decode: value / 10000
```

**If future versions require floats:**
- v0.4+ may specify IEEE-754 canonical encoding
- Requires explicit normalization rules (−0 → +0, NaN → error, etc.)

---

## 4. Proof-of-Work Canonical Definition (NORMATIVE)

**Replaces:** v0.2 Section 2.4.1 (CostProof_PoW) with precise wire format

### 4.1 PoW Algorithm

```
Algorithm: SHA-256 based proof-of-work

Input:
  event_id: bytes (32-byte SHA-256 hash)
  difficulty: uint (number of leading zero bits required)

Output:
  nonce: uint64
  proof_hash: bytes (32-byte SHA-256 hash)

Procedure:
  for nonce in [0, 2^64):
      challenge = event_id || nonce_bytes
      proof_hash = SHA-256(challenge)
      
      if leading_zero_bits(proof_hash) >= difficulty:
          return (nonce, proof_hash)
  
  # If exhausted: increase difficulty or fail
```

---

### 4.2 Wire Format

```
CostProof_PoW {
    "type": "pow",
    "difficulty": uint,        # bits of leading zeros
    "nonce": uint,             # 64-bit unsigned integer
    "proof_hash": bytes        # 32-byte SHA-256 result (for verification)
}

CBOR encoding (canonical):
{
  "difficulty": 16,
  "nonce": 1847263,
  "proof_hash": h'00003FA2...',  # 32 bytes
  "type": "pow"
}
```

---

### 4.3 Nonce Encoding (NORMATIVE)

**Nonce MUST be encoded as:**

```
Type: Unsigned 64-bit integer (uint64)
Byte Order: Big-endian (network byte order)
Length: 8 bytes (even if value < 256)

Example:
  nonce = 1847263 (decimal)
  = 0x001C31DF (hex)
  
  nonce_bytes = 0x00 0x00 0x00 0x00 0x00 0x1C 0x31 0xDF
                |                               |
                MSB                            LSB
```

**Challenge Construction:**

```
challenge = event_id || nonce_bytes
          = (32 bytes) || (8 bytes)
          = 40 bytes total

proof_hash = SHA-256(challenge)
```

---

### 4.4 Difficulty Definition (NORMATIVE)

**Difficulty is defined as:**

> The minimum number of leading zero BITS in `proof_hash` (binary representation).

**NOT:**
- Leading zero bytes (ambiguous)
- Leading zero hex digits (base-dependent)

**Verification:**

```python
def leading_zero_bits(hash_bytes: bytes) -> int:
    """Count leading zero bits in hash."""
    count = 0
    for byte in hash_bytes:
        if byte == 0:
            count += 8
        else:
            # Count leading zeros in this byte
            count += (byte.bit_length() ^ 8)
            break
    return count

# Example:
proof_hash = bytes.fromhex('00003FA2...')
#             0000 0000  0000 0000  0000 0000  0011 1111 ...
#             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#             18 leading zero bits

leading_zero_bits(proof_hash) = 18
```

---

### 4.5 Verification Algorithm (NORMATIVE)

```
Input:
  event_id: bytes (32 bytes)
  difficulty: uint
  nonce: uint64
  claimed_proof_hash: bytes (32 bytes)

Verification:
  1. nonce_bytes = nonce.to_bytes(8, byteorder='big')
  2. challenge = event_id || nonce_bytes
  3. computed_hash = SHA-256(challenge)
  
  4. IF computed_hash ≠ claimed_proof_hash:
       REJECT (hash mismatch)
  
  5. IF leading_zero_bits(computed_hash) < difficulty:
       REJECT (insufficient difficulty)
  
  6. ACCEPT
```

---

### 4.6 Difficulty Calibration (Informative)

**Expected time on 2025-era consumer CPU (single-threaded):**

| Difficulty | Expected Time | Hashes Required |
|------------|---------------|-----------------|
| 10 | ~1 ms | ~1,000 |
| 16 | ~65 ms | ~65,000 |
| 20 | ~1 second | ~1,000,000 |
| 24 | ~16 seconds | ~16,000,000 |

**CostPolicy_v0 default: difficulty = 16**

---

## 5. Temporal Correlation Normalization (NORMATIVE)

**Replaces:** v0.2 Section 3.2.1 with precise bucket definitions

### 5.1 Window Definition

```
Window is a ROLLING time range:
  
  start = current_time - window_duration
  end = current_time

window_duration MUST be expressed in days (integer)

Example:
  current_time = 2026-01-23 14:30:00 UTC
  window = 30 days
  
  start = 2025-12-24 14:30:00 UTC
  end   = 2026-01-23 14:30:00 UTC
```

**NOT aligned windows** (e.g., "January 2026")  
**Ensures consistency across clients computing at different times.**

---

### 5.2 Bucket Definition (NORMATIVE)

```
Parameters:
  window_duration: days (e.g., 30)
  bucket_count: uint (MUST be 100 for canonical signal)

bucket_size = window_duration / bucket_count
            = 30 days / 100
            = 0.3 days
            = 7.2 hours

Bucket boundaries:
  bucket[0]: [start, start + bucket_size)
  bucket[1]: [start + bucket_size, start + 2×bucket_size)
  ...
  bucket[99]: [start + 99×bucket_size, end)
```

---

### 5.3 Activity Vector Construction (NORMATIVE)

```
For actor A:

activity_vector(A, window, buckets=100):
  
  events = all events by A in [start, end)
  vector = [0, 0, ..., 0]  # 100 elements
  
  for event in events:
      t = event.timestamp
      bucket_index = floor((t - start) / bucket_size)
      
      if 0 <= bucket_index < 100:
          vector[bucket_index] += 1
  
  return vector
```

**Edge case: Event exactly at boundary**
```
If event.timestamp == start + k × bucket_size:
  → Assign to bucket[k] (inclusive lower bound)
```

---

### 5.4 Pearson Correlation (NORMATIVE)

```
temporal_corr(A, B, window=30, buckets=100):
  
  v_A = activity_vector(A, window, buckets)
  v_B = activity_vector(B, window, buckets)
  
  r = pearson_correlation(v_A, v_B)
  
  return max(0, r)  # Clip to [0, 1]
```

**Pearson formula:**

```
r = Σ((v_A[i] - mean_A) × (v_B[i] - mean_B)) 
    / (std_A × std_B × n)

where:
  mean_A = Σ v_A[i] / n
  std_A = sqrt(Σ(v_A[i] - mean_A)² / n)
  n = bucket_count = 100
```

---

### 5.5 Zero-Variance Handling (NORMATIVE)

**Case 1: One or both vectors have zero variance**

```
If std_A == 0 OR std_B == 0:
  return 0.0
  
Rationale:
  - Pearson is undefined when variance is zero
  - Zero activity or constant activity → no coordination signal
  - Clipping handles this gracefully
```

**Case 2: Both vectors are zero**

```
If all(v_A[i] == 0) AND all(v_B[i] == 0):
  return 0.0
  
Rationale:
  - No activity from either actor
  - No evidence of correlation
```

---

## 6. Canonical Signals Registry (NORMATIVE)

**This section establishes the EXACT parameters for "canonical signals".**

**Any client claiming "Protocol v0.3 compliance" MUST compute these signals with these exact parameters.**

---

### 6.1 overlap(A, B, window)

```
Function: overlap(actor_A, actor_B, window_days)

Parameters:
  window_days: 30 (FIXED for canonical)
  
Algorithm:
  targets_A = {nodes that A created edges to/from in last 30 days}
  targets_B = {nodes that B created edges to/from in last 30 days}
  
  jaccard = |targets_A ∩ targets_B| / |targets_A ∪ targets_B|
  
  return jaccard  # ∈ [0, 1]

Edge cases:
  - If both sets empty: return 0.0
  - If one set empty: return 0.0
```

**Window computation:**
```
current_time = time_of_computation
start = current_time - 30 days
end = current_time

Include edge if: start <= edge.timestamp < end
```

---

### 6.2 temporal_corr(A, B, window)

```
Function: temporal_corr(actor_A, actor_B, window_days)

Parameters:
  window_days: 30 (FIXED for canonical)
  bucket_count: 100 (FIXED for canonical)

Algorithm:
  As defined in Section 5.4
  
  return max(0, pearson_correlation(v_A, v_B))  # ∈ [0, 1]
```

---

### 6.3 cost_budget(A, window)

```
Function: cost_budget(actor_A, window_days)

Parameters:
  window_days: 30 (FIXED for canonical)

Algorithm:
  events = all events by A in last 30 days
  
  budget = 0
  for event in events:
      budget += cost_difficulty(event.cost_proof)
  
  return budget  # ∈ [0, ∞)

where:
  cost_difficulty(PoW) = 2^difficulty
  cost_difficulty(Stake) = stake_amount  # (if supported)
  cost_difficulty(RateLimit) = 1.0
```

**Normalization (informative):**
```
Typical values for legitimate actor:
  - 100 events/month at difficulty=16
  - budget = 100 × 2^16 = 6,553,600

Typical values for spam bot:
  - 10,000 events/month at difficulty=10 (easier)
  - budget = 10,000 × 2^10 = 10,240,000
  
→ Absolute budget alone doesn't distinguish
→ Must combine with overlap and temporal_corr
```

---

### 6.4 Canonical Signals Summary Table

| Signal | Window | Buckets | Normalization | Range | Deterministic |
|--------|--------|---------|---------------|-------|---------------|
| `overlap` | 30d | N/A | Jaccard | [0, 1] | ✓ |
| `temporal_corr` | 30d | 100 | max(0, Pearson) | [0, 1] | ✓ |
| `cost_budget` | 30d | N/A | Sum of 2^difficulty | [0, ∞) | ✓ |

**"Deterministic" means:**
> Given identical event logs and computation time, all clients produce identical float64 results (within IEEE-754 rounding).

---

## 7. CBOR Canonical Encoding Clarifications (NORMATIVE)

**Extends:** RFC 8949 Section 4.2 with protocol-specific rules

### 7.1 Map Key Ordering

```
CBOR maps MUST have keys sorted by:
  1. Byte length (shorter first)
  2. Lexicographic byte order (if same length)

Example:
  {"z": 1, "aaa": 2, "b": 3}
  
  Sorted:
    "b"   (1 byte)
    "z"   (1 byte) 
    "aaa" (3 bytes)
  
  CBOR: A3 61 62 01 61 7A 01 63 61 61 61 02
```

---

### 7.2 Integer Encoding

```
MUST use smallest possible encoding:
  
  0-23:        1 byte (major type 0, value in low 5 bits)
  24-255:      2 bytes (major type 0, additional info 24)
  256-65535:   3 bytes (major type 0, additional info 25)
  etc.

Example:
  Value: 100
  MUST encode as: 0x18 0x64 (2 bytes)
  MUST NOT encode as: 0x19 0x00 0x64 (3 bytes)
```

---

### 7.3 Text String Encoding

```
MUST use UTF-8 encoding (CBOR major type 3)
MUST apply NFC normalization BEFORE encoding

Example:
  Text: "café" (after NFC)
  UTF-8: 0x63 0x61 0x66 0xC3 0xA9
  CBOR: 0x65 0x63 0x61 0x66 0xC3 0xA9
        ^^^^
        major type 3, length 5
```

---

### 7.4 Byte String Encoding

```
MUST use major type 2 for raw bytes (hashes, keys, etc.)

Example:
  SHA-256 hash (32 bytes)
  CBOR: 0x58 0x20 [32 bytes of hash]
        ^^^^
        major type 2, length 32
```

---

### 7.5 Prohibited Encodings

**The following MUST NOT appear in canonical CBOR:**

- Indefinite-length strings/arrays/maps
- Duplicate map keys
- Non-canonical integers (oversized encoding)
- Float16/32 (use float64 only, if absolutely necessary)
- Tags (unless explicitly defined in future protocol versions)

---

## 8. Event Hash Computation (NORMATIVE ALGORITHM)

**Complete deterministic procedure for `event_id`:**

```
Input: Event object (all fields except signature)

Procedure:

1. Construct event_payload map:
   {
     "event_type": text,
     "actor": {
       "public_key": bytes (32),
       "nonce": uint
     },
     "payload": {...},  # node or edge
     "metadata": {
       "timestamp": uint,
       "declared_topic": text | null,
       "subgraph_id": bytes | null
     },
     "cost_proof": {...}
   }

2. Apply NFC to all text fields

3. Encode to CBOR canonical:
   cbor_bytes = CBOR_canonical_encode(event_payload)

4. Hash:
   event_id = SHA-256(cbor_bytes)

5. Sign:
   signature = Ed25519_sign(cbor_bytes, actor.private_key)

6. Final Event:
   {
     ...event_payload...,
     "event_id": event_id,
     "signature": signature
   }
```

---

## 9. Compliance Test Vectors (NORMATIVE)

**All v0.3-compliant implementations MUST pass these vectors.**

### 9.1 Text Normalization Vector

```
Input text: "café" (U+0063 U+0061 U+0066 U+0065 U+0301)

Expected:
  NFC: "café" (U+0063 U+0061 U+0066 U+00E9)
  UTF-8: 0x63 0x61 0x66 0xC3 0xA9
  content_id: SHA-256(0x636166c3a9)
            = 0x8A3F... (32 bytes, deterministic)
```

---

### 9.2 PoW Verification Vector

```
event_id = 0xA1B2C3D4... (32 bytes, example)
difficulty = 16
nonce = 1847263 (0x001C31DF)

Expected:
  nonce_bytes = 0x00000000001C31DF (8 bytes, big-endian)
  challenge = event_id || nonce_bytes (40 bytes)
  proof_hash = SHA-256(challenge)
             = 0x00003FA2... (must have ≥16 leading zero bits)
  
  Verification: PASS
```

---

### 9.3 Temporal Correlation Vector

```
Actor A events (timestamps in Unix seconds):
  [1705881600, 1705881700, 1705881800]  # 3 events, 100s apart

Actor B events:
  [1705881610, 1705881710, 1705881810]  # 3 events, 100s apart, offset by 10s

Window: 30 days
Buckets: 100
Bucket size: 7.2 hours = 25920 seconds

Expected:
  All 6 events fall in bucket[0] (within first 7.2 hours)
  
  v_A = [3, 0, 0, ..., 0]
  v_B = [3, 0, 0, ..., 0]
  
  Pearson correlation = 1.0 (perfect, but trivial - same bucket)
  temporal_corr = max(0, 1.0) = 1.0
```

**Better test (informative): spread across buckets for meaningful correlation**

---

### 9.4 CBOR Canonical Encoding Vector

```
Object:
{
  "timestamp": 1705881600,
  "type": "node_create",
  "actor": {
    "nonce": 42,
    "public_key": h'A1B2...' (32 bytes)
  }
}

Expected CBOR (hex):
  A3                         # map(3)
    65 6163746F72           # text(5) "actor"
    A2                       # map(2)
      65 6E6F6E6365         # text(5) "nonce"
      18 2A                  # uint(42)
      6A 7075626C69635F6B6579  # text(10) "public_key"
      58 20                  # bytes(32)
      [32 bytes]
    69 74696D657374616D70   # text(9) "timestamp"
    1A 65B3E7C0             # uint(1705881600)
    64 74797065             # text(4) "type"
    6B 6E6F64655F637265617465  # text(11) "node_create"

SHA-256(above) = event_id (deterministic)
```

---

## 10. Migration from v0.2 to v0.3

**For existing v0.2 implementations:**

### 10.1 Breaking Changes

**None.** v0.3 is a clarification, not a redesign.

**However, implementations MAY have been non-compliant if they:**
- Used non-canonical CBOR (keys unsorted)
- Skipped NFC normalization
- Used floats in event payloads
- Computed PoW incorrectly

---

### 10.2 Action Items

#### Immediate (Before Generating New Events)

- [ ] Verify CBOR encoder produces canonical output (test with vector 9.4)
- [ ] Add NFC normalization to text content pipeline
- [ ] Update PoW implementation to use big-endian nonce encoding
- [ ] Remove any floats from event payloads

#### Retroactive (For Existing Data)

**Existing events from v0.2:**
- IF they were created with non-canonical encoding → hashes are wrong
- MUST recompute `event_id` with canonical encoding
- Signatures remain valid (if original signing used canonical bytes)

**Migration strategy:**
```
1. For each stored event:
   - Re-encode payload using v0.3 canonical rules
   - Recompute event_id
   - IF new event_id ≠ stored event_id:
     - Log as "non-canonical v0.2 event"
     - Update database with canonical event_id
     - Notify peers of hash update
```

---

### 10.3 Compatibility

**v0.3 events are compatible with v0.2 clients IF:**
- v0.2 client was already using canonical encoding (many were)

**v0.2 events are compatible with v0.3 clients IF:**
- They happen to be canonically encoded (verify with hash check)

**Recommended:**
- All new events: v0.3 canonical
- Old events: grandfather clause (accept if signature valid, even if non-canonical)
- Phase out non-canonical events over 6 months

---

## Appendix A: Complete Diff v0.2 → v0.3

| Section | Change | Type | Rationale |
|---------|--------|------|-----------|
| **1.1** | Added content canonicalization rules | Addition | Eliminates encoding ambiguity |
| **1.1.1** | Mandated NFC normalization for text | Normative | UTF-8 has multiple representations |
| **1.1.3** | Specified URI normalization | Normative | RFC 3986 allows variations |
| **2.1** | Defined NFC application points | Clarification | v0.2 was silent on when to normalize |
| **3.1** | Prohibited floats in event payloads | Restriction | IEEE-754 edge cases cause non-determinism |
| **4.3** | Specified big-endian nonce encoding | Normative | v0.2 didn't specify byte order |
| **4.4** | Defined difficulty as BIT count, not byte | Clarification | Prevents off-by-8 errors |
| **5.2** | Fixed bucket count to 100 for canonical | Normative | v0.2 said "buckets=100" but wasn't enforced |
| **5.5** | Specified zero-variance handling | Normative | Pearson undefined when std=0 |
| **6.x** | Created canonical signals registry | Addition | Makes "canonical" actually canonical |
| **7.x** | Clarified CBOR encoding rules | Clarification | RFC 8949 allows some flexibility |
| **9.x** | Added compliance test vectors | Addition | Enables interoperability testing |

---

## Appendix B: Rationale for Each Har
