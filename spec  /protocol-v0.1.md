Status: Superseded by v0.2
Kept for historical reference only

Protocol Specification v0.1
Graph Ledger for Distributed Knowledge
Status: Draft
Date: 2026-01-21
Authors: Conceptual Framework
Goal: A permissionless, censorship-resistant infrastructure for knowledge as emergent structure
1. Core Principles
1.1 Design Philosophy
This protocol treats truth as a physical property, not a social consensus:
Permissionless ingestion: Any assertion can enter the graph
Emergent selection: Stability arises from structure, not gatekeeping
Plural interpretation: Clients choose their own metrics, but share foundational data
Cost-based filtering: Spam is expensive, truth is cheap (over time)
1.2 What This Is Not
Not: A blockchain (no global ordering requirement)
Not: A consensus system (no voting on truth)
Not: A platform (no central authority)
Not: A replacement for the web (a layer on top)
1.3 What This Is
A protocol for encoding assertions, relations, and their temporal evolution in a verifiable, distributed ledger — such that stability emerges from graph topology, not authority.
2. Fundamental Objects
2.1 Nodes
A node is an atomic assertion or artifact:
Types:
Text (claim, observation, definition)
Media (image, video, audio)
Data (dataset, measurement)
Code (algorithm, proof)
Properties:
Node {
    hash: SHA-256(content + metadata),
    content: bytes | URI,
    creator: PublicKey,
    timestamp: UnixTimestamp,
    metadata: {
        type: enum { text, image, data, code, ... },
        language: ISO-639-1 | null,
        topic: string | null,
        schema_version: semver
    }
}
Immutability: Once created, nodes cannot be changed (only deprecated via updates edges).
2.2 Edges (Relations)
An edge is a typed relation between nodes:
Core Relation Types:
Type
Meaning
Symmetry
supports
Provides evidence for
Asymmetric
contradicts
Conflicts with
Symmetric
refines
Adds precision to
Asymmetric
updates
Supersedes (new version)
Asymmetric
cites
References neutrally
Asymmetric
derived_from
Logical/causal dependency
Asymmetric
similar_to
Analogous content
Symmetric
Properties:
Edge {
    source_node: Hash,
    target_node: Hash,
    relation_type: enum { ... },
    creator: PublicKey,
    timestamp: UnixTimestamp,
    strength: float ∈ [0, 1] | null,  // optional confidence weight
    context: string | null             // optional explanation
}
2.3 Events (Ledger Entries)
Every action in the graph is an event — the fundamental unit of the protocol.
Event Schema:
Event {
    // Identity
    event_id: SHA-256(event_payload + signature),
    event_type: enum {
        node_create,
        edge_create,
        edge_retract,    // withdraw a relation (rare, but allowed)
    },
    
    // Actor
    actor: {
        public_key: Ed25519PublicKey,
        nonce: u64,                    // prevents replay attacks
    },
    
    // Payload
    payload: {
        node: Node | null,             // if node_create
        edge: Edge | null,             // if edge_create/retract
    },
    
    // Context
    metadata: {
        timestamp: UnixTimestamp,
        declared_topic: string | null,  // optional: "climate", "covid", ...
        subgraph_id: Hash | null,       // optional: local graph context
    },
    
    // Anti-Spam
    cost_proof: CostProof,
    
    // Verification
    signature: Ed25519Signature(event_payload, actor.private_key)
}
2.4 Cost Proofs (Anti-Spam Mechanism)
Every event must carry a cost proof — evidence that creating this event was non-trivial.
Types:
A) Proof-of-Work (Light)
CostProof_PoW {
    type: "pow",
    difficulty: u32,               // number of leading zeros required
    nonce: u64,
    hash: SHA-256(event_id + nonce),
    
    valid_if: hash has >= difficulty leading zero bits
}
Calibration:
difficulty = 16 → ~0.1s on consumer CPU
difficulty = 20 → ~1.6s
Adjustable per client/network
Purpose: Make 1M spam events = 27 CPU-hours (economically unfeasible).
B) Stake-Weighted Rate Limit
CostProof_Stake {
    type: "stake",
    stake_amount: u64,             // tokens locked
    rate_tier: enum {
        new_identity: 10/day,
        established: 100/day,
        trusted: unlimited
    },
    proof: Signature(stake_tx_id, actor.private_key)
}
Purpose: Reputation builds over time; Sybil attacks require capital.
C) Hybrid (Recommended for v0)
CostProof_Hybrid {
    pow: CostProof_PoW,
    rate_limit: {
        events_in_last_24h: u32,
        max_allowed: u32,
        proof: Merkle_proof(recent_events)
    }
}
Logic:
New identities: PoW + 10/day
After 30 days without spam: PoW + 100/day
After endorsements: PoW + unlimited
3. Canonical Signals (Minimum Feature Suite)
These are the primitive observables that ALL protocol-compliant clients MUST be able to compute.
3.1 Target Overlap (Jaccard Index)
Definition:
overlap(A, B, window) = 
    |targets(A, window) ∩ targets(B, window)| 
    / 
    |targets(A, window) ∪ targets(B, window)|

