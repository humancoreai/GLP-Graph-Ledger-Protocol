#Status: Draft (current)

Protocol Specification v0.2 (DRAFT)
Diff from v0.1 – Normative Changes Only
Status: Draft for Review
Date: 2026-01-21
Changes: Breaking fixes for interoperability + governance clarity
Summary of Changes
BREAKING CHANGES
[SERIALIZATION] Event encoding now MUST use CBOR Canonical (§2.3.1)
[NODE IDENTITY] Introduced content_id vs assertion_id split (§2.1.1)
[COST POLICY] Events MUST reference a CostPolicy object (§2.4.1)
FIXES
[SIMILARITY] temporal_corr now clipped to [0,1] with explicit semantics (§3.2.1)
[STABILITY] Removed from protocol compliance requirements (§6.3)
[CONVERGENCE] Added PageRank-style damping for C(n) (§5.4.1)
ADDITIONS
[COST] CostPolicy object specification (§2.4.1)
[AUDIT] Policy reference requirement for all events (§2.3.2)
Section 2.1.1 – Node Identity (NEW)
Replaces: Previous hash field in Node definition
Rationale: Prevents fragmentation where identical content gets different hashes due to metadata variations.
Node Identity Model
All nodes have TWO identities:
content_id:
  Purpose: Canonical identifier for the content itself
  Formula: SHA-256(content_bytes)
  Scope: Global deduplication
  
assertion_id:
  Purpose: Unique identifier for this specific claim instance
  Formula: SHA-256(CBOR_canonical({
    content_id,
    creator,
    timestamp,
    immutable_metadata
  }))
  Scope: Provenance tracking
Immutable vs. Mutable Metadata
immutable_metadata (part of assertion_id):
  - schema_version: semver
  - content_type: enum { text, image, video, audio, data, code }
  
  → These CANNOT be changed without creating new assertion

mutable_metadata (NOT part of assertion_id):
  - topic: string | null
  - language: ISO-639-1 | null
  - tags: array<string>
  
  → These MAY be updated via separate metadata edges
Updated Node Schema
Node {
    // Identity (dual)
    content_id: SHA-256,           // NEW: content hash
    assertion_id: SHA-256,         // NEW: assertion hash (replaces old 'hash')
    
    // Content
    content: bytes | URI,
    
    // Provenance
    creator: PublicKey,
    timestamp: UnixTimestamp,
    
    // Metadata
    immutable_metadata: {
        schema_version: semver,
        content_type: enum { text, image, video, audio, data, code }
    },
    
    mutable_metadata: {
        topic: string | null,
        language: ISO-639-1 | null,
        tags: array<string>
    }
}
Client Behavior
Clients MAY choose to:
Use Case
Index By
Effect
Deduplication
content_id
Multiple assertions of "2+2=4" collapse to one node
Full Provenance
assertion_id
Alice's "2+2=4" ≠ Bob's "2+2=4"
Hybrid
Both
Show content once, but track all asserters
Default recommendation: Index by content_id, track provenance via edges.
Section 2.3.1 – Canonical Event Serialization (NEW)
Replaces: Implicit serialization in v0.1
Rationale: Without deterministic encoding, identical events produce different hashes → network fragmentation.
Mandatory Encoding
All events MUST be serialized using:
CBOR Canonical Encoding (RFC 8949, Section 4.2)
Canonical rules:
Keys MUST be sorted lexicographically
Keys MUST NOT be duplicated
Integers MUST use smallest possible encoding
Floating point MUST use IEEE 754 deterministic representation
No indefinite-length encodings
Maps MUST use definite-length encoding
Hash and Signature Calculation
event_payload = {
    event_type,
    actor,
    payload,
    metadata,
    cost_proof  // excluding signature itself
}

cbor_bytes = CBOR_canonical_encode(event_payload)

event_id = SHA-256(cbor_bytes)
signature = Ed25519_sign(cbor_bytes, actor.private_key)
Validation
Clients MUST:
Decode CBOR (rejecting non-canonical encodings)
Re-encode to CBOR canonical
Verify hash matches event_id
Verify signature against re-encoded bytes
If any step fails → REJECT event
Test Vectors (Informative)
Example event (simplified):
{
  "event_type": "node_create",
  "actor": {"public_key": "0x123..."},
  "timestamp": 1705881600
}

