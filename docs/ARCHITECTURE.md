# TutorMatch BD — Production Architecture & Security Blueprint

> Hybrid tutor–student matchmaking platform for Bangladesh (Web / iOS / Android),
> agency model, discovery/"match" workflow. Zero-Trust, privacy-first, MFS-integrated.
>
> Document owner: Platform Architecture | Status: v1.0 baseline | Review cadence: quarterly

---

## 0. Executive Summary & Guiding Principles

| Principle | What it means here |
|---|---|
| **Zero-Trust** | No implicit trust by network location. Every request is authenticated, authorized, and encrypted. "Never trust, always verify." |
| **Privacy by Design / Default** | PII is segregated, encrypted, minimized, and access-logged from day one — not bolted on. |
| **Least Privilege** | Every human, service, and token gets the *minimum* scope to do its job, time-boxed where possible. |
| **Defense in Depth** | WAF → API GW → AuthN/Z → service mesh → row-level security → field-level encryption. A single failure must not be catastrophic. |
| **Safety as a Feature** | A "dating-app" discovery UX over minors and home visits demands harassment controls as first-class architecture, not moderation afterthoughts. |

**Compliance anchors:** Bangladesh *Digital Security Act / proposed Data Protection Act (PDPO drafts)*, Bangladesh Bank MFS guidelines (for bKash/Nagad flows), and PCI-DSS-aligned handling even though MFS offloads card data. Where local law is immature, we default to **GDPR-grade** controls.

---

## 1. High-Level System Architecture & Tech Stack

### 1.1 Logical Architecture (decoupled, defense-in-depth)

```
                    ┌──────────────────────────────────────────────┐
                    │                  CLIENTS                       │
                    │  Flutter app (iOS/Android)  +  Flutter Web/PWA │
                    └───────────────┬──────────────────────────────┘
                                    │  TLS 1.3 only, cert pinning
                                    ▼
                    ┌──────────────────────────────────────────────┐
                    │  EDGE:  CDN (CloudFront)  +  WAF  +  Shield/   │
                    │         Cloud Armor (L3/L4/L7 DDoS, rate-lim)  │
                    └───────────────┬──────────────────────────────┘
                                    ▼
                    ┌──────────────────────────────────────────────┐
                    │  API GATEWAY  (AuthN, JWT verify, throttling,  │
                    │  request validation, schema enforcement)       │
                    └───────────────┬──────────────────────────────┘
                                    │  mTLS inside mesh
        ┌───────────────┬──────────┼───────────┬────────────────┬───────────────┐
        ▼               ▼          ▼           ▼                ▼               ▼
 ┌────────────┐ ┌────────────┐ ┌────────┐ ┌──────────┐ ┌──────────────┐ ┌──────────┐
 │ Identity   │ │ Profile/   │ │ Match  │ │ Verify   │ │ Payment      │ │ Chat/    │
 │ (OTP, JWT, │ │ Catalog    │ │ Engine │ │ (KYC/    │ │ (MFS, ledger)│ │ Messaging│
 │ sessions)  │ │ svc        │ │ + Geo  │ │ docs)    │ │              │ │ + AI bot │
 └─────┬──────┘ └─────┬──────┘ └───┬────┘ └────┬─────┘ └──────┬───────┘ └────┬─────┘
       │              │            │           │              │              │
       ▼              ▼            ▼           ▼              ▼              ▼
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │ DATA TIER (private subnets, no public IP)                                       │
 │  • Postgres "core" (relational, PostGIS for geo)  • Postgres "PII vault" (sep.) │
 │  • Redis (OTP, sessions, rate-limit, feed cache)  • S3/GCS (encrypted docs)     │
 │  • Kafka/PubSub (events: match, payment, audit)   • OpenSearch (search/audit)   │
 │  • KMS/HSM (envelope keys)                         • Append-only audit log       │
 └──────────────────────────────────────────────────────────────────────────────┘
```

**Why microservices (not a monolith) here:** the *Verification*, *Payment*, and *PII* domains have radically different compliance/blast-radius profiles than the discovery feed. Isolating them lets you apply stricter network policy, separate KMS keys, and independent audit. Start as a **modular monolith** if the team is small, but keep these three as separately deployable services from day one.

### 1.2 Recommended Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Mobile + Web | **Flutter** (single codebase → iOS/Android/Web) | See §1.3. One Dart codebase, strong perf, mature. |
| API Gateway | AWS API Gateway / Kong / GCP API Gateway | Centralized authZ, throttling, schema validation. |
| Backend services | **Node.js (NestJS, TypeScript)** or **Go** | NestJS = fast delivery + strong typing; Go for payment/match if perf-critical. Python (FastAPI) fine for AI/verification services. |
| Identity | **Keycloak** (self-host) or **AWS Cognito / Auth0** | OTP, JWT, MFA, OIDC. Self-host Keycloak for data residency. |
| Primary DB | **PostgreSQL 16 + PostGIS** | ACID for money + native geospatial. |
| Cache/ephemeral | **Redis** | OTP store, rate limiting, session, feed cache. |
| Object store | **S3 / GCS** (SSE-KMS) | Encrypted docs, signed URLs. |
| Events | **Kafka / GCP Pub/Sub / AWS SNS+SQS** | Async match, payment reconciliation, audit. |
| Secrets | **AWS Secrets Manager / GCP Secret Manager + KMS** | No secrets in code or client. |
| IaC | **Terraform** | Reproducible, reviewable infra. |
| Observability | OpenTelemetry → Grafana/Datadog; centralized SIEM | Trust requires audit. |

