# Armorray Platform — Plain-English Overview

*A non-technical reference document for the new project owner, written after a full read-through of every piece of documentation in this repository and a verified, line-by-line review of the actual backend, frontend, and viewer code. Prepared 2026-09-24.*

---

## 1. What This Application Does

Armorray is a web-based **teleradiology platform** — it lets a hospital or scanning center send a patient's CT/MRI/X-ray images to a radiologist working remotely, tracks that image through a quality-check and reporting workflow, and delivers a finished, signed diagnostic report back to the hospital. Think of it as a digital "assembly line" for reading medical scans: images come in from a scanner, get checked and routed by a coordination team, get read and reported on by a doctor, get double-checked, and go back out — with billing, chat, and an audit trail running underneath the whole thing.

It is built from three separate pieces of software that work together: a **backend** (the engine and database), a **frontend** (the staff-facing website everyone logs into), and a **viewer** (a separate, industry-standard medical image viewer, used to actually look at the scans).

---

## 2. Who Uses It and How

The platform has five roles. Four are real, active staff logins; the fifth exists in the code but isn't finished.

- **Technician** — Works at (or on behalf of) the hospital/scan center. Logs in, creates or selects a patient, uploads the study (either automatically via a direct PACS connection, or manually), fills in patient details and clinical history, and can mark a case as an emergency. Can chat with the radiologist (without seeing their phone number), track report status, and — if the internet drops — the upload gets queued locally and retried automatically once the connection is back.
- **QA (Quality Analyst)** — The gatekeeper in the middle. Reviews every incoming case for correct patient data and usable image quality, rejects bad studies back to the technician with a reason, and assigns good studies to an available radiologist. After the radiologist submits a report, QA reviews it again before it's finalized and locked. QA is effectively the traffic controller of the whole workflow.
- **Radiologist** — Sees only the cases assigned to them. Opens the case, launches the image viewer to review the scan, and writes the report in a rich-text editor (with templates, keyboard-shortcut text expansion, and voice dictation available). Can accept or reject an assigned case, and can revise a report if QA sends it back.
- **Admin** — Runs the platform itself: creates/manages user accounts and roles, configures billing rates, monitors turnaround times and system health, manages security settings and API keys for outside partners, and can control which viewer tools each role is allowed to use.
- **"User"** *(generic/unfinished role)* — A route and dashboard exist for a fifth role called simply "user," but today it just shows a name and email with no real functionality. It's not clear from the documentation whether this was meant to become a hospital self-service portal, a patient-facing view, or something else — it was never specified or finished.

There is also a **non-human "user"**: an external reporting partner (referred to in the docs as "5C Network") can connect via a secure API key to pull assigned cases and push finished reports back in — without a person ever logging into the website.

---

## 3. The Big Pieces

