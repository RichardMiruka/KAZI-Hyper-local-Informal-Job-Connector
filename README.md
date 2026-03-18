# 🔨 KAZI — Hyper-local Informal Job Connector

> Connect workers with employers in real-time. USSD + WhatsApp + REST API.
> Built for low-data environments. Works on any phone.

---

## 📋 Table of Contents
- [Architecture](#architecture)
- [Quick Start](#quick-start)
- [API Reference](#api-reference)
- [USSD Flow](#ussd-flow)
- [WhatsApp Flow](#whatsapp-flow)
- [Environment Variables](#environment-variables)
- [Deployment](#deployment)

---

## Architecture

```
Client Layer:  USSD (*XXX#)  |  WhatsApp Bot  |  REST API  |  Web App
                              ↓
Gateway:       Express + Nginx (rate limiting, JWT auth, routing)
                              ↓
Services:      Auth  |  Jobs  |  Matching  |  Payments  |  Ratings
                              ↓
Data:          PostgreSQL  |  Redis (cache + sessions)
                              ↓
External:      Africa's Talking (USSD+SMS)  |  M-Pesa  |  WhatsApp API
```

---

## Quick Start

### Option A — Docker (recommended)
```bash
git clone <your-repo>
cd kazi-app

# Copy and fill in your keys
cp .env.example .env

# Start everything
docker compose up -d

# Run migrations + seed
docker compose exec api node scripts/migrate.js
docker compose exec api node scripts/seed.js
```

### Option B — Local
```bash
# Prerequisites: Node 18+, PostgreSQL 14+, Redis 7+

npm install
cp .env.example .env
# Edit .env with your credentials

# Create DB
createdb kazi_db
createuser kazi_user
psql -c "GRANT ALL ON DATABASE kazi_db TO kazi_user"

# Run migrations
node scripts/migrate.js

# Seed demo data
node scripts/seed.js

# Start dev server
npm run dev
```

Server starts at: `http://localhost:3000`

---

## API Reference

All authenticated routes require: `Authorization: Bearer <JWT>`

### Auth

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/send-otp` | Send OTP to phone number |
| POST | `/api/auth/verify-otp` | Verify OTP, returns JWT |
| PUT  | `/api/auth/profile` | Update name, location, skills |
| GET  | `/api/auth/me` | Get current user |

**Send OTP:**
```json
POST /api/auth/send-otp
{
  "phone_number": "0712345678",
  "role": "worker"
}
```

**Verify OTP:**
```json
POST /api/auth/verify-otp
{
  "phone_number": "0712345678",
  "code": "123456"
}
→ { "user": {...}, "token": "eyJ..." }
```

---

### Jobs

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/jobs` | Post a job (employer) |
| GET  | `/api/jobs` | Search/list open jobs |
| GET  | `/api/jobs/:id` | Get job details |
| POST | `/api/jobs/:id/apply` | Apply for job (worker) |
| POST | `/api/jobs/:id/accept/:workerId` | Accept worker (employer) |
| POST | `/api/jobs/:id/complete` | Mark job complete |
| GET  | `/api/jobs/employer/mine` | Employer's jobs |

**Post a job:**
```json
POST /api/jobs
Authorization: Bearer <token>
{
  "skill_id": 1,
  "title": "House cleaning needed",
  "pay_amount": 500,
  "location_name": "Kibera",
  "latitude": -1.3133,
  "longitude": 36.7890,
  "workers_needed": 1
}
```

**Search jobs near me:**
```
GET /api/jobs?skill_id=1&latitude=-1.3133&longitude=36.789&radius_km=5
```

---

### Skills

```
GET /api/skills
→ [{ id, name, icon, ussd_code }, ...]
```

---

### Matching

```
GET /api/matching/workers?skill_id=1&latitude=-1.3133&longitude=36.789
→ Workers sorted by proximity + rating
```

---

### Payments

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/payments/stk-push` | Initiate M-Pesa STK push |
| POST | `/api/payments/mpesa/callback` | M-Pesa callback (Safaricom) |
| GET  | `/api/payments/history` | Payment history |

**STK Push:**
```json
POST /api/payments/stk-push
{
  "job_id": "uuid",
  "phone_number": "254712345678"
}
```

---

### Ratings

```json
POST /api/ratings
{
  "job_id": "uuid",
  "ratee_id": "uuid",
  "score": 5,
  "comment": "Very reliable, showed up on time"
}

GET /api/ratings/:userId
→ { summary: { avg_score, total_ratings }, ratings: [...] }
```

---

## USSD Flow

Dial your USSD code (configured as `AT_USSD_CODE`):

```
*XXX#
├── [New user]
│   ├── 1. Register as Worker → Enter name → Done
│   └── 2. Register as Employer → Enter name → Done
│
├── [Worker]
│   ├── 1. Find Work Today
│   │   ├── 1. Cleaning  2. Construction  3. Delivery ...
│   │   └── [Select job] → Apply → SMS confirmation
│   ├── 2. My Applications → List with status
│   ├── 3. My Profile → Rating, jobs done, skills
│   └── 4. My Earnings → Total earned
│
└── [Employer]
    ├── 1. Post a Job → Skill → Pay → Location → Confirm
    ├── 2. My Jobs → List with applicant counts
    └── 3. Find Workers → Skill → Top-rated nearby workers
```

**Africa's Talking setup:**
1. Register at africastalking.com
2. Create USSD service, set callback to: `https://yourdomain.com/ussd`
3. Set `AT_API_KEY` and `AT_USERNAME` in `.env`

---

## WhatsApp Flow

**Setup:**
1. Create Meta Business account
2. Set up WhatsApp Business App
3. Set webhook URL: `https://yourdomain.com/whatsapp/webhook`
4. Set verify token = `WHATSAPP_VERIFY_TOKEN` in `.env`

**Available commands:**
- `menu` / `hi` / `hello` — Main menu
- `help` — Help message

All other interactions are button/list driven — no typing needed.

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `DB_HOST` | ✅ | PostgreSQL host |
| `JWT_SECRET` | ✅ | Secret for JWT signing (min 32 chars) |
| `AT_API_KEY` | ✅ | Africa's Talking API key |
| `AT_USERNAME` | ✅ | AT username (`sandbox` for testing) |
| `MPESA_CONSUMER_KEY` | ✅ | Safaricom Daraja consumer key |
| `MPESA_CONSUMER_SECRET` | ✅ | Safaricom Daraja consumer secret |
| `MPESA_PASSKEY` | ✅ | Safaricom STK push passkey |
| `MPESA_CALLBACK_URL` | ✅ | Public URL for M-Pesa callback |
| `WHATSAPP_TOKEN` | ✅ | WhatsApp Business API token |
| `WHATSAPP_PHONE_NUMBER_ID` | ✅ | WA phone number ID |
| `WHATSAPP_VERIFY_TOKEN` | ✅ | Webhook verification token |
| `PLATFORM_FEE_KES` | ❌ | Platform fee per job (default: 20) |

---

## Deployment

### Production checklist
- [ ] Set `NODE_ENV=production`
- [ ] Use strong `JWT_SECRET` (32+ random chars)
- [ ] Enable HTTPS (Let's Encrypt via certbot)
- [ ] Set `MPESA_ENV=production`
- [ ] Configure `ALLOWED_ORIGINS` in `.env`
- [ ] Set up PM2 or systemd for process management
- [ ] Configure log rotation
- [ ] Set up DB backups (pg_dump cron)

### PM2 (production process manager)
```bash
npm install -g pm2
pm2 start src/index.js --name kazi-api --instances 2
pm2 save
pm2 startup
```

### Recommended server specs (MVP)
- VPS: 2 vCPU, 2GB RAM (DigitalOcean / Linode ~$12/month)
- DB: Same server or managed PostgreSQL
- OS: Ubuntu 22.04 LTS

---

## Monetization

| Model | Amount | Trigger |
|-------|--------|---------|
| Job posting fee | KES 10–20 | Employer posts a job |
| Success fee | KES 20 | Job is completed |
| Verified badge | KES 50/month | Worker premium feature |

**Rule:** Never charge workers to find work. Revenue comes from employers.

---

## Demo Data

After seeding, test with:
- **Worker:** `254700000001` (John Kamau — Cleaning + Delivery)
- **Employer:** `254711000001` (Amina Hassan)
- All OTPs in sandbox: use Africa's Talking sandbox simulator

Area coverage: Kibera, Nairobi (`-1.31, 36.79`)
