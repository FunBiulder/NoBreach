# NoBreach

### Private storage without the usual password surface.

**$NBP** - the token powering the NoBreach ecosystem.

CA : 0x9e545052593BC326f84f257B7e4f73Cf6A8C2Cb3

NoBreach is a wallet-authenticated encrypted file vault built on **Robinhood Chain**.

Instead of uploading files in plaintext and protecting them with a traditional email/password account, NoBreach uses your wallet as the identity layer and encrypts files **in the browser before they ever reach the storage layer**.

The basic idea is simple:

> **Your wallet proves who you are. Your browser encrypts the file. The server stores the ciphertext.**

---

## What is NoBreach?

NoBreach is designed around a different approach to private cloud storage.

There are no traditional password accounts.

There is no plaintext file sitting in the object store waiting for an application server to retrieve it.

When you upload a file:

```text
Your File
   ↓
Browser
   ↓
AES-GCM 256-bit encryption
   ↓
Encrypted payload
   ↓
API
   ↓
S3-compatible object storage
```

The original file contents are encrypted locally using the Web Crypto API before the upload takes place.

The storage layer receives encrypted bytes rather than the original file.

NoBreach currently uses:

* **Robinhood Chain** for wallet-based identity and payment verification
* **Wallet signatures** for authentication
* **AES-GCM 256-bit** for file encryption
* **PostgreSQL + Prisma** for application records
* **S3-compatible R2 storage** for encrypted objects
* **Express** for the API layer

---

## Why wallet authentication?

NoBreach does not create another password database.

Your wallet address acts as the identity of your vault.

A login works roughly like this:

```text
Connect wallet
     ↓
Request one-time nonce
     ↓
Sign nonce
     ↓
Server verifies signature
     ↓
Temporary session created
     ↓
Vault access
```

The nonce is deleted after successful verification, helping prevent reuse of the same authentication challenge.

Sessions are held in backend memory, meaning a backend restart invalidates active sessions and requires users to authenticate again.

---

## Client-side encryption

This is one of the most important parts of NoBreach.

Encryption happens **before upload**.

The browser uses Web Crypto and AES-GCM with 256-bit keys to encrypt file content locally.

Conceptually:

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

The resulting encrypted payload is what gets uploaded.

The server does not need the original plaintext file to store it.

The encrypted object is then stored under a generated UUID.

---

## Vault secrets

Each vault has its own secret.

The vault secret is not stored as plaintext.

Instead, it is encrypted locally using material derived from the wallet unlock signature.

This creates a separation between:

* wallet identity
* vault access
* encrypted file data
* storage records

The backend stores the encrypted representation required by the application rather than simply keeping the vault secret in plaintext.

---

## Storage architecture

NoBreach currently uses three main backend layers.

### 1. API

The API is responsible for:

* wallet signature verification
* sessions
* payment verification
* ownership checks
* vault-secret operations
* file operations

### 2. PostgreSQL

PostgreSQL stores application records such as:

* users
* wallet identities
* nonces
* payments
* file records
* encrypted vault secrets

Prisma is used as the database layer.

### 3. Object storage

Encrypted file objects are stored in S3-compatible storage.

Each uploaded object receives a generated UUID.

The database keeps the relationship between the encrypted object and its wallet owner.

---

## Ownership boundaries

File operations are owner-scoped.

For example:

```text
Authenticated wallet
       ↓
Session
       ↓
Owner verification
       ↓
Requested file
       ↓
Encrypted object
```

Listing, downloading, deleting files and accessing vault-secret endpoints require an authenticated session.

A requested file must belong to the authenticated wallet before the backend will operate on it.

---

# For Developers

## Tech stack

| Layer               | Technology                           |
| ------------------- | ------------------------------------ |
| Network             | Robinhood Chain                      |
| Chain ID            | 4663                                 |
| Authentication      | EIP-1193 wallet + message signatures |
| Encryption          | Web Crypto / AES-GCM 256-bit         |
| API                 | Express                              |
| Database            | PostgreSQL                           |
| ORM                 | Prisma                               |
| Object storage      | S3-compatible R2                     |
| Frontend encryption | Browser-side                         |

---

## Encryption flow

A simplified upload flow looks like this:

```text
1. Connect wallet
2. Authenticate with a signed nonce
3. Unlock/create vault
4. Select file
5. Encrypt file in browser
6. Encrypt required metadata
7. Send encrypted payload to API
8. Generate UUID for object
9. Store encrypted object
10. Store ownership metadata in PostgreSQL
```