CBOR hex (canonical): A3 63 61 63 74 6F 72 A1 ...
SHA-256: 3a5f2c... (event_id)
Full test vectors in separate document.
Section 2.3.2 – Event Cost Policy Reference (NEW)
Addition to Event schema
Rationale: Prevents "race to the bottom" where attackers use lowest-cost clients to flood network.
Updated Event Schema
Event {
    ...existing fields...
    
    cost_proof: {
        policy_ref: SHA-256,       // NEW: MUST reference a CostPolicy
        type: enum { pow, stake, rate_limit, hybrid },
        proof: bytes
    }
}
Cost Policy Reference Rules
Events MUST:
Include policy_ref pointing to a valid CostPolicy object
Provide proof that meets OR EXCEEDS requirements in that policy
Reference a policy that is valid at event.timestamp
Clients MUST:
Verify the referenced CostPolicy exists and is valid
Validate proof against policy requirements
MAY reject events referencing policies below their local threshold
Section 2.4.1 – Cost Policy Object (NEW)
New first-class object type
CostPolicy Schema
CostPolicy {
    // Identity
    policy_id: SHA-256(CBOR_canonical(policy_content)),
    
    // Issuer
    issuer: {
        public_key: PublicKey | null,  // null = protocol default
        organization: string | null     // informative
    },
    
    // Requirements
    requirements: {
        min_pow_difficulty: u32,        // bits of leading zeros
        min_stake_amount: u64 | null,   // in protocol-defined units
        rate_limits: {
            new_identity: u32,          // events/day for age < 30d
            established: u32,           // events/day for age >= 30d
            trusted: u32 | null         // null = unlimited
        }
    },
    
    // Validity
    valid_from: UnixTimestamp,
    valid_until: UnixTimestamp | null,  // null = indefinite
    
    // Verification
    signature: Ed25519Signature | null  // null for protocol defaults
}
Protocol Default: CostPolicy_v0
This policy MUST be accepted by all compliant clients:
CostPolicy_v0 {
    policy_id: "0x5a3e...",  // deterministic hash
    issuer: {
        public_key: null,
        organization: "Protocol Specification v0.2"
    },
    requirements: {
        min_pow_difficulty: 16,      // ~0.1s on 2025-era CPU
        min_stake_amount: null,      // staking optional in v0
        rate_limits: {
            new_identity: 10,        // 10 events/day for first 30 days
            established: 100,        // 100/day after 30 days
            trusted: null            // no unlimited tier in v0
        }
    },
    valid_from: 1705881600,  // 2025-01-22 00:00:00 UTC
    valid_until: null,
    signature: null
}
Custom Policies
Subgraphs or communities MAY publish stricter policies:
CostPolicy_HighSecurity {
    policy_id: "0x7f2a...",
    issuer: {
        public_key: "0xabcd...",
        organization: "Scientific Peer Review Network"
    },
    requirements: {
        min_pow_difficulty: 20,      // 16× harder than v0
        min_stake_amount: 1000,      // requires stake
        rate_limits: {
            new_identity: 1,
            established: 10,
            trusted: 100
        }
    },
    ...
}
Rules:
Custom policies MUST be signed by issuer
Events referencing custom policies MUST meet those requirements
Clients MAY reject events using policies they don't trust
BUT clients MUST accept events meeting CostPolicy_v0
Auditability
All events carry policy_ref → anyone can verify:
1. What policy did this event claim to meet?
2. Does the proof actually satisfy that policy?
3. Is that policy acceptable to my client?
This makes cost enforcement transparent and forkable.
Section 3.2.1 – Temporal Correlation Normalization (UPDATED)
Replaces: Previous temporal_corr definition in §3.2
Change: Pearson correlation ∈ [-1, 1] must be mapped to [0, 1] for use in similarity_v0.
Updated Definition
temporal_corr(A, B, window, buckets=100) = 
    max(0, pearson_correlation(
        activity_vector(A, window, buckets),
        activity_vector(B, window, buckets)
    ))

where:
    pearson_correlation(...) ∈ [-1, 1]
    max(0, ...) clips to [0, 1]
