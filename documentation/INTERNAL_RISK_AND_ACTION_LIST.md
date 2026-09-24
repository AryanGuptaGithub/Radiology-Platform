# Internal Risk & Action List — Armorray Platform

**Audience: internal team / leadership only. Do not share with the client.**
**Prepared:** 2026-09-24, from a full documentation review + verified code audit (backend, frontend, OHIF Viewers fork).

This is a prioritized punch list, not a narrative. Effort estimates are rough (trivial / small / medium / large) based on what's visible in the code — treat them as a starting point for scoping, not a quote.

---

## 🔴 Critical — do before any client/auditor conversation, ideally this week

| # | Issue | Why it matters | Effort |
|---|---|---|---|
| C1 | **A live-looking login email and password, plus the production URL, are hardcoded in `productiontesting/viewer-load-test.js`**, checked into source control. | If this repo is ever shared, cloned by a contractor, or breached, that's a working credential to a real account sitting in plain text. This is the single most embarrassing thing an outside auditor or new hire could stumble on first. | Trivial — rotate that account's password immediately, delete the hardcoded values from the file, load them from an environment variable instead. |
| C2 | **Orthanc (the image/PACS server) runs with `AuthenticationEnabled: false`** in the repo's config files (`orthanc.json`, `docker-compose.yml`), with `RemoteAccessAllowed: true`. | Anyone with direct network access to that server's port can pull patient DICOM images with no login at all. This is a real PHI exposure risk if it isn't fully blocked at the firewall/network layer in production — and we should not assume it is, since it's committed as the default. | Small–Medium: turn on Orthanc auth with real credentials, confirm the Nginx proxy config still injects the right auth header (it already does per the deployment doc), and verify port 8042/4242 aren't reachable from outside the server. |
| C3 | **The internal endpoint the AI service calls back to (`/api/internal/integrity/results`) has no shared secret or auth check** — it trusts any request that hits it. | Anyone who can reach this endpoint on the network could inject fake "image quality" results into a real case record. Low likelihood if network is locked down, but it's an easy, cheap fix for a real gap. | Small — add a shared-secret header check between the Node backend and the Python AI service. |
| C4 | **The real-time chat "room" authorization check is a stub that always returns `true`** (`utils/socketAuth.js`, `authorizeRoomAccess()`). | Any logged-in user — technician, QA, radiologist, regardless of hospital or assignment — could theoretically join and read/write the live chat for a case they have no business seeing, even though the equivalent REST API endpoints are properly locked down. This is a real per-patient privacy gap, not theoretical. | Small–Medium — wire this up to the same case-assignment check already used elsewhere in the codebase. |
| C5 | **Share-link tool restrictions have a legacy loophole**: in the OHIF viewer's share-link code path, an empty "allowed tools" list for a non-`user` role is treated as *all tools allowed* ("for backward compatibility"), rather than none. | We tell technicians (and implicitly clients) that shared image links are **view-only** with no editing/export. If a share link is ever generated with a role that hits this code path, a recipient outside our organization could get full measurement/annotation tools instead of the view-only experience we promise. This directly touches a claim we make to clients. | Small — close the loophole so an empty/missing tool list always means "view only," not "everything." |

---

## 🟠 High — fix soon (blocks clean onboarding or risks real confusion)

