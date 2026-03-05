# Slopsquatting Detection Framework

A two-layer security framework for detecting and preventing slopsquatting attacks in software development pipelines.

## What is Slopsquatting?

AI coding assistants like GitHub Copilot and ChatGPT sometimes hallucinate package names — suggesting dependencies that don't actually exist. Attackers exploit this by registering those hallucinated names on PyPI or npm with malicious payloads, waiting for developers to blindly install them.

This attack vector was coined **"slopsquatting"** by Python Software Foundation researcher Seth Larson, and represents an emerging supply chain threat as AI-generated code becomes the norm.

Research from USENIX Security 2025 found that across 576,000 code samples, LLMs hallucinate package names in predictable patterns:
- **51%** are pure fabrications
- **38%** are conflations of two real packages (e.g. `express-mongoose`)
- **13%** are typo variants of real packages

## Detection Architecture

This framework takes a **defence-in-depth** approach with two independent layers:

```
Developer writes AI-assisted code
            │
            ▼
┌─────────────────────────────┐
│  LAYER 1: GitHub Actions    │  ← Pre-install gate
│  Package trust scorer       │    Blocks suspicious packages
│  Runs on every code push    │    before they enter the repo
└─────────────────────────────┘
            │
            │ (if something slips through)
            ▼
┌─────────────────────────────┐
│  LAYER 2: Microsoft         │  ← Post-install CCTV
│  Sentinel KQL Rule          │    Detects malicious runtime
│  Continuous monitoring      │    behaviour after install
└─────────────────────────────┘
```

> **Layer 1 prevents. Layer 2 detects. Because prevention alone is never sufficient.**

---

## Layer 1 — GitHub Actions (Pre-Install)

**File:** `.github/workflows/slopsquatting-check.yml`  
**Script:** `slopsquatting_check.py`

Runs automatically on every push or pull request. Reads `requirements.txt` and scores each package against the PyPI registry using five suspicion signals:

| Signal | Score |
|--------|-------|
| Package does not exist on PyPI | +100 (instant block) |
| Registered within the last 30 days | +40 |
| Fewer than 500 downloads in 30 days | +25 |
| No description or summary | +15 |
| No author or maintainer listed | +10 |
| No source code link | +10 |

**Threshold:** Any package scoring 60 or above fails the build. The code cannot be merged until a developer manually reviews and clears the flagged package.

### Example Output

```
============================================================
  SLOPSQUATTING DETECTION — Package Trust Check
============================================================

🔍 Checking 2 package(s) from requirements.txt...

  Checking: requests
    ✔  No suspicious signals detected
    ✅ Score: 0/100 — OK

  Checking: fastapi-aws-auth-helper
    🆕 NEW: Package only 4 days old (registered 2026-03-01)
    📉 LOW DOWNLOADS: Only 12 downloads in the last 30 days
    📭 NO DESCRIPTION: Package has no meaningful summary
    👤 NO AUTHOR: No author or maintainer listed
    🔗 NO SOURCE LINK: No repository or source code link found
    🚨 SUSPICION SCORE: 100/100 — BUILD BLOCKED

============================================================

🚨 BUILD FAILED — 1 suspicious package(s) detected
  → A developer must review these before merging.
  → Were these suggested by an AI coding assistant?
```

---

## Layer 2 — Microsoft Sentinel KQL (Post-Install)

**File:** `slopsquatting_sentinel.kql`

A Sentinel analytics rule that correlates two data sources from Microsoft Defender for Endpoint:

1. **Package install events** — pip or npm install commands
2. **Outbound network connections** — Python or Node processes calling external IPs

If a network connection is made within **10 minutes** of a package install on the same machine, an alert fires. Legitimate utility packages have no reason to phone home immediately after installation.

**To deploy in Sentinel:**
1. Go to Microsoft Sentinel → Analytics → Create Rule
2. Paste the KQL query
3. Schedule: run every 1 hour, look back 24 hours
4. Threshold: trigger if results > 0
5. Severity: High

---

## Why This Matters for DevSecOps

Traditional SCA tools (Dependabot, WhiteSource) check packages against known CVE databases — they have no mechanism to detect newly registered hallucinated packages that haven't been flagged yet. This framework addresses that blind spot by focusing on **behavioural signals** rather than known signatures.

As vibe coding becomes standard and AI agents autonomously install dependencies, the window for human review keeps shrinking. Automated detection at both the pipeline and runtime layer is no longer optional.

---

## Research References

- Larson, S. (2024). *Slopsquatting* — Python Software Foundation
- USENIX Security 2025 — *We Have a Package for You!* (576,000 sample LLM hallucination study)
- Trend Micro (2025). *Slopsquatting: When AI Agents Hallucinate Malicious Packages*
- Snyk (2025). *Slopsquatting Mitigation Strategies*

---

## Author

**Yuhan Perera**  
Cybersecurity Student & Analyst — Deakin University  
[LinkedIn](https://www.linkedin.com/in/yuhanhb)