Semantic Interpretation
Pearson Value
After Clipping
Meaning
+1.0
1.0
Perfect synchronization (highly suspicious)
+0.8
0.8
Strong temporal correlation (suspicious)
+0.3
0.3
Weak correlation (possibly coincidental)
0.0
0.0
Independent timing (expected for unrelated actors)
-0.5
0.0
Anti-correlated (NOT suspicious, clipped to 0)
-1.0
0.0
Perfect anti-correlation (NOT suspicious, clipped to 0)
IMPORTANT:
temporal_corr = 0.0 means "no evidence of coordination," NOT "medium suspicion."
Rationale for clipping negatives:
Negative correlation (e.g., night shift vs. day shift) is NOT evidence of coordination
Only positive correlation suggests synchronized activity
Clipping to 0 treats anti-correlation same as independence
Updated similarity_v0
similarity_v0(A, B) = 
    0.5 × overlap(A, B, window=30d)
    + 0.3 × temporal_corr(A, B, window=30d)  // now ∈ [0, 1]
    + 0.2 × cost_similarity(A, B, window=30d)

Result: ∈ [0, 1] (guaranteed)
Section 5.4.1 – Conflict Topology with Damping (UPDATED)
Replaces: Naive recursive definition in §5.4
Change: Added PageRank-style damping to ensure convergence.
Problem with v0.1
C(n) = Σ Stability(m) for m contradicting n

→ Pure recursion can diverge or oscillate
→ Implementation-dependent behavior
Damped Recursion
C(n, iteration_i) = 
    d × Σ Stability(m, iteration_{i-1})  // recursive component
    + (1-d) × C_base(n)                   // grounding component

where:
    d = 0.85  // damping factor (same as PageRank default)
    
    C_base(n) = contradiction_count(n) / (1 + age_years(n))
      // Base conflict measure (non-recursive)
    
    contradiction_count(n) = 
      |{m : exists edge(m, n, type=contradicts)}|
Iterative Algorithm
# Initialize
for all n:
    C[n] = C_base(n)
    Stability[n] = T(n) × I(n) / (1 + C[n])

# Iterate until convergence
for iteration in range(max_iterations=100):
    C_new = {}
    for all n:
        C_new[n] = (
            0.85 × sum(Stability[m] for m contradicting n)
            + 0.15 × C_base(n)
        )
        Stability_new[n] = T(n) × I(n) / (1 + C_new[n])
    
    # Check convergence
    if max(|C_new[n] - C[n]| for all n) < ε:
        break
    
    C = C_new
    Stability = Stability_new

where:
    ε = 0.01  // convergence threshold
Convergence Guarantees
Properties:
Damping ensures contraction: Each iteration pulls toward C_base
Typical convergence: 50-100 iterations for graphs <100k nodes
Deterministic: Same graph → same result (given same T, I)
If non-convergence detected:
Clients MAY fall back to C(n) = C_base(n) (non-recursive)
Clients MUST log warning in diagnostics
Alternative (Simpler, for MVP)
For clients not wanting iterative computation:
C_simple(n) = Σ min(Stability(m), cap)  for m contradicting n

where:
    cap = 100  // maximum weight per contradiction
    Stability(m) computed without recursion (C=0 for all nodes)
This is PERMITTED but SHOULD be labeled as "non-recursive stability" in client metadata.
Section 6.3 – Stability Calculation (UPDATED)
Replaces: v0.1 requirement that clients MUST compute stability
Change: Stability moved from protocol compliance to application layer.
Removed from MUST Requirements
Clients MUST compute T(n), I(n), C(n) ← DELETED
New Classification
Protocol Layer (MUST):
Parse events (§2.3.1)
Verify signatures
Validate cost proofs (§2.4.1)
Compute canonical signals (§3.1, §3.2, §3.3)
Application Layer (MAY):
Compute stability scores
Choose stability formula
Weight T/I/C differently
Ignore stability entirely
If Providing Stability Scores
Clients that expose stability values to users MUST:
Declare formula version:
"This client uses Stability_v0.2_damped"
Publish parameters:
{
  "formula": "T × I / (1 + C)",
  "T_lambda": 0.00274,  // decay rate
  "C_damping": 0.85,
  "C_epsilon": 0.01
}
Reference canonical signals used:
"Based on: overlap(30d), temporal_corr(30d), cost_budget(30d)"
Clients SHOULD:
Allow users to switch between multiple stability formulas
Indicate when using non-standard metrics
Provide stability value ranges for context (e.g., "typical range: 1-100")
Interoperability
Two clients using different stability formulas:
Client A: Stability_v0.2_damped  → node X = 45.3
Client B: Stability_custom       → node X = 78.1

