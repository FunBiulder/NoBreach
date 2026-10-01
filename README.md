# NoBreach

**$NBP** · Encrypted file storage powered by your wallet.

NoBreach is a wallet-authenticated file vault built on **Robinhood Chain**, designed around a simple idea:

> **Encrypt it in the browser. Store the ciphertext. Use your wallet as your identity.**

Instead of sending plaintext files to a traditional cloud storage backend, NoBreach encrypts files client-side using **AES-GCM 256-bit** before they are uploaded.

Your wallet handles authentication, while the backend handles access control, encrypted storage, payments, and metadata.

---

## ✨ What is NoBreach?

NoBreach combines wallet authentication with client-side encrypted storage.

```text
                    NoBreach

       ┌──────────────────────────┐
       │          Wallet          │
       │    Identity + Signing    │
       └────────────┬─────────────┘
                    │
                    ▼
       ┌──────────────────────────┐
       │         Browser          │
       │                          │
       │   Encrypt file locally   │
       │       AES-GCM 256        │
       └────────────┬─────────────┘
                    │
                    │ ciphertext
                    ▼
       ┌──────────────────────────┐
       │           API            │
       │                          │
       │ Auth · Ownership · API   │
       └───────┬──────────┬───────┘
               │          │
               ▼          ▼
        ┌───────────┐  ┌─────────────┐
        │ PostgreSQL│  │ Object Store│
        │  Metadata │  │  Ciphertext │
        └───────────┘  └─────────────┘
```

The important boundary is the browser:

**the original file is encrypted before the upload reaches the storage layer.**

---

## 🔐 How it works

### 1. Connect your wallet

NoBreach uses a wallet address as the identity for your vault.

There is no traditional email/password account.

### 2. Sign a one-time nonce

The backend generates a nonce.

The wallet signs the nonce.

The backend verifies the signature and creates a session.

```text
Wallet
  │
  │ request nonce
  ▼
API
  │
  │ nonce
  ▼
Wallet
  │
  │ sign nonce
  ▼
API
  │
  │ verify signature
  ▼
Authenticated session
```

The nonce is removed after successful verification so the same authentication challenge cannot simply be reused.

### 3. Encrypt the file

When you select a file, encryption happens in the browser using the Web Crypto API.

```text
Original file
     │
     ▼
AES-GCM 256-bit
     │
     ▼
Encrypted bytes
     │
     ▼
Upload
```

The backend receives the encrypted payload rather than the original plaintext file.

### 4. Store the encrypted object

The encrypted file is stored in S3-compatible object storage.

Each object receives a generated UUID.

PostgreSQL stores the application metadata and ownership relationship.

```text
PostgreSQL
├── wallet identity
├── file metadata
├── ownership
├── payment records
└── encrypted vault data

Object Storage
└── encrypted file objects
```

### 5. Verify ownership on access

File operations are scoped to the authenticated wallet.

A download request must pass the ownership check before the backend retrieves the object.

---

# 🧱 Tech Stack

| Component       | Technology                           |
| --------------- | ------------------------------------ |
| Blockchain      | Robinhood Chain                      |
| Chain ID        | `4663`                               |
| Authentication  | EIP-1193 wallet + message signatures |
| Encryption      | Web Crypto API / AES-GCM 256-bit     |
| Backend         | Express                              |
| Database        | PostgreSQL                           |
| ORM             | Prisma                               |
| Object storage  | S3-compatible R2                     |
| File encryption | Client-side                          |
| Payments        | ETH / configured USDG                |

---

# 🚀 Getting Started

## Requirements

You'll need:

* Node.js
* PostgreSQL
* A compatible wallet
* Robinhood Chain configuration
* S3-compatible object storage
* Prisma
* A frontend environment for the client application

---

## Clone the repository

```bash
git clone https://github.com/<your-org>/<your-repo>.git

cd <your-repo>
```

Install dependencies:

```bash
npm install
```

---

## Environment variables

Create a `.env` file:

```env
DATABASE_URL=
DIRECT_URL=

R2_ENDPOINT=
R2_BUCKET=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=

PAYMENT_RECEIVER=
USDG_TOKEN_ADDRESS=

FRONTEND_URL=
```

Never expose storage credentials to the frontend.

In particular, credentials such as:

```env
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
```

must remain server-side.

---

# 🗄️ Database

NoBreach uses PostgreSQL with Prisma.

After configuring your database:

```bash
npx prisma generate
```

Then run the appropriate migrations for your environment:

```bash
npx prisma migrate dev
```

For production deployments, use your normal Prisma migration deployment workflow rather than development migrations.

---

# 🔌 API

The current API is organized around authentication, payments, vault secrets and files.

## Authentication

### `POST /nonce`

Creates a wallet authentication nonce.

### `POST /login`

Verifies the wallet signature and creates an authenticated session.