> **Bangladesh data-residency note:** If regulators require local hosting of KYC/financial data, run the **PII vault + Verification + Payment ledger** in a local/regional DC or AWS/GCP region nearest (Mumbai `ap-south-1` / `asia-south1` is the pragmatic default; Singapore as fallback) and keep the discovery feed wherever latency is best.

### 1.3 Flutter vs. React Native — decision + security analysis

**Recommendation: Flutter.** True single codebase across Web+iOS+Android, compiled to native ARM (harder to reverse than JS bundles), consistent rendering, and a strong security tooling ecosystem. React Native is viable but its JS bundle is comparatively easier to inspect and its web story is weaker for a discovery-feed UX.

**Hybrid-app threat model & mitigations (applies to both, Flutter specifics noted):**

| Vulnerability | Risk | Mitigation |
|---|---|---|
| **Reverse engineering / decompilation** | Attacker extracts business logic, endpoints, weak checks | Flutter AOT-compiles Dart → native (better baseline). Add **code obfuscation** (`flutter build apk --obfuscate --split-debug-info=...`). Enable R8/ProGuard on Android. Strip symbols on iOS. |
| **Secrets in the binary** | API keys, signing secrets extracted from APK/IPA | **Never** ship server secrets in the client. SMS/MFS/AI keys live **server-side only** (see §4, §5). Client holds only public identifiers. |
| **Insecure local storage** | Tokens/PII readable on rooted/jailbroken device | Use **`flutter_secure_storage`** (Keychain / Keystore-backed). Never store PII or refresh tokens in `SharedPreferences`/`localStorage`. Short-lived access tokens in memory. |
| **MITM / traffic interception** | Token theft, request tampering | **TLS 1.3 + certificate pinning** (`http` client with pinned SPKI / `dio` + `certificate_pinning`). Reject user-added CAs. |
| **Tampering / repackaging** | Modified app distributed with malware/backdoor | **Play Integrity API** (Android) + **App Attest / DeviceCheck** (iOS). Server rejects requests from unattested clients for sensitive ops (payments, verification). |
| **Root/jailbreak** | Hooking (Frida), runtime patching | Root/JB detection (`flutter_jailbreak_detection`) + anti-Frida checks; degrade gracefully (block payments, allow read-only). Treat as signal, not absolute. |
| **Web (PWA) XSS / token theft** | Stolen session via injected script | CSP headers, HttpOnly+Secure+SameSite cookies for web sessions, DOM sanitization, Subresource Integrity. |
| **Screenshots of sensitive data** | KYC docs / contact leakage | `FLAG_SECURE` on Android, secure-screen on iOS for KYC + chat-with-contact screens. |

> **Key mindset:** the client is **untrusted**. Every control that matters (authZ, masking, rate limits, integrity) is *re-enforced server-side*. Client-side hardening only raises the cost of attack.

### 1.4 Database schema — geospatial + strict PII isolation

**Core idea: two-vault split.** A `core` database holds operational, low-sensitivity data and *tokens/references* to PII. A separate **`pii_vault`** database (different credentials, different KMS key, tighter network policy, field-level encryption) holds names, NID numbers, phone numbers, addresses, and document pointers. The core DB never stores raw PII — only a `pii_ref` UUID.

#### `core` database (PostgreSQL + PostGIS)

```sql
-- Users: NO PII here. Identity + role + a pointer into the vault.
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role            TEXT NOT NULL CHECK (role IN ('guardian','tutor','staff','admin')),
    status          TEXT NOT NULL DEFAULT 'active',
    pii_ref         UUID NOT NULL,                 -- FK conceptually to pii_vault.identities
    phone_hash      BYTEA NOT NULL UNIQUE,         -- HMAC-SHA256(phone, pepper) for lookup/login
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_login_at   TIMESTAMPTZ
);

-- Tutor public-ish profile (still gated by match/RBAC at API layer)
CREATE TABLE tutor_profiles (
    user_id            UUID PRIMARY KEY REFERENCES users(id),
    display_name       TEXT,                       -- chosen alias, NOT legal name
    headline           TEXT,
    bio                TEXT,
    subjects           TEXT[],
    levels             TEXT[],                     -- e.g. {class_6, hsc, admission}
    hourly_rate_bdt    INTEGER,
    gender_pref        TEXT,
    verification_state TEXT NOT NULL DEFAULT 'unverified'
                        CHECK (verification_state IN
                        ('unverified','pending','verified','rejected','suspended')),
    rating_avg         NUMERIC(2,1),
    -- Coarsened location only: snapped to a neighborhood cell, never exact home
    area_id            UUID REFERENCES areas(id),
    geo_cell           GEOGRAPHY(POINT, 4326),     -- centroid of cell, NOT real coords
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tutor_geo ON tutor_profiles USING GIST (geo_cell);
CREATE INDEX idx_tutor_subjects ON tutor_profiles USING GIN (subjects);

-- Tuition requirements posted by guardians
CREATE TABLE tuition_posts (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guardian_id   UUID NOT NULL REFERENCES users(id),
    subjects      TEXT[] NOT NULL,
    level         TEXT NOT NULL,
    budget_bdt    INTEGER,
    schedule      JSONB,
    gender_pref   TEXT,
    area_id       UUID REFERENCES areas(id),
    geo_cell      GEOGRAPHY(POINT, 4326),
    status        TEXT NOT NULL DEFAULT 'open',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_post_geo ON tuition_posts USING GIST (geo_cell);

-- Neighborhood reference grid (Dhaka thana/ward etc.). Public, non-sensitive.
CREATE TABLE areas (
    id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name_en   TEXT, name_bn TEXT,
    district  TEXT,
    boundary  GEOGRAPHY(POLYGON, 4326)   -- used for geofencing / area matching
);
CREATE INDEX idx_area_boundary ON areas USING GIST (boundary);

-- Swipe/discovery interactions
CREATE TABLE swipes (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id    UUID NOT NULL REFERENCES users(id),
    target_id   UUID NOT NULL REFERENCES users(id),
    direction   TEXT NOT NULL CHECK (direction IN ('like','pass')),
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (actor_id, target_id)
);

-- A confirmed match unlocks reveal + chat
CREATE TABLE matches (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guardian_id   UUID NOT NULL REFERENCES users(id),
    tutor_id      UUID NOT NULL REFERENCES users(id),
    state         TEXT NOT NULL DEFAULT 'matched'
                   CHECK (state IN ('matched','contact_unlocked','closed','blocked')),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (guardian_id, tutor_id)
);

-- Safety: reports & blocks
CREATE TABLE reports (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reporter_id  UUID NOT NULL REFERENCES users(id),
    reported_id  UUID NOT NULL REFERENCES users(id),
    category     TEXT NOT NULL,   -- harassment, scam, fake_profile, inappropriate
    detail       TEXT,
    status       TEXT NOT NULL DEFAULT 'open',
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE blocks (
    blocker_id   UUID NOT NULL REFERENCES users(id),
    blocked_id   UUID NOT NULL REFERENCES users(id),
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (blocker_id, blocked_id)
);
```

