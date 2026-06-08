# TutorMatch BD — Cost Breakdown & Budget Model

> Estimates for building and operating the platform described in `ARCHITECTURE.md`.
> Currency: **USD** with **BDT** equivalents at **৳120 = $1** (2026 approximate).
> These are planning estimates — actual costs vary with team, scale, and vendor negotiation.
> Status: v1.0 | Pair with the architecture & UI preview docs.

---

## 0. TL;DR — Three budget scenarios

| Scenario | One-time build | Monthly run cost (infra + tools, pre-marketing) | Best for |
|---|---|---|---|
| **A. Lean / Bootstrap MVP** | **$8k–18k** (৳10–22 lakh) | **$500–900/mo** (৳60k–1.1 lakh) | Founder-led, small contract team, prove product-market fit |
| **B. Funded Launch** | **$35k–70k** (৳42–84 lakh) | **$2k–4k/mo** (৳2.4–4.8 lakh) | In-house team, polished launch, 10k+ users |
| **C. Scale (post-traction)** | ongoing | **$5k–12k/mo** (৳6–14 lakh) | 50k–200k users, multi-region, ML, full security ops |

> **Variable costs (SMS, MFS fees, KYC, AI) are on top** and scale with usage — see §3. At small scale they're modest ($150–600/mo); they grow with transaction volume but are largely **revenue-coupled** (MFS fees come out of payments you collect).

---

## 1. One-Time Development Cost

### 1.1 Option A — In-house / contract team (Bangladesh rates, MVP ~3–4 months)

Monthly salaries are mid-market Dhaka 2026 estimates; an MVP needs ~3–4 months.

| Role | Monthly (BDT) | Monthly (USD) | MVP months | MVP cost (USD) |
|---|---|---|---|---|
| Flutter developer (mid-senior) | ৳120k–170k | $1,000–1,420 | 3–4 | $3,000–5,700 |
| Backend developer (Node/Go) | ৳110k–160k | $920–1,330 | 3–4 | $2,800–5,300 |
| UI/UX designer | ৳70k–100k | $580–830 | 1.5–2 | $900–1,700 |
| QA / tester (part-time) | ৳50k–70k | $420–580 | 2 | $840–1,160 |
| DevOps / security (contract) | ৳140k–200k | $1,170–1,670 | 1–1.5 | $1,200–2,500 |
| Product/PM (often the founder) | — | — | — | $0 (founder) |
| **Subtotal (in-house MVP)** | | | | **~$8,700–16,400** |

> A scrappy founder-developer can compress this dramatically. A 1–2 person technical founding team building it themselves shifts most of this to opportunity cost / equity rather than cash.

### 1.2 Option B — Local software agency (Bangladesh)

| Build type | Typical agency quote (USD) | Notes |
|---|---|---|
| Basic MVP (auth, profiles, simple matching) | $6,000–12,000 | Cuts corners on security — **not recommended** given KYC/payments |
| Production MVP (per this blueprint: secure, MFS, KYC) | $18,000–40,000 | Realistic for the security bar you've set |
| Full polished v1 (all 6 modules, AI bot, hardened) | $45,000–90,000 | Funded-launch quality |

### 1.3 Option C — Offshore/international agency
$80,000–200,000+ — included only for comparison; **not advised** for a BD-market product where local talent and context are cheaper and better aligned.

### 1.4 One-time setup fees (any option)

| Item | Cost |
|---|---|
| Apple Developer Program | $99/year |
| Google Play Console | $25 one-time |
| Domain name (.com) | $10–15/year |
| Logo / brand kit | $100–500 (or DIY) |
| Company registration / trade license (BD) | ৳10k–50k ($85–420) |
| MFS merchant onboarding (bKash/Nagad/SSLCommerz) | Usually free setup; some require deposit/agreement |
| **Subtotal** | **~$250–1,000 + reg fees** |

---

## 2. Fixed Monthly Cloud Infrastructure (AWS `ap-south-1` / Mumbai)

GCP equivalents are within ±15%. Numbers assume the tiered-VPC, two-vault design from the blueprint.

### 2.1 MVP / early stage (0–5k users)

