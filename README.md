# Hospital Node Boilerplate

A lightweight FastAPI application that simulates a single, siloed hospital database node for **The Open Accelerator Healthcare Hackathon — Track 1: Federated Medical Imaging Search**.

> **First time?** Start with the [Pre-Hackathon Setup Guide](PRE_HACK_SETUP.md) to get Python, Git, and everything else installed before the event.

## What This Is

This boilerplate represents an intentionally "dumb" hospital edge node. Each instance:

- Blindly serves its own local study data
- Has **zero awareness** of other hospitals
- Lacks **any authentication**
- Intentionally **leaks PII** (patient names, birthdates)

You will run **three separate instances** on different ports to simulate a disconnected, multi-hospital network. Your challenge is to build the overarching aggregation, privacy, and access-control layer on top.

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  BCH  :8001  │   │  MGH  :8002  │   │  BWH  :8003  │
│  900 studies │   │  900 studies │   │  900 studies │
│  No auth     │   │  No auth     │   │  No auth     │
│  PII exposed │   │  PII exposed │   │  PII exposed │
└──────────────┘   └──────────────┘   └──────────────┘
       ↑                  ↑                  ↑
       └──────────────────┼──────────────────┘
                          │
                    YOUR SOLUTION
              (aggregator, auth, redaction)
```

## Quick Start

### 1. Install dependencies

```bash
cd hospital-node-boilerplate
pip install -r requirements.txt
```

### 2. Start the three hospital nodes

Open three separate terminal windows:

```bash
# Terminal 1 — Boston Children's Hospital
HOSPITAL_NODE=BCH uvicorn main:app --port 8001 --reload

# Terminal 2 — Massachusetts General Hospital
HOSPITAL_NODE=MGH uvicorn main:app --port 8002 --reload

# Terminal 3 — Brigham and Women's Hospital
HOSPITAL_NODE=BWH uvicorn main:app --port 8003 --reload
```

### 3. Verify they're running

```bash
curl http://localhost:8001/health
# {"status":"healthy","node":"BCH"}

curl http://localhost:8002/health
# {"status":"healthy","node":"MGH"}

curl http://localhost:8003/health
# {"status":"healthy","node":"BWH"}
```

## API Reference

Each node exposes three endpoints. All responses are JSON.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/health` | Health check — returns node name and status |
| `GET` | `/api/studies` | Returns all study records on this node |
| `GET` | `/api/studies/{study_id}` | Returns a single study by StudyID, or 404 |

**Examples:**

```bash
# Get all studies from BCH
curl http://localhost:8001/api/studies

# Get a specific study by ID
curl http://localhost:8001/api/studies/BR-7721
```

> **Note:** There is no search endpoint — that's intentional. Building search, filtering, and cross-node querying is part of your challenge.

### Interactive API Docs

FastAPI auto-generates interactive Swagger UI docs for each running node:

- BCH: http://localhost:8001/docs
- MGH: http://localhost:8002/docs
- BWH: http://localhost:8003/docs

## Data Schema

Each study record contains these fields (all strings):

| Field | Format | Example |
|-------|--------|---------|
| `PatientName` | `LastName^FirstName` | `Harrington^Lucas` |
| `PatientID` | `PREFIX-NNNNN` | `CHB-99214` |
| `PatientBirthDate` | `YYYYMMDD` | `20181104` |
| `PatientAge` | `NNNY` / `NNNM` / `NNND` | `007Y` |
| `PatientSex` | `M` / `F` | `M` |
| `InstitutionName` | Full hospital name | `Boston Children's Hospital` |
| `StudyID` | `PREFIX-NNNN` | `BR-7721` |
| `StudyInstanceUID` | DICOM UID format | `1.3.12.2.1107.5.2.19.45152...` |
| `StudyDate` | `YYYYMMDD` | `20260715` |
| `Modality` | DICOM modality code | `MR` |
| `BodyPartExamined` | `BRAIN` / `HEART` / `FETAL` | `BRAIN` |
| `Diagnosis` | Full radiology report | Multi-paragraph clinical text |

## Data Overview

Each hospital has 900 pre-generated study records (300 brain, 300 heart, 300 fetal):

- **BCH** (Boston Children's Hospital) — Pediatric patients, ages 0–21
- **MGH** (Massachusetts General Hospital) — Adult patients, ages 22–85
- **BWH** (Brigham and Women's Hospital) — Adult patients, ages 18–75

Conditions overlap across hospitals, so a federated search for something like "hydrocephalus" will return results from multiple nodes.

## Regenerating Data

The pre-generated data files are in `data/`. If you want to regenerate or expand the dataset:

```bash
cd scripts
pip install -r requirements.txt
GEMINI_API_KEY=your-key-here python generate_data.py
```

Use `--dry-run` to preview what would be generated without calling the API:

```bash
python generate_data.py --dry-run
```

Adjust batch size if you hit rate limits:

```bash
GEMINI_API_KEY=your-key python generate_data.py --batch-size 25
```

## Architecture

```
hospital-node-boilerplate/
├── main.py              # FastAPI app — reads HOSPITAL_NODE env var to pick data file
├── models.py            # Pydantic StudyRecord schema
├── requirements.txt     # Runtime dependencies (fastapi, uvicorn, pydantic)
├── data/
│   ├── bch_data.json    # 900 records — Boston Children's Hospital
│   ├── mgh_data.json    # 900 records — Massachusetts General Hospital
│   └── bwh_data.json    # 900 records — Brigham and Women's Hospital
└── scripts/
    ├── generate_data.py # Gemini-powered data generator
    └── requirements.txt # Generation-only dependencies
```

The `HOSPITAL_NODE` environment variable controls which JSON file gets loaded into memory at startup. The app is completely stateless — no database, no external services.

## What to Build Next

This boilerplate gives you three isolated hospital nodes. Your hackathon challenge is to build:

1. **A central aggregator** that queries all three nodes and merges results
2. **Zero-trust access control** — authentication and authorization for who can query what
3. **PII redaction** — strip or mask patient names, birthdates, and other identifiers before returning results
4. **Cross-node routing** — intelligent query distribution and result aggregation
5. **Privacy-preserving search** — federated queries that never move raw patient data

## Tech Stack

- Python 3.10+
- FastAPI
- Uvicorn
- Pydantic
