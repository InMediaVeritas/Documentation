# inMediaVeritas

A decentralized news verification application (DApp) that combats misinformation through cryptographic provenance, privacy-preserving community voting, and on-chain consensus — without any central arbiter of truth.

> For the full theoretical background, cryptographic design, and formal evaluation see the accompanying paper **inMediaVeritas (2026)** by Jakub Ivácek, Patrik Baran, Tomáš Tisovský, Vojtech Babinský, Tomáš Miština, Matúš Hrkeľ, Richard Gazdík, Ivan Homoliak — STU FIIT, Slovakia Bratislava.
>
> Live demo & landing page: https://jakubivacek.github.io/inMediaVeritas/

---

## What it does

inMediaVeritas builds a trustless pipeline from raw media capture to verified, licensable content through three ideas:

- **Cryptographic provenance** — media is hashed and its IPFS content identifier (CID) is committed on-chain, so any tampering is immediately detectable.
- **Privacy-preserving community verification** — geographically eligible users vote on the credibility of reported media, proving their eligibility with **zk-SNARKs** without revealing who they are or exactly where they are.
- **No central arbiter of truth** — credibility is established by on-chain consensus rather than by an editorial board.

### Three actor tiers

| Tier | App role | What they do |
|---|---|---|
| Reporter ("roaming / journalist") | `journalist`, `newsAgency` | Capture media, attach a location, publish an event on-chain |
| Witness ("local user") | `localUser` | Confirm or deny an event by voting, proving local eligibility with a zk-SNARK |
| Reader / News Agency | `reader`, `newsAgency` | Browse verified events and their vote breakdown; purchase licensed access |

---

## Architecture

```
Authority (EdDSA VC issuer)
       │ 1. Issue Verifiable Credential
       ▼
   ID App (identity wallet)
       │ 2. User holds VC + wallet
       ▼
IMV FullApp (report / vote / zk-proof generation)
       │  ◄──► Media Voting Backend (IPFS relay, read-only REST API)
       │                    │
       │           reads on-chain state
       ▼                    ▼
  Anvil RPC Node  ◄─────────
  (MediaVoting.sol + Groth16 verifiers)
```

**Key architectural invariant:** the mobile wallet signs every chain write. The backend is a read-only convenience relay — it can neither forge media nor influence vote outcomes.

---

## Repository layout

This project is **not a single git repository**. Each component directory is its own independent git repo:

| Directory | Role | Stack |
|---|---|---|
| `media-voting-smart-contract/` | `MediaVoting.sol` — on-chain events, media items, votes, ZK verifier hooks | Solidity + Foundry |
| `media-voting-backend/` | Read API for on-chain events + IPFS upload/gateway relay; local infra via Docker Compose | TypeScript, Express, ethers v6 |
| `IMV_FullApp/` | Main voting/reporting app — npm-workspaces monorepo (`IMV_app` UI + `IMV_ClientLogic` + `packages/shared-types`) | Expo / React Native |
| `ID_app/` | Identity-credential wallet (camera ID capture, wallet + ZK key generation, holds the VC) | Expo / React Native |
| `AUTHORITY/` | Issuer/official portal that attests users and issues Verifiable Credentials | Node.js + Express, Vue 3 + Vite, SQLite |
| `ZK-SNARKs/` | Circom circuits: `identity-verifier` and `location-verifier` | Circom + snarkjs (Groth16) |

---

## How it works end-to-end

### A. Onboarding & credential issuance
1. User opens **ID_app**, generates a blockchain wallet (secp256k1) and ZK keys (BLS12-381), photographs their ID document, and fills in the credential fields.
2. ID_app submits an `issue-vc-request` (with ID images) to **AUTHORITY**.
3. An official logs into the AUTHORITY Vue frontend, reviews the request, and accepts it.
4. AUTHORITY computes `vcHash = Poseidon(address, cityHash, districtHash, metaHash)` and signs it with an EdDSA/Baby-JubJub key; the signed VC is stored and ID_app is notified via FCM.
5. ID_app polls `get-vcs/:request_id`, receives the signed VC, and stores it on the device.

### B. Reporting an event (journalist role)
6. Reporter captures media, fills in event metadata and location.
7. IMV_app uploads media to **IPFS** via `POST /api/ipfs/images` and collects the CIDs.
8. Reporter generates an **identity zk-SNARK proof** on-device via `generateProof`.
9. Mobile client broadcasts `createEvent(...)` — with media CIDs and the identity proof — directly to the **Anvil RPC**. `MediaVoting.sol` stores the event and emits `EventCreated` / `MediaItemAdded`.

### C. Witness voting (local-user role)
10. Witness browses events via the backend `/api/events`.
11. IMV_app builds the circuit input from the stored VC and the event's required locality, then calls `generateProof(...)` to produce a Groth16 proof **on-device** (under 300 ms on recent phones).
12. **Demo mode:** wallet calls `voteOnMedia(eventId, mediaId, voterHash, support, isLocalVoter)`. **ZK mode:** wallet calls `voteOnMediaWithProof(...)` and `MediaVoting.sol` verifies the proof through the verifier adapter before counting the vote.
13. The contract deduplicates by `voterHash`, updates the global/local tallies, and emits `MediaVoted`.

### D. Consumption (reader / agency role)
14. Readers and agencies browse verified events and the per-media vote breakdown through the backend read API; media is fetched from the IPFS gateway by CID.

---

## Cryptographic design