#### `pii_vault` database (separate instance/credentials/KMS key)

```sql
-- Each PII column is encrypted at the application layer with envelope encryption.
-- Stored as ciphertext (BYTEA). Searchable fields also keep a blind index (HMAC).
CREATE TABLE identities (
    pii_ref          UUID PRIMARY KEY,             -- referenced by core.users.pii_ref
    legal_name_enc   BYTEA NOT NULL,
    phone_enc        BYTEA NOT NULL,
    phone_bidx       BYTEA NOT NULL,               -- blind index for exact-match lookup
    email_enc        BYTEA,
    address_enc      BYTEA,
    exact_geo_enc    BYTEA,                         -- real coordinates, only here
    dek_id           TEXT NOT NULL,                 -- which data-encryption-key/version
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE kyc_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pii_ref         UUID NOT NULL REFERENCES identities(pii_ref),
    doc_type        TEXT NOT NULL CHECK (doc_type IN ('nid','passport','certificate','photo')),
    object_key      TEXT NOT NULL,                  -- S3 key (object itself SSE-KMS encrypted)
    sha256          BYTEA NOT NULL,                 -- integrity check of stored file
    nid_number_bidx BYTEA,                          -- blind index, never plaintext NID
    review_state    TEXT NOT NULL DEFAULT 'pending',
    reviewer_id     UUID,
    reviewed_at     TIMESTAMPTZ,
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Append-only access log for PII reads (who saw what, when, why)
CREATE TABLE pii_access_log (
    id          BIGSERIAL PRIMARY KEY,
    actor_id    UUID NOT NULL,
    pii_ref     UUID NOT NULL,
    field       TEXT NOT NULL,
    purpose     TEXT NOT NULL,        -- match_unlock, support_ticket#123, kyc_review
    at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Why blind indexes?** You can't `WHERE phone = '01...'` on encrypted data. A **blind index** = `HMAC(phone, index_key)` lets you do exact-match lookups without ever storing plaintext, and without enabling range/partial scans that would leak data.

**Geospatial privacy trick:** the `core` DB only stores `geo_cell` — the centroid of a ~1–2 km neighborhood cell (or area centroid). Exact coordinates live only in `pii_vault.exact_geo_enc` and are *never* returned to clients. Matching uses `ST_DWithin(geo_cell, :searcher_cell, radius)` on coarsened points. See §3.2.

---

## 2. Rigorous Security & Trust Framework (Zero-Trust)

### 2.1 Tutor Verification Pipeline (NID / passport / certificates)

**Goal:** ingest highly sensitive identity documents, verify them, and ensure they're encrypted, access-controlled, retention-limited, and never exposed to other users — while supporting a semi-automated reviewer workflow.

**Data flow (ingest → store → verify → decide):**

```
 Tutor app                Backend (Verify svc)            Storage / Review
 ─────────                ────────────────────            ────────────────
 1. Request upload  ──►   Issue PRE-SIGNED PUT URL
    slot                  (S3, 5-min expiry, content-type
                          + size + SSE-KMS enforced)
 2. PUT file directly ─────────────────────────────►  S3 "kyc-raw" bucket
    to S3 (TLS)                                        (Object Lock, SSE-KMS,
                                                        private, no public ACL)
 3. S3 event ─────────►   Verify worker:
                          - virus/malware scan (ClamAV/GuardDuty)
                          - compute sha256, store in pii_vault.kyc_documents
                          - OCR + face match (auto pre-screen)
                          - NID format/checksum validation
                          - de-dup via nid_number_bidx
 4.                       If auto-confident → 'verified'
                          else queue for HUMAN reviewer ──► Reviewer console
                                                            (time-limited signed
                                                             URL, watermarked,
                                                             FLAG_SECURE, logged)
 5.                       Decision writes verification_state
                          + emits audit event. Raw doc moves to
                          cold, access-restricted tier or is purged
                          per retention policy.
