# Folder Structure — AI-Powered IPsec VPN Protocol Analyzer

## 1. Recommended repository layout

```text
ipsec-vpn-analyzer/
├── README.md
├── architecture.md
├── design.md
├── folder-structure.md
├── .gitignore
├── .env.example
├── docker-compose.yml                 # Optional, add when container setup is needed
│
├── backend/
│   ├── pyproject.toml                 # Or requirements.txt; choose one primary dependency format
│   ├── requirements-dev.txt           # Optional development/test dependencies
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                    # FastAPI app creation and router registration
│   │   ├── core/
│   │   │   ├── __init__.py
│   │   │   ├── config.py               # Environment settings and safe defaults
│   │   │   ├── logging_config.py       # Structured application logging
│   │   │   └── errors.py               # Shared exceptions and API error mapping
│   │   ├── api/
│   │   │   ├── __init__.py
│   │   │   ├── deps.py                 # Shared API dependencies
│   │   │   └── v1/
│   │   │       ├── __init__.py
│   │   │       ├── router.py           # Combines v1 endpoints
│   │   │       └── endpoints/
│   │   │           ├── __init__.py
│   │   │           ├── health.py       # GET /health
│   │   │           ├── analyses.py     # Upload, status, delete analysis
│   │   │           ├── findings.py     # Finding list/detail endpoints
│   │   │           └── reports.py      # Report download endpoint
│   │   ├── schemas/
│   │   │   ├── __init__.py
│   │   │   ├── analysis.py             # Request/response models
│   │   │   ├── protocol.py             # Protocol observation schemas
│   │   │   ├── finding.py              # Security finding schema
│   │   │   └── traffic.py              # Traffic summary schema
│   │   ├── db/
│   │   │   ├── __init__.py
│   │   │   ├── session.py              # Database connection/session
│   │   │   ├── models.py               # SQLAlchemy models
│   │   │   └── migrations/             # Add Alembic migrations when needed
│   │   ├── services/
│   │   │   ├── __init__.py
│   │   │   ├── analysis_service.py     # Orchestrates one analysis
│   │   │   ├── capture_service.py      # Capture intake and safe file handling
│   │   │   ├── protocol_service.py     # Protocol/flow analysis coordination
│   │   │   ├── assessment_service.py   # Rules + score aggregation
│   │   │   └── report_service.py       # Report generation
│   │   ├── analyzer/
│   │   │   ├── __init__.py
│   │   │   ├── parser_base.py          # Parser interface
│   │   │   ├── tshark_parser.py        # tshark-backed parser
│   │   │   ├── scapy_parser.py         # Optional alternative/parser helper
│   │   │   ├── normalizer.py           # Parser output -> normalized records
│   │   │   ├── flow_features.py        # Flow statistics and feature extraction
│   │   │   └── evidence.py              # Evidence references and provenance
│   │   ├── security/
│   │   │   ├── __init__.py
│   │   │   ├── engine.py               # Runs all enabled security rules
│   │   │   ├── scoring.py              # Transparent score calculation
│   │   │   ├── policy.py               # Configurable security policy values
│   │   │   └── rules/
│   │   │       ├── __init__.py
│   │   │       ├── algorithms.py       # Known algorithm policy checks
│   │   │       ├── key_exchange.py     # DH/PFS checks when evidence exists
│   │   │       ├── sa_lifetime.py      # SA lifetime checks
│   │   │       ├── replay_protection.py# Replay protection evidence checks
│   │   │       └── metadata_exposure.py# Observable metadata assessment
│   │   ├── ml/
│   │   │   ├── __init__.py
│   │   │   ├── features.py             # ML feature schema
│   │   │   ├── predict.py              # Load model and produce predictions
│   │   │   ├── train.py                # Offline training script
│   │   │   └── evaluate.py              # Metrics and validation
│   │   └── reporting/
│   │       ├── __init__.py
│   │       ├── templates/
│   │       │   └── report.html         # HTML report template
│   │       └── render.py               # Populate and export report
│   │
│   ├── tests/
│   │   ├── conftest.py
│   │   ├── unit/
│   │   │   ├── test_normalizer.py
│   │   │   ├── test_evidence.py
│   │   │   ├── test_scoring.py
│   │   │   └── test_security_rules.py
│   │   ├── api/
│   │   │   └── test_analyses_api.py
│   │   └── integration/
│   │       └── test_analysis_pipeline.py
│   └── scripts/
│       ├── check_tshark.py             # Verify parser dependency
│       └── create_demo_capture.md       # Notes for generating controlled test data
│
├── frontend/
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── index.html
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── api/
│       │   ├── client.ts               # API base URL and shared request handling
│       │   └── analyses.ts             # Analysis-related API calls
│       ├── components/
│       │   ├── layout/
│       │   ├── charts/
│       │   ├── findings/
│       │   └── common/
│       ├── pages/
│       │   ├── OverviewPage.tsx
│       │   ├── NewAnalysisPage.tsx
│       │   ├── AnalysisResultsPage.tsx
│       │   ├── FindingsPage.tsx
│       │   ├── TrafficPage.tsx
│       │   └── ReportPage.tsx
│       ├── types/
│       │   ├── analysis.ts
│       │   ├── finding.ts
│       │   └── traffic.ts
│       └── utils/
│           ├── formatters.ts
│           └── severity.ts
│
├── data/
│   ├── README.md                       # Explain safe local data handling
│   ├── samples/                        # Small, sanitized, authorized sample captures only
│   ├── datasets/
│   │   ├── README.md                   # Dataset provenance, labels, split strategy
│   │   └── .gitkeep
│   └── models/
│       └── .gitkeep                    # Do not commit large model artifacts by default
│
├── storage/
│   ├── uploads/
│   │   └── .gitkeep                    # Runtime uploads; ignored by Git
│   ├── reports/
│   │   └── .gitkeep                    # Runtime reports; ignored by Git
│   └── .gitkeep
│
├── lab/
│   ├── README.md                       # Authorized testbed setup notes
│   ├── strongswan/
│   │   └── README.md                   # Lab configuration guidance; no real secrets
│   └── scenarios/
│       └── README.md                   # Tunnel/Transport and other scenario definitions
│
└── docs/
    ├── api.md
    ├── scoring-methodology.md
    ├── dataset-methodology.md
    ├── testing.md
    └── demo-script.md
```