| Concern | Mechanism | Where |
|---|---|---|
| Credential authenticity | EdDSA signature over Poseidon hash, Baby JubJub curve | AUTHORITY signs; circuits verify |
| Credential privacy | zk-SNARK (Groth16, BN254) proves VC validity without revealing attributes | ZK-SNARKs/, on-device |
| Locality eligibility | Location circuit enforces `vc_city == req_city`, `vc_district == req_district` privately | `LocationCircuit.circom` |
| Vote deduplication | `voterHash` (bytes32) keyed mapping per media item | `MediaVoting.sol` |
| Vote integrity | On-chain transaction signature (secp256k1 wallet) | mobile wallet → RPC |
| Media integrity | IPFS CID is the content hash; SHA-256 committed on-chain | IPFS + contract `uri` |
| Key custody | Secure Enclave / Keystore via `expo-secure-store`; biometric gate | ID_app |

### zk-SNARK circuits (Circom 2.1.5, Groth16, BN254)

**Identity circuit** (`IdentityCircuit.circom`) — proves the prover holds a VC validly signed by the authority and bound to their address.

**Location circuit** (`LocationCircuit.circom`) — proves credential validity *and* that the credential's locality matches the requested filter (city/district), without revealing the exact attributes.

On-chain proof verification costs ~215k–228k gas per proof. On mobile (Pixel 6 Pro / Samsung S23 Ultra) end-to-end proving completes in **187–277 ms**.

---

## Technology stack

- **Smart contracts:** Solidity ^0.8.13, Foundry (`forge`), optimizer + `via_ir`
- **Zero-knowledge:** Circom 2.1.5, circomlib (Poseidon, EdDSA), Groth16, snarkjs (desktop), `@iden3/react-native-circom-witnesscalc` + `@iden3/react-native-rapidsnark` (mobile)
- **Backend:** Node.js, TypeScript (ESM), Express 5, ethers v6, `ipfs-http-client`, multer; Docker Compose (Anvil + Kubo IPFS + API)
- **Mobile (IMV_app & ID_app):** Expo SDK 54, React Native 0.81, React 19, expo-router; ethers v6 / v5; `@noble/curves`, `poseidon-lite`, `expo-secure-store`, `expo-local-authentication`, Firebase (ID_app)
- **Authority:** Node.js + Express, Vue 3 + Vite, SQLite, circomlibjs, bcrypt, JWT, Zod, Firebase Admin
- **Storage:** IPFS (Kubo); **Chain:** local Anvil dev chain (chain ID 31337)
- **Host environment:** Windows 11 with WSL 2 (Ubuntu) for Foundry; PowerShell automation scripts

---

## Implementation status

| Capability | Status |
|---|---|
| `MediaVoting` contract (events, media, per-media voting, dedup, local/global tallies) | Implemented & tested |
| ZK verifier hooks + mode switch (`setZKVerifier`, `setZKMode`, mock verifier) | Implemented |
| Identity & location Circom circuits + Groth16 proving (desktop & mobile, benchmarked in paper §VIII) | Implemented |
| On-device proof generation (`witnesscalc` + `rapidsnark`) | Implemented — **identity circuit only** |
| Location proof wired into mobile app | Not yet — circuit exists and is benchmarked, but not integrated into on-device proving |
| IPFS upload/read relay + read-only event API | Implemented |
| Identity wallet (camera capture, manual entry, wallet + ZK keys, VC storage, biometrics, FCM) | Implemented |
| Automatic OCR text recognition in the wallet | Stub only (`extractTextFromImage` returns empty; awaiting cloud OCR integration) |
| Authority portal (review, EdDSA VC issuance, notifications) | Implemented |
| Demo (non-ZK) voting path | Working end-to-end |
| Full ZK voting path with real on-chain Groth16 verification | Partial — identity proof path implemented; location proof and combined ZK voting not yet wired |
| Incentive pool / subscription economics | Design only |
| News agency subscription & licensing — payments | Not implemented (no on-chain payment, no licensing backend) |

---

## Limitations & future work

- **Sparse witness coverage** — correctness depends on enough geographically proximate witnesses being present; quorum/resolution logic is not yet implemented.
- **Authority centralization risk** — credential issuance reintroduces some centralization at the identity layer; federated governance and cross-authority credential recognition are future work.
- **Backend relay dependency** — IPFS uploads/reads route through the backend for convenience; it is not a trust anchor but remains an availability dependency. Durable/remote pinning (Filecoin, Arweave) is future work.
- **Vote-signature gap in the location circuit** — binding the vote to the user's key inside the circuit is disabled because the wallet key is secp256k1; vote integrity currently relies on the on-chain transaction signature.
- **Incentive layer** — rewards, staking, and slashing are designed but not coded.
- **Scalability** — single L1/L2 deployment; high-volume news cycles would stress gas and pinning; Layer-2 rollups and sharded IPFS pinning clusters are natural directions.

---

## Glossary

| Term | Definition |
|---|---|
| **VC** (Verifiable Credential) | A signed claim about a user issued by the authority and held in the ID_app wallet |
| **voterHash** | A `bytes32` value derived from the VC, used on-chain to deduplicate votes without revealing identity |
| **zk-SNARK / Groth16** | A succinct zero-knowledge proof system; used to prove VC validity and locality without revealing private attributes |
| **Poseidon** | A hash function efficient inside arithmetic circuits |
| **EdDSA / Baby JubJub** | The signature scheme/curve the authority uses, verifiable inside Circom |
| **BN254 (alt_bn128)** | The proving curve, with native Ethereum precompiles for cheap on-chain verification |
| **CID** | IPFS content identifier; the content hash that addresses a stored file |
| **Anvil** | Foundry's local Ethereum development node |
| **Kubo** | The reference Go implementation of an IPFS node |
