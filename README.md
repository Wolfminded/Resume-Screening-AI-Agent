# 🤖 Resume Screening AI Agent — n8n Workflow

<p align="center">
  <img src="https://img.shields.io/badge/Built%20With-n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white"/>
  <img src="https://img.shields.io/badge/LLM-Gemini%202.5%20Pro-4285F4?style=for-the-badge&logo=google&logoColor=white"/>
  <img src="https://img.shields.io/badge/Storage-Google%20Drive-34A853?style=for-the-badge&logo=googledrive&logoColor=white"/>
  <img src="https://img.shields.io/badge/Database-Notion-000000?style=for-the-badge&logo=notion&logoColor=white"/>
  <img src="https://img.shields.io/badge/Email-Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white"/>
  <img src="https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Status-Active-brightgreen?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Agents-4%20LLM%20%2B%202%20Code-blueviolet?style=for-the-badge"/>
</p>

<p align="center">
  An end-to-end, multi-agent AI pipeline built on <strong>n8n</strong> that fully automates the resume screening process — from PDF ingestion to structured hiring decisions — powered by <strong>Google Gemini 2.5 Pro</strong>.
</p>

---

## 🗺️ Workflow Diagram

![Resume Screening AI Agent Workflow](./workflow_diagram.svg)

---

## 🧠 Architecture Overview

The pipeline follows a strict multi-agent design pattern where each agent has a **single, well-defined responsibility**. No agent scores and parses at the same time — separation of concerns is enforced at every step.

```
Form Submission
      │
      ▼
Google Drive (Upload + Download Resume)
      │
      ▼
PDF Text Extraction (Resume + Job Description)
      │
      ▼
🧩 Parser Agent         → Extracts structured candidate & JD data (Gemini 2.5 Pro)
      │
      ▼
🧩 Matching Agent       → Compares resume skills vs. JD requirements (Gemini 2.5 Pro)
      │
      ▼
🧩 Scoring Engine       → Computes composite score (JS: 65% skill match + 35% experience)
      │
      ▼
🧩 Decision Agent       → Generates strengths, weaknesses, risk/reward, exec summary (Gemini 2.5 Pro)
      │
      ▼
🧩 Confidence Validator → Runs 7 structural checks; routes to retry if confidence < 0.6
      │
      ├── [PASS] → Merge Final Output → Send Candidate Email (Gmail) → Save to Notion
      └── [FAIL] → Low Confidence Log & Retry Notice
```

---

## ⚙️ Agent Breakdown

### 🧩 Parser Agent
Extracts raw structured data from resume and job description text. Returns a strict JSON object with two top-level keys: `candidate` and `job`. Never infers or assumes — extraction only.

**Candidate fields extracted:**
- `name`, `email`, `total_experience_years`, `current_role`, `education_level`
- `technical_skills`, `soft_skills`, `job_titles_held`

**JD fields extracted:**
- `required_skills`, `preferred_skills`, `required_experience_years`, `key_responsibilities`

---

### 🧩 Matching Agent
Performs strict case-insensitive skill comparison between `candidate.technical_skills` and `job.required_skills`.

**Output:**
| Field | Description |
|---|---|
| `matched_skills` | Required skills found in resume |
| `missing_skills` | Required skills absent from resume |
| `bonus_skills` | Candidate skills beyond JD requirements |
| `match_percentage` | `(matched / total_required) * 100` |

---

### 🧩 Scoring Engine (JavaScript)
Deterministic scoring node — no LLM involved. Produces a `composite_score` using a weighted formula:

```
composite_score = (skill_match_pct × 0.65) + (experience_score × 0.35)
```

**Tier mapping:**

| Score | Tier | Next Step |
|---|---|---|
| ≥ 80 | Strong Hire | Advance to Technical Round |
| ≥ 65 | Hire | Advance to HR Round |
| ≥ 45 | Maybe | Request Portfolio / Work Samples |
| < 45 | No Hire | Reject |

---

### 🧩 Decision Agent
Acts as a **senior technical recruiter**. Given full resume text, JD, skill match, and scoring result, it produces:

- `candidate_strengths` (2–4 evidence-backed strengths)
- `candidate_weaknesses` (2–3 gaps with severity: Minor / Moderate / Critical)
- `risk_factor` (Low / Medium / High + worst-case explanation)
- `reward_factor` (Low / Medium / High + fit horizon + best-case explanation)
- `justification_rating` (0–10 score + confidence level + red flags + supporting evidence)
- `overfit_rating` (is_overfit, severity, retention_risk)
- `executive_summary` (2–3 sentence hiring rationale)

---

### 🧩 Confidence Validator (JavaScript)
Runs **7 structural integrity checks** on the Decision Agent output before allowing it to proceed:

