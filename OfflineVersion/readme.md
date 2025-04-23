# Offline Hybrid Encryption Demo

An all-in-browser tool for hybrid (asymmetric + symmetric) encryption using Curve25519 keys. No server, no blockchain—everything runs offline.

## Features

- **Keypair Management**  
  - Import or paste your Curve25519 keypair (public & secret) in Base64.  
  - Export your keypair to a JSON file for safe offline backup.  
  - Reload your keypair from file at any time.

- **Recipient Management**  
  - Add as many recipients as you like by pasting their public keys (Base64).  
  - Always include yourself automatically when encrypting.

- **Encryption**  
  - Hybrid “envelope” scheme: for each recipient, generates a one-time ephemeral key and nonce, seals the message symmetrically with `nacl.box`, and bundles into a Base64 blob.  
  - Outputs a JSON mapping each recipient’s public key → Base64 envelope.

- **Decryption**  
  - Paste the envelopes JSON, and it finds the blob intended for your public key, opens it with your secret key, and reveals the original plaintext.

- **Debug Log**  
  - Real-time operation logs to help you troubleshoot or learn how each step works.

## Setup & Usage

1. **Download** or **clone** this repository.  
2. **Open** `index.html` in a modern browser (Chrome, Firefox, Edge).  
   - No web server required—just double-click the file.

3. **Load or Import Your Keypair**  
   - Click **Import Keypair from File** to load a previously saved JSON.  
   - OR paste your Base64 public & secret keys into the textareas.  
   - Once loaded, **Save Keypair to File** enables for offline backup.

4. **Add Recipients**  
   - Paste each recipient’s Base64 public key and click **Add Recipient**.  
   - They will appear in the recipients list.

5. **Encrypt**  
   - Enter your plaintext and click **Encrypt ▶️**.  
   - You’ll see a JSON object mapping each recipient key → Base64 envelope.

6. **Decrypt**  
   - Paste the envelopes JSON into the decrypt textarea.  
   - Click **Decrypt ▶️** to recover your original plaintext.

7. **Review Debug Log**  
   - The bottom pane shows step-by-step logs of key loading, encryption, decryption, and errors.

## How It Works

1. **Keypair**  
   - Uses [TweetNaCl.js](https://github.com/dchest/tweetnacl-js) to generate Curve25519 keypairs.  
   - Public & secret keys are Base64-encoded for transport/storage.

2. **Hybrid Scheme**  
   - **Ephemeral Key**: For each recipient, generate a fresh `ephKeyPair`.  
   - **Symmetric Seal**: `ct = nacl.box(utf8enc(plaintext), nonce, recipientPub, ephSecret)`.  
   - **Bundle**: `[ephPub‖nonce‖ct]`, Base64-encoded → “envelope”.

3. **Security Properties**  
   - **Confidentiality**: Only the holder of the matching secret key can open their envelope.  
   - **Forward Secrecy**: Each envelope uses a new ephemeral key.  
   - **Offline Use**: No secrets or plaintext ever leave your browser.

## Dependencies

- [TweetNaCl.js](https://github.com/dchest/tweetnacl-js) (via CDN)
- [tweetnacl-util.js](https://github.com/dchest/tweetnacl-util-js) for Base64/text encoding

Both are included via `<script>` tags; no build or package manager required.

## Security & Caveats

- **Protect Your Secret Key**: If someone steals it, they can decrypt any envelope addressed to you.  
- **Backup & Recovery**: Always export your keypair JSON before clearing browser storage.  
- **No Authentication**: This demo does _not_ authenticate the sender; use in trusted contexts or add signatures if needed.  
- **Browser Environment**: Ensure you run on `file://` or a local server over HTTPS for maximum security (some browsers restrict cryptography on insecure contexts).

---

Feel free to fork, adapt, and extend for your own offline encryption needs!
