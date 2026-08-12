# AI Candidate Screening Pipeline

> **An AI-powered recruitment workflow that transforms incoming resumes into structured candidate profiles, applies transparent scoring, routes candidates by fit, and automates recruiter notifications and candidate communication.**

Built with **n8n, Groq, Google Sheets, Google Drive, Gmail, Slack, and Calendly**.

[![n8n](https://img.shields.io/badge/n8n-Workflow-EA4B71?logo=n8n\&logoColor=white)](https://n8n.io)
[![Groq](https://img.shields.io/badge/Groq-Llama%203.3%2070B-orange)](https://groq.com)
[![Google Sheets](https://img.shields.io/badge/Google%20Sheets-Pipeline-34A853?logo=google-sheets\&logoColor=white)](https://sheets.google.com)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## Overview

Recruitment teams often spend significant time reviewing resumes, comparing candidates against job requirements, and manually communicating screening outcomes.

This project automates that first-pass process.

A new application triggers an event-driven n8n workflow that:

1. Retrieves the candidate's resume.
2. Extracts structured candidate information using an LLM.
3. Loads the requirements for the relevant position.
4. Calculates an explainable candidate-fit score.
5. Routes the candidate into one of three outcomes.
6. Logs the decision to the recruitment pipeline.
7. Sends the appropriate communication.
8. Notifies the hiring team through Slack.

The result is a **repeatable, auditable first-pass screening pipeline** rather than a black-box AI decision.

> **AI extracts and structures the information. Deterministic code makes the scoring and routing decision.**

---

## Workflow

```text
Candidate Application
        │
        ▼
Google Sheets Trigger
        │
        ▼
Resume / File Retrieval
        │
        ▼
PDF Text Extraction
        │
        ▼
Candidate Object
        │
        ├──────────────► Job Requirements
        │
        ▼
Groq LLM
        │
        ▼
Structured Candidate JSON
        │
        ▼
Parse AI Response
        │
        ▼
Merge Candidate + Requirements
        │
        ▼
Candidate Scoring Engine
        │
        ▼
     ┌──┴───────────────┐
     │                  │
     ▼                  ▼
Score / Status       Pipeline Log
     │
     ▼
Route by Status
     │
 ┌───┼────────────┐
 ▼   ▼            ▼
Shortlisted   Screening   Rejected
 │              │           │
 ▼              ▼           ▼
Scheduling   Manual       Decline
Email         Review       Email
 │
 ▼
Calendly
 │
 ▼
Slack Notification
```

---

## Key Features

### AI Resume Extraction

Candidate resumes are processed by an LLM and converted into a consistent structured schema containing:

* Skills
* Programming languages
* Frameworks
* Databases
* Cloud platforms
* Tools
* Experience
* Education
* Certifications
* Projects
* Strengths
* Weaknesses

The extraction prompt requires structured JSON output and avoids inventing missing candidate information.

---

### Explainable Candidate Scoring

The final hiring route is **not determined solely by the LLM**.

The workflow uses a deterministic JavaScript scoring engine.

| Component        |   Weight |
| ---------------- | -------: |
| Required Skills  |      60% |
| Preferred Skills |      20% |
| Experience       |      20% |
| **Maximum**      | **100%** |

Skills are matched across the candidate's extracted technical profile using normalized/fuzzy matching.

This provides a score that can be inspected, reproduced, and explained.

---

### Candidate Routing

The default minimum score is **70**.

|  Score | Status        | Recommendation | Action           |
| -----: | ------------- | -------------- | ---------------- |
| 70–100 | `Shortlisted` | Interview      | Scheduling email |
|  49–69 | `Screening`   | Manual Review  | Recruiter review |
|   0–48 | `Rejected`    | Reject         | Decline email    |

The threshold can be configured per role through the requirements sheet.

---

## Why Separate AI From Scoring?

A key architectural decision in this project is separating **unstructured AI extraction** from **deterministic business logic**.

### AI is responsible for:

* Understanding resume content
* Extracting candidate information
* Normalizing skills
* Structuring unstructured text

### Code is responsible for:

* Calculating the score
* Applying thresholds
* Determining status
* Routing candidates
* Producing audit information

This makes the workflow easier to debug and reduces the risk of an LLM making an inconsistent final routing decision.

---

## Pipeline Logging

Every processed candidate is written to the recruitment pipeline with information such as:

* Candidate name
* Position
* Fit score
* Status
* Matched skills
* Missing skills
* Reasoning
* Processing information

This creates an audit trail for screening decisions.

---

## Automated Communication

### Shortlisted candidates

Candidates who meet the configured threshold receive a branded HTML email containing a Calendly scheduling link for the next stage.

### Screening candidates

Candidates in the middle score band remain available for manual review rather than being automatically rejected.

### Rejected candidates

Candidates below the review threshold receive a professional decline email.

This preserves a human-review path instead of forcing every candidate into an automated accept/reject decision.

---

## Recruiter Notifications

Slack notifications keep the hiring team informed when candidates move through the screening pipeline.

This removes the need for recruiters to continuously monitor the underlying spreadsheet.

---

# Architecture

```text
                 ┌──────────────────────┐
                 │   Google Sheets       │
                 │   Application Intake  │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │  Resume Retrieval    │
                 │  Google Drive        │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │   PDF Text Extraction│
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │ Candidate Object     │
                 └──────────┬───────────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
     ┌──────────────────┐       ┌──────────────────┐
     │ Candidate Data   │       │ Job Requirements │
     └────────┬─────────┘       └────────┬─────────┘
              │                          │
              └────────────┬─────────────┘
                           ▼
                 ┌──────────────────────┐
                 │ Groq LLM             │
                 │ Structured Extraction │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │ Parse AI JSON        │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │ Merge Data           │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │ Scoring Engine       │
                 │ Deterministic JS     │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │ Status Router        │
                 └───────┬──┬──┬───────┘
                         │  │  │
               ┌─────────┘  │  └─────────┐
               ▼            ▼            ▼
          Shortlisted    Screening    Rejected
               │            │            │
               ▼            ▼            ▼
          Calendly      Manual Review  Decline
            Email                       Email
               │
               ▼
             Slack
```

---

# Data Flow

### Application Intake

The workflow expects candidate application data from Google Sheets.

Example fields:

| Field               | Example                                     |
| ------------------- | ------------------------------------------- |
| Full Name           | Jane Doe                                    |
| Email Address       | [jane@example.com](mailto:jane@example.com) |
| Position            | Senior Backend Engineer                     |
| Years of Experience | 5                                           |
| LinkedIn            | linkedin.com/in/janedoe                     |
| GitHub              | github.com/janedoe                          |
| Location            | Remote                                      |
| Resume              | Google Drive PDF                            |

---

### Job Requirements

Each position can have its own requirements.

| Field              | Example                    |
| ------------------ | -------------------------- |
| Required Skills    | Python, PostgreSQL, Docker |
| Preferred Skills   | Kubernetes, Redis          |
| Minimum Experience | 3 years                    |
| Minimum Score      | 70                         |

This allows the same workflow to screen candidates for different positions without changing the underlying scoring engine.

---

# AI Extraction Schema

The LLM returns structured candidate information:

```json
{
  "candidateName": "",
  "email": "",
  "phone": "",
  "position": "",
  "location": "",
  "summary": "",
  "skills": [],
  "programmingLanguages": [],
  "frameworks": [],
  "databases": [],
  "cloudPlatforms": [],
  "tools": [],
  "education": "",
  "certifications": [],
  "yearsExperience": 0,
  "projects": [],
  "strengths": [],
  "weaknesses": []
}
```

Missing information should remain empty rather than being fabricated.

---

# Technology Stack

| Layer                         | Technology              |
| ----------------------------- | ----------------------- |
| Workflow Orchestration        | n8n                     |
| AI / LLM                      | Groq — Llama 3.3 70B    |
| Candidate Intake              | Google Sheets           |
| Resume Storage                | Google Drive            |
| Resume Processing             | n8n document extraction |
| Candidate Database / Pipeline | Google Sheets           |
| Email                         | Gmail                   |
| Scheduling                    | Calendly                |
| Notifications                 | Slack                   |
| Scoring                       | JavaScript              |

---

# Project Structure

```text
candidate-screening-pipeline/
│
├── README.md
├── LICENSE
│
├── workflows/
│   └── 02 - Candidate Screening Pipeline.json
│
├── docs/
│   └── images/
│       ├── workflow-overview.png
│       ├── ai-extraction.png
│       ├── scoring-routing.png
│       ├── shortlisted-email.png
│       └── rejection-email.png
│
└── examples/
    └── sample-candidate.json
```

---

# Quick Start

## Requirements

You will need:

* n8n self-hosted or cloud
* Google account
* Google Sheets
* Google Drive
* Gmail OAuth
* Groq API key
* Slack workspace — optional
* Calendly link

## Import the Workflow

1. Open n8n.
2. Select **Workflows → Import from File**.
3. Import the workflow JSON from `workflows/`.
4. Configure the required credentials.
5. Verify the Google Sheets and Drive references.
6. Configure your Calendly link.
7. Configure the Slack destination.
8. Run the workflow using a test candidate.

---

# Configuration

### Minimum Score

Set the `Minimum Score` field in the Requirements sheet.

Default:

```text
70
```

### Calendly

Update the scheduling URL inside the shortlist email.

### AI Prompt

The candidate extraction prompt can be customized inside the LLM node.

### Email Templates

The shortlist and rejection email templates can be modified directly in their respective n8n nodes.

### Slack

Configure the destination channel used for recruiter notifications.

---

# Testing

Use synthetic candidate information while testing the workflow.

Verify:

* [ ] Resume is successfully retrieved
* [ ] Resume text is extracted
* [ ] LLM returns valid JSON
* [ ] Candidate data is merged with requirements
* [ ] Fit score is calculated
* [ ] Candidate status is correct
* [ ] Shortlisted branch works
* [ ] Screening branch works
* [ ] Rejected branch works
* [ ] Pipeline entry is created
* [ ] Emails contain the correct candidate information
* [ ] Calendly link works
* [ ] Slack notification is delivered

---

# Limitations

This project is designed for **first-pass screening**, not autonomous hiring.

The workflow does not replace recruiter judgment or make final employment decisions.

### Current limitations

* Text-based PDFs work out of the box.
* Scanned/image-only resumes require OCR.
* Skill matching is rule-based fuzzy matching and may require domain-specific tuning.
* LLM extraction quality depends on the resume format and model.
* Production deployments should implement appropriate candidate-data security, retention, access control, and compliance policies.

---

# Roadmap

* [ ] OCR support for scanned resumes
* [ ] Recruiter approve/reject actions from Slack
* [ ] Multi-role screening
* [ ] ATS integrations
* [ ] Analytics dashboard
* [ ] Candidate skill-gap analysis
* [ ] Recruiter feedback loop for scoring calibration

---

# Before & After

| Traditional Process          | Automated Pipeline                |
| ---------------------------- | --------------------------------- |
| Manual resume review         | Automated first-pass extraction   |
| Subjective initial screening | Transparent weighted scoring      |
| Manual candidate routing     | Rule-based routing                |
| Individual email handling    | Automated candidate communication |
| Spreadsheet maintenance      | Automated pipeline logging        |
| Recruiter checks for updates | Slack notifications               |

---

# Responsible Use

This system is intended to **assist recruiters with first-pass screening**, not make final employment decisions.

Recruitment workflows should be reviewed for:

* Bias and fairness
* Data privacy
* Local employment regulations
* Candidate consent
* Data retention
* Human oversight

For demonstrations and development, use synthetic candidate data rather than real applicant information.

---

# Extending the Workflow

The architecture is intentionally modular.

The intake layer can be replaced with:

* Webhooks
* ATS platforms
* Application forms
* CRM systems
* Custom recruitment portals

The AI provider can also be replaced without changing the scoring and routing architecture.

The scoring engine remains independent from the LLM layer, allowing the workflow to evolve without rewriting the business logic.

---

# Author

## Nexoryn AI

**AI Automation • Workflow Engineering • Custom Software**

Built by **Nexoryn AI** as a demonstration of production-oriented workflow orchestration, AI-assisted data extraction, deterministic business logic, and automated operational systems.

* Portfolio: *Add your portfolio URL*
* LinkedIn: *Add your LinkedIn*
* GitHub: *Add your GitHub profile*

---

# License

MIT License.

See [LICENSE](LICENSE) for details.
