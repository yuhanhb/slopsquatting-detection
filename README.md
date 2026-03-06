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

This framework takes a defence-in-depth approach across two independent pipeline stages, addressing a gap that traditional SCA tools currently miss: they check packages against known CVE databases, but have no mechanism to catch newly registered hallucinated packages that haven't been flagged yet.

### Layer 1 — CI/CD Gate (Pre-Install)

Integrated directly into GitHub Actions, this layer intercepts every push and pull request before code can be merged. It reads the project's `requirements.txt`, queries the PyPI registry for each package, and runs a trust scoring algorithm across five behavioural signals: package existence, registration age, download volume, documentation presence, and source code transparency. Any package scoring above the suspicion threshold fails the build automatically. The developer is forced to investigate before the pipeline continues.

This is the prevention layer. It operates on the assumption that most slopsquatting attempts will be caught here — either because the hallucinated package doesn't exist yet, or because a recently registered attacker package exhibits low-trust signals that legitimate packages don't.

### Layer 2 — Runtime Monitoring (Post-Install)

No prevention layer is perfect. Packages can enter a codebase through local installs, dependency chains, or CI/CD misconfigurations that bypass Layer 1. The Sentinel KQL rule addresses this by continuously correlating two data sources from Microsoft Defender for Endpoint: package install events and outbound network connections. If a Python or Node process makes an external network call within ten minutes of a package being installed on the same machine, an alert fires and an incident is created for SOC review.

Legitimate utility packages have no reason to initiate network connections immediately after installation. This behavioural baseline is what makes the detection reliable — it doesn't depend on signatures or known-bad lists, only on anomalous behaviour relative to expected package functionality.

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

  Checking: azure-keyvault-session-helper
    ❌ CRITICAL: Package does not exist on PyPI
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

## Usage

### Requirements

- Python 3.9+
- A `requirements.txt` file in the root of your project

### Running Locally

Clone the repository and install dependencies:

```bash
git clone https://github.com/yuhanhb/slopsquatting-detection.git
cd slopsquatting-detection
pip install requests python-dateutil
```

Add the packages you want to check to `requirements.txt`, then run:

```bash
python slopsquatting_check.py
```

### Adding to Your Own Project

To integrate this into any existing Python project:

1. Copy `slopsquatting_check.py` into your project root
2. Copy `.github/workflows/slopsquatting-check.yml` into your project's `.github/workflows/` folder
3. Push to GitHub — the check will run automatically on every push and pull request

No configuration needed. The workflow will read your existing `requirements.txt` and score each package automatically.

### Real-World Test

This framework has been tested against [Vulpy](https://github.com/yuhanhb/vulpy), a deliberately vulnerable Python web application used for security training. All 9 of Vulpy's real production dependencies passed cleanly, confirming the framework does not generate false positives against legitimate packages.

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
Cybersecurity Analyst  
[LinkedIn](https://www.linkedin.com/in/yuhanhb)
