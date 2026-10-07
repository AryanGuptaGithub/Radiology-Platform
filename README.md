# Armorray — Teleradiology & RIS Platform

Armorray is a full-stack teleradiology platform that connects imaging centers, technicians, QA reviewers, and radiologists in a single workflow — from the moment a scan leaves the modality to the moment a signed report reaches the referring doctor.

It combines a role-based case management system, a DICOM/PACS pipeline built on **Orthanc**, a browser-based diagnostic viewer (an **OHIF** fork), real-time collaboration via **Socket.io**, and an AI-assisted image quality check service.

---

## Table of Contents

- [Architecture](#architecture)
- [Core Features](#core-features)
- [Case Lifecycle](#case-lifecycle)
- [Roles](#roles)
- [Tech Stack](#tech-stack)
- [Repository Layout](#repository-layout)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Running with Docker](#running-with-docker)
- [API Surface](#api-surface)
- [Known Limitations](#known-limitations)
- [Documentation](#documentation)

---

## Architecture

```
┌─────────────┐      DICOM (C-STORE)       ┌──────────────┐
│  Modality /  │ ─────────────────────────▶ │   Orthanc    │
│   Scanner    │                             │  PACS Server │
└─────────────┘                             └──────┬───────┘
                                                    │ DICOMweb
┌─────────────┐   custom DICOM SCP (4243)          │
│  Modality /  │ ───────────────┐                  │
│   Scanner    │                 ▼                  ▼
└─────────────┘        ┌─────────────────┐   ┌─────────────┐
                        │  Node.js Backend │◀──│   OHIF      │
                        │  Express 5 +     │   │   Viewer    │
                        │  MongoDB         │   │ (browser)   │
                        └──────┬───────────┘   └─────────────┘
                               │ Socket.io (realtime)
                               │ REST API
                        ┌──────▼───────────┐   ┌──────────────────┐
                        │  React Frontend  │   │  AI Quality-Check │
                        │  (Staff Portal)  │   │  Microservice     │
                        └──────────────────┘   │  (Python/FastAPI) │
                                                └──────────────────┘
```

Three independently deployable applications share one MongoDB database and one PACS layer:

| Service | Role |
|---|---|
| **backend** | Express 5 API, MongoDB models, DICOM SCP listener, PACS (C-FIND/C-MOVE/C-STORE) integration, Socket.io realtime layer, report generation, auth. |
| **frontend** | React + TypeScript + Vite staff portal — the dashboards technicians, QA, radiologists, and admins actually work in. |
| **Viewers** | A customized fork of the open-source [OHIF Viewer](https://ohif.org/) for DICOM rendering, MPR, and annotation tools, with role-based tool restrictions enforced both client- and server-side. |

Images arrive through **two parallel DICOM intake paths** — a custom Node DICOM SCP on port `4243` and Orthanc's native receiver on `4242`/`8042` — routed by AE title. See [`PACS-Documentation.md`](./PACS-Documentation.md) for the full intake design.

---

## Core Features

- **Role-based case workflow** — technician upload → QA review → radiologist reporting → QA audit → medico-legal finalization, with full status tracking.
- **DICOM & PACS integration** — direct modality push, Orthanc-backed archive, C-FIND/C-MOVE querying of remote PACS nodes, and DICOMweb for viewer streaming.
- **Non-DICOM image intake** — JPEG/PNG upload path for clinics without DICOM-capable equipment, with manual scan-type tagging.
- **Integrated diagnostic viewer** — OHIF-based MPR/2D viewer embedded in the workflow, with per-role tool access control enforced server-side.
- **Structured reporting** — rich-text report editor (TipTap), DOCX/PDF export, report templates and macros.
- **Real-time collaboration** — Socket.io-powered live case updates, notifications, and chat, backed by a Redis adapter for multi-instance scaling.
- **AI-assisted image quality check** — a FastAPI microservice that flags metadata, structural, and clinical issues on incoming studies (currently opt-in rather than automatic — see [Known Limitations](#known-limitations)).
- **Audit logging & security** — JWT access/refresh tokens with rotation and reuse detection, audit trails on sensitive actions, rate limiting, and per-role access control.
- **Billing & payouts** — institution billing and radiologist payout tracking.
- **Shareable reports** — tokenized external share links for delivering results outside the platform.
- **Admin console** — user management, site/institution configuration, banners, templates, webhook configuration, and system analytics.

## Case Lifecycle

```
Uploaded → QA_Pending → QA_Review → Assigned → In_Progress → QA_Audit → Finalized
                                        │                         │
                                        └──────── Rep_Correction ◀┘
                             (QA_Review) → Rejected   ·   (PACS) → Incomplete
```

A case's `status` field (`backend/models/Case.js`) is the single source of truth for where it sits in the pipeline — every dashboard view filters on it.

## Roles

| Role | Responsibility |
|---|---|
| `technician` | Uploads studies (DICOM folder, PACS import, or image upload), enters patient/clinical details, submits for QA. |
| `qa` | Reviews incoming studies for completeness/quality, assigns radiologists, audits submitted reports before finalization. |
| `radiologist` | Opens assigned studies in the viewer, writes structured reports. |
| `admin` | Manages users, sites, institutions, templates, billing, and platform configuration. |
| `institution` | External-facing role for partner hospitals/clinics. |
| `user` | Base authenticated account type. |

## Tech Stack

**Backend**
- Node.js 18+, Express 5, MongoDB + Mongoose
- Socket.io 4 (+ Redis adapter for horizontal scaling)
- `dicom-dimse-native` / `dcmjs-dimse` for DICOM networking
- JWT auth (RS256) with refresh-token rotation, Helmet, rate limiting, Mongo sanitization
- `docx` / `html-to-docx` / `pdf-lib` / `puppeteer` for report generation and export
- Winston logging, Swagger/OpenAPI docs, `node-cron` for scheduled jobs
- Python 3 + FastAPI AI quality-check microservice (`backend/ai-service`)

**Frontend**
- React 18 + TypeScript + Vite
- Radix UI + Tailwind-based component system
- TanStack Query for data fetching
- TipTap for the structured report editor
- Cornerstone3D for client-side DICOM parsing utilities

**Viewer**
- OHIF Viewer fork (Cornerstone3D rendering engine)
- DICOMweb against Orthanc

**Infrastructure**
- MongoDB
- Orthanc (PACS / DICOM archive)
- Redis (Socket.io scaling)
- Docker Compose for local infra (Orthanc + Redis)

## Repository Layout

```
Radiology-Project/
├── backend/              Express API, DICOM SCP, AI quality-check service
│   └── ai-service/       Python/FastAPI image quality microservice
├── frontend/             React staff portal (technician/QA/radiologist/admin)
├── Viewers/              OHIF viewer fork
├── documentation/        Design docs, user-flow docs, architecture notes
├── productiontesting/    Load/smoke test scripts
├── docker-compose.yml    Local Orthanc + Redis infra
├── orthanc.json          Orthanc server configuration
└── PACS-Documentation.md DICOM intake & PACS integration reference
```

> `backend`, `frontend`, and `Viewers` are each their own git repository, checked into this one as submodules.

## Getting Started

### Prerequisites
- Node.js 18+
- MongoDB (local or remote)
- Docker (for Orthanc + Redis), or standalone installs of both
- Python 3.10+ (for the AI quality-check microservice, optional)

### 1. Clone

```bash
git clone --recurse-submodules <repo-url>
cd Radiology-Project
```

### 2. Start infrastructure

```bash
docker-compose up -d
```
Brings up Orthanc (`:8042` web UI / DICOMweb, `:4242` DICOM) and Redis (`:6379`).

### 3. Backend

```bash
cd backend
npm install
cp .env.example .env   # fill in the values — see Environment Variables below
npm run dev
```

### 4. Frontend

```bash
cd frontend
npm install
npm run dev
```

### 5. Viewer

```bash
cd Viewers
yarn install
yarn dev
```

## Environment Variables

The backend is configured entirely through `backend/.env`. Required keys:

| Variable | Purpose |
|---|---|
| `PORT`, `NODE_ENV` | Server port and environment mode |
| `MONGO_URI` | MongoDB connection string |
| `JWT_PRIVATE_KEY`, `JWT_PUBLIC_KEY`, `JWT_SECRET` | Auth token signing |
| `JWT_ACCESS_EXPIRY`, `JWT_REFRESH_EXPIRY` | Token lifetimes |
| `FRONTEND_URL`, `BASE_URL`, `ALLOWED_ORIGINS` | CORS / cookie scoping |
| `REDIS_URL` | Socket.io adapter (optional — falls back to single-instance mode) |
| `DICOM_SCU_AE_TITLE`, `DICOM_SCU_PORT`, `DICOM_INCOMING_DIR` | Custom DICOM SCP listener |
| `ORTHANC_URL`, `ORTHANC_USER`, `ORTHANC_PASSWORD` | Orthanc API access |
| `MONGODUMP_PATH` | Path to `mongodump` binary, for backups |

No secrets are committed to this repository — populate `.env` locally per environment.

## Running with Docker

`docker-compose.yml` currently provisions local infrastructure only (Orthanc + Redis). The Node/React/Python services are run directly with `npm`/`yarn`/`uvicorn` during development; containerizing the full stack for production deployment is tracked as future work.

## API Surface

The backend exposes a REST API organized by domain, with interactive docs served via Swagger:

`auth · patient · case · pacs · orthanc · qa · radiologist · report · template · macro · billing · payout · admin · analytics · performance · banner · webhook · apiKey · shareLink · backup · internal`

Run the backend and visit its `/api-docs` route for the full OpenAPI reference.

## Known Limitations

Documented honestly, so nothing here is mistaken for a finished guarantee:

- **No automated test suite or CI/CD pipeline** — deployment is a manual process.
- **AI quality-check is opt-in**, not automatically run on every incoming study.
- **WhatsApp notification on report finalization** is not yet implemented.
- **Automated radiologist assignment ("smart matching")** is not implemented — assignment is manual.
- Submodules must be initialized with `--recurse-submodules`; a plain clone leaves `backend`, `frontend`, and `Viewers` empty.

## Documentation

- [`PACS-Documentation.md`](./PACS-Documentation.md) — DICOM intake & PACS integration design
- [`documentation/PLATFORM_OVERVIEW_PLAIN_ENGLISH.md`](./documentation/PLATFORM_OVERVIEW_PLAIN_ENGLISH.md) — non-technical platform overview
- [`documentation/USER_FLOWS.md`](./documentation/USER_FLOWS.md) — role-by-role user flows
- [`documentation/system_architecture.md`](./documentation/system_architecture.md) — architecture notes

---

<p align="center">© Armorray</p>