---

## Payments

### `GET /payment/options`

Returns the currently configured activation options.

### `POST /payment/verify`

Verifies an activation transaction on-chain.

The backend checks relevant transaction information rather than trusting a submitted transaction hash alone.

---

## Vault

### `GET /secret`

Retrieves the authenticated vault's encrypted secret data.

### `POST /secret`

Stores the encrypted vault secret.

---

## Files

### `GET /files`

Lists files belonging to the authenticated wallet.

### `POST /files`

Creates file metadata.

### `POST /upload`

Uploads an encrypted file payload.

### `GET /download/:id`

Downloads an encrypted file after ownership verification.

### `DELETE /files/:id`

Deletes a file belonging to the authenticated wallet.

---

# 🔑 Encryption Model

NoBreach uses **AES-GCM with 256-bit keys** for file encryption.

The browser performs the encryption operation using the Web Crypto API.

A simplified version looks like:

```javascript
const encrypted = await crypto.subtle.encrypt(
  {
    name: "AES-GCM",
    iv
  },
  key,
  fileData
);
```

The encrypted result is then sent to the backend.

The architecture intentionally keeps the encryption boundary on the client.

---

# 🗝️ Vault Secrets

Each vault has a secret used as part of the encryption architecture.

The secret is not intended to be stored as plaintext by the backend.

Instead, the application works with an encrypted representation of the vault secret.

Wallet signing is used as part of the unlock/authentication flow.

This means losing access to the wallet can also mean losing access to the corresponding vault.

---

# 💳 Activation

NoBreach currently uses a one-time activation payment before protected vault functionality becomes available.

Supported activation assets include:

* ETH
* configured USDG

The API verifies the transaction on Robinhood Chain before recording the payment.

The verification process checks information such as:

```text
chain
transaction receipt
transaction status
sender
recipient
amount
```

A transaction hash by itself is not treated as sufficient proof of payment.

---

# 📦 File Storage

Files are encrypted before being uploaded.

The storage layer therefore receives encrypted objects.

A simplified upload flow:

```text
File selected
     │
     ▼
Browser
     │
     ├── Encrypt
     │
     ▼
Ciphertext
     │
     ▼
API
     │
     ├── authenticate
     ├── validate
     ├── create UUID
     │
     ▼
Object Storage
```

The current upload limit is **50 MB**.

---

# 🌐 Network

NoBreach runs on:

```text
Network:  Robinhood Chain
Chain ID: 4663
```

Wallet authentication uses EIP-1193-compatible injected wallets and personal message signatures.

---

# 🏗️ Architecture

At a high level:

```text
                    ┌─────────────┐
                    │    Wallet   │
                    └──────┬──────┘
                           │
                     signature
                           │
                           ▼
┌────────────────────────────────────────────┐
│                  Browser                   │
│                                            │
│   Wallet Auth      File Encryption         │
│                       │                    │
└───────────────────────┼────────────────────┘
                        │
                    ciphertext
                        │
                        ▼
               ┌────────────────┐
               │      API       │
               │                │
               │ Authentication │
               │ Sessions       │
               │ Ownership      │
               │ Payments       │
               │ File API       │
               └───────┬────────┘
                       │
              ┌────────┴─────────┐
              │                  │
              ▼                  ▼
       ┌─────────────┐   ┌──────────────┐
       │ PostgreSQL  │   │ R2 / S3      │
       │             │   │              │
       │ Metadata    │   │ Ciphertext   │
       │ Ownership   │   │ Objects      │
       │ Payments    │   │              │
       └─────────────┘   └──────────────┘
```

---

# 🛡️ Security Principles

NoBreach is built around a few straightforward boundaries.

### Client-side encryption

Files are encrypted before upload.

### Wallet-based authentication

Wallet signatures provide the authentication mechanism instead of traditional passwords.

### Owner-scoped access

Authenticated users can only operate on files associated with their wallet.

### Server-side credentials

Object-storage credentials remain on the backend.

### Transaction verification

Activation payments are verified against the blockchain rather than blindly trusting client-submitted information.

---

# ⚠️ Recovery

NoBreach does not provide a backdoor for recovering a lost wallet.

If you lose your wallet, recovery phrase or signing capability, the application cannot recover those credentials for you.

**Back up your wallet securely.**

The system also depends on the availability of its application, database, RPC and storage infrastructure.

---

# 🧪 Development

When working on security-sensitive parts of the application, pay particular attention to:

* authentication
* nonce handling
* signature verification
* ownership checks
* encryption boundaries
* vault-secret handling
* payment verification
* object-storage credentials

Avoid moving plaintext file handling to the backend unless there is a specific architectural reason to do so.

---

# 🤝 Contributing

Contributions are welcome.

Before submitting a pull request:

1. Keep security-sensitive changes small
