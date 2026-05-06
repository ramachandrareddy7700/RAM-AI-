# RAM AI — Phase 1
### Encrypted Personal AI Assistant | Self-Hosted | CPU-Only | Jarvis/Friday Mode

---

## What's Running in Phase 1

| Service | Role | Port |
|---------|------|------|
| **Nginx** | TLS 1.3 reverse proxy | 443 (public) |
| **FastAPI** | Encrypted REST API | internal only |
| **llama.cpp** | Mistral 7B LLM inference | internal only |
| **PostgreSQL** | Encrypted message storage | internal only |
| **Redis** | Sessions + rate limiting | internal only |

---

## Quick Start

### 1. Clone & enter directory
```bash
git clone <your-repo> ram-ai
cd ram-ai
```

### 2. Run setup (Ubuntu 24.04 only)
```bash
sudo bash setup.sh
```
This installs Docker, generates secrets, sets firewall rules, and starts all services.

### 3. Download the LLM model (~4.1 GB)
```bash
wget -c \
  https://huggingface.co/bartowski/Mistral-7B-Instruct-v0.3-GGUF/resolve/main/Mistral-7B-Instruct-v0.3-Q4_K_M.gguf \
  -O models/mistral-7b-instruct-v0.3.Q4_K_M.gguf
```

### 4. Generate TLS certs (local dev)
```bash
bash nginx/gen_certs.sh
```
For production, use Let's Encrypt:
```bash
sudo certbot certonly --standalone -d yourdomain.com
```

### 5. Start everything
```bash
docker compose up -d
```

### 6. Check health
```bash
curl http://localhost:8000/health
```

---

## Manual Setup (without setup.sh)

```bash
# 1. Generate master secret
python3 -c "import os; open('secrets/master_secret.bin','wb').write(os.urandom(32))"

# 2. Generate JWT RS256 key pair
openssl genrsa -out secrets/jwt_private_key.pem 4096
openssl rsa -in secrets/jwt_private_key.pem -pubout -out secrets/jwt_public_key.pem

# 3. Copy and fill .env
cp .env.example .env
# Edit .env: set DB_PASSWORD and REDIS_PASSWORD to strong random values

# 4. Generate TLS certs
bash nginx/gen_certs.sh

# 5. Build and start
docker compose up -d

# 6. Apply database schema
docker compose exec backend python -m db.migrate
```

---

## API Usage

### Register a new user
```bash
curl -X POST https://localhost/auth/register \
  -H "Content-Type: application/json" \
  -k -d '{
    "username": "boss",
    "password": "YourStr0ngPassword!",
    "display_name": "Boss"
  }'
```
Response includes a **TOTP QR code** — scan it with Google Authenticator or Authy.

### Login (requires TOTP code)
```bash
curl -X POST https://localhost/auth/login \
  -H "Content-Type: application/json" \
  -k -d '{
    "username": "boss",
    "password": "YourStr0ngPassword!",
    "totp_code": "123456"
  }'
```
Response: `{ "access_token": "...", "refresh_token": "..." }`

### Create a chat session
```bash
TOKEN="your_access_token"
curl -X POST https://localhost/chat/sessions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -k -d '{"title": "First Session"}'
```

### Send a message to Ram
```bash
SESSION_ID="your_session_id"
curl -X POST https://localhost/chat/sessions/$SESSION_ID/send \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -k -d '{
    "content": "Hey Ram, what can you do?",
    "nonce": "'$(python3 -c "import uuid; print(uuid.uuid4())")'"
  }'
```

### Stream a response (SSE)
```bash
NONCE=$(python3 -c "import uuid; print(uuid.uuid4())")
curl -N "https://localhost/chat/sessions/$SESSION_ID/stream?message=Hello+Ram&nonce=$NONCE" \
  -H "Authorization: Bearer $TOKEN" \
  -k
```

---

## Encryption Summary

Every layer of Ram is encrypted independently:

```
User password       → Argon2id hash (never stored in plain)
User message        → XSalsa20-Poly1305 before DB write
User memory         → XSalsa20-Poly1305 before DB write
TOTP secret         → XSalsa20-Poly1305 encrypted in DB
Per-user keys       → Wrapped with master-derived key (HMAC-SHA256)
Master secret       → Loaded from file/Docker secret, never in DB
JWT tokens          → RS256 asymmetric signing (4096-bit RSA)
All traffic         → TLS 1.3 only (no TLS 1.2, no legacy ciphers)
Redis data          → Password-protected, no external port
PostgreSQL          → No external port, password-protected
LLM inference       → Plaintext in RAM only, never written to disk
```

---

## Project Structure

```
ram-ai/
├── backend/
│   ├── main.py                 ← FastAPI app entry point
│   ├── config.py               ← All settings (from .env)
│   ├── requirements.txt
│   ├── Dockerfile
│   ├── crypto/
│   │   ├── message_crypto.py   ← XSalsa20-Poly1305 encrypt/decrypt
│   │   ├── key_management.py   ← Master key, per-user key derivation
│   │   └── token_crypto.py     ← JWT RS256 + refresh tokens
│   ├── db/
│   │   ├── init.sql            ← Schema (pgvector + pgcrypto)
│   │   ├── session.py          ← Async SQLAlchemy session
│   │   ├── models.py           ← ORM models
│   │   └── migrate.py          ← Migration runner
│   ├── routers/
│   │   ├── auth.py             ← Register / Login / Refresh / Logout
│   │   └── chat.py             ← Sessions / Send / History / Stream
│   ├── llm/
│   │   └── ram_inference.py    ← llama.cpp interface + Mistral prompt format
│   └── middleware/
│       ├── auth_middleware.py  ← JWT validation dependency
│       ├── rate_limiter.py     ← Sliding window rate limiter
│       └── audit_logger.py     ← Security event logging
├── llama_server/
│   └── Dockerfile              ← Builds llama.cpp from source
├── nginx/
│   ├── nginx.conf              ← TLS 1.3, security headers, rate limits
│   └── gen_certs.sh            ← Self-signed cert generator (dev)
├── models/                     ← Put your .gguf model here
├── secrets/                    ← Generated by setup.sh (never commit)
├── docker-compose.yml
├── setup.sh                    ← One-command Ubuntu setup
├── .env.example
└── .gitignore
```

---

## Monitoring

```bash
# Watch all logs live
docker compose logs -f

# Watch specific service
docker compose logs -f backend
docker compose logs -f llama_server

# Check service health
docker compose ps

# Database shell
docker compose exec postgres psql -U ram -d ram

# Redis shell
docker compose exec redis redis-cli -a $REDIS_PASSWORD
```

---

## Next Phase

Phase 2 adds the **ReAct Agent** — Ram's agentic brain:
- Shell / file / app control
- Autonomous web browsing (Playwright)
- Email reading & sending (IMAP/SMTP)
- Calendar integration (CalDAV)
- Private web search (SearXNG)

Phase 3 adds **Voice**:
- Wake word detection ("Hey Ram")
- Whisper.cpp speech-to-text
- Piper TTS voice responses

---

*Ram AI — Phase 1 Complete. All systems online, Boss.*
