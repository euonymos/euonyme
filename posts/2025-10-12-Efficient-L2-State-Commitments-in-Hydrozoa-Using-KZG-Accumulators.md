---
title: Efficient L2 State Commitments in Hydrozoa Using KZG Accumulators
---

## What is Hydrozoa?

Hydrozoa is a Hydra-like state channel protocol for groups of people to manage funds and information via a virtual Layer 2 ledger with rules they collectively enforce. The group holds funds collectively on Cardano's Layer 1, but in the L2 ledger, funds can be allocated to any individual, group, or script.

Transactions in Hydrozoa's L2 are confirmed as soon as every peer signs off that they comply with the ledger rules, enabling quick, cheap, and private transactions. However, if peers cannot reach consensus, Hydrozoa shifts to a "rule-based regime" where funds are controlled by Plutus scripts. In this regime, each peer votes on the latest confirmed L2 state, and funds can be withdrawn according to that state without requiring all signatures.

This is where KZG accumulators become essential—representing the L2 UTXO state succinctly so it can be verified on-chain efficiently.

## The Problem: Representing L2 UTXO State

Layer 2 solutions face a fundamental challenge: how do you prove the current state of thousands of UTXOs to the Layer 1 blockchain without posting all of them on-chain? For Hydrozoa, every block on our L2 needs to create a succinct, verifiable representation of the entire UTXO set that can be efficiently verified by Cardano's base layer.

