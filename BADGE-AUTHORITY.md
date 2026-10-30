

### Design in BADGE-AUTHORITY (Conceptual)

From `ZERO-KNOWLEDGE BADGE VALIDATION.md` and `vertical tracks.md`:

1. **Badge JSON → canonical hash** (SHA-256 off-chain).
2. Hash is inserted into a **Merkle tree**.
3. Only the **Merkle root** is stored on-chain.
4. A user generates a **zero-knowledge proof of Merkle inclusion** (proves their badge hash is in the tree without revealing the badge contents or which leaf it is).
5. Smart contract verifies the proof against the public root.

**Conceptual circuit structure** (from the docs):

```
Inputs (private):
  - leafHash          // the badge hash
  - pathElements[]    // Merkle siblings
  - pathIndices[]     // left/right path bits

Input (public):
  - root              // current Merkle root on-chain

Constraint:
  MerkleProof(leafHash, pathElements, pathIndices) == root
```

There is also a simplified credit-score example (not pure badge hashing):

```circom
// Simplified example from the docs (credit_score.circom style)
template CreditScoreThreshold() {
    signal input creditScore;
    signal input threshold;
    signal output isAboveThreshold;

    // Proves score > threshold without revealing the actual score
    isAboveThreshold <-- creditScore > threshold ? 1 : 0;
}
```

### Related Project: ZK-5D Cryptographic Badge Authority

The companion repository [ZK-5D-Cryptographic-Badge-Authority-app](https://github.com/GitDigital-Solana/ZK-5D-Cryptpgrapgic-Badge-Authority-app) describes more concrete circuits:

| Circuit              | Purpose                                      | Key Inputs (examples)                          |
|----------------------|----------------------------------------------|------------------------------------------------|
| `identity.circom`    | Prove ownership of a GitHub identity         | githubId (public), privateKey (private)        |
| `badgeProof.circom`  | Prove contribution criteria are met          | mergedPRs / reviews (private), threshold (public) |
| `revocation.circom`  | Prove a badge is still valid (not revoked)   | current timestamp (public), revocation time (private) |

These are documented but the actual `.circom` source files do not appear to be present on the main branch (paths return 404).

### Summary

| Aspect                        | Status in BADGE-AUTHORITY                  |
|-------------------------------|--------------------------------------------|
| Badge hashing circuit         | Conceptual only (Merkle inclusion)         |
| Actual `.circom` files        | None present                               |
| Hashing method                | Off-chain SHA-256 → Merkle leaf            |
| Proof type                    | ZK Merkle membership proof                 |
| On-chain component            | Root storage + verifier contract           |
| Credit / threshold proofs     | Sketch only (`credit_score.circom` style)  |

If you need production-ready circuits, the standard approach is to use **circomlib** (or **zk-kit**) Merkle inclusion templates + Poseidon hashing (preferred over SHA-256 inside circuits for efficiency). The current docs describe the architecture but do not ship the circuits.




---






Here are the ZK circuits created for **GitDigital Badge Authority** badge hashing.

### Created files

```
circuits/
├── badge_merkle_inclusion.circom   # Core Merkle inclusion proof
├── badge_commitment.circom         # Poseidon commitment of badge fields → leaf
├── badge_threshold.circom          # Prove value ≥ threshold (score / count)
├── badge_full_proof.circom         # Recommended: commitment + Merkle + threshold
└── README.md                       # Full usage, compilation & trusted-setup guide
```

### What each circuit does

| Circuit | Purpose |
|---------|---------|
| **badge_merkle_inclusion** | Proves a private `badgeHash` is a leaf in a Merkle tree whose `root` is public. This is the exact design described in the Badge Authority docs. |
| **badge_commitment** | Builds a Poseidon commitment from badge fields (`badgeIdHash`, `recipientHash`, `issuedAt`, `version`). The output becomes the Merkle leaf. |
| **badge_threshold** | Proves a private score/count is ≥ a public threshold without revealing the exact value (matches the `credit_score.circom` sketch in the docs). |
| **badge_full_proof** | **Production circuit** – combines commitment + Merkle inclusion + threshold check in a single proof. |

### Design alignment

- Off-chain: canonicalize badge JSON → hash / commitment  
- Insert commitment into Merkle tree → publish root on-chain  
- User generates ZK proof of membership (+ optional score threshold)  
- On-chain verifier checks the proof against the public root  

All circuits use **Poseidon** (efficient inside R1CS) and force path indices to be binary. Default tree depth is **20** (~1 M leaves).