| Service | Spec | Monthly (USD) |
|---|---|---|
| Compute (ECS Fargate / Cloud Run, 2–3 small services) | ~2 vCPU aggregate | $80–200 |
| RDS PostgreSQL + PostGIS (core) | db.t4g.small, single-AZ + backups | $40–90 |
| RDS PostgreSQL (PII vault, isolated) | db.t4g.micro/small | $30–70 |
| ElastiCache Redis (OTP/session/cache) | cache.t4g.micro | $25–50 |
| S3 (docs, media) + lifecycle | low volume | $5–20 |
| CloudFront CDN | low traffic | $10–40 |
| AWS WAF | managed rules + requests | $25–50 |
| Shield **Standard** (DDoS) | included free | $0 |
| KMS (multiple CMKs) | per-key + requests | $10–25 |
| Secrets Manager | ~10 secrets | $5–10 |
| NAT Gateway (egress) | 1 NAT | $35–55 |
| CloudWatch / logging | basic | $20–50 |
| Data transfer | modest | $20–60 |
| **MVP infra subtotal** | | **~$340–770/mo** |

### 2.2 Growth stage (5k–50k users)

| Bucket | Monthly (USD) |
|---|---|
| Compute (autoscaling, more services) | $400–900 |
| Databases (multi-AZ core + vault + read replica) | $400–800 |
| Redis (larger, HA) | $100–250 |
| S3 + CDN (more media/docs) | $80–250 |
| WAF + observability + SIEM (Sentry/Datadog tier) | $150–400 |
| NAT / networking / transfer | $100–250 |
| **Growth infra subtotal** | **~$1,200–2,850/mo** |

### 2.3 Scale stage (50k–200k+ users)

| Bucket | Monthly (USD) |
|---|---|
| Compute (HA microservices) | $1,500–4,000 |
| Databases (multi-AZ, replicas, larger instances) | $1,500–3,500 |
| Caching / search (OpenSearch) | $400–1,000 |
| Media/CDN | $400–1,200 |
| Security (Shield Advanced optional $3,000, WAF, SIEM) | $500–3,500 |
| Networking / transfer | $300–800 |
| **Scale infra subtotal** | **~$4,600–14,000/mo** |

> **Cost lever:** Shield **Advanced** is $3,000/mo flat — only adopt it once you're a real DDoS target. Shield Standard + WAF rate-limiting covers MVP/growth.

---

## 3. Variable / Per-Transaction Costs (scale with usage)

These are the ones founders underestimate. They grow with users and transactions.

### 3.1 SMS / OTP (⚠️ biggest sleeper cost — SMS pumping risk)

| Item | Rate (BD bulk SMS) |
|---|---|
| Non-masking SMS | ~৳0.25–0.40 each ($0.002–0.003) |
| Masking SMS (branded sender) | ~৳0.40–0.55 each ($0.003–0.005) |

**Example monthly cost by OTP volume (masking):**

| OTPs / month | Approx cost |
|---|---|
| 10,000 | ৳4,000–5,500 ($35–46) |
| 50,000 | ৳20,000–27,500 ($170–230) |
| 200,000 | ৳80,000–110,000 ($670–920) |

> Each login/signup may send 1–2 SMS. **Without the anti-pumping controls from §3.1 of the blueprint (CAPTCHA, rate limits, attestation), a fraud attack can spike this into lakhs of taka overnight.** Budget the controls — they pay for themselves.

### 3.2 MFS payment gateway fees (deducted from revenue, not a cash burn)

| Provider | Typical merchant fee |
|---|---|
| bKash (merchant/payment gateway) | ~1.5%–2.0% per transaction |
| Nagad | ~1.4%–1.8% per transaction |
| SSLCommerz (aggregator: cards + MFS) | ~2.5%–3.0% + VAT |

> On ৳1,000,000/month in payments at ~2%, that's **~৳20,000 ($170)** in fees — but it comes *out of money you've collected*, so it's a margin line, not upfront burn. Negotiate rates as volume grows.

### 3.3 KYC / NID verification

| Method | Cost |
|---|---|
| Porichoy / NID gateway (automated EC verification) | ~৳2–10 per check ($0.02–0.08) — partner/agreement needed |
| Manual reviewer (semi-automated workflow) | labor: ~৳15k–35k/mo for a part-time reviewer at low volume |
| Third-party KYC SaaS (OCR + face match) | $0.10–0.50 per verification |

At 1,000 tutor verifications/month with automated gateway: **~$20–80**. Manual review dominates at low volume; automate as you scale.

### 3.4 AI chatbot (LLM API, with PII-redaction layer)

| Volume | Monthly cost |
|---|---|
| Light (FAQ, ~10k short messages) | $30–120 |
| Moderate (~50k messages, RAG context) | $150–500 |
| Heavy (~200k messages) | $600–2,000 |

> Use a smaller/cheaper model (e.g., Haiku-class / GPT-mini-class) for routing + a capable model only when needed. Cache common answers. The PII-redaction layer adds negligible compute cost.

### 3.5 Other variable

| Item | Cost |
|---|---|
| Push notifications (FCM/APNs) | Free |
| Email (transactional, SES/SendGrid) | $0–20/mo at MVP |
| Maps (if used — minimize via coarsened geo / OSM) | $0–200/mo |