```

**Hardening specifics:**

- **Direct-to-S3 pre-signed upload** — the document never transits through (or is logged by) the API servers. The pre-signed URL enforces `Content-Type`, `Content-Length` range, and `x-amz-server-side-encryption: aws:kms`.
- **Dedicated KMS key** for the KYC bucket with a tight key policy: only the Verify service role and a break-glass admin role can decrypt; key usage is CloudTrail-logged.
- **S3 Object Lock (compliance mode)** + versioning to prevent tampering/deletion during the review window.
- **No raw doc ever returned to clients.** Reviewers see them only via short-lived (e.g., 60-second) signed URLs, dynamically **watermarked** with reviewer ID + timestamp, on a `FLAG_SECURE` screen. Every view writes to `pii_access_log`.
- **Automated pre-screen** (OCR + liveness/face-match against selfie) reduces human exposure to PII and speeds throughput; humans only adjudicate edge cases.
- **Retention & minimization:** once verified, you may keep a *verification result + hash + masked NID*, and **purge or deep-archive the raw image** after the legal retention window. Don't hoard original documents.
- **Third-party option:** Bangladesh-aware KYC vendors / Porichoy (NID verification via Bangladesh Election Commission gateway) can validate NID authenticity server-to-server — call it from the Verify service with mTLS + IP allowlist, never from the client.

### 2.2 Data Protection & Privacy

**Encryption in transit (TLS 1.3):**
- TLS 1.3 only (disable TLS ≤1.2 where possible; minimum 1.2 if a legacy MFS endpoint requires it). Strong ciphers only (AEAD), HSTS with preload, OCSP stapling.
- **mTLS inside the mesh** (service-to-service) so a compromised pod can't freely call the payment/PII services.
- **Certificate pinning** on mobile clients.

**Encryption at rest (AES-256):**
- **Storage-level:** S3 SSE-KMS, RDS/Postgres encryption, EBS, Redis, and backups all AES-256 via KMS CMKs (separate keys per domain: `core`, `pii`, `kyc`, `payments`).
- **Field-level (envelope encryption) for PII:** generate a per-record/per-tenant **Data Encryption Key (DEK)**, encrypt the field with `AES-256-GCM`, then encrypt the DEK with the KMS **Key Encryption Key (KEK)**. Store ciphertext + wrapped DEK. KMS never exposes the KEK.

```typescript
// Envelope encryption (Node/TS) — conceptual, library-agnostic
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

async function encryptField(plaintext: string, kms: Kms, keyId: string) {
  const { Plaintext: dek, CiphertextBlob: wrappedDek } =
    await kms.generateDataKey({ KeyId: keyId, KeySpec: 'AES_256' });
  const iv = randomBytes(12);
  const cipher = createCipheriv('aes-256-gcm', dek, iv);
  const ct = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  const tag = cipher.getAuthTag();
  dek.fill(0); // zero the plaintext DEK in memory ASAP
  return Buffer.concat([wrappedDek, iv, tag, ct]); // store this blob
}
```

- **Key rotation:** KMS automatic annual KEK rotation; DEKs versioned (`dek_id`) so re-encryption can be lazy/background.
- **Blind indexes** use a separate HMAC pepper stored in Secrets Manager (not the DB).

**RBAC model (least privilege):**

| Role | Can see / do | Cannot see |
|---|---|---|
| **Guardian** | Own posts; tutor *public* profile (blurred pre-match); contact of tutor **only after match + unlock** | Tutor legal name, NID, exact address, other guardians' data |
| **Tutor** | Own profile/docs; guardian *public* requirement (area-level, no contact pre-match); contact only post-match | Other tutors' PII, guardian contact pre-match, raw docs of others |
| **Support staff** | Operational metadata, tickets; PII **only via just-in-time, purpose-bound, logged access** | Bulk PII export, payment secrets, raw KYC unless escalated |
| **KYC reviewer** | KYC docs **only for assigned, pending cases**, time-limited, watermarked, logged | Payment data, unrelated cases, bulk download |
| **Finance** | Payment ledger, reconciliation (masked user identifiers) | KYC docs, chat content |
| **Admin/SRE** | Infra, deploys; **no standing access to plaintext PII** — break-glass only, dual-control, alerting | Decrypted PII by default |

Implementation: OIDC roles → **ABAC policies** (e.g., OPA/Rego or Cedar) evaluated at the API gateway *and* service layer + **Postgres Row-Level Security (RLS)** so even a leaked DB credential can't read across tenants.

```sql
-- Row-level security: a guardian only reads their own posts
ALTER TABLE tuition_posts ENABLE ROW LEVEL SECURITY;
CREATE POLICY guardian_own_posts ON tuition_posts
  USING (guardian_id = current_setting('app.current_user_id')::uuid);
```

**Audit:** every PII read (`pii_access_log`), every privileged action, and every KMS decrypt is logged to an **append-only** store and shipped to SIEM with anomaly alerts (e.g., a support agent reading 200 profiles in an hour → page on-call).

### 2.3 User Safety & Anti-Harassment (critical for a "discovery" model with minors)

Because this resembles a dating-app flow but involves **children and home visits**, safety controls are architecture, not moderation afterthoughts.

| Control | How it's enforced |
|---|---|
| **Blurred/aliased profiles pre-match** | Photos served only as blurred derivatives until `matches.state >= matched`. Legal names never shown; only `display_name` alias. Enforced server-side (the original photo URL is never sent to the unmatched client). |
| **Mutual-consent reveal** | Contact details unlock **only** when *both* parties opt in (`state = contact_unlocked`). One-sided "like" reveals nothing. |
| **Contact-number masking / proxy** | Even after unlock, prefer **in-app chat first**. For phone, use a **masking/proxy number** (telco/Twilio-style relay) so real numbers stay hidden; can be revoked on report. |
| **No exact location ever** | Only neighborhood/area shown (§3.2). Exact address shared *only* at booking confirmation, optionally via a one-time reveal, logged. |
| **Reporting & block triggers** | `reports`/`blocks` tables; a block instantly hides both parties from each other's feed/chat. Repeated reports auto-suspend (`verification_state='suspended'`) pending review. |
| **Rate-limited interactions** | Cap likes/messages per hour to throttle spammers/predators; abnormal patterns flagged. |
| **Content moderation in chat** | Pipe messages through a moderation classifier (abuse/PII-leak/grooming-pattern detection); flag + queue for human review. Image moderation (CSAM hashing via known hash sets, e.g., PhotoDNA-style) on uploads — **mandatory** given minors. |
| **Guardian-mediated minor accounts** | Students who are minors don't hold independent accounts; the **guardian** is the account holder. No direct tutor↔minor private channel without guardian visibility. |
| **Verified badges** | Only `verified` tutors appear in default feed; unverified are limited/hidden. |
| **Panic/safety center** | In-app report, block, emergency info, and "share booking with trusted contact." |

> **Design rule:** every "reveal" (photo, phone, address) is a **server-side authorization decision** based on match state + consent + not-blocked + not-suspended — never a client-side toggle.

---

## 3. Core Feature Implementation

### 3.1 Secure passwordless OTP login (BD mobile numbers)

**Threats:** SMS pumping/toll fraud (attacker triggers thousands of OTPs to premium ranges to earn revenue), OTP brute force, OTP interception, session hijacking, account takeover.

**Flow:**

```
1. Client POST /auth/otp/request { phone }   (+ device attestation token,
                                               + CAPTCHA/Turnstile token,
                                               + app-integrity assertion)
