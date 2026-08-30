# Specification 10: Capability Assessment — e-Sushrut@Clinic vs. ClinuxFlow

## 1. Objective

Reference site analyzed: `https://muhseclinic.uat.dcservices.in/` — a UAT deployment of **e-Sushrut@Clinic**, run by a state health university ("MUHS" in the hostname) on infrastructure operated by C-DAC's `dcservices.in`. This is the direct government-backed competitor for ClinuxFlow's target segment: small clinics, single-doctor practices, and small nursing homes. The strategic framing matters — e-Sushrut@Clinic is free/subsidized (a National Health Authority + C-DAC initiative to drive ABDM adoption), so ClinuxFlow, as a private entity charging for the same category of service, cannot compete on price alone. This spec exists to make that competitive reality explicit and actionable: where ClinuxFlow must reach parity to be credible, where it already leads, and where the honest answer is "not worth matching."

This is a **reference document, not a build plan**. No roadmap decisions are made here — capabilities are assessed and gaps are surfaced; prioritization and what (if anything) gets built is a separate decision the user makes after reviewing this.

## 2. Reference system profile — e-Sushrut@Clinic

Sourced from the live UAT site plus C-DAC/NHA's own public materials (the UAT login page itself only exposes a pre-auth landing screen, so the module-level detail below is corroborated against C-DAC's public launch coverage, not just the login page's marketing copy).

**Identity**: C-DAC (Centre for Development of Advanced Computing) — a Government of India institution — built e-Sushrut@Clinic as a lightweight SaaS derivative of their flagship **e-Sushrut** HMIS, which is already deployed across 17 AIIMS campuses and 4,000+ health facilities nationwide. Launched under NHA/ABDM sponsorship specifically to onboard small providers who couldn't otherwise afford or justify a full HMIS.

**Target users**: General practitioners, specialists, clinic administrators, pharmacists, lab technicians, and allied health workers at small/medium clinics, PHCs, sub-centres, health & wellness centres, and private OPD facilities.

**Modules and capabilities, as documented:**