---

## 4. Ongoing Operational Costs (post-launch, monthly)

| Item | Lean (USD/mo) | Funded (USD/mo) |
|---|---|---|
| Engineering / maintenance team | founder-led / $1,000–2,500 (1 dev) | $4,000–9,000 (small team) |
| Customer support + KYC reviewers | $200–500 (1 part-timer) | $800–2,000 |
| Cloud infra (from §2) | $340–770 | $1,200–4,000 |
| Variable (SMS/AI/KYC, from §3) | $150–400 | $400–1,500 |
| Tools (GitHub, Google Workspace, Sentry, monitoring) | $30–100 | $150–400 |
| **Operational subtotal (ex-marketing)** | **~$900–2,300/mo** | **~$6,500–17,000/mo** |
| Marketing / user acquisition (optional, highly variable) | $200–1,000 | $2,000–20,000+ |

---

## 5. Security & Compliance (periodic)

| Item | Cost | Cadence |
|---|---|---|
| SSL/TLS certificates | Free (Let's Encrypt / ACM) | — |
| Penetration test (external) | $2,000–8,000 | Annual (once handling real PII/money) |
| Security audit / compliance review | $1,500–6,000 | Annual |
| Bug bounty program | variable / pay-per-finding | Once mature |
| Cyber-liability insurance | $500–3,000/yr | Annual (recommended given KYC data) |

> **Don't skip the pen-test before scaling.** You're holding NID documents and processing payments — a breach is existential, both legally (Digital Security Act / data-protection rules) and reputationally.

---

## 6. Worked Example — Year 1 Budget (Lean Bootstrap Path)

Assumes founder + 1 contract dev, MVP in 4 months, then ~10k users by year-end.

| Line item | Year 1 total (USD) | BDT (~৳120/$) |
|---|---|---|
| One-time development (lean in-house) | $10,000 | ৳12,00,000 |
| Setup fees (stores, domain, registration) | $700 | ৳84,000 |
| Cloud infra (avg $600/mo × 12) | $7,200 | ৳8,64,000 |
| Variable: SMS + AI + KYC (avg $300/mo × 8 post-launch) | $2,400 | ৳2,88,000 |
| Maintenance dev (post-launch, $1,500/mo × 6) | $9,000 | ৳10,80,000 |
| Support / KYC reviewer ($300/mo × 8) | $2,400 | ৳2,88,000 |
| Tools | $700 | ৳84,000 |
| Pen-test (one, pre-scale) | $3,000 | ৳3,60,000 |
| Contingency (~15%) | $5,300 | ৳6,36,000 |
| **Year 1 total (ex-marketing)** | **~$40,700** | **~৳48.8 lakh** |

> MFS fees are excluded here — they're netted against revenue. Add a marketing budget per your go-to-market ambition.

---

## 7. Cost-Optimization Levers (how to keep it lean early)

1. **Start as a modular monolith** (one deployable, isolated modules) — cuts compute/DB count roughly in half vs full microservices. Split later.
2. **Single-AZ + automated backups at MVP**; move to multi-AZ only when uptime SLAs matter.
3. **Serverless where bursty** (Cloud Run / Fargate scale-to-low) so you pay for actual usage early on.
4. **Aggressively defend SMS** — anti-pumping controls are the highest-ROI spend; they prevent runaway taka loss.
5. **Tiered AI models** — cheap model for routing/FAQ, premium only for complex queries; cache answers.
6. **Skip Shield Advanced** ($3k/mo) until you're genuinely targeted; WAF + rate limiting suffices.
7. **Automate KYC early** to avoid linear reviewer-labor growth.
8. **Reserved instances / savings plans** once usage is predictable (20–40% off compute/DB).
9. **Cloud credits** — AWS Activate / Google for Startups can grant **$1k–100k+** in credits; apply before launch.

---

## 8. Summary Table

| Phase | One-time | Monthly fixed | Monthly variable | When |
|---|---|---|---|---|
| **MVP (Lean)** | $8k–18k | $340–770 | $150–400 | Months 0–6 |
| **Growth** | (iterate) | $1.2k–2.9k | $400–1.5k | 5k–50k users |
| **Scale** | (iterate) | $4.6k–14k | $1.5k–5k+ | 50k–200k+ users |

> **Bottom line:** You can credibly launch a secure MVP for **~$10–18k upfront and run it for under $1k/month** until you find traction. The architecture's security investments (vault split, anti-pumping, signed webhooks) are *cost-control measures as much as safety measures* — they prevent the two failure modes that bankrupt early platforms: SMS-fraud bills and breach liability.