2. Server:
   - validate phone is a real BD MSISDN (+8801[3-9]XXXXXXXX), normalize E.164
   - check rate limits (per-phone, per-IP, per-device) in Redis
   - check SMS-pumping heuristics (geo/number-range anomalies, velocity)
   - generate 6-digit OTP, store HASH only: Redis key otp:{phoneHash}
       value = { hash: HMAC(otp, pepper), attempts:0, exp:120s }
   - send OTP via server-side SMS gateway (key never on client)
   - return opaque request_id (no info leak about whether phone exists)
3. Client POST /auth/otp/verify { request_id, otp, device_id }
4. Server:
   - lookup by request_id, constant-time compare HMAC(otp)
   - increment attempts; after 5 fails → invalidate + lockout/backoff
   - on success: issue short-lived access JWT (5–15 min) + rotating refresh token
   - bind tokens to device fingerprint; store refresh token hash server-side
```

**Anti–SMS-pumping defenses (must-have, this is real money loss in BD):**
- **CAPTCHA / Cloudflare Turnstile** before the *first* OTP send.
- **Multi-dimension rate limits:** per phone (e.g., 3/hour, 5/day), per IP, per device, plus a **global circuit breaker** if total OTP volume spikes.
- **Number-range / geo anomaly detection:** block premium/known-fraud ranges; alert on bursts to sequential numbers.
- **Progressive friction / backoff** (exponential delay after repeated requests).
- **Device attestation** (Play Integrity / App Attest) required before OTP send for new devices.
- **Cost alarms** on the SMS spend metric → auto-throttle.

**Anti–brute-force:**
- 6-digit OTP, **120-second TTL**, **max 5 attempts**, then invalidate.
- Store only `HMAC(otp, pepper)`; constant-time comparison.
- Lock the phone+IP for a cooldown after lockout.

**Anti–session-hijacking:**
- Short-lived access JWT (RS256, kid-rotated keys) + **refresh-token rotation** with reuse detection (if an old refresh token is replayed → revoke the whole family, force re-auth).
- Bind sessions to device + (coarse) context; **store refresh token in `flutter_secure_storage`** (web: HttpOnly+Secure+SameSite=strict cookie).
- TLS + cert pinning; reject tokens on integrity-failed devices for sensitive ops.
- Server-side session/refresh registry in Redis enables instant global logout/revocation.

```typescript
// Rate-limit + pumping guard (sketch)
async function requestOtp(phone: string, ctx: ReqCtx) {
  const e164 = normalizeBd(phone);          // throws if not +8801[3-9]\d{8}
  await turnstile.verify(ctx.captchaToken); // bot/pumping gate
  await integrity.verify(ctx.attestation);  // Play Integrity / App Attest
  await rl.consume(`otp:phone:${e164}`, { points: 3, durationSec: 3600 });
  await rl.consume(`otp:ip:${ctx.ip}`,   { points: 10, durationSec: 3600 });
  if (await fraud.isSuspiciousRange(e164)) throw new Forbidden();
  const otp = randomDigits(6);
  await redis.setex(`otp:${hmac(e164)}`, 120,
      JSON.stringify({ h: hmac(otp), attempts: 0 }));
  await sms.send(e164, otpTemplate(otp));    // server-side key only
  return { request_id: opaqueId(e164) };
}
```

### 3.2 The Matchmaking Engine (privacy-preserving geofencing)

**Principle: exact coordinates never leave the device-of-origin's vault.** The client may capture GPS, but it's immediately **coarsened** (snapped to a neighborhood cell or area) before anything is stored in `core` or shown to others.

**Location handling:**
1. Client gets GPS (with permission) → **snaps locally** to the nearest area/cell centroid (or sends raw GPS once to the server which snaps and stores exact only in `pii_vault.exact_geo_enc`, returning a `geo_cell`).
2. `core` stores only `geo_cell` (centroid) + `area_id`. Feeds and other users see **area name only** ("Dhanmondi", "Mirpur-10"), never coordinates or distance-to-the-meter.
3. Matching query operates on coarsened geography:

```sql
-- Tutors near a guardian's post: area + radius on coarsened cells
SELECT t.user_id, t.display_name, t.subjects, t.hourly_rate_bdt, t.rating_avg,
       a.name_en AS area