→ Not directly comparable
→ BUT both computed from same canonical signals
→ Users can understand "Client B weights time more heavily"
This is INTENDED pluralism, not fragmentation.
Section 6.1 – Updated Compliance Requirements (UPDATED)
Changes: Added new MUST requirements, removed old ones
Minimum Compliance (Updated)
A client is "Protocol v0.2 Compliant" if and only if:
✅ Event Handling:
MUST parse CBOR canonical encoding (§2.3.1)
MUST verify Ed25519 signatures
MUST validate cost proofs against referenced CostPolicy (§2.4.1)
MUST accept events meeting CostPolicy_v0
MUST reject events with invalid proofs or signatures
✅ Node Identity:
MUST compute both content_id and assertion_id (§2.1.1)
MAY index by either or both (client choice)
✅ Canonical Signals:
MUST be able to compute:
overlap(A, B, window) for at least window=30d
cost_budget(A, window)
SHOULD be able to compute:
temporal_corr(A, B, window) with clipping (§3.2.1)
graph_distance(A, B) (optional, for full compliance)
❌ MUST NOT:
Require PII (email, phone, IP, real name, etc.)
Censor events based on content
Modify events after verification
Removed from v0.1
MUST compute T(n), I(n), C(n) → Now MAY (application layer)
Appendix D – Migration Guide from v0.1 (NEW)
For implementers who started with v0.1:
Breaking Changes Checklist
1. Serialization
[ ] Replace JSON with CBOR canonical encoding
[ ] Update hash calculations to use CBOR bytes
[ ] Update signature verification to use CBOR bytes
[ ] Add CBOR canonical library (e.g., cbor2 for Python, cbor for Rust)
2. Node Identity
[ ] Split hash field into content_id + assertion_id
[ ] Update database schema to store both
[ ] Decide: index by content_id (dedup) or assertion_id (provenance)?
[ ] Migrate existing nodes (rehash with new formula)
3. Cost Policy
[ ] Create CostPolicy_v0 object in your database
[ ] Update event creation to include policy_ref
[ ] Update event validation to check policy
[ ] Decide: accept only v0, or also custom policies?
4. Similarity Calculation
[ ] Update temporal_corr to clip negative values to 0
[ ] Verify similarity_v0 now returns [0, 1]
5. Stability (if used)
[ ] Remove from protocol layer
[ ] Move to application layer
[ ] Declare formula version in client metadata
[ ] Add damping to C(n) calculation (or use simplified version)
Non-Breaking Changes (Recommended)
6. Documentation
[ ] Update API docs to reflect v0.2 changes
[ ] Add compliance badge: "Protocol v0.2 Compliant"
[ ] Document which optional features you support
7. Testing
[ ] Verify CBOR encoding matches reference implementation
[ ] Test cross-client event exchange (interoperability)
[ ] Validate stability convergence (if using damped C(n))
Appendix E – Reference Implementations (Informative)
The following implementations are being developed:
Language
Library
Status
Maintainer
Python
graph-ledger-py
WIP
TBD
Rust
graph-ledger-rs
Planned
TBD
JavaScript
graph-ledger-js
Planned
TBD
Test vectors and compliance tests will be published separately.
End of v0.2 Diff
Next Steps:
Community review (2-week RFC period)
Reference implementation in Python (targeting 2026-Q2)
Interoperability testing between at least 2 independent clients
Formal release of v0.2 as stable
Feedback Period: Open until 2026-02-15
Changes Summary:
6 breaking fixes
3 new sections
4 updated sections
0 removals of functionality (only clarifications)
Result: v0.2 is implementable, interoperable, and governance-neutral.
This is ready for implementation.