Traditional approaches like Merkle trees seem appealing at first—they provide logarithmic proof sizes and are well-understood. However, updating the tree structure is somewhat inconvenient: every time UTXOs are removed (when users withdraw to L1 in Hydrozoa's rule-based regime—see [specification](https://cardano-hydrozoa.github.io/hydrozoa/hydrozoa.pdf)), you must recalculate portions of the tree, update the root hash, and recompute many intermediate nodes—all of which must happen on-chain in the validator.

We chose KZG commitments because they offer a crucial property: **the membership proof itself is the new accumulator state**. When users withdraw UTXOs from our L2:

1. They provide a KZG proof that their UTXOs exist in the current accumulator
2. L1 verifies this proof
3. This proof becomes the new accumulator state with those UTXOs removed

No tree recalculation. No recomputing intermediate nodes. The proof and the state update are one and the same operation. Additionally, both the commitment and the proof are constant-size—just 48 bytes each (a single BLS12-381 G1 point)—regardless of how many UTXOs are in the set.

However, KZG accumulators come with their own challenges: **computing the commitment itself is expensive** (every L2 block needs a fresh commitment representing the current UTXO set), and **they require a trusted setup**—a one-time ceremony to generate the cryptographic parameters needed for the scheme. As we'll see, we address the setup requirement by leveraging Ethereum's existing ceremony, while the commitment computation becomes the main bottleneck we need to optimize.

## What is a KZG Commitment?

KZG (Kate-Zaverucha-Goldberg) commitments are a type of polynomial commitment scheme. Here's the intuition:

Imagine you have a set of UTXOs. You can encode this entire set as a polynomial—a mathematical function where each UTXO contributes to the polynomial's structure. A KZG commitment is a cryptographic "fingerprint" of this polynomial that:

- **Compresses the entire set** into a single point on an elliptic curve (just 48 bytes for BLS12-381)
- **Allows proving membership** of any UTXO in the set with a small proof
- **Prevents tampering** - you cannot create false proofs without breaking hard cryptographic problems

The beauty of KZG is that it uses a one-time trusted setup (more on this below) to enable these efficient proofs. The commitment itself is computed as:

```
commitment = g₁^f(τ)
```

where `f(x)` is your polynomial encoding the UTXO set, `g₁` is a generator point on the elliptic curve, and `τ` (tau) is a secret value from the trusted setup that nobody knows.

## The Setup: Learning from Ethereum's KZG Ceremony

To use KZG commitments, you need a trusted setup—a one-time ceremony that generates public parameters. Rather than running our own ceremony, Hydrozoa uses the parameters from [Ethereum's KZG Ceremony](https://ceremony.ethereum.org/) (also called the "Powers of Tau" ceremony).

This 14-month ceremony involved over 141,000 contributions from participants across the globe, using diverse equipment and multiple independent software implementations. This massive collaborative effort produced a highly secure setup for KZG-BLS commitments—as long as at least one participant honestly deleted their secret contribution, the setup is secure.

This setup is **universal**, meaning it can be used by any application that needs polynomial commitments up to a certain degree. The Ethereum ceremony generated:

- **32,768 powers in G1**: g₁^τ⁰, g₁^τ¹, g₁^τ², ..., g₁^τ³²⁷⁶⁷
- **65 powers in G2**: g₂^τ⁰, g₂^τ¹, g₂^τ², ..., g₂^τ⁶⁴

This allows us to support L2 UTXO sets with up to 32,000 UTXOs, which should be sufficient for most Hydrozoa heads in typical usage. And if the 32,000 limit becomes restrictive, we have ideas for expanding capacity while reusing the same setup parameters.

### Why the Asymmetry?

You might wonder: why 32,768 elements in G1 but only 65 in G2? This design choice reflects where the computational bottlenecks actually occur in practice.

In standard KZG schemes:

- **Commitments are computed in G1** - this happens frequently (every block in our case)
- **Verification uses G2** for pairing checks - this happens less frequently

Elliptic curve operations in G2 are roughly **3× slower** than in G1 on the BLS12-381 curve. By putting the large setup (32k elements) in G1, Ethereum optimized for the frequent operation (commitment computation) while keeping G2 small since verification happens rarely.

This asymmetry is perfect for Hydrozoa's needs: we compute commitments frequently (every minor block) but verification by L1 validators happens less often. We benefit from:

- **Fast G1 arithmetic** for computing commitments
- **Large G1 setup** supporting up to 32,768 UTXOs
- **Reusing battle-tested parameters** from Ethereum's ceremony

## Current Approach: The MVP Strategy

For our MVP, Hydrozoa targets L2 virtual ledgers with **1,000-3,000 UTXOs**. We're using a straightforward but effective approach: rebuild the commitment from scratch for each block.

### The Commitment Pipeline

Computing a KZG commitment for the UTXO set involves several steps:

1. **Take all UTXOs** in the current L2 state
2. **Convert to Plutus representation** (so L1 validators can verify membership proofs on-chain)
3. **Convert to `Data` and serialize into `ByteString`**
4. **Hash each UTXO** using blake2b-224
5. **Build the polynomial** from these hashes
6. **Evaluate** the commitment using the trusted setup

Steps 1-4 are standard serialization and hashing—relatively fast operations. The real computational work happens in steps 5 and 6.

### Performance Characteristics

Here's what our benchmarks show for the naive approach:

**For 1,000 UTXOs:**

- Hashing (steps 1-4): ~50ms
- Building polynomial: ~265ms
- Evaluating commitment: ~104ms
- **Total: ~420ms**

**For 10,000 UTXOs:**

- Hashing: ~307ms
- Building polynomial: ~23 seconds ⚠️
- Evaluating commitment: ~1.3 seconds
- **Total: ~25 seconds**

The bottleneck is clear: building the polynomial has O(N²) complexity with our current naive implementation. For 1-3k UTXOs this is acceptable, but it won't scale to larger sets.

**Note on JVM optimizations**: Hydrozoa is implemented in Scala using [Scalus](https://github.com/nau/scalus), a full-fledged Cardano development framework. Hydrozoa leverages Scalus' ledger rules for its custom L2 ledger, uses it for building transactions, and uses Scalus for its on-chain scripts. One advantage of running on the JVM is that the Just-In-Time (JIT) compiler significantly optimizes repetitive calculations like hashing—our benchmarks show substantial improvements after warm-up iterations as the JVM optimizes hot code paths. This means that in production, where the same hashing operations occur frequently across many blocks, performance will be better than these cold-start benchmarks suggest.

### Near-Term Optimization

The good news: we can easily improve hashing performance by computing hashes upfront when L2 transactions are first processed, storing them alongside UTXOs in the ledger state. This eliminates redundant hashing work, complementing the JIT optimizations mentioned above.

With this optimization, commitment time for 1,000 UTXOs drops to approximately **350ms**—entirely acceptable for L2 block production.

If we need to further optimize the bottleneck steps (building the polynomial and evaluating the commitment) in the MVP phase, we have straightforward optimizations available:

- **Divide-and-conquer** for polynomial construction (still O(N²) but with better constant factors)
- **Parallelization** across multiple cores
- **Multi-scalar multiplication** (MSM) optimizations using the `blst` library (potential 25× speedup)

## On-Chain Verification: Bringing KZG to Plutus

While computing KZG commitments happens off-chain in our L2, the Cardano base layer needs to verify membership proofs when users withdraw UTXOs back to L1.

### How Verification Works

When users submit a withdrawal transaction:

1. **The redeemer specifies** which UTXOs to withdraw (by their refs: transaction IDs and indices)
2. **The validator rebuilds the commitment pipeline in G2**: It takes the withdrawal UTXOs, converts them to Plutus representation, serializes them, hashes them, and builds a polynomial—all the same steps as commitment generation, but now computing in G2 instead of G1
3. **Pairing verification**: The script checks that the withdrawals were actually in the accumulator using a pairing equation:

   ```
   e(acc, g₂) = e(proof, g₂^withdrawal)
   ```

   where `acc` is the stored accumulator (in G1), `proof` becomes the new accumulator state after withdrawal, and `g₂^withdrawal` is the G2 commitment to the withdrawal polynomial.

This verification guarantees that the withdrawn UTXOs existed in the L2 state, and the proof simultaneously becomes the new accumulator representing the remaining UTXOs.

### The Challenge: Limited G2 Setup

Here's where we hit a constraint: the Ethereum ceremony only generated 65 powers in G2. This means the withdrawal polynomial is limited to degree 64, which practically restricts us to withdrawing around 60-65 UTXOs per transaction.

For the MVP targeting heads with 1-3k UTXOs, this is acceptable—users can batch reasonable numbers of withdrawals. However, for larger-scale usage, this becomes a bottleneck.

### The Need for Efficient Primitives

Plutus does have support for BLS12-381 curve operations, though they're not as widely used in the ecosystem yet. More critically, efficient finite field arithmetic primitives are still needed.

### The Challenge: Plutus Arithmetic Operations

KZG verification requires operations in finite fields—specifically, arithmetic modulo the large prime that defines the scalar field of the BLS12-381 curve. Unfortunately, Plutus currently lacks native support for modular arithmetic over large integers.

This means we must implement these operations manually in Plutus script, which consumes significant execution budget and limits how many UTXOs can be verified per transaction.

### CIP Proposals for Efficient On-Chain Verification

To address this, we've submitted [a CIP proposal](https://github.com/cardano-foundation/CIPs/pull/1087) for native Plutus primitives for finite field arithmetic based on **Montgomery multiplication**—an efficient algorithm specifically designed for modular arithmetic.

Montgomery multiplication is the standard approach used in high-performance cryptography libraries. Instead of performing expensive modular reduction after each operation, it works in a transformed "Montgomery space" where operations are cheaper, only converting back to normal representation when needed.

With native Montgomery multiplication in Plutus, KZG verification becomes dramatically more efficient, allowing significantly more UTXO proofs per transaction and reducing L1 costs.

Complementing our finite field primitives, [CIP-0133](https://github.com/cardano-foundation/CIPs/blob/master/CIP-0133/README.md) proposes native **multi-scalar multiplication (MSM)** for elliptic curves. During verification, we need to compute expressions like `a₁·P₁ + a₂·P₂ + ... + aₙ·Pₙ`. The naive approach processes each term sequentially in a loop, while MSM algorithms (like Pippenger's) compute this much more efficiently by batching operations.

Together, these CIPs transform on-chain verification from an expensive operation requiring manual implementation into an efficient, practical one with native cryptographic primitives. This benefits not just Hydrozoa, but any Cardano project using elliptic curve cryptography—zkSNARKs, other L2s, bridges, and more.

### Making KZG Commitments Practical on Cardano

By submitting these CIPs, we're working to make KZG commitments practical and efficient on Cardano. Once implemented, any project can leverage:

- Efficient polynomial commitments
- Succinct state proofs
- Standardized verification logic

This creates a foundation for a new generation of scalability and cryptographic applications on Cardano.

## Future Work: Scaling to Larger UTXO Sets

For UTXO sets beyond a few thousand elements, we need incremental commitment updates rather than rebuilding from scratch. We're exploring two complementary approaches:

### 1. Hot-Spot Optimization

In practice, most ledger activity concentrates in a "hot spot"—newer UTXOs are much more likely to be spent than older ones. Hydrozoa's consensus provides total ordering of L2 transactions, letting us:

- **Sort UTXOs by age** into generations (similar to generational garbage collectors)
- **Reuse commitments** for unchanged "cold" portions of the ledger
- **Only recompute** commitments for the active "hot" region

Think of it like a database index: you don't rebuild the entire index when you add a few records, you update only the affected parts.

### 2. Incremental Updates with NTT

Each block removes some UTXOs (spent) and adds others (created). Instead of rebuilding the entire polynomial, we could:

- **Start with** the previous block's polynomial f_{n-1}(x)
- **Divide out** spent UTXOs (remove their corresponding factors)
- **Multiply in** new UTXOs (add their factors)

The challenge: polynomial division is slow in the standard coefficient representation we currently use.

**Enter NTT**: The Number-Theoretic Transform (NTT) is essentially the Fast Fourier Transform (FFT) adapted for finite fields. It converts polynomials between two representations:

- **Coefficient form**: [a₀, a₁, a₂, ...] representing a₀ + a₁x + a₂x² + ...
- **Point-value form**: [f(ω⁰), f(ω¹), f(ω²), ...] the polynomial evaluated at special points

In point-value form, multiplying and dividing polynomials becomes pointwise operations on the arrays. However, division in point-value form requires that divisor polynomials don't evaluate to zero at the chosen points. We expect this can be managed through careful choice of evaluation points, or as a fallback, we can perform a full rebuild when a zero divisor is encountered. The BLS12-381 curve was specifically designed to support efficient NTT operations.

Our incremental approach would work like this:

1. **First block**: Build initial polynomial and convert to point-value form using NTT
2. **Each subsequent block**:

    - Build small polynomials for added and removed UTXOs
    - Convert them to point-value form (fast, only depends on their size)
    - Perform pointwise multiplication/division (very fast)
    - Convert back to coefficient form using inverse NTT
    - Compute commitment using multi-scalar multiplication

NTT operations run in **O(n log n)** time—a massive improvement over O(n²) for large sets. This would let us scale to tens of thousands of UTXOs while keeping block times reasonable.

## The Path Forward

Hydrozoa's approach to L2 state commitments reflects a practical engineering philosophy:

**For MVP (1-3k UTXOs)**: Use the simplest approach that works. Rebuilding commitments from scratch is straightforward to implement and debug. Performance is acceptable with basic optimizations.

**For scaling (10k+ UTXOs)**: Invest in incremental updates using NTT-based polynomial arithmetic. The BLS12-381 curve and Ethereum's trusted setup give us excellent tools for this.

By leveraging Ethereum's KZG ceremony and the mathematical properties of BLS12-381, we can provide succinct, verifiable L2 state commitments that scale with Cardano's growing ecosystem.

## Learn More

- **Hydrozoa Project**: [cardano-hydrozoa.github.io/hydrozoa](https://cardano-hydrozoa.github.io/hydrozoa)
- **Hydrozoa Specification**: [hydrozoa.pdf](https://cardano-hydrozoa.github.io/hydrozoa/hydrozoa.pdf)
- **Project Catalyst**: [Hydrozoa Milestones](https://milestones.projectcatalyst.io/projects/1200059/milestones)
- **CIP Draft - Native Finite Field Arithmetic**: [github.com/cardano-foundation/CIPs/pull/1087](https://github.com/cardano-foundation/CIPs/pull/1087)
- **Ethereum KZG Ceremony**: [ceremony.ethereum.org](https://ceremony.ethereum.org/)
- **KZG Commitments Explained**: [dankradfeist.de/ethereum/2020/06/16/kate-polynomial-commitments.html](https://dankradfeist.de/ethereum/2020/06/16/kate-polynomial-commitments.html)
- **BLS12-381 Curve**: [hackmd.io/@benjaminion/bls12-381](https://hackmd.io/@benjaminion/bls12-381)

---

*Hydrozoa is a Hydra-like state channel protocol designed by [George Flerovsky](https://x.com/SwiftlyUnmoving) 
and funded by Catalyst Fund 12.
We're grateful to the Cardano community for making this work possible. 
Hydrozoa is building scalable Layer 2 infrastructure for Cardano using modern cryptographic techniques.*