FROM tutor_profiles t
JOIN areas a ON a.id = t.area_id
WHERE t.verification_state = 'verified'
  AND t.subjects && :wanted_subjects                       -- array overlap
  AND :wanted_level = ANY (t.levels)
  AND ST_DWithin(t.geo_cell, :post_cell, :radius_m)        -- coarse proximity
  AND t.user_id NOT IN (SELECT blocked_id FROM blocks WHERE blocker_id = :me)
  AND t.user_id NOT IN (SELECT target_id FROM swipes WHERE actor_id = :me)
ORDER BY ST_Distance(t.geo_cell, :post_cell), t.rating_avg DESC
LIMIT 30;
```

**Discovery feed:**
- Server builds a ranked, **already-filtered** feed (subject/level/area/availability/rating + safety filters: verified-only, not-blocked, not-already-swiped). Ranking can be a simple weighted score now, ML later.
- **Photos in the feed are blurred derivatives**, contact fields absent (enforced server-side). The client literally never receives the data it isn't allowed to reveal.
- A **match** (`like` from both sides, or guardian-likes-tutor + tutor-accepts) creates a `matches` row → unlocks unblurred photo + in-app chat → mutual consent unlocks masked contact.
- Cache feed pages in Redis (short TTL) keyed by (user, filters) for performance; never cache PII.

---

## 4. Localized Integrations & Financial Security

### 4.1 MFS Payment Gateway (SSLCommerz / bKash / Nagad)

**Golden rules:** all credentials and signing keys live **server-side only**; the client never talks to the MFS API directly; every callback is **verified cryptographically and server-confirmed** before granting value.

**Secure payment flow (SSLCommerz pattern; bKash/Nagad analogous with their token APIs):**

```
1. Client → POST /payments/initiate { match_id / booking_id, amount }
2. Payment svc:
   - create local txn row state='INITIATED', generate unique tran_id (idempotency key)
   - server-to-server call to gateway init with store_id/store_passwd (from Secrets Mgr)
   - persist gateway session/val_id ↔ tran_id
   - return gateway redirect URL / SDK token to client
3. User completes payment in MFS (bKash/Nagad/card) via gateway-hosted page/SDK
4. Gateway → IPN/Webhook → POST /payments/webhook (server-to-server)
   - VERIFY: signature/hash (verify_sign / checksum) using secret
   - VERIFY: re-call gateway "validation API" (validator) with val_id over server-to-server
             to independently confirm amount, currency, status == VALID/VALIDATED
   - VERIFY: amount + currency + tran_id match our INITIATED record
   - idempotent: if tran_id already SETTLED → ignore (prevents double-credit)
   - on success → state='PAID', write immutable ledger entry, emit event
5. Client polls /payments/{tran_id}/status (never trusts its own redirect result)
```

**Securing webhooks against tampering:**
- **Verify the signature/checksum** on every callback (SSLCommerz `verify_sign`/`verify_key` hash; bKash/Nagad signed responses). Reject mismatches.
- **Never trust webhook amount/status alone** — always **re-validate server-to-server** against the gateway's validation API using `val_id`/transaction reference. This is the canonical anti-tamper step.
- **IP allowlist** the gateway's callback IPs at the WAF; require HTTPS; optionally mTLS.
- **Constant-time** signature comparison; reject stale/expired callbacks (timestamp window).

**Preventing double-spending / replay:**
- **Idempotency key** = your unique `tran_id`; a `UNIQUE` constraint on it. Processing is idempotent — a repeated webhook for a settled txn is a no-op.
- **State machine** with allowed transitions only (`INITIATED → PAID → SETTLED`; no backward jumps). Use a DB transaction + row lock when settling.
- **Append-only double-entry ledger** for auditability and reconciliation.

```typescript
// Webhook verification (SSLCommerz-style, sketch)
app.post('/payments/webhook', async (req, res) => {
  const p = req.body;
  // 1. signature check
  if (!verifySslczHash(p, secrets.storePass)) return res.sendStatus(400);
  // 2. independent server-to-server validation (authoritative)
  const v = await sslcz.validate(p.val_id, secrets); // GET validator API
  if (v.status !== 'VALID' && v.status !== 'VALIDATED') return res.sendStatus(400);
  // 3. match our record + amount/currency
  const txn = await db.tx(async t => {
    const row = await t.oneOrNone(
      `SELECT * FROM payments WHERE tran_id=$1 FOR UPDATE`, [p.tran_id]);
    if (!row) return null;
    if (row.state === 'PAID' || row.state === 'SETTLED') return row; // idempotent
    if (Number(v.amount) !== Number(row.amount_bdt) || v.currency !== 'BDT')
      throw new Error('amount mismatch');
    await t.none(`UPDATE payments SET state='PAID', val_id=$2 WHERE tran_id=$1`,
                 [p.tran_id, p.val_id]);
    await t.none(`INSERT INTO ledger(tran_id,debit,credit,amount) VALUES(...)`);
    return row;
  });
  return res.sendStatus(txn ? 200 : 404);
});
```

**Reconciliation:** nightly job pulls the gateway's settlement report and reconciles against the ledger; discrepancies alert finance. Refund/chargeback flows are explicit ledger reversals, never row deletes.

### 4.2 SMS Gateway Security (Greenweb / BulkSMSBD / etc.)

- **API keys live only on the server** (Secrets Manager / env injected at runtime), **never in the Flutter binary or web bundle.** The client calls *your* `/auth/otp/request`; your backend calls the SMS provider.
- Outbound SMS goes through a **single internal SMS service** (one egress point) with: provider failover, rate limiting, per-message audit, cost metering, and template management.
- **mTLS / IP allowlist** to the provider where supported; rotate keys regularly; scope keys to the sending number/route.
- **Don't log OTP contents.** Log message metadata (to-hash, template id, provider, status) only.
- Monitor delivery rates + spend; auto-throttle on anomalies (ties into anti-pumping, §3.1).

---

## 5. Secure AI Chatbot Infrastructure

### 5.1 Blueprint

```
User ─► App ─► /ai/chat (your backend, authenticated)
                  │
                  ▼
        ┌───────────────────────────────────────────┐
        │  AI GATEWAY (your service)                  │
        │  1. AuthZ + rate limit + abuse filter       │
        │  2. PII DETECTION & REDACTION (pre-LLM)     │
        │  3. Prompt-injection / jailbreak filtering  │
        │  4. Language detect (en / bn / bangl­ish)    │
        │  5. RAG: fetch only NON-PII KB context      │
        │  6. Call LLM (OpenAI/Vertex/Claude) w/      │
        │     server-side key, no-train/zero-retention│
        │  7. Output filter + RE-INSERT safe tokens   │
        │  8. Log (redacted) for audit                │
        └───────────────────────────────────────────┘