where:
    targets(A, window) = {
        all nodes that A created an edge to/from 
        in time interval [now - window, now]
    }
Parameters:
window ∈ {7d, 30d, 365d} (protocol-defined standard windows)
Returns: float ∈ [0, 1]
Interpretation:
0.0 → A and B work on completely different topics
1.0 → A and B target identical nodes (potential coordination)
3.2 Temporal Correlation
Definition:
temporal_corr(A, B, window, buckets=100) = 
    pearson_correlation(
        activity_vector(A, window, buckets),
        activity_vector(B, window, buckets)
    )

where:
    activity_vector(A, window, buckets) = [
        count(events by A in bucket_1),
        count(events by A in bucket_2),
        ...
        count(events by A in bucket_100)
    ]
    
    bucket_size = window / buckets
Example:
window = 30d, buckets = 100 → each bucket ≈ 7.2 hours
Returns: float ∈ [-1, 1]
Interpretation:
> 0.8 → Highly synchronous activity (suspicious)
≈ 0.0 → Independent timing
< -0.5 → Anti-correlated (rare, but possible)
3.3 Cost Budget
Definition:
cost_budget(A, window) = Σ cost_difficulty(event_i)
    for all events by A in [now - window, now]

where:
    cost_difficulty(PoW) = 2^difficulty
    cost_difficulty(Stake) = stake_amount
    cost_difficulty(RateLimit) = 1.0  // normalized baseline
Returns: float ≥ 0
Interpretation:
Low budget + high activity → likely Sybil
High budget + high activity → legitimate heavy user
3.4 Graph Distance (Optional, but Recommended)
Definition:
graph_distance(A, B) = shortest_path_length(A, B, identity_graph)

where:
    identity_graph = secondary graph where:
        nodes = all actor public keys
        edge(A, B) exists if overlap(A, B, 30d) > 0.1
        edge_weight = overlap(A, B, 30d)
Returns: int ≥ 0 | ∞
Interpretation:
distance = 1 → Direct collaboration
distance > 5 → Likely independent clusters
distance = ∞ → No connection
Why Optional:
Computationally expensive (requires global view)
Not all clients want to run community detection
But valuable for research/analysis
4. Reference Metric v0
This is the DEFAULT similarity metric — a Schelling point for interoperability.
4.1 Formula
similarity_v0(A, B) = 
    α × overlap(A, B, window=30d)
    + β × temporal_corr(A, B, window=30d)
    + γ × cost_similarity(A, B, window=30d)

where:
    α = 0.5
    β = 0.3
    γ = 0.2
    
    cost_similarity(A, B) = {
        1.0   if cost_budget(A) < threshold AND cost_budget(B) < threshold
        0.5   if exactly one has cost_budget < threshold
        0.0   if both have cost_budget ≥ threshold
    }
    
    threshold = 100 × median_cost_budget(all_actors, 30d)
Returns: float ∈ [0, 1]
4.2 Rationale
Why this weighting?
Term
Weight
Reason
overlap
50%
Most direct signal of coordination
temporal_corr
30%
Catches synchronized campaigns
cost_similarity
20%
Flags cheap bot farms
Why threshold = 100 × median?
Adapts to network conditions
Penalizes actors far below typical cost investment
Avoids absolute thresholds that become outdated
4.3 Non-Prescriptive Nature
Important:
v0 is a REFERENCE, not a REQUIREMENT.
Clients MAY:
Use v0 as-is (recommended for MVP)
Fork v0 → v1, v2, ... with different weights
Create entirely new metrics
Use multiple metrics simultaneously
But clients MUST:
Compute the canonical signals (§3)
Document deviations from v0 (for comparability)
5. Stability Metrics
Now we combine the primitives into a stability score.
5.1 The Three Orthogonal Dimensions
Recall from core design:
Stability(n) = T(n) × I(n) / (1 + C(n))

