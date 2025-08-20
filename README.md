# TPRM Agent

A minimal backend to power a ChatGPT Custom GPT for **third-party risk assessments** with:
- Web discovery for trust/privacy/security docs
- Evidence capture (download + hash + local store)
- Parsers (LLM-assisted): **SOC 2**, **ISO 27001**, **Privacy policy**
- Passive posture checks: SPF/DMARC, TLS/HSTS, security headers, security.txt
- Threat signals: **NVD CVEs** + **CISA KEV**
- Scoring + **Evidence Pack** (ZIP)

> This project **only performs passive checks**. No vuln scans, no auth bypass, no intrusive actions.

---

## Quickstart

### Option A: pip + venv (simplest)

    python -m venv .venv && source .venv/bin/activate
    pip install -r requirements.txt
    cp .env.example .env   # set OPENAI_API_KEY
    uvicorn backend.main:app --reload --port 8080

### Option B: Conda (recommended for data science workflows)

    conda env create -f environment.yml
    conda activate tprm
    cp .env.example .env   # set OPENAI_API_KEY
    uvicorn backend.main:app --reload --port 8080

Open Swagger: http://localhost:8080/docs

---

### Example call

    curl -X POST http://localhost:8080/assess_vendor \
      -H 'content-type: application/json' \
      -d '{"id":"vndr_acme","name":"Acme Corp","domain":"acme.com","criticality":"high"}'

---

### Connect to ChatGPT (Actions)

1. Expose locally with **Cloudflare Tunnel** or **ngrok**:

       cloudflared tunnel --url http://localhost:8080

2. In your Custom GPT, add an Action with your public base URL and import **`infra/openapi.yaml`**.

---

## Project layout

    tprm-agent/
      backend/
        __init__.py
        main.py            # FastAPI endpoints (/healthz, /assess_vendor)
        config.py          # env vars (OPENAI_API_KEY, storage, discovery hints)
        models.py          # Pydantic models for Vendor, Evidence, Findings, etc.
        storage.py         # store_bytes() + sha256; local evidence store
        fetcher.py         # discover trust/privacy/security links, fetch URLs
        parsers/
          soc2.py          # SOC 2 parser (LLM assisted)
          iso27001.py      # ISO 27001 certificate parser (regex + LLM fallback)
          policy.py        # Privacy/policy extractor (LLM)
        checks/
          dns_email.py     # SPF/DMARC presence; MX check
          tls_headers.py   # TLS min version; HSTS/CSP/headers
          securitytxt.py   # /.well-known/security.txt presence
        threats/
          nvd.py           # NVD CVE keyword search
          cisa_kev.py      # CISA Known Exploited Vulns feed
          news.py          # placeholder for NewsAPI/GDELT (optional)
        scoring.py         # simple transparent rubric
        pack.py            # builds Evidence Pack ZIP (docs + JSON + README)
      infra/
        openapi.yaml       # Import into ChatGPT Actions
        monitor.yml        # GitHub Actions example for weekly monitoring
      tests/
        fixtures/          # put sample SOC2/ISO PDFs here (not included)
      requirements.txt
      environment.yml      # Conda environment setup
      .env.example
      .gitignore
      README.md

---

## Endpoint details

### `POST /assess_vendor`

**Input JSON**

    {
      "id": "vndr_acme",
      "name": "Acme Corp",
      "domain": "acme.com",
      "criticality": "high",
      "data_types": ["PII"],
      "region": "US"
    }

**What happens**

1. **discover_links** → gather likely trust/security/privacy URLs.  
2. **fetch_url_to_evidence** → download HTML/PDF → store with SHA-256.  
3. **parse** → SOC2, ISO 27001, Privacy policy (as available).  
4. **posture** → SPF/DMARC, TLS/HSTS/CSP, security.txt.  
5. **threats** → NVD CVEs; mark **CISA KEV** hits.  
6. **score** → transparent rubric to Low/Medium/High.  
7. **pack** → ZIP with raw docs + structured JSON findings.  

**Output JSON (abridged)**

    {
      "links_checked": ["https://acme.com/security", "..."],
      "soc2": { ... },
      "iso": { ... },
      "policy": { ... },
      "posture": { ... },
      "threats": { ... },
      "decision": { "overall": "Medium", ... },
      "evidence_pack": "./evidence/vndr_acme/evidence_pack.zip"
    }

---

## How each file works (brief)

- **backend/main.py** — Orchestrates the full assessment flow for a vendor.  
- **backend/config.py** — Centralizes environment config; modify discovery hints & storage here.  
- **backend/models.py** — Strongly-typed data contracts for everything the agent emits or consumes.  
- **backend/storage.py** — Saves evidence artifacts; records SHA-256 for immutability.  
- **backend/fetcher.py** — Finds and downloads trust/privacy/security pages and PDFs.  
- **backend/parsers/soc2.py** — Uses OpenAI to extract SOC 2 opinion, period, exceptions, CUECs, etc.  
- **backend/parsers/iso27001.py** — Extracts ISO 27001 certificate info (regex) with LLM fallback.  
- **backend/parsers/policy.py** — Pulls high-value policy fields (data categories, retention, transfers).  
- **backend/checks/dns_email.py** — Verifies SPF/DMARC presence and MX (email hygiene).  
- **backend/checks/tls_headers.py** — Reads TLS version + key security headers from HTTPS response.  
- **backend/checks/securitytxt.py** — Looks for a security.txt disclosure with contact channels.  
- **backend/threats/nvd.py** — Lightweight CVE keyword search from NVD.  
- **backend/threats/cisa_kev.py** — Marks any matched CVEs as known exploited.  
- **backend/threats/news.py** — Placeholder to wire a news feed later.  
- **backend/scoring.py** — Simple rubric that turns signals → overall risk + mitigations.  
- **backend/pack.py** — Creates a ZIP (“Evidence Pack”) containing all raw docs and structured outputs.  
- **infra/openapi.yaml** — Import this into a Custom GPT Action for one-click wiring.  
- **infra/monitor.yml** — Example GitHub Actions job to re-run assessments on a schedule.  

---

## Security & legal guardrails

- **Passive only.** No scanning, no auth bypass, no login scraping.  
- Respect **robots.txt** & site TOS; throttle requests; log URLs & timestamps.  
- **Evidence immutability.** Keep hashes and original files; cite sources in briefs.  
- If a document is gated behind a portal, the Action should ask the user to upload it manually.  

---

## Roadmap ideas

- Subprocessor list diffing & alerts  
- Better CVE/product mapping via CPEs  
- News/OSINT integration (NewsAPI/GDELT)  
- Evidence store on S3/R2; signed download links  
- GRC sync (OneTrust/Archer/ServiceNow) webhooks  
- Front-end dashboard + reviewer queue  

---

## Troubleshooting

- **No SOC 2 found** → Upload the PDF manually; ensure it’s not redacted/scanned or add OCR.  
- **Empty ISO fields** → Some certs are images; add OCR or rely on LLM fallback.  
- **TLS shows Unknown** → Domain might block raw socket; header checks will still work.  
- **NVD rate limits** → Request an NVD API key and set `NVD_API_KEY`.  