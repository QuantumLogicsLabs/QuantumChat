# QuantumChat

A MERN messaging app with real end-to-end encryption. Every account gets a **pool of 5 X25519 keypairs**: public halves live in MongoDB, private halves never leave the browser. Messages and file attachments are sealed client-side with [`tweetnacl`](https://github.com/dchest/tweetnacl-js)'s `nacl.box` (Curve25519-XSalsa20-Poly1305) — the server only ever stores and relays ciphertext.

See also: [`backend/README.md`](backend/README.md) and [`frontend/README.md`](frontend/README.md) for package-specific setup.

## How the encryption works

- **Keys are real, not random strings.** `nacl.box.keyPair()` produces a 32-byte public key and a 32-byte private key that are mathematically linked (X25519). Hex-encoded, each is exactly 64 characters.
- **Sealed boxes: a public key can only encrypt, a private key can only decrypt.** This is enforced structurally, not by convention — `sealMessage(plaintext, targetPublicKey)` takes no secret-key argument at all (it generates a disposable one-time keypair internally), and `unsealMessage(envelope, myPrivateKey)` requires a private key; passing a public key into that slot fails the authentication check and returns nothing. This is the standard "sealed box" construction (as in libsodium's `crypto_box_seal`). The tradeoff: the ciphertext no longer cryptographically proves who sent it — the app authenticates the sender at the API layer instead, via the JWT required on every `/messages` POST.
- **Each user has a static pool of 5 public keys, not one** (`User.publicKeys`), generated once at registration and unchanged thereafter — login doesn't touch them. Senders pick a random entry from the recipient's pool per message/attachment, so a single conversation's ciphertext is spread across multiple keys instead of always the same one. Each of the 5 has exactly one matching private key; decrypting just means finding the one that matches the envelope's `targetPublicKey`, not "trying keys until one works."
- **A local keyring, not a single stored key.** All 5 private keys a device generated at registration live in `localStorage` (`frontend/src/crypto/keyStorage.js`). Every sealed envelope names the exact public key (`targetPublicKey`) it was sealed to, so the right one is looked up directly.
- **A device can still request a brand-new 5-key pool** (`regenerateKeys` in `AuthContext.jsx`, `PATCH /users/me/public-keys`) if its local keyring is lost — e.g. cleared storage or a new device — but this is manual recovery, not automatic rotation. History sealed under the old pool becomes unreadable unless this device already holds those old private keys.
- **`keys.txt` backup and restore.** Right after registration, the app shows the 5 freshly-generated private keys and lets you download them as a plain-text `keys.txt` (`frontend/src/crypto/keyFile.js`). Logging in on a device with no local keyring (a new browser, or one that's been cleared) offers "Import keys.txt" as an alternative to generating a brand-new pool — upload the file, and each imported private key is validated by deriving its public key (`derivePublicKey`, X25519 secret keys deterministically produce one specific public key) and checking it against the account's actual published `publicKeys`. A file from the wrong account, or one that's stale because the pool was since regenerated, is rejected rather than silently imported.
- **Because sealing is one-way, text messages are sealed twice**: once to the recipient's current key (so they can read it) and once to the sender's own current key (so the sender can read their own sent history back — the ephemeral key used to seal is discarded immediately and can't be recovered otherwise).
- **Attachments are sealed once, to the recipient only** — not doubled like text, to avoid uploading every file twice. This means the sender cannot re-open a file after sending it (they already have the original locally); the UI shows "only the recipient can open this" for the sender's own sent attachments.
- **The server never has a private key.** It can't decrypt anything; it stores ciphertext + nonce + envelope metadata and relays new messages over Socket.IO where available.
- **Losing local storage means losing history.** If a device's keyring is wiped, there's no way to recover previously received messages — that's inherent to true E2E encryption, not a bug. The app lets you generate a fresh 5-key pool to keep chatting going forward (see above).

## Tech stack

- **Backend**: Node.js, Express, MongoDB/Mongoose, Socket.IO (local dev), JWT auth, `helmet` + rate limiting.
- **Frontend**: React 18, Vite, React Router, Axios, Socket.IO client, `tweetnacl`.

## Project structure

```
backend/
  server.js                  # local-dev entry point (persistent server + Socket.IO)
  api/index.js                # Vercel serverless entry point (no Socket.IO there)
  vercel.json                  # rewrites all paths to api/index for Vercel
  src/
    app.js                    # express app, middleware, routes
    config/db.js               # cached mongoose connection (safe for serverless reuse)
    models/                    # User (publicKeys[5]), Message (forRecipient/forSender), Attachment
    controllers/                # auth, users, messages, attachments
    routes/
    middleware/                 # auth (JWT), upload (multer), rateLimiter
    socket/index.js             # authenticated Socket.IO wiring (local dev only)
frontend/
  src/
    crypto/keys.js              # keypair/key-set generation + sealMessage/unsealMessage
    crypto/keyStorage.js        # local keyring (append-only) + session storage
    context/AuthContext.jsx     # register/login/regenerateKeys/logout
    pages/                      # Register, Login, Chat
    components/                  # UserList, MessageBubble, AttachmentBubble
```

## Setup

### Backend

```bash
cd backend
npm install
cp .env.example .env   # then edit MONGODB_URI / JWT_SECRET
npm run dev             # http://localhost:5000
```

### Frontend

```bash
cd frontend
npm install
cp .env.example .env   # VITE_API_URL, defaults to http://localhost:5000
npm run dev             # http://localhost:5173
```

You'll need a MongoDB instance running (local `mongod`, Docker, or Atlas) — point `MONGODB_URI` at it.

### Environment variables (backend)

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | 5000 | HTTP/Socket.IO port (local dev only; unused on Vercel) |
| `MONGODB_URI` | — | Mongo connection string, including the database name |
| `JWT_SECRET` | — | JWT signing secret — set a long random value |
| `JWT_EXPIRES_IN` | 7d | Token lifetime |
| `UPLOAD_DIR` | uploads (or `/tmp/uploads` on Vercel) | Where encrypted attachment blobs are stored on disk |

CORS is intentionally wide open (`app.use(cors())`, no origin allowlist) — auth is JWT-bearer, not cookie-based, so there's no CSRF exposure from allowing any origin, and it removes a whole class of "env var didn't match" deployment failures.

## API reference

Base URL: `http://localhost:5000/api` (or your deployed backend + `/api`). Authenticated routes take `Authorization: Bearer <jwt>`.

### Auth

- `POST /auth/register` — `{ username, email, password, publicKeys }` (`publicKeys`: array of exactly 5 64-char hex keys, generated once and fixed thereafter). Returns `{ token, user }`.
- `POST /auth/login` — `{ email, password }`. Doesn't touch `publicKeys`. Updates `lastLoginAt`. Returns `{ token, user }`.
- `GET /auth/me` — current user profile.

### Users

- `GET /users` — everyone except yourself, with each user's `publicKeys` (array of 5) and `lastLoginAt`.
- `GET /users/:id` — a single user's public profile.
- `PATCH /users/me/public-keys` — `{ publicKeys }` (array of 5). Manual recovery only — replaces the whole pool when a device's local keyring is lost. Not called automatically.

### Messages

- `POST /messages` — `{ to, forRecipient, forSender, attachmentId? }`, where each envelope is `{ ciphertext, nonce, ephemeralPublicKey, targetPublicKey }` (sealed-box output from `sealMessage`). Broadcasts `message:new` to the recipient over Socket.IO where available.
- `GET /messages/:userId` — full conversation history with that user, oldest first, with attachment metadata populated.

### Attachments

- `POST /attachments` — `multipart/form-data`: `file` (pre-sealed ciphertext bytes), `recipientId`, `nonce`, `ephemeralPublicKey`, `targetPublicKey`. Returns `{ id, filename, mimetype, size }`.
- `GET /attachments/:id/raw` — raw encrypted bytes, only for the sender or recipient. Decryption (and the "only the recipient can actually open it" check) happens client-side.

### Socket.IO

Local dev only (`server.js`) — connect with `{ auth: { token: <jwt> } }`. Each user joins a room named after their own id; `message:new` events carry the full message document (both envelopes, attachment ref — decrypt client-side). Not available on the Vercel deployment; REST send/fetch still works there, just without the instant push.

## Deploying to Vercel

Both `backend/` and `frontend/` deploy as separate Vercel projects (each is its own GitHub repo, per `.gitmodules`).

- **Backend** needs `vercel.json` + `api/index.js` (already in the repo) because Vercel only runs stateless serverless functions — `server.js`'s `app.listen()` doesn't work there. `api/index.js` exports the Express app as a request handler and reuses a cached DB connection across warm invocations.
- **Set environment variables in the Vercel project dashboard** (Settings → Environment Variables) — a local `.env` file is never used by Vercel. At minimum: `MONGODB_URI`, `JWT_SECRET`.
- `app.set('trust proxy', 1)` is required in `src/app.js` — Vercel's proxy sets `X-Forwarded-For`, and without this `express-rate-limit` throws on every request.
- **Known limitations on Vercel**: no Socket.IO (serverless functions can't hold persistent connections — messages still send/receive over REST, just without instant push), and attachments are unreliable (Vercel's filesystem is read-only outside `/tmp`, and `/tmp` isn't shared or durable across invocations — a file uploaded in one invocation may be gone by the time a later request tries to download it). A persistent host (Render, Railway, Fly.io, a VPS) or object storage (S3 etc.) would fix both; not implemented here.
- If `vercel.json` ever needs a `functions` key, note that Vercel rejects an empty config object (`{}`) — it must contain at least one real property, or omit the key entirely.