where:
    T(n) = temporal persistence
    I(n) = independent confirmation
    C(n) = conflict topology
5.2 Temporal Persistence
T(n) = log₂(1 + age_days(n)) × decay_factor(n)

where:
    age_days(n) = (now - n.timestamp) / 86400
    
    decay_factor(n) = exp(-λ × time_since_last_reference(n))
    
    λ = 1 / 365  // half-life of 1 year
    
    time_since_last_reference(n) = 
        min(time since any edge pointed to/from n)
Properties:
Logarithmic age reward (diminishing returns)
Penalty for "dead" nodes (no recent activity)
Self-correcting: old lies with growing contradictions lose T
Example Values:
Age
Last Reference
T(n)
1 day
yesterday
~0.0 × 1.0 = 0.0
30 days
yesterday
~4.9 × 1.0 = 4.9
365 days
yesterday
~8.5 × 1.0 = 8.5
365 days
365 days ago
~8.5 × 0.37 = 3.1
3650 days
yesterday
~11.8 × 1.0 = 11.8
5.3 Independent Confirmation
I(n) = effective_diversity(supporters(n))

where:
    supporters(n) = {all actors who created edges of type 'supports' to n}
    
    effective_diversity = N / (1 + avg_similarity)
    
    avg_similarity = mean(similarity_v0(A, B) for all pairs A, B in supporters)
Alternative (more sophisticated):
I(n) = 1 / Σᵢ pᵢ²    // Simpson Diversity Index

where:
    pᵢ = proportion of supporters in cluster i
    
    clusters = community_detection(identity_graph)  // e.g., Leiden algorithm
Example Values:
Situation
N
Avg Similarity
I(n)
100 independent researchers
100
0.1
~91
100 bots from one farm
100
0.95
~5
50-person echo chamber
50
0.6
~31
5.4 Conflict Topology
C(n) = Σ Stability(m) × weight(edge_type)
    for all m where exists edge(m, n, type=contradicts)

where:
    weight(contradicts) = 1.0
    weight(refines) = 0.1      // mild correction, not full contradiction
    weight(updates) = -0.5     // updates INCREASE stability (evolution)
Recursive Definition:
Like PageRank, this requires iterative computation
Stable contradictions hurt a node
Unstable spam contradictions are ignored
Convergence:
Initialize: C(n) = 0 for all n
Iterate until convergence:
    C_new(n) = Σ Stability(m) for all m contradicting n
    Stability_new(n) = T(n) × I(n) / (1 + C_new(n))
