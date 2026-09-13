# Secure File Sharing System 

## Stack
- Python / Flask, PostgreSQL, SQLAlchemy, Alembic (migrations)
- JWT auth (PyJWT), Argon2 password hashing, AES-256-GCM file encryption
- TOTP MFA (pyotp), QR codes (qrcode + Pillow), rate limiting (Flask-Limiter)

---

## Quick Start

```bash
cd secure_file_share
pip install -r requirements.txt

# Set env vars (or create a .env file)
export DATABASE_URL=postgresql://user:pass@localhost:5432/secure_files
export SECRET_KEY=your-secret
export JWT_SECRET_KEY=your-secret
export MASTER_KEK=$(python -c "import os, base64; print(base64.b64encode(os.urandom(32)).decode())")
export BASE_URL=https://yourdomain.com
export ALLOWED_ORIGINS=http://localhost:3000

# Initialize database
flask --app wsgi db init
flask --app wsgi db migrate -m "initial"
flask --app wsgi db upgrade

# Create admin user (Module 6)
python create_admin.py

# Run server
python wsgi.py   # dev only — use gunicorn + TLS in production
```

---

## Run with Docker

### Using Docker Compose
```bash
# Start PostgreSQL database and the API container
docker compose up --build -d

# Run database migrations
docker compose exec app flask --app wsgi db upgrade

# (Optional) Create admin user
docker compose exec app python create_admin.py

# Tail application logs
docker compose logs -f app
```

### Using the Multi-Stage Dockerfile
```bash
# Build the production image
docker build -t secure-file-share:latest .

# Run the container (connecting to an external Postgres instance)
docker run -p 5000:5000 \
  -e DATABASE_URL="postgresql://user:pass@host:5432/secure_files" \
  -e SECRET_KEY="your-secret-key-at-least-32-chars" \
  -e JWT_SECRET_KEY="your-jwt-secret-at-least-32-chars" \
  -e MASTER_KEK="$(python -c 'import os, base64; print(base64.b64encode(os.urandom(32)).decode())')" \
  -e ALLOWED_ORIGINS="http://localhost:3000" \
  secure-file-share:latest
```

---

## Sharing Flow

```
1. Owner uploads a file  →  POST /api/files
2. Owner creates a share →  POST /api/files/<id>/share
      - selects recipient(s) by UUID (search via GET /api/auth/users/search?q=)
      - sets permissions: can_view / can_download / can_edit / can_reshare
      - optionally sets a password and/or expiry datetime
3. Server returns share_token + share_url + (optionally) QR code
4. Recipient opens share_url  →  GET /api/shares/<token>/access
      - if password-protected: POST /api/shares/<token>/verify-password → grant_token
      - send grant_token as X-Share-Grant header on subsequent requests
5. Recipient downloads file  →  GET /api/shares/<token>/download
      - file is decrypted in-memory and streamed; no plaintext hits disk
6. Owner can update or revoke at any time via PATCH / DELETE /api/shares/<share_id>
```

---

## Permission Model

| Permission    | What it allows                                      |
|---------------|-----------------------------------------------------|
| `can_view`    | View file metadata and access the share link        |
| `can_download`| Download the decrypted file                         |
| `can_edit`    | Signals edit rights (enforced by your edit endpoint)|
| `can_reshare` | Recipient may create their own share for this file  |

Permissions are checked server-side on every request — never inferred from the frontend.

---

## Security Notes

- Share tokens are generated with `secrets.token_urlsafe(32)` (256-bit entropy).
- Share-link passwords are hashed with Argon2id before storage; plaintext is never persisted.
- Password verification is rate-limited to **5 attempts per 15 minutes per IP**.
- Every access attempt (success or failure) is logged to `access_logs` for audit.
- Files are decrypted in-memory only; no plaintext copy is written to disk.
- Expired (`expires_at`) and revoked (`is_revoked=true`) shares are rejected with HTTP 410.
- Recipient-scoped shares reject requests from other authenticated users (403).
- **Envelope Encryption**: Each file has an ephemeral AES-256-GCM Data Encryption Key (DEK). Each DEK is wrapped with AES-256-GCM under a server-side Master Key Encryption Key (`MASTER_KEK`, 32-byte Base64) before being saved to the database (`wrapped_key_hex`).
  - *Note*: A server-side static KEK is a stopgap protecting against offline database compromise. Production systems should use a dedicated Key Management Service (AWS KMS, GCP KMS, or HashiCorp Vault).
- **CORS Configuration**: Cross-Origin Resource Sharing is managed via `flask-cors` and configured through the `ALLOWED_ORIGINS` environment variable (comma-separated list, e.g. `http://localhost:3000,http://localhost:5173`, defaulting safely to `http://localhost:3000`). Because the API uses `Authorization` headers with credentials, `ALLOWED_ORIGINS` must never be set to wildcard `*`.
- **API Documentation**: Interactive Swagger UI is available at `GET /api/docs` and the raw OpenAPI 3.1 specification is served at `GET /api/openapi.yaml`. Both routes are enabled in dev/test or when `ENABLE_API_DOCS=true` or `DEBUG=true`, and safely disabled (404) by default in production.
- **ClamAV Malware Scanning**: Pre-encryption anti-malware inspection is integrated into `POST /api/files` and `POST /api/files/<id>/versions` via `clamd`. Uploads flagged as infected are rejected with HTTP 422. Configurable via `SCAN_UPLOADS=true/false` (default `false` for development/testing without ClamAV), `CLAMD_HOST` (default `127.0.0.1`), and `CLAMD_PORT` (default `3310`). **Malware scanning should always be enabled (`SCAN_UPLOADS=true`) in production environments.**

---

## Running Tests

```bash
pytest tests/ -v
```

---

## Database Schema

### `shares`
| Column        | Type                    | Notes                          |
|---------------|-------------------------|--------------------------------|
| id            | UUID PK                 |                                |
| file_id       | UUID FK → files.id      |                                |
| owner_id      | UUID FK → users.id      |                                |
| recipient_id  | UUID FK → users.id NULL | NULL = open link               |
| can_view      | boolean                 |                                |
| can_download  | boolean                 |                                |
| can_edit      | boolean                 |                                |
| can_reshare   | boolean                 |                                |
| share_token   | varchar UNIQUE INDEX    | CSPRNG, never sequential       |
| password_hash | text NULL               | Argon2id                       |
| expires_at    | timestamptz NULL        |                                |
| is_revoked    | boolean default false   |                                |
| created_at    | timestamptz             |                                |
| updated_at    | timestamptz             |                                |

### `access_logs`
| Column      | Type             | Notes                                    |
|-------------|------------------|------------------------------------------|
| id          | UUID PK          |                                          |
| share_id    | UUID FK          |                                          |
| accessed_by | UUID FK NULL     | NULL for anonymous                       |
| ip_address  | varchar(45)      | IPv4 or IPv6                             |
| action      | varchar(32)      | view / download / edit / reshare / password_verify |
| success     | boolean          |                                          |
| detail      | text NULL        | failure reason                           |
| timestamp   | timestamptz      | indexed                                  |
