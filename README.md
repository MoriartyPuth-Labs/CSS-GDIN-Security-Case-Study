<div align="center">

# 🛡️ API Broken Access Control & PII Exposure: Government Complaint Support System (CSS)
### Ministry of Interior — Cambodia

[![Target](https://img.shields.io/badge/Target-CSS_GDIN_Portal-blue?style=for-the-badge&logo=globe)](https://css-gdin.interior.gov.kh/home)
[![Vuln](https://img.shields.io/badge/Vulnerability-IDOR_%2F_BAC-red?style=for-the-badge)]()
[![CVSS](https://img.shields.io/badge/CVSS_v3.1-7.5_High-orange?style=for-the-badge)]()
[![Status](https://img.shields.io/badge/Status-Patched_%26_Verified-success?style=for-the-badge&logo=checkmarx)]()
[![Disclosure](https://img.shields.io/badge/Disclosure-Responsible-blueviolet?style=for-the-badge)]()

</div>

---

## 📋 Executive Summary

This case study documents the discovery, responsible disclosure, and successful remediation of a
**Broken Access Control** vulnerability in a live government API handling citizen complaint records.
The flaw allowed any unauthenticated actor to issue bulk `HTTP GET` requests against the
`/api/v1/complaints` collection endpoint and receive structured PII for over **1,000+ citizen entries**
including full names, phone numbers, and sensitive allegation descriptions.

The root cause was a **Default Allow** posture inherited from the public submission flow —
the API correctly permitted anonymous `POST` requests for citizen submissions but failed to restrict
`GET` access at the method level, exposing the entire complaints collection to unauthenticated reads.

> Responsibly disclosed to MOI IT stakeholders. Patch verified **February 19, 2026**.

---

## 🎯 Target Profile

| Field | Detail |
|---|---|
| System | Government Complaint Support System (CSS) |
| API Stack | Spring Boot, Spring Security |
| Endpoint | `/api/v1/complaints` |
| Method | HTTP GET (unauthenticated) |
| Data Exposed | Citizen PII — names, phone numbers, complaint descriptions |
| Records Exposed | 1,000+ entries |

---

## 🗺️ Vulnerability Flow
```
┌──────────────────────────────────────────────────────────────┐
│               CSS API ACCESS CONTROL FLAW                    │
│                                                              │
│  [Citizen]  ──POST──▶  /api/v1/complaints  ✅ permitAll()   │
│                                │                             │
│  [Attacker] ──GET───▶  /api/v1/complaints  ❌ no restriction │
│                                │                             │
│                                ▼                             │
│              Full JSON dump of 1,000+ PII records            │
│              { name, phone, description, status }            │
└──────────────────────────────────────────────────────────────┘
```
---

## 🔬 Methodology

### Phase 1 — Reconnaissance

Deployed [Bubble-Scanner](https://github.com/MoriartyPuth/bubble-scanner) to map the Spring Boot
API surface. The tool's **Bubble-Dive Engine** recursively fuzzes endpoints and triggers
sub-scanners on every `200 OK` response, including:

- Source code scavenging (regex patterns for leaked API keys, tokens, hardcoded credentials)
- RCE vector identification (file upload forms, unvalidated input fields)
- Basic error-based SQLi probing on discovered parameters

```bash
bubble_dive() {
    head -n 1000 "$WORDLIST" | while read -r path; do
        url="${base_url%/}/${path}"
        res=$(curl -s -o /dev/null -w "%{http_code}" "$url")
        if [ "$res" == "200" ]; then
            scan_source_code "$url"   # hunt for leaked secrets
            check_upload_vuln "$url"  # identify RCE vectors
        fi
    done
}
```

### Phase 2 — Traffic Analysis

Observed a discrepancy between the authenticated admin dashboard (correctly access-controlled)
and the raw API routes beneath it. The frontend enforced auth; the backend did not.

### Phase 3 — Vulnerability Confirmation

Unauthenticated `GET` to the complaints collection returned a full database dump instead of
the expected `403 Forbidden`:

```bash
# Unauthenticated collection probe (pre-patch)
curl -i -k 'https://[TARGET_REDACTED]/api/v1/complaints'
```

**Response (anonymized sample):**
```json
[
  {
    "id": 7,
    "plaintiffName": "ហុង [REDACTED]",
    "phoneNumber": "086******",
    "description": "Sensitive allegation description...",
    "status": "PENDING"
  }
]
```

### Phase 4 — Disclosure & Remediation

Delivered a Root Cause Analysis to MOI IT stakeholders identifying the missing
method-level security configuration. The recommended fix:

```java
// Spring Security fix (conceptual — applied by MOI IT team)
.requestMatchers(HttpMethod.POST, "/api/v1/complaints").permitAll()  // public submissions
.requestMatchers(HttpMethod.GET,  "/api/v1/complaints").hasRole("ADMIN")  // restrict reads
```

### Phase 5 — Patch Verification

Regression tested all previously successful extraction vectors on **February 19, 2026**.
All endpoints returned `HTTP 403 Forbidden` for unauthenticated `GET` requests.
Public `POST` submission flow remained functional.

---

## 🔍 Vulnerability Analysis

### CWE Classification

| CWE | Description |
|---|---|
| CWE-284 | Improper Access Control |
| CWE-200 | Exposure of Sensitive Information to an Unauthorized Actor |
| CWE-285 | Improper Authorization (method-level GET/POST distinction missing) |

### CVSS v3.1 Scoring

**Score: 7.5 (High)**
`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N`

| Metric | Value | Rationale |
|---|---|---|
| Attack Vector | Network | Exploitable remotely over HTTP |
| Attack Complexity | Low | Single unauthenticated GET request |
| Privileges Required | None | No auth token needed |
| User Interaction | None | Fully automated extraction possible |
| Confidentiality | High | Full PII collection exposed |
| Integrity | None | Read-only — no write access demonstrated |
| Availability | None | No service disruption |

---

## 🎖️ Outcome

| Metric | Result |
|---|---|
| Discovery date | February 2026 |
| Report submitted | February 18, 2026 |
| Patch verified | February 19, 2026 |
| Time to remediation | < 24 hours |
| Disclosure type | Responsible — coordinated with MOI IT |
| PII records secured | 1,000+ entries |

---

## 🛠️ Tools Used

| Tool | Purpose |
|---|---|
| [Bubble-Scanner](https://github.com/MoriartyPuth/bubble-scanner) | Endpoint discovery, source recon, SQLi probing |
| curl | Manual endpoint probing & verification |
| Spring Security docs | Remediation guidance |

---

> **Disclaimer:** This research was conducted for educational and security-hardening purposes.
> No data was retained or exfiltrated. All findings were reported directly to the system owner
> prior to publication.

---

## 👤 Author

<div align="center">

**Eav Puthcambo**
<br/>
AUPP Cybersecurity Programme
<br/>
American University of Phnom Penh

[![GitHub](https://img.shields.io/badge/GitHub-MoriartyPuth--Labs-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/MoriartyPuth-Labs)

</div>