Typically converges in <100 iterations.
5.5 Final Stability Formula
Stability(n) = T(n) × I(n) / (1 + C(n))
Returns: float ≥ 0
Interpretation:
Stability
Meaning
< 1
Unstable (new, contested, or weak support)
1–10
Emerging consensus
10–100
Established knowledge
> 100
Foundational truth
Note: These ranges are heuristic and domain-dependent.
6. Minimum Compliance Requirements
For a client to be "protocol-compliant," it MUST:
6.1 Event Handling
✅ MUST be able to:
Parse Event schema (§2.3)
Verify signatures (Ed25519)
Validate cost proofs (at least one type: PoW, Stake, or RateLimit)
Reject events with invalid proofs
❌ MUST NOT:
Censor events based on content
Require authentication beyond cryptographic signatures
Demand PII (email, phone, IP, real name, etc.)
6.2 Canonical Signal Computation
✅ MUST support:
overlap(A, B, window) for at least one standard window (30d recommended)
cost_budget(A, window)
✅ SHOULD support:
temporal_corr(A, B, window)
graph_distance(A, B) (if claiming "full protocol support")
❌ MAY add additional signals (encouraged)
6.3 Similarity Metric
✅ MUST either:
Use similarity_v0 (§4.1), OR
Document deviations in client metadata
✅ MAY:
Offer multiple similarity options to users
Implement experimental metrics (labeled as such)
6.4 Stability Calculation
✅ MUST compute:
At least T(n) and I(n)
C(n) if claiming full compliance
✅ MAY:
Adjust weights (α, β, γ)
Use different time windows
Apply domain-specific corrections
But MUST disclose these changes in client metadata.
7. Network Architecture (Informative, Not Prescriptive)
The protocol does NOT mandate a specific topology. These are options:
7.1 Fully Decentralized (IPFS-style)
- Events stored in content-addressed storage (IPFS, Dat, BitTorrent)
- Clients sync events via DHT
- No central servers
Pros: Censorship-resistant, permissionless
Cons: Slow bootstrap, garbage collection hard
7.2 Federated (Mastodon-style)
- Instances host subgraphs
- Instances sync via defined protocols (ActivityPub, custom)
- Users choose their home instance
Pros: Easier UX, faster sync
Cons: Instance admins have power
7.3 Hybrid (Recommended for v0)
- Public "anchor" nodes (universities, archives, NGOs) host full history
- Clients sync from anchors + peer-to-peer
- Light clients query via API
- Heavy clients validate locally
Pros: Best of both worlds
Cons: Anchor nodes become targets (but replaceable)
8. Open Questions & Future Work
8.1 Governance of the Protocol
Question: Who decides on updates to this spec?
Proposed Answer (not final):
Reference implementation maintained as open-source
Changes via RFC process (like IETF)
Forks permitted (clients choose which spec version to follow)
8.2 Incentives for Hosting
Question: Why would anyone host Events/nodes?
Options:
Altruism (like Archive.org)
Micropayments (query fees)
Freemium (basic access free, premium analytics paid)
Proof-of-Storage rewards (like Filecoin)
Not decided in v0 — implementations can experiment.
8.3 Privacy vs. Transparency
Tension:
Full transparency → easy Sybil detection
Privacy (Tor, VPNs) → harder to detect coordination
Proposed Balance:
Identity = public key (pseudonymous)
Behavior patterns analyzable (via canonical signals)
No PII required
Clients MAY offer privacy tools (onion routing for events)
8.4 Cross-Domain Stability
Challenge:
A node might be stable in "climate science" but unstable in "climate policy."
Proposed Solution:
Stability is ALWAYS computed relative to a subgraph filter
Clients can ask: "What's the stability of node X in subgraph 'peer-reviewed climate science'?"
No global "Truth Score" — always contextualized
9. Implementation Checklist (MVP)
For someone building the first client:
Phase 1: Local Graph Editor
[ ] Create nodes (hash, timestamp, signature)
[ ] Create edges (typed relations)
[ ] Export as JSON-LD or Protocol Buffers
[ ] Compute overlap() for two actors
Phase 2: Event Log
[ ] Generate Events from user actions
[ ] Validate cost proofs (PoW implementation)
[ ] Sign events with Ed25519
[ ] Store events in append-only log
Phase 3: Canonical Signals
[ ] Implement overlap(A, B, 30d)
[ ] Implement temporal_corr(A, B, 30d)
[ ] Implement cost_budget(A, 30d)
Phase 4: Stability Calculation
[ ] Implement T(n) (temporal persistence)
[ ] Implement I(n) (effective diversity)
[ ] Implement C(n) (iterative conflict resolution)
[ ] Combine into Stability(n)
Phase 5: Sync
[ ] Sync events with another client (local network)
[ ] Verify signatures from remote events
[ ] Reject invalid events
Phase 6: UI
[ ] Visualize graph (D3.js, Cytoscape, etc.)
[ ] Color nodes by Stability
[ ] Show conflicting nodes
[ ] Allow filtering by topic/time
10. Reference Implementation Pseudo-Code
class Event:
    def __init__(self, actor, payload, cost_proof):
        self.actor = actor
        self.payload = payload
        self.cost_proof = cost_proof
        self.timestamp = now()
        self.signature = actor.sign(self.serialize())
        self.event_id = hash(self.serialize() + self.signature)
    
    def validate(self):
        assert self.actor.verify(self.signature, self.serialize())
        assert self.cost_proof.is_valid()