| Area | What e-Sushrut@Clinic provides |
|---|---|
| Registration/Identity | Own-portal account creation; HPR ID generation/validation; HFR ID generation/validation; ABHA creation and search |
| OPD / Clinical documentation | Patient registration, structured "Doctor Desk" consultation documentation, prescriptions, digitized patient records |
| IPD | Basic inpatient tracking ("day-to-day activities related to an admitted patient") — present even in the lite variant |
| Pharmacy | A pharmacy module (dispensing workflow; the full e-Sushrut's pharmacy module includes stock/inventory, unconfirmed whether the Clinic-lite variant does) |
| Nursing | A dedicated nursing module |
| Lab / Radiology | "Integrated with pharmacy, lab, and radiology" — order/result integration, not necessarily an in-app viewer |
| Physical workflow | Barcode generation for patient registration and lab samples — scan-based lookup for revisits, billing, pharmacy, IPD |
| CDSS | AIIMS Clinical Decision Support System integration for hypertension and diabetes management, at no extra cost |
| Speech-to-text | Built in, for consultation documentation |
| Telemedicine | Telemedicine service delivery is explicitly listed |
| ABDM integration | ABHA creation/search, Scan-and-Share, HFR/HPR onboarding, ABDM M1–M4 milestone compliance |
| Standards | HL7 and MDDS (Metadata and Data Standards) compliance |
| Reporting | Dashboards, real-time reporting, MIS reporting explicitly framed for "audits and public health planning" |
| Communication | SMS and email integration |
| Connectivity | Both online and offline operation (mechanism undocumented publicly) |
| Access | Web app, multi-device (laptop/mobile) |
| Authentication | HPR ID login, username/password, mobile OTP, CAPTCHA |
| Accessibility | Text-to-speech, dyslexia-friendly mode, contrast adjustment |
| Support | 24/7 dedicated support (government-staffed) |
| Cost | ₹499/month for up to 5 users; ABDM-subsidized to ₹299/month (NHA covers ₹200); first 3 months free |

**Full e-Sushrut** (the hospital-grade parent product, referenced here only as the ceiling of what C-DAC's platform family can eventually reach): ~20 modules including OPD, IPD, OT (Operation Theatre scheduling/equipment tracking), Labs, Blood Bank, Pharmacy, Referrals, Medical Examinations, Reimbursement of Medical Claims, integrated OPD/IPD/ancillary billing.

**Sources**: [NHA & C-DAC Partner to Roll Out e-Sushrut@Clinic](https://www.digitalhealthnews.com/new-e-sushrut-clinic-to-drive-abdm-adoption-across-healthcare), [e-Sushrut@Clinic — Vajiram & Ravi Current Affairs](https://vajiramandravi.com/current-affairs/e-sushrutclinic/), [e-Shushrut HMIS — e-Gov AppStore](https://apps.nic.in/apps/government/e-shushrut-hospital-management-information-system), live fetch of `muhseclinic.uat.dcservices.in`.

## 3. Capability matrix

Legend: ✅ built & live-verified this session or earlier · 🟡 partial/adjacent capability exists · ❌ not built.

| Capability | e-Sushrut@Clinic | ClinuxFlow (current) | Assessment |
|---|---|---|---|
| OPD registration + consultation documentation | ✅ | ✅ (Front Desk → Consultation Desk, merged Encounter composition) | Parity |
| Vitals capture | ✅ | ✅ (repeating-group vitals in the Encounter form) | Parity |
| SOAP-style clinical notes | ✅ (Doctor Desk module) | ✅ | Parity |
| Prescriptions | ✅ | ✅ (structured MedicationRequest, PDF generation) | Parity |
| Billing / invoicing | ✅ (integrated OPD/IPD/ancillary) | ✅ (Checkout's Billing step, AG Grid payments table) | Parity |
| **IPD / ward / bed / admission tracking** | ✅ (present even in the lite variant) | ❌ | **Gap** — ClinuxFlow is OPD-only; no admission, bed, or ward concept exists anywhere in the data model |
| **Pharmacy stock/inventory management** | 🟡 (module exists; inventory depth in the lite variant unconfirmed) | ❌ | **Gap** — ClinuxFlow's "prescription" is a medication order with no linked stock, reorder, or expiry tracking |
| **Lab order management** (order → sample → result workflow) | ✅ | ❌ | **Gap** — no lab-order entity; results, if any, arrive only as generic document attachments |
| **Barcode-based physical workflow** (patient ID bands, lab sample tags) | ✅ | ❌ | **Gap** — no barcode generation/scanning anywhere |
| Radiology / medical imaging viewing | 🟡 (integration, not confirmed as an in-app viewer) | ✅ (`CornerstoneViewer.vue`, DICOM-capable) | **ClinuxFlow ahead** — an actual in-app DICOM viewer is a materially more advanced capability than a referral-style lab/radiology "integration" |
| Telemedicine / video consultation | ✅ | ✅ (RealtimeKit-based `VideoCallPanel.vue`) | Parity |
| CDSS (clinical decision support) | ✅ (AIIMS CDSS for hypertension/diabetes, free) | 🟡 (Cübo's NLP/keyword slot-filling exists; no clinical-guideline safety-check layer yet — SPEC-07 Part C's MedGemma/MedSAM tier is planned, not built) | **Gap today, roadmap exists** — e-Sushrut's CDSS is live and free; ClinuxFlow's equivalent (SPEC-06/07) is specified but unbuilt |
| Speech-to-text for documentation | ✅ | 🟡 (`test-scribe` Workers AI endpoint exists per earlier session work; not confirmed wired into the live consultation flow this session) | Needs verification, likely a small gap |
| ABHA creation/search | ✅ | 🟡 (`clinuxflow-abdm-gateway`'s `abha.js` route exists; full UI-driven citizen ABHA creation/search flow not confirmed live-verified) | Partial — backend plumbing exists, UI completeness unconfirmed |
| HFR/HPR registration | ✅ | ✅ (SPEC-09's canonical schema + dedicated Staff registration flow, live-verified this session) | Parity, arguably ahead on UX (plain-language fields + inline validation vs. a generic government form) |
| ABDM Scan-and-Share / consent-based record exchange | ✅ | 🟡 (ClinuxFlow's own P2P chat-based transfer + DigiLocker PDF/QR export cover an adjacent need, but not the actual ABDM Health Information Exchange consent flow) | **Gap** — real HIE participation (acting as a full HIP within ABDM's consent-manager flow) is not built; SPEC-09 §5 explicitly defers this to Enterprise tier |
| DigiLocker integration | Not explicitly documented | ✅ (PDF+QR export, citizen self-stores via DigiLocker's own Scan/Upload — built and live-verified this session) | **ClinuxFlow ahead**, if e-Sushrut@Clinic indeed lacks this |
| MIS / audit reporting dashboards | ✅ (explicitly framed for public-health reporting) | ❌ | **Gap** — no analytics/reporting dashboard exists in ClinuxFlow at all yet |
| SMS/email notifications | ✅ | ❌ | **Gap** |
| Multi-user / role-based staff accounts | ✅ | ✅ (multi-account-per-clinic, Staff registry, worklist assignment/locking) | Parity |
| Real-time chat (staff-to-staff) | Not documented | ✅ (P2P WebRTC chat, live-verified) | **ClinuxFlow ahead** |
| Offline operation | ✅ (claimed, mechanism undocumented) | ✅ (genuinely offline-first: local TanStack DB + Tauri LAN shared-server mode, not just "works when the network briefly drops") | Likely **ClinuxFlow ahead** on depth, though e-Sushrut's actual mechanism is unverified so this is not a confirmed comparison |
| Accessibility (text-to-speech, dyslexia mode, contrast) | ✅ | ❌ | **Gap** |
| Authentication (OTP, CAPTCHA, HPR-ID login) | ✅ | 🟡 (email/password auth exists; OTP/CAPTCHA/HPR-ID-as-login not built) | **Gap** |
| FHIR compliance | Not primary claim (HL7/MDDS instead) | ✅ (explicitly FHIR R4-native architecture, reference-only data model per SPEC-05 §4) | **ClinuxFlow ahead** on standards modernity — FHIR R4 is the current international standard; HL7v2/MDDS is the older, India-specific baseline e-Sushrut targets |
| Multi-tier pricing / deployment model (local/cloud/on-prem AI) | N/A (single free government product) | ✅ (SPEC-05's 3-tier model; SPEC-07 Part C's nano-DC/on-prem AI tier) | **ClinuxFlow-specific differentiator** — not a like-for-like comparison, since e-Sushrut has no such tiering, but relevant to ClinuxFlow's business model |
| 24/7 dedicated support | ✅ (government-staffed) | ❌ | **Gap** — a resourcing/business decision, not a technical one |

## 4. Gap summary, by severity

**Adoption-blocking gaps** (a clinic evaluating both products would likely notice these immediately):
1. **No IPD/admission tracking at all.** Even a single-room nursing home doing occasional admissions has no path in ClinuxFlow today. This is the largest structural gap — it's a missing data model (Encounter needs an admission/discharge/ward concept), not a UI tweak.
2. **No lab-order workflow.** Order → sample → result is a core clinic loop e-Sushrut covers and ClinuxFlow doesn't model at all.
3. **No reporting/MIS dashboard.** Clinics increasingly need this for their own operations, and it's explicitly what public-sector buyers (and increasingly private ones) expect to see demonstrated.
4. **No SMS/email notifications** (appointment reminders, results-ready alerts) — a basic patient-facing expectation.

**Credibility gaps** (matter for compliance conversations and larger accounts, less for a first demo):
5. Pharmacy inventory/stock tracking.
6. Barcode-based physical workflows (patient wristbands, sample tube labels) — matters more for nursing homes/labs than solo-GP clinics.
7. OTP/CAPTCHA/HPR-ID login options — e-Sushrut's auth story is more government-compliance-flavored; ClinuxFlow's is simpler but less aligned with what buyers evaluating against a government benchmark will expect to see.
8. Accessibility features — increasingly a checkbox item in government/institutional procurement even for private vendors.

**Lower urgency / needs verification, not confirmed gaps**:
9. Speech-to-text in the live consultation flow (backend endpoint reportedly exists; not confirmed wired up and verified this session).
10. Full ABHA creation/search UI completeness (gateway routes exist; UI-driven end-to-end flow not independently re-verified this session).
11. Real ABDM Health Information Exchange participation (already an explicitly deferred Enterprise-tier item per SPEC-09 §5 — not a surprise gap, just worth naming here for completeness).

## 5. Where ClinuxFlow already leads

- **In-app DICOM/medical imaging viewer** (`CornerstoneViewer.vue`) — e-Sushrut's public materials describe lab/radiology "integration," not an in-app viewer; if accurate, this is a real, demonstrable ClinuxFlow advantage.
- **DigiLocker PDF+QR citizen export** — not documented as an e-Sushrut@Clinic feature.
- **True peer-to-peer staff chat** (WebRTC, nothing stored server-side) — not documented for e-Sushrut@Clinic.
- **Genuinely offline-first architecture** (local-first TanStack DB collections + a real LAN shared-server mode for connectivity-poor sites) vs. e-Sushrut's undocumented "supports offline" claim.
- **FHIR R4-native data model** vs. e-Sushrut's HL7/MDDS baseline — FHIR is the modernization direction India's own ABDM ecosystem is moving toward; being FHIR-native from the start is a forward-looking advantage, not just a technical curiosity.
- **Tiered pricing with an on-prem/nano-DC AI option** (SPEC-07 Part C) — no equivalent exists for a free government product, but this is ClinuxFlow's own differentiation lever for larger/security-conscious private accounts, not a response to something e-Sushrut offers.
- **UX quality on the ABDM registration flow itself** — SPEC-09's plain-language, inline-validated Staff/Hospital registration (built and live-verified this session) is a genuinely more polished experience than a typical government-portal registration form, though this is a subjective/qualitative claim, not something independently benchmarked against the live e-Sushrut UI (which wasn't accessible without credentials).

## 6. Codebase and licensing

Checked explicitly (the user asked directly): **no public source code repository exists for e-Sushrut or e-Sushrut@Clinic.** A direct GitHub search turns up nothing from C-DAC or NHA — only unrelated third-party/student projects that happen to share the name (e.g. a UP-HMIS-themed student repo, an unrelated "SushrutHealth" event-client project). C-DAC's own site (`cdac.in`) documents the product's existence and rollout milestones but never publishes source, a repository link, or an open-source license — every reference found treats it as C-DAC's own proprietary, internally-maintained codebase.

**Licensing/delivery model, not open source**: e-Sushrut is distributed as a **SaaS subscription**, not licensed/sold as installable or forkable software:
- **e-Sushrut@Clinic** (the small-clinic variant this spec compares against): ₹499/month for up to 5 users, subsidized to an effective ₹299/month under the NHA–C-DAC partnership (NHA covers ₹200/month), with the first 3 months free. NHA separately bears cloud-hosting and patient SMS-notification costs; C-DAC retains responsibility for software maintenance and upgrades — i.e., C-DAC owns and operates the code, clinics pay for access to the running service.
- **Full e-Sushrut** (hospital-grade, 80+ government hospitals / 17 AIIMS): offered in both "stand-alone" and SaaS deployment modes per C-DAC's own product listing, but no public commercial pricing exists for this tier — these are government-to-government deployments, not open commercial licensing.

Net for competitive positioning: e-Sushrut is not a code base ClinuxFlow could inspect, fork, or benchmark against source-for-source — only its live product surface and public claims are assessable, which is exactly the constraint this whole spec already works under (§2's sourcing note).

## 7. Open items / not assessed

- Could not access any authenticated e-Sushrut@Clinic screens (OPD/IPD/pharmacy workflows themselves) — this whole assessment is built from public marketing/launch coverage plus the pre-auth landing page, not a hands-on comparison of the actual working software. Treat module-level claims as "documented," not "independently verified."
- No visibility into e-Sushrut@Clinic's actual FHIR/HL7 interoperability depth, its real barcode hardware requirements, or how its "offline" mode actually behaves — all listed as capabilities here on the strength of public claims only.

## 8. Next step

Awaiting direction — no roadmap or build decisions made in this spec.
