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
    images/           # Stock device photos (Unsplash, dark-background product style)
scripts/              # One-time utilities: generate_key, init_db, migrate_db, enroll_card, import_devices
tests/                # pytest suite with in-memory DB and mock NFC data
```

### Data Flow

1. `NFCReader` (pyscard `CardMonitor` + `ReaderMonitor` in background threads) detects a card tap and queues an event.
2. `SmartLockerApp.run()` retrieves the event and calls `Authenticator.authenticate(uid_hex)`.
3. Authenticator computes `HMAC-SHA256(uid, hmac_key)` and does an indexed lookup in `users.uid_hmac` — no decryption needed for auth.
4. On success, `SessionManager` starts a session. The card is removed from the reader; the session persists on the touch display. A second card tap ends the session explicitly. An inactivity timeout (default 120 s) silently clears the session as a security backstop.
5. `LockerService.borrow_device()` / `return_device()` enforces the per-user borrow limit (default 5), updates device status, and writes a `TransactionLog`.

### Security Design

- **Two independent keys**: `SMART_LOCKER_ENC_KEY` (AES-256-GCM) and `SMART_LOCKER_HMAC_KEY` (HMAC-SHA256), both 32-byte base64-encoded values in `.env`.
- **HMAC for lookup**: Deterministic digest stored in `users.uid_hmac` (indexed) enables O(1) auth without decrypting rows.
- **AES-GCM for storage**: Random nonce per encryption — same UID produces different ciphertext every time; only admins can decrypt.
- **UID never logged**: Raw UIDs are never written to log files — card events are logged as "Card inserted on <reader>" with no UID information.

### Database Schema

| Table | Key Columns |
|---|---|
| `users` | `uid_hmac` (indexed), `encrypted_card_uid`, `role` (USER/ADMIN), `is_active` |
| `devices` | `status` (AVAILABLE/BORROWED/MAINTENANCE), `current_borrower_id` (FK), `serial_number` (unique) |
| `transaction_logs` | `transaction_type` (BORROW/RETURN), `performed_by_id` (for admin returns) |

### Frontend (Kiosk UI)

Static files in `smart_locker/frontend/`:
- **index.html** — 6 screen definitions (Idle, Auth Failed, Main Menu, Borrow Grid, Return Grid, Device Detail)
- **style.css** — Dark theme (`#080c10` / `#00d4ff` cyan), landonorris.com-inspired transitions and animations
- **app.js** — Client-side state machine; includes a demo mode with mock data for testing without backend
- **images/** — Dark-background stock photos from Unsplash (placeholder; will be replaced with actual electrical test equipment photos)

#### Animation System (landonorris.com-inspired)
- **Circle reveal page transitions** — screens expand from click origin via CSS `clip-path: circle()`; JS tracks `lastClickX/Y` and sets `--reveal-x/--reveal-y` CSS custom properties
- **Diagonal wipe overlays** — device detail and inactivity overlays use polygon clip-path wipes
- **Ellipse-reveal images** — device card images reveal through expanding `clip-path: ellipse()` on entrance; action button background images use the same technique
- **NFC texture breathe** — idle screen NFC icon has a looping ellipse reveal of a tech texture behind it
- **Character split text** — headline text is split into individual `<span>` characters with staggered `rotateX + translateY` entrance animations
- **Slot-machine numbers** — inactivity countdown digits animate with vertical slide transitions
- **Magnetic hover** — buttons shift subtly toward cursor (clamped to ±4px on action buttons to prevent overlap)
- **Image parallax** — device card images shift opposite to mouse direction on hover
- **Cursor glow** — custom cursor dot expands and glows when hovering interactive elements
- **Easing curves** — `--ease` (snappy) for small interactions, `--ease-smooth: cubic-bezier(0.65, 0.05, 0, 1)` (Norris-style) for major transitions

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

**Complete:** NFC reader (pyscard, retry logic), authentication (HMAC lookup, session management), security (AES-256-GCM, key management), database + ORM (SQLAlchemy 2.0, WAL), business logic (borrow/return, admin override, transaction logging), unit tests (52 tests), frontend UI (6 screens, landonorris.com-inspired animations, stock images, demo mode).

**Not yet built:** REST API routes connecting frontend to backend, SSE event bridge for pushing card-tap events to the browser, replace stock images with actual electrical test equipment photos, kiosk deployment (Chromium kiosk mode, Windows auto-start).
