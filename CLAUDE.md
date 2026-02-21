# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Smart Locker is an NFC-based equipment borrowing/returning kiosk system for Windows. Users tap an ACR1252U NFC card reader to authenticate, then interact with a touch-display browser UI to borrow or return devices. All data is persisted in SQLite with AES-256-GCM encrypted card storage.

## Commands

### Setup

```bash
python -m venv venv
source venv/Scripts/activate  # Windows: .\venv\Scripts\Activate
pip install -r requirements.txt

# Generate encryption + HMAC keys (one-time)
python -m scripts.generate_key

# Copy .env.example to .env and paste the generated keys
cp .env.example .env

# Initialize database
python -m scripts.init_db

# Enroll an NFC card (requires physical reader)
python -m scripts.enroll_card --name "User Name" --role admin
```

### Run

```bash
python -m smart_locker.app
```

### Tests

```bash
# All tests (52 tests, no hardware required)
python -m pytest tests/ -v

# Single test file
python -m pytest tests/test_services.py -v

# Single test
python -m pytest tests/test_services.py::TestLockerService::test_borrow_device -v
```

No linter is configured; no build step is needed.

## Architecture

### Package Layout

```
config/               # Settings (DB path, reader name, timeouts, limits) and logging
smart_locker/
  app.py              # Entry point: starts NFC reader threads + main event loop
  nfc/                # pyscard interface: reader, card/reader observers, APDU commands
  auth/               # Card UID → user lookup (HMAC) + session lifecycle
  security/           # AES-256-GCM encryption, HMAC-SHA256 hashing, key loading
  database/           # SQLAlchemy 2.0: models, engine (WAL mode), repositories
  services/           # Business logic: borrow/return with limit enforcement
  frontend/           # Static kiosk UI (index.html, style.css, app.js)
scripts/              # One-time utilities: generate_key, init_db, enroll_card, import_devices
tests/                # pytest suite with in-memory DB and mock NFC data
```

### Data Flow

1. `NFCReader` (pyscard `CardMonitor` + `ReaderMonitor` in background threads) detects a card tap and queues an event.
2. `SmartLockerApp.run()` retrieves the event and calls `Authenticator.authenticate(uid_hex)`.
3. Authenticator computes `HMAC-SHA256(uid, hmac_key)` and does an indexed lookup in `users.uid_hmac` — no decryption needed for auth.
4. On success, `SessionManager` starts a session; inactivity timeout (default 120 s) is reset on each user action via `touch()`.
5. `LockerService.borrow_device()` / `return_device()` enforces the per-user borrow limit (default 5), updates device status, and writes a `TransactionLog`.

### Security Design

- **Two independent keys**: `SMART_LOCKER_ENC_KEY` (AES-256-GCM) and `SMART_LOCKER_HMAC_KEY` (HMAC-SHA256), both 32-byte base64-encoded values in `.env`.
- **HMAC for lookup**: Deterministic digest stored in `users.uid_hmac` (indexed) enables O(1) auth without decrypting rows.
- **AES-GCM for storage**: Random nonce per encryption — same UID produces different ciphertext every time; only admins can decrypt.
- **UID masking in logs**: Raw UIDs are never logged (shown as `04**********80`).

### Database Schema

| Table | Key Columns |
|---|---|
| `users` | `uid_hmac` (indexed), `encrypted_card_uid`, `role` (USER/ADMIN), `is_active` |
| `devices` | `status` (AVAILABLE/BORROWED/MAINTENANCE), `current_borrower_id` (FK), `serial_number` (unique) |
| `transaction_logs` | `transaction_type` (BORROW/RETURN), `performed_by_id` (for admin returns) |

### Frontend (Kiosk UI)

Three static files in `smart_locker/frontend/`:
- **index.html** — 6 screen definitions (Idle, Auth Failed, Main Menu, Borrow Grid, Return Grid, Device Detail)
- **style.css** — Dark theme (`#080c10` / `#00d4ff` cyan), `clip-path` wipe transitions, staggered card animations
- **app.js** — Client-side state machine; includes a demo mode with mock data for testing without backend

The frontend currently has no HTTP connection to the backend. The next implementation step is to add a REST API layer (FastAPI routes + SSE event stream) so the frontend can receive card-tap events and invoke borrow/return operations.

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `SMART_LOCKER_ENC_KEY` | (required) | 32-byte AES key, base64-encoded |
| `SMART_LOCKER_HMAC_KEY` | (required) | 32-byte HMAC key, base64-encoded |
| `SMART_LOCKER_DB_PATH` | `smart_locker.db` | SQLite file path |
| `SMART_LOCKER_READER_NAME` | `ACR1252` | Substring matched against connected reader names |
| `SMART_LOCKER_SESSION_TIMEOUT` | `120` | Inactivity timeout in seconds |
| `SMART_LOCKER_MAX_BORROWS` | `5` | Maximum concurrent borrows per user |

## What's Built vs. What's Next

**Complete:** NFC reader (pyscard, retry logic), authentication (HMAC lookup, session management), security (AES-256-GCM, key management), database + ORM (SQLAlchemy 2.0, WAL), business logic (borrow/return, admin override, transaction logging), unit tests (52 tests), frontend UI (6 screens, animations, demo mode).

**Not yet built:** REST API routes connecting frontend to backend, SSE event bridge for pushing card-tap events to the browser, kiosk deployment (Chromium kiosk mode, Windows auto-start).
