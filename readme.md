# 🔐 Hybrid On-Chain Encryption

A self-contained web app demonstrating secure, end-to-end encryption for Ethereum wallets using X25519 keypairs, on-chain public-key registration, and off-chain message envelopes. This document provides a deep dive into the underlying algorithms, key management strategies, feature set, usage patterns, and security considerations.

---

## Table of Contents

1. [Algorithm Overview](#algorithm-overview)  
2. [Keypair Generation & Recovery](#keypair-generation--recovery)  
3. [On-Chain Public Key Registry](#on-chain-public-key-registry)  
4. [Hybrid Envelope Construction](#hybrid-envelope-construction)  
5. [Browser App Features](#browser-app-features)  
6. [User Workflow & Usage](#user-workflow--usage)  
7. [Key Handling & Security](#key-handling--security)  
8. [Future Enhancements](#future-enhancements)  

---

## Algorithm Overview

1. **X25519 Key Exchange**  
   - Each user holds an X25519 keypair `(pk, sk)`. X25519 is a widely used elliptic-curve Diffie–Hellman (ECDH) scheme on Curve25519, chosen for performance and resilience.  

2. **ECDH to Derive Symmetric Key**  
   - To send a message to recipient with public key `pk_rec`, the sender generates an ephemeral X25519 keypair `(epk, esk)`.  
   - Both parties compute a shared secret:  
     ```
     shared = X25519(esk, pk_rec)
     ```  
   - The shared secret is hashed (SHA-256) to produce a 32-byte symmetric key `K`.  

3. **Symmetric Encryption (Secretbox)**  
   - The plaintext is encrypted with `K` using NaCl’s `secretbox` (XSalsa20-Poly1305 AEAD).  
   - A random nonce is prepended to the ciphertext; the result is Base64-encoded as the **body cipher**.

4. **Envelope for Key Transport**  
   - The sender’s ephemeral public key `epk` is prepended to the nonce and ciphertext to form the **envelope** for each recipient.  
   - Recipients recover `K` via `X25519(sk_rec, epk)`, then decrypt the body cipher.

<p align="center">
  <a href="https://www.youtube.com/watch?v=F3zzNa42-tQ">
    <img src="https://img.youtube.com/vi/F3zzNa42-tQ/0.jpg" alt="▶ Watch the video">
  </a>
  <br>
  <a href="https://www.youtube.com/watch?v=F3zzNa42-tQ">Click to view on YouTube</a>
</p>
---

## Keypair Generation & Recovery

- **Browser-Local Generation**  
  - On first use, the app generates `nacl.box.keyPair()`, yielding a 32-byte Curve25519 secret key and corresponding public key.  

- **Wallet-Signed Binding**  
  - To prove ownership of the on-chain identity, the app signs the Base64 public key with the user’s EOA via `signMessage()`.  
  - The signature is stored alongside the keypair in `localStorage`, so one can audit that “EOA X registered pubkey Y.”

- **Persistent Storage**  
  - Keypairs are stored in `localStorage` under a map keyed by the lowercased wallet address.  
  - Users may **export** their keypair to a JSON file (`<address>-enc-key.json`) for offline backup or **import** it on another browser.

- **Recovery on New Browser**  
  1. **Import JSON**: Load the backup file.  
  2. **LocalStorage Mapping**: The app writes the imported keypair into `localStorage` for future automatic loading.  
  3. **Signature Display**: The stored signature re-affirms that the key belongs to the connected EOA.

---

## On-Chain Public Key Registry

A Solidity contract exposes:

```solidity
function isPublicKeyRegistered(address user) view returns (bool);
function registerUserPublicKey(string b64PubKey) external;
function getUserPublicKey(address user) view returns (string);
event UserPublicKeyRegistered(address indexed user);
```

- **Register / Update**  
  - Users push their Base64 X25519 public key on-chain in a transaction.  
  - The UI listens for `UserPublicKeyRegistered` events to build a local **registry** of address → public-key mappings.

- **Privacy**  
  - Only public keys live on-chain; secrets and message data remain off-chain.

---

## Hybrid Envelope Construction

1. **Body Cipher**  
   - A single symmetric encryption of the plaintext under `K` (derived from ephemeral & recipient keys).

2. **Per-Recipient Envelope**  
   - Each recipient gets an envelope containing:
     - Ephemeral public key (`epk`, 32 bytes)
     - Nonce (`24 bytes`)
     - Secretbox ciphertext
   - The envelope is Base64-encoded and stored in a JSON map keyed by recipient address.

3. **Multi-Recipient Efficiency**  
   - The body cipher is computed once; recipients each get only the envelope containing their wrapped key.

---

## Browser App Features

- **Wallet Connect (Ethers.js)**  
  - Detects `window.ethereum`, requests accounts, and switches to the configured chain (Sonic Blaze Testnet, `0xdede`).

- **Keypair UI**  
  - **Load Existing**: Restores from `localStorage` if present.  
  - **Generate New**: Prompts overwrite if a key already exists.  
  - **Copy to Clipboard**: Public and secret keys.  
  - **Save to File**: Triggers JSON download of your keypair.  
  - **Import from File**: Loads a backup keypair JSON into `localStorage`.

- **Registry Management**  
  - Fetch past `UserPublicKeyRegistered` events to populate the recipients list.  
  - Register or update your on-chain public key with a single button click.

- **Encrypt & Decrypt Workflows**  
  - **Encrypt**: Type message, pick recipients (multi-select), click **Encrypt**, get a JSON map of envelopes.  
  - **Decrypt**: Paste that JSON, click **Decrypt**, and view your recovered plaintext automatically.

- **Per-Wallet Storage Isolation**  
  - Under the hood, keys are stored at `enc_kp_map[<lowercase_address>]` so multiple wallet users can share one browser without collisions.

---

## User Workflow & Usage

1. **Serve Locally**  
   ```bash
   git clone <repo>
   cd <repo>
   npx serve .             # or any static server
   ```
2. **Open in Browser**  
3. **Connect Wallet** → Approve in MetaMask (or any injected EVM wallet).  
4. **Load vs Generate Keypair**  
   - Click **Load** if you’ve used this page before or **Import** backup file.  
   - Otherwise, click **Generate** (overwriting only with confirmation).  
5. **Register Public Key On-Chain** → Sign & send transaction.  
6. **Encrypt**  
   - Enter plaintext  
   - Select registered recipients (always includes you)  
   - Click **Encrypt** → Copy the JSON envelopes  
7. **Decrypt**  
   - Paste the JSON envelopes  
   - Click **Decrypt** → Plaintext appears  

---

## Key Handling & Security

- **Secret Key Confidentiality**  
  - Never transmitted to any server.  
  - Only exposed in-browser when clicking **Show** and copied/exported explicitly.

- **LocalStorage Risks**  
  - If an attacker gains filesystem or browser access, they can steal your secret key.  
  - Mitigation: **Export** and then **clear** your secret key from `localStorage` when not in active use.

- **Signature Verification**  
  - The on-chain contract does not enforce signature checks; the UI displays your wallet’s signature over your pubkey as an audit aid.  
  - For full security, clients should verify this signature off-chain or extend the contract to do so.

- **Envelope Replay & Integrity**  
  - Each envelope is bound to a fresh ephemeral key; replaying old envelopes yields no new secret.  
  - Secretbox’s built-in MAC prevents tampering of ciphertext.

---

## Future Enhancements

- **On-Chain Signature Enforcement**  
  - Modify the registry contract to verify that only the legitimate wallet owner (via ECDSA recover) can register a pubkey.

- **Encrypted Message Storage**  
  - Integrate with IPFS or decentralized storage to submit/retrieve encrypted data blobs.

- **Group Messaging & Key Rotation**  
  - Support rotating user keypairs and retroactively re-encrypting for new keys.

- **UI Polish & UX**  
  - Add countdown timers for auto-logout, secret key expiration, or “burn after reading.”

- **Interoperability**  
  - Build NPM/ESM modules so this functionality can be integrated into larger dApps and frameworks.

---

With this foundation, you can build rich, private messaging or data-sharing apps on top of public blockchains—ensuring only authorized recipients can decrypt sensitive content, while leveraging on-chain identity and auditability.
