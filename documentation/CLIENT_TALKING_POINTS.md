# Client Talking Points — Armorray Platform

**Audience: the client.** Assume non-technical, but invested and paying attention. Tone: honest, calm, in control — not defensive, not apologetic about the past.

---

## 1. The Elevator Pitch

> "Armorray is the digital backbone that gets a scan from the moment it's taken to a signed, ready-to-use report in your hands — as quickly and reliably as possible. Images come in from your scanning equipment automatically, get quality-checked by our team before a radiologist ever sees them, get read and reported on, get double-checked again, and go straight back to you — with every step tracked, timestamped, and accounted for along the way. You always know where a case stands, and once a report is signed, it's locked, so you can trust it as the final word."

---

## 2. What's Working Well Today

These are real, verified, currently-functioning capabilities — described in terms of what they mean for the client, not how they're built.

- **Scans arrive automatically.** If your imaging equipment supports it, studies can flow straight from your scanner into our system with patient details already attached — no manual re-typing, fewer transcription errors.
- **A dedicated quality check happens before a radiologist ever opens the case.** Our QA team verifies patient identity and image completeness up front, catching problems (wrong patient, incomplete study, poor image quality) early instead of after a radiologist has already spent time on it.
- **Radiologists read images in a proven, hospital-grade viewer** — no software installation required, works directly in the browser. It includes all standard tools: zoom, contrast adjustment, measurements, and multi-angle reconstruction (viewing a scan from top-down, side-on, or front-on, not just the original angle it was taken at).
- **Reports are fast to produce and hard to tamper with.** Radiologists use templates, shortcuts, and voice dictation to write reports quickly, export them as PDF or Word, and sign them digitally. Once a report is finalized, it's locked — it cannot be silently edited afterward, which matters for legal and clinical defensibility.
- **You always know where a case stands.** Every case has a live, timestamped status — uploaded, in QA, assigned, in progress, finalized — visible in real time, with built-in chat between your team, our QA team, and the reporting radiologist.
- **Billing is calculated automatically and locked to the finalized report**, so every invoice is always backed by a specific, signed piece of work — no ambiguity about what you're being billed for.
- **If your team's internet connection drops mid-upload, nothing is lost.** The upload is queued locally and finishes automatically once the connection returns — built specifically with lower-connectivity locations in mind.
- **Outside partner organizations can connect securely**, via a controlled, permission-based integration, to exchange cases and reports programmatically when that's part of an agreed workflow.
- **Every action is logged for audit** — who did what, and when — supporting accountability and compliance reviews.

---

## 3. What to Say If Asked About a Specific Feature

Honest, professional one-liners for features that are genuinely unbuilt, manual-only, or still in progress. None of these are lies or spin — they're accurate descriptions of where things stand.

- **"Does it send WhatsApp notifications to the doctor?"**
  → *"Notifications go out in-app and by email today. WhatsApp delivery is on our roadmap but isn't turned on yet."*

- **"Does the system automatically pick the best available radiologist for a case?"**
  → *"Today, our QA team manually assigns each case to the right radiologist based on specialty and availability — that human judgment is actually a strength, not a gap. An automated suggestion tool is something we're evaluating for the future."*

- **"Does the AI check every scan for quality automatically?"**
  → *"We have an AI tool that can flag issues like blurry images or an incomplete series. Today our QA team runs it as needed rather than on every single scan automatically — making it fully automatic is a near-term improvement we're looking at."*

- **"I heard you were building your own custom viewer — is that live?"**
  → *"We evaluated building a fully custom viewer in-house for tighter control. For now we've standardized on [the OHIF viewer], which is a proven, widely-used medical imaging viewer already trusted across the industry, rather than maintaining two separate viewers. That was a deliberate choice to keep things reliable."*

- **"We had a report that some scans were slow to load on older computers — is that fixed?"**
  → *"Yes — we identified the cause and applied a server-side fix so images are delivered in a format that works reliably even on lower-powered computers."*

---

## 4. Anticipated Questions & Suggested Answers

| # | Likely Question | Suggested Answer |
|---|---|---|
| 1 | "Is our patients' data secure? Who can actually see it?" | "Access is role-based — a technician, a QA reviewer, a radiologist, and an administrator each see only what their role needs, and case access is scoped to the people actually assigned to it. Every action is logged for audit." |
| 2 | "What happens if the internet goes down while we're uploading a scan?" | "The upload is queued automatically and picks back up the moment the connection returns — no manual re-upload needed." |
| 3 | "Can we see where a case is in the process at any time?" | "Yes — every case has a live status and full timeline, visible to your team in real time, along with built-in chat with our QA and reporting team." |
| 4 | "Once a report is signed, can it still be changed?" | "No — once finalized, a report is locked. Any correction after that goes through a formal, logged amendment process rather than a silent edit." |
| 5 | "Do we need special equipment to use this, or does it work with what we already have?" | "It's built to work with standard scanning equipment and hospital image systems using industry-standard protocols — in most cases there's nothing new to install on your end." |
| 6 | "Do we have to install anything to view images?" | "No — the image viewer runs entirely in a web browser, nothing to install." |
| 7 | "Can we share a scan with an outside specialist who isn't on the platform?" | "Yes, there's a secure, time-limited link option designed for exactly that — sharing a specific study with someone outside the platform for viewing." |
| 8 | "How is billing calculated, and can it change after the fact?" | "Pricing is set per scan type and locked in automatically the moment the report is finalized, so what you're invoiced always matches a specific, completed piece of work." |

**Flag before the meeting — I genuinely can't answer these confidently from the platform itself, and would rather find out than guess:**
- The exact contracted turnaround-time (SLA) commitment per scan type/urgency level — that's a business/contract term, not something set in the software.
- Our formal regulatory/compliance certification status (e.g., a HIPAA-equivalent certification) — the platform has audit logs and access controls, but that's not the same as a certification, and I don't have confirmation of one on file.
- Any specific uptime/hosting guarantee we've committed to.
- Whether two-factor login is currently required for this client's accounts specifically.

---

## 5. Glossary (client-relevant terms only)

| Term | Plain-English meaning |
|---|---|
| **DICOM** | The standard file format medical scans are stored in — bundles the image with patient/scan details together. |
| **PACS** | The general industry term for a medical image server/archive — where scans are stored and retrieved from. |
| **Zero-footprint viewer** | A scan viewer that runs entirely in a web browser — nothing to install on any computer. |
| **Turnaround Time (TAT)** | How long it takes from a scan arriving to a finished, signed report going back out. |
| **Audit trail** | A complete, timestamped record of who did what and when, for every case. |
| **Role-based access** | Each person only sees and can do what their specific job role requires — nothing more. |
| **Medico-legal lock** | Once a report is signed off, it becomes permanently read-only unless a formal, logged correction process is used. |
| **MPR (multi-angle reconstruction)** | The ability to view a 3D scan from a different angle than it was originally taken — e.g., top-down instead of side-on. |
| **Secure share link** | A time-limited link that lets someone outside the platform view a specific study without a full account. |