- **Case Workflow Engine** — The core of the backend. Every study is a "Case" that moves through a fixed set of stages (uploaded → QA review → assigned to a radiologist → in progress → QA audit → finalized, with reject/correction loops along the way). Every change is timestamped into an audit trail.
- **PACS / Image Intake Pipeline** — The part that receives scans. It can accept images two ways: (1) a hospital's scanner/PACS pushes images directly over the network, and the system automatically figures out which technician they belong to based on the sending machine's ID; or (2) a technician manually searches a hospital's PACS and pulls a study in, or uploads files directly. Either way, images are archived in an open-source image server called **Orthanc**.
- **The Image Viewer** — A separate, well-known open-source medical image viewer called **OHIF Viewer** (used by hospitals worldwide) is deployed as its own mini-website. When a radiologist clicks "View Study," it opens in a new browser tab and streams the images directly from Orthanc. It's been lightly customized: a genuine role-based permission system locks down certain tools (e.g., measurement tools) for certain roles, checked both in the viewer and re-checked on the server for safety.
- **Reporting Module** — A rich-text report editor (built on a well-known editor called Tiptap) with saved templates, text-expansion shortcuts ("macros"), and voice dictation. Reports export to PDF/Word and get digitally signed. Once finalized, a report is legally locked from further editing.
- **Real-Time Layer** — Live chat (scoped to each case, viewable by QA/Admin for oversight), live notifications, and live case-status updates across all dashboards, powered by a technology called Socket.io — no page refreshing needed.
- **Billing & Payouts** — Configurable pricing per scan type/hospital, an automatic billing snapshot locked in at the moment a report is finalized (so past invoices can't silently change), hospital invoicing, and radiologist payout tracking.
- **Admin & Security Console** — User/role management, audit logs, login-activity tracking, backup management, and management of API keys for outside partners.
- **AI Image Quality-Check Service** — A separate small Python service that can analyze uploaded images for blurriness, motion artifacts, poor contrast, or the wrong body part being scanned. It works, but — see Section 7 — it does **not** currently run automatically on every upload; it has to be triggered manually.
- **External Partner API** — A secured, key-based API that lets an outside radiology reporting company fetch assigned cases and submit finished reports programmatically, without a login.
- **A second, currently unused image viewer** — There is also a large amount of custom, in-house viewer code (built on the same underlying technology as OHIF) sitting in the frontend. It is fully written and was extensively performance-tuned, but every part of it is presently disabled ("commented out"). See Section 7 — this is one of the more important things to understand before talking to a client.

---

## 4. How Data Moves Through the System

1. **A scan happens** at a hospital or imaging center on a CT/MRI/X-ray machine.
2. **The images arrive** at Armorray one of two ways: the hospital's own PACS system pushes them straight over the network to Armorray's intake listener, or a technician manually imports/uploads them through the website.
3. **The system files it away**: the images are saved to disk, a "Case" record is created in the database, the case is automatically routed to the correct technician (based on which hospital sent it), and the images are also copied into Orthanc, the image archive that the viewer reads from.
4. **The technician enriches the case** — adding/confirming patient details, clinical history, urgency level, and any supporting documents or photos.
5. **QA reviews it** — checking patient identity, image completeness, and quality. Bad cases bounce back to the technician; good cases move forward.
6. **QA assigns a radiologist**, who gets notified, opens the case in their dashboard, and launches the image viewer (which talks directly to Orthanc to stream the actual pixel data — the main website itself never has to handle the heavy image data).
7. **The radiologist writes and submits a report** using the built-in report editor.
8. **QA does a final check** and, if satisfied, finalizes the report — which locks it, generates a billing record, and notifies the hospital.
9. **Every step along the way** is logged for audit, and pushed live to relevant dashboards via the real-time chat/notification layer, so nobody has to refresh a page to see updates.

---

## 5. Key Technologies Used (and Why)

| Technology | Where it's used | Why it's a sensible choice |
|---|---|---|
| **Node.js + Express** | Backend engine | Fast to build with, huge ecosystem, works naturally with real-time features like chat and live notifications. |
| **MongoDB** | Main database | Flexible record structure — useful because different scan types/hospitals need slightly different fields on a "case" without a rigid, hard-to-change database structure. |
| **Orthanc** | Image archive / PACS server | A mature, free, open-source medical image server — avoids the enormous cost and risk of building image storage and retrieval from scratch. |
| **OHIF Viewer** | The actual scan viewer | A free, open-source, browser-based viewer used by hospitals worldwide — building a competitive medical image viewer from zero would be a multi-year undertaking on its own. |
| **React + TypeScript + Vite** | Staff-facing website | The current industry standard for building fast, maintainable web interfaces; TypeScript catches a class of bugs before they reach users. |
| **Socket.io** | Real-time chat/notifications | Lets the app push instant updates to a browser without the browser having to keep asking "anything new?" |
| **Redis** | Caching, and scaling the real-time layer | Speeds up repeated requests and lets the chat/notification system work correctly even if the app is later run on more than one server. |
| **Docker** | Packaging Orthanc, Redis, and the OHIF Viewer | Makes those services run the same way regardless of which server they're deployed to. |
| **Puppeteer / pdf-lib / docx** | Report and invoice generation | Automatically produces polished PDF/Word documents rather than requiring manual formatting. |
| **A separate Python service** | AI image quality checks | Python has by far the strongest tooling for image/scientific analysis, which is why this one piece is written in a different language than the rest of the backend. |

---

## 6. Radiology-Specific Concepts in Play

- **DICOM** is the international standard file format for medical images — it bundles the picture together with structured medical metadata (patient name, scan type, machine settings, etc.). Almost everything in this system is built around handling DICOM correctly.
- **PACS** ("Picture Archiving and Communication System") is the generic industry term for a hospital's medical-image server. In this project, **Orthanc** plays that role for Armorray itself, and the platform also knows how to talk to a *hospital's own* PACS to pull studies in.
- **DICOMweb** is a modern, web-friendly way of moving DICOM images over the internet using standard web requests, instead of older specialized medical-networking protocols. This is how the OHIF viewer fetches images from Orthanc.
- Alongside that, there's an older-style, direct-network protocol (DIMSE) that's still what most physical scanners and hospital PACS systems actually speak — Armorray runs a small built-in listener that speaks this older protocol specifically so hospital equipment can push scans to it directly.
- A **Study** is one complete scan visit; it's made up of one or more **Series** (e.g., "axial brain, contrast" as one series, "sagittal brain" as another), and each Series is made up of many individual image slices.
- **MPR (Multi-Planar Reconstruction)** is the ability to take a 3D stack of CT/MRI slices and re-slice it into different viewing angles (top-down, side-on, front-on) instead of only the angle it was originally scanned in. This is one of the more computationally demanding features of any modern viewer, and — as detailed in Section 7 — a lot of engineering effort went into making it fast.
- **Window/Level (or "windowing")** is adjusting contrast and brightness so that, for example, lung tissue, bone, or brain tissue each show up clearly — the same raw scan can look completely different depending on this setting, and radiologists switch between presets constantly.
- **Secondary Capture** is a DICOM trick used here to let a regular photograph (like an uploaded JPG of an old paper report) be stored and viewed *alongside* real scan images in the same study.
- A **"medico-legal lock"** is the requirement that once a radiologist's report is finalized, it becomes legally read-only — any later correction has to go through a formal, logged amendment process rather than a silent edit. This is standard practice for diagnostic reports and is implemented here.

---

## 7. What's Unclear, Missing, or Contradictory

This is the section to read carefully before presenting anything to the client. Items are ordered roughly by how much it matters, and each is marked with how confident this assessment is.

**High confidence (directly confirmed by reading the actual code):**

1. **A whole second, unused image viewer exists.** Beyond the OHIF viewer that's actually in use, the codebase contains a large, independently-built custom DICOM viewer (using the same underlying imaging engine as OHIF). Seven separate internal documents describe multiple rounds of serious performance engineering on it ("all phases complete," "production ready," 15-30x speed improvements, etc.). In the actual code today, every part of that custom viewer is commented out and unreachable from the live website. None of that documented work is running for real users right now. This isn't necessarily a problem — it may reflect a deliberate later decision to standardize on OHIF instead — but it means a large fraction of the technical documentation in this repository describes work that isn't currently active, and a client should not be told that the "in-house viewer" is a shipped feature.
2. **The project can't be freshly re-downloaded as-is.** The three main folders (`Viewers`, `backend`, `frontend`) are registered in the main repository as Git submodules, but the file that's supposed to tell Git where to fetch them from (`.gitmodules`) doesn't exist and never has. In practice this means a plain `git clone` of this repository by someone else would produce three empty folders. This should be fixed early — it's an easy fix, but it will trip up anyone else who tries to check this project out from scratch.
3. **A safety claim in the architecture document is backwards.** The planning document describes the viewer's role-based tool restrictions as having a "fail-open" safety mechanism. The actual code does the opposite (and the safer thing): if anything goes wrong checking permissions, it **denies** advanced tools by default rather than allowing them. This is good news from a safety standpoint, but the documentation describes the wrong behavior and should be corrected before it's repeated to anyone.
4. **The AI image-quality checker isn't automatic.** Several documents describe AI-assisted quality flags (blurry images, motion artifacts, wrong body part) as something QA sees automatically on every case. The service that does this work is real and functional, but it is currently configured to run only when manually triggered — not automatically on upload.
5. **WhatsApp notifications appear to be unbuilt, at least in one key place.** Nearly every role document treats WhatsApp notifications (e.g., "notify the doctor by WhatsApp after QA sends a scan") as an existing, core feature. The relevant place in the backend code has a plain, literal to-do marker for this — it hasn't been built yet. Other WhatsApp mentions in the documents should be treated as unconfirmed until specifically checked, not assumed to be working.
6. **The "Smart Doctor Matching Engine"** (an algorithm that's supposed to auto-suggest the best available radiologist by specialty and workload) is described in detail in the QA documentation but was not found anywhere in the backend's case-assignment code, which only supports manual assignment today. Treat this as a future idea, not a built feature.
7. **A server-side "MPR rendering service" was planned in detail but never built.** A full architecture document describes a separate Python service that would pre-render 3D image reconstructions on the server to avoid crashing low-powered computers. It was never implemented. The actual fix that shipped for the underlying crash problem was much simpler: a configuration change that tells the image server to convert images to an uncompressed format before sending them to the viewer.
8. **One document ("Doc2.md," Post-Processing Features) reads like a feature sheet copied from a different, more traditional desktop radiology workstation product** — it lists things like automatic vessel analysis, CD/USB burning, and film-printer queue management, none of which exist anywhere in this codebase. This should be treated as unrelated reference material, not a specification for this platform, and probably shouldn't be shown to a client as "our roadmap" without a caveat.
9. **A load-testing script contains a real-looking login email/password and the live production web address, hardcoded directly in a file that's checked into the source code.** This should be treated as a security cleanup item: that account's password should be rotated, and credentials should never be committed to source control going forward.
10. **The image server (Orthanc) currently runs with authentication turned off** in the configuration files present in this repository (the project's own PACS documentation already flags this as a known gap). Anyone with direct network access to that server's port could see DICOM images without logging in. Worth confirming this is locked down before showing this system to a client as production-ready, if it isn't already handled at the network/firewall level.
11. **There's an unrelated file at the project root** (`install.sh`) that is actually a generic installer script for an unrelated coding-assistant tool, not part of this radiology project at all — almost certainly added by accident and safe to delete.
12. **Documentation is heavily duplicated.** The four role guides (Admin, QA, Radiologist, Technician) each exist twice — once as a `.md` file and once as a nearly identical `.docx` file — plus a third, differently-formatted early draft covering the same four roles. They don't meaningfully contradict each other, but keeping three copies of the same requirements in sync is unnecessary overhead going forward.
13. **No automated test suite or deployment pipeline was found for the backend or frontend.** (The OHIF viewer folder does have its own test suite, but that tests the open-source viewer project itself, not this platform's custom code.) Deployment today is described as a manual, step-by-step VPS process (manually restarting the backend, manually rebuilding and copying the frontend).
14. **Assorted small technical debt**, worth knowing about but low urgency: a leftover/disabled real-time connection file in the frontend that would error if anything ever called it; three different data-validation libraries used inconsistently across the backend; and one admin screen that has a hardcoded viewer web address instead of using the shared configuration setting (harmless today, but would silently break if that address ever changes).

**Lower confidence / open questions (not directly verified — worth asking about or checking further):**

- Whether two-factor login (mentioned as "optional" in the Admin documentation) is actually implemented anywhere.
- Whether report delivery by email or SMS/WhatsApp is wired up for any workflow event, beyond in-app/portal delivery.
- Whether the platform holds, or is pursuing, any formal healthcare data-privacy compliance certification (e.g., an equivalent of HIPAA). The documentation talks about audit trails, consent tracking, and a "medico-legal lock" as concepts and those specific things are implemented, but nothing in the repository indicates a formal compliance certification process.

---

## 8. Glossary

| Term | Plain-English meaning |
|---|---|
| **DICOM** | The standard file format for medical images — bundles the picture with structured patient/scan information. |
| **PACS** | "Picture Archiving and Communication System" — the generic term for a medical image server/archive. |
| **Orthanc** | The specific open-source PACS software this platform uses to store and serve images. |
| **OHIF Viewer** | The open-source, browser-based medical image viewer this platform uses for radiologists to actually look at scans. |
| **DICOMweb** | A modern way of fetching medical images over the regular internet/web, instead of older specialized medical networking. |
| **WADO-RS / QIDO-RS** | Two specific types of DICOMweb request — one for retrieving image data, one for searching/querying what studies exist. |
| **DIMSE** | The older, traditional network protocol that most physical scanners and hospital PACS systems use to talk directly to each other. |
| **AE Title** | A short ID name a DICOM device announces itself with on the network — used here to automatically figure out which hospital a scan came from. |
| **C-ECHO / C-FIND / C-MOVE / C-STORE** | The four basic "verbs" of the older DICOM network protocol: ping, search, retrieve, and send. |
| **Modality** | The type of scanning equipment used — CT, MRI, X-ray, ultrasound, etc. |
| **Study / Series / Instance** | The three-level structure of a scan: one Study (a visit) contains one or more Series (a sequence, like "axial brain"), and each Series contains many individual image slices (Instances). |
| **MPR (Multi-Planar Reconstruction)** | Re-slicing a 3D scan to view it from a different angle than it was originally taken. |
| **Window/Level (WW/WL)** | Adjusting contrast/brightness so a scan is readable for a specific tissue type (bone, lung, brain, etc.). |
| **Hounsfield Unit (HU)** | The specific density scale CT scans use, which windowing settings are based on. |
| **Secondary Capture** | A way of storing a regular photo/document inside a DICOM study, alongside real scan images. |
| **RIS** | "Radiology Information System" — the broader category of software this platform belongs to, covering workflow, scheduling, and reporting around imaging (as distinct from just image storage). |
| **TAT (Turnaround Time)** | How long it takes from a scan being uploaded to a finished report being delivered — a key performance metric in this industry. |
| **SLA (Service Level Agreement)** | A promised/expected turnaround time or standard of service. |
| **Medico-legal lock** | Making a finalized report permanently read-only, so it can't be silently altered after the fact. |
| **RBAC (Role-Based Access Control)** | Giving different users different permissions depending on their role (Technician, QA, Radiologist, Admin). |
| **JWT** | "JSON Web Token" — the technology used here to keep a user securely logged in without the server having to remember every session. |
| **Socket.io / WebSocket** | The technology that lets the website push live updates (chat, notifications, status changes) instantly, without refreshing the page. |
| **Zero-footprint viewer** | A medical image viewer that runs entirely in a web browser — no software installation needed on the user's computer. |
| **Git submodule** | A way of embedding one code repository inside another. This project uses three of them, but is currently missing the configuration file that makes them work correctly on a fresh download (see Section 7). |
