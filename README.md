# CJ Licence — Discord Bot Protection System

A complete licensing and triple-layer protection system for Discord bots.

## Protection Stack

| Layer | Tool | Effect |
|---|---|---|
| 1 — Obfuscation | `javascript-obfuscator` | Scrambles variable names, injects dead code, RC4 string encoding |
| 2 — Bytecode | `bytenode` | JS → V8 binary bytecode `.jsc` (not readable as JS) |
| 3 — EXE Bundle | `nexe` | Node.js runtime + bytecode → single `.exe` |
| 4 — License | Backend API | HWID-locked license keys with expiry |
| 5 — Anti-Debug | In bot code | VM MAC detection, debugger traps, sandbox detection |

---

## Project Structure

```
cj-licence/
├── backend/                  ← Express API server
│   ├── server.js
│   ├── db.js                 ← SQLite (no external DB needed)
│   └── routes/licenseRoutes.js
├── bot-core/                 ← Your bot source (gets protected)
│   ├── index.js              ← Bot logic (replace with your own)
│   ├── auth.js               ← HWID + license check
│   └── config.json           ← Template for end-users
├── build/                    ← Build toolchain
│   ├── obfuscate.js          ← Step A
│   ├── compile.js            ← Step B
│   ├── loader.js             ← Step C entry point
│   └── bundle.js             ← Step D (nexe)
├── dist/                     ← Build output (auto-created)
│   ├── obfuscated.js
│   ├── bot-core.jsc
│   └── bot.exe               ← Final deliverable
├── .env.example
└── package.json
```

---

## Setup

### 1. Install dependencies
```bash
npm install
```

### 2. Configure environment
```bash
copy .env.example .env
# Edit .env — set ADMIN_SECRET to a long random string
```

### 3. Start the backend API
```bash
npm run start:backend
# API runs on http://localhost:3000
```

---

## Backend API Usage

All admin endpoints require the `x-admin-secret` header matching your `ADMIN_SECRET` in `.env`.

### Generate a license key
```bash
curl -X POST http://localhost:3000/api/license/generate \
  -H "Content-Type: application/json" \
  -H "x-admin-secret: YOUR_SECRET" \
  -d '{"note": "Customer Name", "expiresInDays": 30}'
```

Response:
```json
{
  "success": true,
  "key": "A1B2-C3D4-E5F6-G7H8",
  "expiresAt": "2026-06-03T...",
  "note": "Customer Name"
}
```

### Validate a key (called automatically by the bot)
```bash
curl -X POST http://localhost:3000/api/license/validate \
  -H "Content-Type: application/json" \
  -d '{"key": "A1B2-C3D4-E5F6-G7H8", "hwid": "abc123..."}'
```

### List all keys
```bash
curl http://localhost:3000/api/license/list \
  -H "x-admin-secret: YOUR_SECRET"
```

### Revoke a key
```bash
curl -X POST http://localhost:3000/api/license/revoke \
  -H "Content-Type: application/json" \
  -H "x-admin-secret: YOUR_SECRET" \
  -d '{"key": "A1B2-C3D4-E5F6-G7H8"}'
```

---

## Building the Protected .exe

```bash
# All three steps in sequence:
npm run build

# Or individually:
npm run build:obfuscate   # Step A: Obfuscate JS → dist/obfuscated.js
npm run build:compile     # Step B: Compile → dist/bot-core.jsc
npm run build:bundle      # Step C: Bundle → dist/bot.exe
```

> **Note:** The first `build:bundle` run downloads a Node.js binary (~30 MB) and caches it. Subsequent builds are fast.

---

## Distributing to Customers

Give each customer:
1. `dist/bot.exe`
2. A blank `config.json`:
   ```json
   {
     "token": "THEIR_DISCORD_BOT_TOKEN",
     "prefix": "!",
     "licenseKey": "THEIR-KEY-HERE",
     "apiUrl": "http://YOUR_SERVER_IP:3000"
   }
   ```
3. Instructions: place both files in the same folder and run `bot.exe`

---

## Customizing the Bot

Replace the contents of `bot-core/index.js` with your own bot logic. Keep the auth gate:
```js
const config = await authenticate(); // Must be first — exits if invalid
```

After editing, re-run `npm run build` to generate a new protected `bot.exe`.
