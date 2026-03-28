# aegis-vulnops

Autonomous vulnerability lifecycle management agent.

Ingests findings from **DefectDojo** (REST API) or **Excel/CSV** files, verifies them via SSH or Git repository access, proposes AI-generated fixes using Claude, creates Jira/ServiceNow tickets from templates, and posts status updates back to DefectDojo and Checkmarx.

DefectDojo is **optional** — the agent runs fully from an Excel spreadsheet with no external services required beyond the Anthropic API.

---

## Features

| Capability | Detail |
|---|---|
| **Dual ingestion** | DefectDojo REST API (paginated) or `.xlsx`/`.csv` files |
| **Multi-tracking IDs** | Jira, ServiceNow, Checkmarx, BA ticket — all per finding |
| **SSH verification** | Qualys findings — `dpkg`/`rpm`/`pip` checks on target host |
| **Git verification** | Prisma findings — clone repo and pattern-match vulnerable code |
| **AI remediation** | Claude `claude-opus-4-6` proposes unified diffs or routes to team |
| **SLA prioritisation** | Overdue findings first, then by severity (critical → info) |
| **Jinja2 ticketing** | `jira_security.j2`, `jira_dev.j2`, `change_request.j2` |
| **Feedback loop** | Posts comments to DefectDojo, Checkmarx, Jira |
| **Audit trail** | Every action written to SQLite (`vuln_actions` table) |

---

## Quick Start

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Configure environment
cp .env.example .env
# Edit .env — only ANTHROPIC_API_KEY is required for Excel mode

# 3a. Run with an Excel file (no DefectDojo needed)
python -m vulnops.agent --source excel --file findings.xlsx

# 3b. Run against DefectDojo (continuous polling)
python -m vulnops.agent --source defectdojo

# Dry run (no tickets created, no external writes)
python -m vulnops.agent --source excel --file findings.xlsx --dry-run
```

---

## Excel Column Mapping

The ingestor auto-detects columns (case-insensitive). Supported headers:

| Finding Field | Accepted Column Names |
|---|---|
| Title | `Title`, `Vulnerability`, `Finding` |
| Severity | `Severity`, `Risk`, `Risk Level` |
| Scanner | `Scanner`, `Source`, `Tool` |
| Component | `Component`, `Package`, `Library` |
| Version | `Version`, `Pkg Version` |
| Host | `Host`, `Target`, `IP`, `Hostname` |
| Repository | `Repo`, `Repository`, `Git URL` |
| Due Date | `Due Date`, `SLA Date`, `Fix By` |
| Jira ID | `Jira`, `Jira ID`, `Jira Ticket` |
| ServiceNow | `ServiceNow`, `CR`, `Change Request` |
| BA Ticket | `BA Ticket`, `BA`, `BA ID` |

---

## Architecture

```
ingestor_factory(source)
├── DefectDojoIngestor  →  paginated GET /api/v2/findings/
└── ExcelIngestor       →  openpyxl / csv.DictReader
        │
        ▼
AIRemediator.prioritize()   (overdue first → severity)
        │
        ▼ (per finding)
verifier_factory(scanner_type).verify()
├── QualysVerifier  →  SSH → dpkg/rpm/pip
└── PrismaVerifier  →  Git clone → pattern match
        │
  [not confirmed] → false_positive
  [confirmed]     ↓
AIRemediator.analyze()  →  Claude proposes diff or routes to team
        │
  [can_fix]    →  post diff as DefectDojo note / Jira comment
  [manual]     →  TicketingEngine.route_to_team() → Jira / ServiceNow CR
        │
FeedbackEngine  →  DefectDojo note, Checkmarx, Jira comment, SQLite audit
```

---

## Project Structure

```
aegis-vulnops/
├── vulnops/
│   ├── agent.py        — Main async loop + CLI
│   ├── models.py       — Pydantic models
│   ├── ingestor.py     — DefectDojo + Excel ingestion
│   ├── client.py       — DefectDojo async REST client
│   ├── verifier.py     — SSH/Git modular verifiers
│   ├── remediator.py   — Claude AI remediation engine
│   ├── ticketing.py    — Jira/ServiceNow ticket creator
│   ├── feedback.py     — Comment posting + audit log
│   ├── db.py           — SQLite schema bootstrap
│   └── templates/
│       ├── jira_security.j2
│       ├── jira_dev.j2
│       └── change_request.j2
├── common/
│   ├── db.py           — SQLite connection factory
│   └── logger.py       — Structured JSON logging
├── pyproject.toml
├── requirements.txt
└── .env.example
```

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `ANTHROPIC_API_KEY` | Yes | Claude API key |
| `DEFECTDOJO_URL` | No | DefectDojo base URL |
| `DEFECTDOJO_API_TOKEN` | No | DefectDojo API token |
| `JIRA_URL` | No | Jira base URL |
| `JIRA_USER` | No | Jira username (email) |
| `JIRA_TOKEN` | No | Jira API token |
| `JIRA_PROJECT_KEY` | No | Jira project key (default: `SEC`) |
| `SSH_KEY_PATH` | No | SSH private key path |
| `SSH_USERNAME` | No | SSH username |
| `CHECKMARX_URL` | No | Checkmarx base URL |
| `CHECKMARX_TOKEN` | No | Checkmarx API token |
| `VULNOPS_POLL_INTERVAL_SECONDS` | No | Polling interval (default: 300) |
| `VULNOPS_EXCEL_FILE` | No | Default Excel file path |