| # | Check | Confidence Penalty |
|---|---|---|
| 1 | `justification_rating.score` is a valid number | −0.30 |
| 2 | `candidate_strengths` non-empty with title + detail | −0.20 |
| 3 | `candidate_weaknesses` non-empty with valid severity | −0.15 |
| 4 | `executive_summary` ≥ 20 characters | −0.20 |
| 5 | `risk_factor.score` and `reward_factor.score` are valid enums | −0.10 each |
| 6 | `matched_skills` and `missing_skills` are non-empty arrays | −0.25 |
| 7 | `composite_score` is a real number | −0.15 |

Confidence starts at `1.0` and degrades per failed check. If `confidence < 0.6`, the run is flagged to a **Low Confidence log node** instead of proceeding downstream.

---

## 🔗 Integrations

| Service | Purpose |
|---|---|
| **n8n Form Trigger** | Candidate-facing application form |
| **Google Drive** | Resume storage and retrieval |
| **Google Gemini 2.5 Pro** | LLM backbone for all three AI agents |
| **Gmail** | Automated confirmation email to candidate |
| **Notion** | Applicant tracking database (JOB APPLICANT TRACKER) |

---

## 📊 Notion Database Fields Populated

| Field | Type | Source |
|---|---|---|
| Applicant Name | Title | Form |
| First / Last Name | Rich Text | Form (split) |
| Email | Email | Form |
| Phone Number | Phone | Form |
| Application Date | Date | `$now` |
| Overall Fit | Number | Scoring Engine |
| Tier | Select | Scoring Engine |
| Skill Match % | Number | Matching Agent |
| Matched Skills | Rich Text | Matching Agent |
| Confidence Score | Number | Confidence Validator |
| Next Step | Select | Scoring Engine |
| Resume URL | URL | Google Drive |
| Risk Factor | Select | Decision Agent |
| Reward Factor | Rich Text | Decision Agent |
| Executive Summary | Rich Text | Decision Agent |
| Strengths | Rich Text | Decision Agent |
| Weaknesses | Rich Text | Decision Agent |

---

## 🚀 Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- Google Cloud project with Drive + Gmail APIs enabled
- Google Gemini API key
- Notion integration token
- A Notion database matching the schema above

### Steps

1. **Import** the `My_workflow_3.json` file into your n8n instance via **Workflows → Import from file**.

2. **Configure credentials** in n8n under Settings → Credentials:
   - `Google Drive OAuth2`
   - `Google Gemini (PaLM) API`
   - `Gmail OAuth2`
   - `Notion API`

3. **Update the Google Drive folder ID** in the `Upload Resume` node to point to your own Drive folder.

4. **Update the Notion database ID** in the `Save to Notion` node.

5. **(Optional)** Replace the static JD file ID in the `Download JD` node with a dynamic lookup or form-submitted input to support multiple job roles.

6. **Activate** the workflow and share the form URL with candidates.

---

## 🛠 Tech Stack

| Layer | Technology |
|---|---|
| Orchestration | n8n (v1 execution engine) |
| LLM | Google Gemini 2.5 Pro |
| Scoring Logic | JavaScript (n8n Code node) |
| File Storage | Google Drive |
| ATS / CRM | Notion |
| Notifications | Gmail |
| PDF Parsing | n8n Extract From File node |

---

## 📌 Design Decisions

**Structured output parsers on all LLM agents** — Schema-enforced JSON is mandatory. No regex parsing of free-form text is used anywhere in the pipeline.

**Deterministic scoring outside LLM inference** — The composite score is computed in pure JavaScript, making it reproducible, auditable, and free from hallucination risk.

**Confidence gating before write operations** — The Confidence Validator acts as a circuit breaker. Low-quality LLM outputs cannot silently pollute the Notion database.

**Agent isolation** — Each agent can be independently swapped, upgraded, or unit-tested without touching the rest of the pipeline.

**Single responsibility per node** — The Parser never scores. The Scoring Engine never reasons. The Decision Agent never extracts. This is intentional and strictly enforced in every system prompt.

---

## 📂 Repository Structure

```
.
├── My_workflow_3.json      # n8n workflow export (import this)
├── workflow_diagram.svg    # Visual pipeline architecture
├── README.md               # This file
└── CONTRIBUTING.md         # Contribution guidelines
```
DEMO
https://youtu.be/BIj7K1StJOk

SCREENSHOT-
<img width="1366" height="768" alt="Screenshot (61)" src="https://github.com/user-attachments/assets/fc604459-3014-43b4-9b74-9ae25b891824" />

---

## 🤝 Contributing

Contributions are welcome! Please read [CONTRIBUTING.md](./CONTRIBUTING.md) before opening a pull request.

---

## 📄 License

This project is licensed under the MIT License. See [LICENSE](./LICENSE) for details.