| # | Issue | Why it matters | Effort |
|---|---|---|---|
| H1 | **`Viewers`, `backend`, and `frontend` are registered as Git submodules with no `.gitmodules` file anywhere in history.** | A fresh `git clone` of this repo gives three empty folders. The very first thing a new developer (or a new machine/CI runner) does will fail. | Trivial–Small — add a `.gitmodules` file pointing each path at its actual remote, matching the currently-pinned commits. |
| H2 | **Seven internal documents (`MPR_*.md`, `DICOM_PHASED_LOADING_PLAN.md`, `Dicom memory optimization guide.md`, `PERFORMANCE_OPTIMIZATIONS.md`, `Plan for dicom .md`) describe a custom in-house DICOM viewer as actively developed and "production ready," but every line of that viewer's code is commented out and unreachable.** | Any new developer reading these docs (or `CALUDE MD.md`, which also describes it as "in active development") will waste real time trying to find, run, or extend a feature that isn't live, or will "fix" dead code thinking it matters. | Trivial to contain — add a one-line "SUPERSEDED / NOT IN USE" banner at the top of each of those docs. Small–Large if we also want to physically delete the ~2,200 lines of dead viewer components plus the ~25 orphaned supporting service files and the unused `dicomStore.ts`. |
| H3 | **The OHIF Viewers submodule currently has an uncommitted local change pointing its backend URL at `localhost:5000`** (dev mode), with the real production URL commented out in `extensions/default/src/services/BackendService.ts`. | If someone builds/deploys the Viewers image from this exact working copy without noticing, the viewer would silently fail to reach the real API in production. | Trivial — revert or confirm intent before the next Viewers build. |
| H4 | **No automated test suite or CI/CD pipeline exists for `backend` or `frontend`** (the `Viewers` submodule's CI only tests the upstream open-source OHIF project, not our customizations). Deployment is a fully manual VPS process (manual `pm2 restart`, manual `npm run build` + file copy). | Every release currently depends on a human doing the right manual steps in the right order, with no regression safety net. This is a scaling risk the moment more than one person touches the code. | Large — this is a real project, not a quick fix; worth roadmap discussion. |
| H5 | **Heavy documentation duplication**: the four role guides (Admin, QA, Radiologist, Technician) each exist as a `.md` file, a near-identical `.docx` file, *and* a third early-draft version covering all four roles in `Doc1.md`. | Three copies of the same requirements will drift out of sync over time, and it's already unclear which is "current." | Small — pick one canonical version per role, archive/delete the rest. |
| H6 | **`Doc2.md` ("Post Processing Features") lists functionality that doesn't exist anywhere in this codebase** (automatic vessel analysis, CD/USB export, film-printer queue management) — it reads like a copied feature sheet from an unrelated desktop PACS product. | If anyone (including us, under time pressure) mistakes this for our actual roadmap or shows it to a client as "coming soon," we'd be promising things we've never built and have no code basis for. | Trivial — relabel it clearly as reference/inspiration material, or remove it. |

---

## 🟡 Medium / Low — worth knowing, not urgent

| # | Issue | Why it matters | Effort |
|---|---|---|---|
| M1 | `Site.scpAETitle` (the field used to route an incoming scan to the right technician) has no unique-index constraint. Two technicians registering the same hospital AE title would cause unpredictable routing. Already self-flagged in `PACS-Documentation.md`. | Real but low-likelihood operational bug; would show up as "wrong technician got a scan." | Trivial — add a unique index; check for existing duplicates first. |
| M2 | Three different data-validation libraries (`joi`, `zod`, `express-validator`) and two separate Socket.io Redis adapter packages are installed and used inconsistently across the backend. | No functional bug today, but it raises onboarding confusion and maintenance cost as more people touch the code. | Medium — standardize over time, not a sprint item. |
| M3 | A leftover, deprecated real-time service file (`frontend/src/services/socketService.ts`) remains in the tree and would throw an error if anything ever called it — nothing currently does. | Pure dead-code cleanup; no runtime risk today. | Trivial. |
| M4 | One admin screen (`OrthancManager.tsx`) hardcodes `localhost:3000/viewer` instead of using the shared `VITE_OHIF_URL` config and the `/basic` path used everywhere else. | Harmless today; would silently break if the OHIF address ever changes in production. | Trivial. |
| M5 | `backend/ai-service/main.py` has a hardcoded fallback file path from a previous developer's own machine (`c:/Users/admin/Desktop/Varun/...`) as the default upload directory if the environment variable isn't set. | Would silently point at a nonexistent path in a new environment rather than failing loudly, if someone forgets to set the env var. | Trivial — remove the hardcoded fallback, fail loudly instead. |
| M6 | `install.sh` at the repo root is an unrelated installer script for a different coding-assistant CLI tool, not part of this project. Root-level `crash.log` and `dicom.log` (outside `backend/logs/`) also appear to be dead, unused log targets. | Pure clutter, no functional impact. | Trivial — delete. |
| M7 | Placeholder radiologist "specialties"/"modalities" values are hardcoded in `authControllers.js` rather than derived from real profile data. | Cosmetic/data-quality issue — not a security or correctness bug. | Small. |
| M8 | `dicom-dimse-native` is an installed dependency whose actual runtime usage (vs. `dcmjs-dimse`, which is confirmed in active use) wasn't conclusively traced. | Possibly dead weight; worth a five-minute grep before assuming it's needed. | Trivial to check. |

---

## Claims Made in Old Docs That Are NOT True Today

Use this table as the "do not repeat this in a client conversation" reference.

| Claim (and where it's made) | Actual state today | Where verified |
|---|---|---|
| A custom in-house Cornerstone3D DICOM viewer is being actively built/completed, described across 7 docs as "all phases complete" / "production ready." | 100% commented-out, unreachable dead code. The platform only uses the external OHIF viewer. | `frontend/src/components/viewer/DicomViewer.tsx`, `DicomViewerLayout.tsx`, `ViewerRoute.tsx`, `MPRViewport.tsx`, `MPRViewportOverlay.tsx` — every top-level line commented out. |
| Viewer role-tool restrictions have a "fail-open safety mechanism to prevent configuration tampering" (`ProjectFlow.md`). | The opposite: both the viewer and the backend deny-by-default (fail-closed) if a role's permissions can't be loaded or aren't configured. (The one real exception is the narrow share-link loophole — see Critical item C5 above, which is a bug, not the documented "safety mechanism.") | `Viewers/modes/basic/src/index.tsx` (`SAFE_DEFAULT_TOOLS` fallback); `backend/middleware/viewerToolMiddleware.js` ("secure by default" — denies if no restriction record exists). |
| AI-assisted image quality flags (blur, motion, wrong body part) run automatically and are shown to QA before every approval (implied as live in `system_architecture.md` / `QA.MD`). | The AI service exists and works, but `AI_CONFIG.AUTO_RUN` is hardcoded `false` — it only runs when explicitly triggered, not automatically on upload. | `backend/services/StudyIntegrityService.js`. |
| WhatsApp notification to the assigned doctor "after sending scan" is described as a required, built feature in nearly every role document. | Unbuilt — a literal `// TODO: Trigger Notification Service (WhatsApp/In-App)` sits in the relevant controller. | `backend/controllers/qaController.js` (~line 177). |
| A "Smart Doctor Matching Engine" auto-suggests the best available radiologist by specialty, workload, and sub-specialty (`QA.MD`). | Not implemented. QA assignment in the code is manual/dispatch-only — no matching algorithm exists. | `backend/controllers/qaController.js`. |
| A detailed server-side MPR (3D reconstruction) rendering microservice architecture is specified (`server-side-mpr-rendering-phase-2.md`). | Never built — no trace of it anywhere in the codebase. The actual fix for the crash problem it targeted was a much simpler Orthanc configuration flag that converts images to an uncompressed format before sending them to the viewer. | Repo-wide search for `mpr_service`/`mpr-server`/`/mpr/frame` — no matches; fix confirmed in `armorray-production.js` / `orthanc-local.js` (`requestTransferSyntaxUID`). |
| Automatic vessel analysis, CD/USB export, and film-printer queue management are listed as standard features (`Doc2.md`). | Nothing matching this exists in the codebase — appears to be copied from an unrelated desktop PACS product's feature sheet. | No matching code found anywhere in `backend` or `frontend`. |

**Unverified — do not confirm or deny these if asked; check first:**
- Two-factor authentication for any role (mentioned as "optional" in the Admin doc; no 2FA fields found in the `User` model, but this wasn't exhaustively ruled out).
- Any report delivery channel beyond in-app/portal (email specifically wasn't traced end-to-end; WhatsApp beyond the one confirmed-unbuilt case wasn't individually checked).
- Formal regulatory/compliance certification status (HIPAA-equivalent or local equivalent) — the code implements audit logs, RBAC, and a medico-legal lock, but none of that amounts to a certification, and no certification process was found referenced anywhere.