```

The chatbot is for **support/FAQ/discovery help**, not for handling payments or revealing PII. It has **no direct DB access** to PII; it queries scoped, non-sensitive endpoints only.

### 5.2 PII-sanitization layer (before any data leaves to the third-party LLM)

This is the critical control: users *will* paste phone numbers, NIDs, addresses, and amounts.

- **Detect & redact pre-send.** Run a layered detector: regex for BD-specific patterns + an NER/PII model:
  - BD phone: `(?:\+?8801|01)[3-9]\d{8}`
  - NID: 10 / 13 / 17-digit sequences
  - bKash/Nagad numbers, amounts, email, addresses, names.
- **Tokenize, don't just strip:** replace `01712345678` → `«PHONE_1»`. Keep a **request-scoped, in-memory map** so you can re-insert real values into the *final answer* if needately needed (e.g., echoing back the user's own number) — the map never goes to the LLM.
- **Send only redacted text + non-PII KB context** to the LLM.
- **Provider config:** use enterprise endpoints with **zero data retention / no-training** flags (OpenAI org "do not train"/zero-retention, Azure OpenAI, or Vertex/Claude with data-governance terms). Server-side key in Secrets Manager.
- **Output filter:** scan the LLM response for any leaked PII / unsafe content before returning; re-insert only user-owned tokens.
- **Hard guardrails:** the bot refuses to surface other users' contact info, process payments, or give safety-critical advice; escalates to human support for sensitive cases.
- **Never log raw user input to the LLM provider's debugging**; your own logs store the **redacted** version only.

```python
# Pre-LLM redaction (FastAPI sketch)
PII_PATTERNS = {
  "PHONE": r"(?:\+?8801|01)[3-9]\d{8}",
  "NID":   r"\b\d{10}\b|\b\d{13}\b|\b\d{17}\b",
  "EMAIL": r"[\w.+-]+@[\w-]+\.[\w.-]+",
}
def redact(text):
    mapping, idx = {}, {}
    for label, pat in PII_PATTERNS.items():
        for m in re.finditer(pat, text):
            idx[label] = idx.get(label, 0) + 1
            tok = f"«{label}_{idx[label]}»"
            mapping[tok] = m.group(0)
            text = text.replace(m.group(0), tok)
    text = ner_model.redact(text)   # catch names/addresses regex misses
    return text, mapping            # mapping stays server-side, in-memory
```

### 5.3 English + Bangla / Banglish handling

- **Language/script detection** first (Unicode Bangla vs. Latin-script "Banglish"). Choose a multilingual-capable model (GPT-4o/Claude/Gemini handle Bangla + Banglish reasonably).
- **Normalize Banglish** (romanized Bangla) — optionally transliterate to Bangla script for better retrieval; maintain a **bilingual FAQ/KB** so RAG context exists in both.
- **PII patterns must run on both scripts** — Bangla-script names/addresses won't match Latin regex, so the NER model must be multilingual; keep the regex layer for numerals (digits are the highest-risk leak and are script-stable, though watch for Bangla numerals ০–৯).
- Respond in the **user's language**; keep a tone/safety system prompt that's locale-aware (culturally appropriate, no unsafe advice).
- Test with a Banglish red-team set to ensure redaction + refusals hold across scripts.

---

## 6. Cloud Infrastructure & DevOps (CI/CD) Security

### 6.1 Network & deployment topology (AWS-flavored; GCP equivalents noted)

```
Internet
   │
   ▼
[Route53/Cloud DNS] → [CloudFront/CDN] → [AWS WAF + Shield Adv / Cloud Armor]
   │                                         (DDoS L3-7, SQLi/XSS rules, rate limits,
   │                                          geo rules, bot control)
   ▼
[Public subnet: ALB only]  ── no app/db here
   │  (TLS 1.3 termination + re-encrypt to targets)
   ▼
[Private subnet: app/services]  (ECS/EKS/Cloud Run, no public IP)
   │  egress via NAT GW only; service mesh mTLS
   ▼
[Private "data" subnet: RDS/Postgres, Redis, no internet route]
        │
        ▼
[Isolated subnet/account: pii_vault + payments + kyc]  ← strictest SG/NACL,
        separate KMS keys, separate IAM boundary, VPC endpoints to S3/KMS only