The important boundary is step 5:

**The file is encrypted before it reaches the upload/storage layer.**

---

## API

The current API surface includes:

### Authentication

```http
POST /nonce
POST /login
```

Creates/retrieves a wallet nonce and verifies the resulting signature.

### Payments

```http
GET  /payment/options
POST /payment/verify
```

Returns configured activation options and verifies activation transactions on-chain.

### Vault secret

```http
GET  /secret
POST /secret
```

Retrieves or stores the encrypted vault secret.

### Files

```http
GET  /files
POST /files
POST /upload
```

Used for encrypted file records and encrypted uploads.

### Individual files

```http
GET    /download/:id
DELETE /files/:id
```

Downloads or deletes a vault-owned encrypted object.

---

## Payment verification

New vaults require a one-time activation payment before the protected file and secret endpoints become available.

NoBreach does not simply trust a transaction hash.

The API checks the transaction against Robinhood Chain and verifies relevant details including:

* chain
* transaction receipt
* transaction success
* sender
* recipient
* required amount

Only after validation is the payment record persisted.

Supported activation assets currently include:

* ETH
* configured USDG

---

## Configuration

The backend expects configuration for things such as:

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

### Important

Storage credentials are server-only.

Never expose:

```text
R2_ACCESS_KEY_ID
R2_SECRET_ACCESS_KEY
```

through public frontend environment variables such as:

```text
NEXT_PUBLIC_*
```

---

## File limits

Uploads currently use in-memory request handling with a **50 MB file-size limit**.

Encrypted objects are stored using generated UUID keys.

When downloading, the API first verifies that the requested object belongs to the authenticated wallet before streaming it from storage.

---

## Recovery

NoBreach deliberately does not provide a backdoor for recovering a lost wallet.

If you lose:

* your wallet
* recovery phrase
* signing capability

NoBreach cannot recover those credentials for you.

**Keep your wallet backup secure.**

There is also an important operational reality with any hosted application: availability depends on the application, network, database, RPC and storage infrastructure continuing to operate correctly.

---

# Network

NoBreach runs on:

```text
Network: Robinhood Chain
Chain ID: 4663
EIP-155: 4663
```

Wallet authentication uses EIP-1193-compatible injected wallets and personal message signatures.

---

# Project Philosophy

NoBreach is built around a few straightforward principles:

### Encrypt before upload

Don't make the storage layer responsible for protecting plaintext files when the client can encrypt them first.

### Wallet-native identity

A wallet can act as the authentication boundary without creating another password database.

### Minimize trust

The backend should enforce ownership and access control without needing plaintext file contents for ordinary storage operations.

### Keep the architecture understandable

Security is easier to reason about when the boundaries are explicit:

```text
Wallet
  │
  ├── Identity
  │
  └── Signing
       │
       ▼
     Browser
       │
       ├── Encrypt
       │
       ▼
      API
       │
       ├── Ownership
       ├── Sessions
       └── Payments
          │
          ▼
   ┌───────────────┐
   │ PostgreSQL    │
   │ Encrypted DB  │
   └───────────────┘

       +

   ┌───────────────┐
   │ Object Store  │
   │ Ciphertext    │
   └───────────────┘
```

---

# Status

NoBreach is an actively developing project.

Current documentation covers the protocol, security model, encryption flow, storage architecture, payments, API surface and network configuration.

Contract addresses and some public project channels are marked as coming soon in the current documentation.

---

# Security

If you discover a security issue, please report it responsibly rather than publicly exposing an exploitable vulnerability.

When reporting an issue, include:

* affected component
* reproduction steps
* expected behavior
* actual behavior
* potential impact

Please do not include private keys, recovery phrases or other sensitive credentials in an issue report.

---

# Contributing

Contributions, technical feedback and security research are welcome.

Before opening a pull request:

1. Understand the authentication flow.
2. Understand where encryption occurs.
3. Avoid moving plaintext file handling to the backend unnecessarily.
4. Never expose server-side storage credentials.
5. Preserve wallet-owner access boundaries.
6. Add tests for security-sensitive changes.
7. Document changes that affect the protocol or API.

---

# Links

**Website:** https://www.nobreach.site

**Documentation:** https://www.nobreach.site/docs

---

## NoBreach

**Your wallet. Your files. Your encryption boundary.**

Built for a world where private storage doesn't have to begin with another password.