## 2. Folder responsibilities

- `backend/app/api/`: HTTP endpoints only; keep parsing and scoring logic out of route handlers.
- `backend/app/services/`: coordinates workflows across modules.
- `backend/app/analyzer/`: packet parsing, normalization, flow features, and evidence.
- `backend/app/security/`: deterministic checks and transparent score calculation.
- `backend/app/ml/`: offline training, evaluation, and optional prediction.
- `backend/app/reporting/`: report templates and rendering.
- `backend/app/db/`: persistent metadata and migrations.
- `backend/tests/`: unit, API, and end-to-end pipeline tests.
- `frontend/src/`: dashboard pages, reusable components, API client, and TypeScript types.
- `data/`: sanitized sample captures, dataset documentation, and model artifacts.
- `storage/`: runtime uploads and generated reports; never commit private captures.
- `lab/`: controlled IPsec testbed notes and repeatable scenarios.
- `docs/`: project-level technical documentation.

## 3. Important `.gitignore` entries

```gitignore
# Python
__pycache__/
*.py[cod]
.venv/
venv/
.pytest_cache/
.mypy_cache/
.ruff_cache/

# Frontend
node_modules/
dist/
coverage/

# Secrets and environment
.env
.env.*
!.env.example

# Runtime files and captures
storage/uploads/*
!storage/uploads/.gitkeep
storage/reports/*
!storage/reports/.gitkeep
*.pcap
*.pcapng
*.key
*.pem
*.p12
*.pfx

# Local database
*.db
*.sqlite
*.sqlite3

# Model artifacts / large generated data
data/models/*
!data/models/.gitkeep
data/datasets/*
!data/datasets/README.md
!data/datasets/.gitkeep

# OS / editor
.DS_Store
Thumbs.db
.vscode/
.idea/
```

Only add sanitized sample captures to version control when you have permission to share them and have checked that they contain no sensitive information.

## 4. Build the repository incrementally

Do not create every file and implement every module on day one. Begin with this minimum structure:

```text
ipsec-vpn-analyzer/
├── README.md
├── architecture.md
├── design.md
├── folder-structure.md
├── .gitignore
├── backend/
│   ├── requirements.txt
│   └── app/
│       ├── __init__.py
│       └── main.py
├── backend/tests/
└── frontend/
```

Then add analyzer, security, database, ML, reporting, and lab folders as each milestone begins. Empty folders are not tracked by Git unless they contain a file such as `.gitkeep`.

## 5. Naming conventions

- Python modules/functions: `snake_case`.
- Python classes: `PascalCase`.
- React components: `PascalCase.tsx`.
- API endpoints: plural nouns, under `/api/v1`.
- Test files: `test_*.py`.
- Configuration: environment variables with safe defaults; secrets only in local `.env`, never committed.