```

- **VPC with tiered private subnets**; databases and the PII/payment tier have **no route to the internet**. Access S3/KMS/Secrets via **VPC endpoints** (no public traffic).
- **WAF (AWS WAF / Cloud Armor):** managed rule sets for SQLi/XSS/OWASP Top 10, rate-based rules (per-IP throttling), bot control, geo rules, and custom rules for OTP/payment endpoints. **Shield Advanced / Cloud Armor** for DDoS.
- **Security groups / firewall = default-deny**, explicit allow only. NACLs as a second layer.
- **Separate AWS accounts / GCP projects** per environment (dev/stage/prod) and ideally an isolated account for the PII/payments tier (blast-radius containment) under AWS Organizations / GCP folders with SCPs.

### 6.2 Secure media storage (KYC docs, photos)

- S3/GCS buckets: **private, block all public access, SSE-KMS, versioning, Object Lock** for KYC.
- **Signed, time-limited URLs** for *every* access (upload via pre-signed PUT, view via short-lived GET, e.g., 60s for KYC, a few minutes for profile photos). No object is ever public.
- Separate buckets/prefixes + separate KMS keys for `kyc-raw`, `profile-photos`, `chat-media`. Profile photos served as **blurred derivatives pre-match** (a transform pipeline generates blurred + clear variants; the clear URL is only signed for matched users).
- Malware scan on upload; image moderation (incl. CSAM hash matching) before any serve.
- CloudTrail/Audit Logs on all object + KMS access.

### 6.3 Secure CI/CD

| Stage | Control |
|---|---|
| **Source** | Branch protection, required reviews, signed commits, CODEOWNERS on payment/PII/verify dirs. |
| **Secrets** | **No secrets in repo.** Scan with gitleaks/trufflehog in CI. Runtime secrets via Secrets Manager + OIDC-federated short-lived cloud creds (no long-lived CI keys). |
| **SAST/DAST** | Static analysis (Semgrep/CodeQL), dependency scanning (Dependabot/Snyk), container image scanning (Trivy), IaC scanning (Checkov/tfsec). Fail build on criticals. |
| **Supply chain** | Pin dependencies, generate SBOM, verify image signatures (cosign/Sigstore), use minimal/distroless base images. |
| **Build** | Reproducible builds; Flutter `--obfuscate`; sign Android (Play App Signing) + iOS in a hardened runner. |
| **Deploy** | IaC (Terraform) with plan review; least-privilege deploy roles; immutable infra; blue/green or canary; auto-rollback on health/error-budget breach. |
| **Runtime** | Container runtime security (read-only FS, non-root, seccomp), secrets injected at runtime, network policies, pod security standards. |
| **Observability/IR** | Centralized logs → SIEM, OpenTelemetry traces, alerting, anomaly detection on PII access + payment + OTP volume. Documented incident-response runbook + breach-notification process. |

### 6.4 Operational security baseline

- **MFA + SSO** for all staff; **no shared accounts**; admin access via short-lived, audited sessions (e.g., AWS SSO/IAM Identity Center, just-in-time elevation).
- **Break-glass** for PII/keys: dual-control, time-boxed, heavily alerted.
- **Backups:** encrypted, tested restores, cross-region; PITR for Postgres; backups inherit the same KMS isolation.
- **DR:** defined RPO/RTO; multi-AZ by default; documented failover.
- **Regular pen-tests** + bug bounty once mature; quarterly access reviews; data-retention & deletion jobs (right-to-erasure).

---

## 7. Phased Rollout (pragmatic for a startup)

| Phase | Focus | Don't over-build |
|---|---|---|
| **MVP (0–3 mo)** | Flutter app, OTP auth, profiles, area-level matching, in-app chat, manual KYC review, SSLCommerz, core+PII split, WAF, KMS, signed URLs. | Skip full microservices; modular monolith with PII/Payment/Verify as isolated modules + separate DB/keys. |
| **Growth (3–9 mo)** | Automated KYC pre-screen, contact masking, AI chatbot with PII redaction, ledger reconciliation, observability/SIEM, RLS everywhere. | Add bKash/Nagad direct, image moderation, ML ranking. |
| **Scale (9+ mo)** | Split hot services, multi-region, advanced fraud/anti-pumping ML, bug bounty, formal compliance audit. | Premature multi-region before product-market fit. |

---

## 8. Security Control Checklist (quick reference)

- [ ] TLS 1.3 everywhere + HSTS + cert pinning (mobile)
- [ ] AES-256 at rest (storage) + field-level envelope encryption for PII
- [ ] `core` ↔ `pii_vault` split; blind indexes; no plaintext PII in core
- [ ] KMS per-domain keys; rotation; break-glass dual-control
- [ ] RBAC/ABAC at gateway + service + Postgres RLS
- [ ] Append-only PII access log + SIEM anomaly alerts
- [ ] OTP: hashed, TTL, attempt cap, rate limits, CAPTCHA, attestation, anti-pumping
- [ ] Refresh-token rotation + reuse detection; instant revocation
- [ ] Coarsened geo only in core; exact coords encrypted in vault
- [ ] Blurred photos + mutual-consent reveal + contact masking
- [ ] Reporting/blocking + auto-suspend + chat & image moderation (incl. CSAM hashing)
- [ ] Payments: server-side keys, webhook signature + independent validation, idempotency, ledger
- [ ] SMS/MFS/AI keys server-side only; never in client binary
- [ ] AI: pre-LLM PII redaction (en/bn/banglish), zero-retention provider, output filter
- [ ] WAF + DDoS + private subnets + VPC endpoints + default-deny SGs
- [ ] Signed time-limited URLs for all media; private buckets; Object Lock for KYC
- [ ] CI/CD: secret scanning, SAST/DAST/SCA, SBOM, image signing, OIDC short-lived creds
- [ ] Backups encrypted + tested; DR with RPO/RTO; data-retention & erasure jobs
- [ ] Staff MFA/SSO, JIT access, quarterly access reviews, IR + breach-notification runbook