class Graph:
    def __init__(self):
        self.nodes = {}
        self.edges = []
        self.events = []
    
    def ingest(self, event):
        if not event.validate():
            raise InvalidEvent
        self.events.append(event)
        # Update nodes/edges based on event payload
    
    def overlap(self, A, B, window_days=30):
        targets_A = set(self.targets_by(A, window_days))
        targets_B = set(self.targets_by(B, window_days))
        if not (targets_A or targets_B):
            return 0.0
        return len(targets_A & targets_B) / len(targets_A | targets_B)
    
    def stability(self, node_hash, iterations=100):
        # Initialize
        T = self.temporal_persistence(node_hash)
        I = self.effective_diversity(node_hash)
        C = 0.0
        
        # Iterative convergence for C
        for _ in range(iterations):
            C_new = sum(
                self.stability(m, iterations=0)  # one-step recursion
                for m in self.contradicts(node_hash)
            )
            if abs(C_new - C) < 0.01:
                break
            C = C_new
        
        return (T * I) / (1 + C)
11. Versioning
This is Protocol Specification v0.1
Future versions will:
Be backward-compatible where possible
Introduce new canonical signals (opt-in)
Refine cost proof mechanisms
Add optional privacy features
Clients MUST declare which spec version they support in metadata.
12. License & Contributions
Proposed License: CC0 (public domain) or MIT
Contributions:
Anyone can fork this spec
Improvements welcome via RFC process
Reference implementation: open-source (language TBD)
Appendix A: Worked Example
Scenario: A new COVID treatment claim appears.
Step 1: Node Creation
Node {
    hash: "abc123...",
    content: "Ivermectin reduces COVID mortality by 50%",
    creator: PublicKey(Alice),
    timestamp: 2025-02-01,
    metadata: { type: "text", topic: "covid", language: "en" }
}
Step 2: Early Support
Bob creates edge: supports(Bob → abc123)
Carol creates edge: supports(Carol → abc123)

At t=1 week:
    T(abc123) = log₂(8) × 1.0 = 3.0
    I(abc123) = 2 / (1 + similarity(Bob, Carol))
              = 2 / (1 + 0.2) = 1.67
    C(abc123) = 0  (no contradictions yet)
    
    Stability = 3.0 × 1.67 / 1 = 5.0 (emerging)
Step 3: Contradictions Emerge
Dave creates node: "Meta-analysis shows no effect"
Eve creates edge: contradicts(Dave's node → abc123)

At t=1 month:
    T(abc123) = log₂(31) × 1.0 = 4.95
    I(abc123) = 2 / (1 + 0.2) = 1.67  (unchanged)
    C(abc123) = Stability(Dave's node) = 8.0  (well-supported meta-analysis)
    
    Stability = 4.95 × 1.67 / (1 + 8.0) = 0.92 (now unstable)
Step 4: Long-term Outcome
At t=1 year:
    Original node has no new supports, many contradictions
    T(abc123) = log₂(366) × exp(-1.0) = 8.5 × 0.37 = 3.1
    C(abc123) = 50 (many stable contradictions)
    
    Stability = 3.1 × 1.67 / 51 = 0.10 (discredited)
Meanwhile, the meta-analysis node:
T(meta) = 8.5 × 1.0 = 8.5  (still active)
    I(meta) = 50 / (1 + 0.15) = 43.5  (50 diverse supporters)
    C(meta) = 0.5  (weak contradictions)
    
    Stability = 8.5 × 43.5 / 1.5 = 246 (foundational)
Appendix B: Attack Resistance Analysis
Attack Vector
Defense Mechanism
Attacker Cost
Sybil farm (1000 bots)
Low I(n) via similarity detection
1000× PoW cost
Astroturfing
Temporal correlation + cost budget
Coordination overhead
Slow poison (10-year lie)
C(n) via stable contradictions
10 years × ongoing cost
DDoS (spam events)
Cost proofs rate-limit
Computational/capital cost
Censorship (delete node)
Immutable log
Not possible
Reputation farming
Time-weighted T(n)
Time (not buyable)
Only "slow poison" works — and it's the most honest attack (building a coherent alternative narrative over years).
Appendix C: Comparison to Existing Systems
System
Similarity
Difference
Wikipedia
Collaborative knowledge
Centralized, deletable, editor hierarchy
Blockchain
Immutable ledger
No semantic relations, just transactions
Semantic Web (RDF)
Typed relations
No stability metric, no cost mechanism
PageRank
Graph-based importance
Measures attention, not robustness
Git
Versioned content
No cross-repo relations, no stability
IPFS
Content-addressed storage
No relations, no meaning layer
This protocol = semantic web + blockchain properties + evolutionary selection.
End of Specification v0.1
