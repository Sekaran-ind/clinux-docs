# Blog Series Guide: "Building Clinical Software That Deserves Trust"

Internal guide for the ClinuxFlow blog series. Not for publication.

## Audience and voice

Readers are people who build or buy clinical software: CTOs and engineering leads at health-tech
companies, clinical informatics teams, founders, and technically minded clinic owners, mostly in
India. They are skeptical of AI and "platform" claims and have read a lot of marketing.

Each post is an engineering essay about a problem every healthcare application has to solve. It
uses ClinuxFlow's real decisions and real mistakes as the worked example, and ends with an honest
account of where ClinuxFlow stands. The series earns trust by being specific and by saying what
isn't built yet.

Rules for every post:
- No claim about ClinuxFlow that the code doesn't support. "Planned" and "designed" are said out
  loud. Check against SPEC-00's status matrix before publishing.
- No vendor names used as borrowed credibility (the earlier drafts leaned on Smile CDR, which
  ClinuxFlow does not use).
- No absolute promises ("impossible to take offline", "zero hallucination", "perfect compliance").
- Regulatory statements are general guidance with the instrument named, not legal advice.

## Reading order

| # | File | Title | Primary reader |
|---|---|---|---|
| 1 | BLOG-01 | FHIR as the Data Model, Not the Export Format | CTOs, informaticists |
| 2 | BLOG-02 | AI in the Clinical Loop: Where Models Belong and Where They Don't | CTOs, clinical leads, AI engineers |
| 3 | BLOG-03 | Local-First Clinical Software: Offline, Sync and Conflict Without Losing Data | Engineers, architects |
| 4 | BLOG-04 | Security and Privacy for Clinical Apps in India, Without the Buzzwords | CISOs, CTOs, DPOs |
| 5 | BLOG-05 | Sovereign Inference at the Clinic Edge: A Realistic Nano Data Centre | CIOs, infrastructure leads |
| 6 | BLOG-06 | Designing a Conversational Workspace for Clinicians: Lessons from Cübo | Product, design, clinical leads |
| 7 | BLOG-07 | Making Local Models Emit Valid Clinical Data: Grammar-Constrained Extraction with MedGemma | AI engineers |
| 8 | BLOG-08 | Building on ABDM: Lessons from Integrating ABHA, HPR and HFR | Engineers at Indian health-tech companies |
| 9 | BLOG-09 | State Machines in Clinical Software: Where They Help and Where They Hurt | Engineers, architects |

## Before publishing any post

1. **BLOG-04 must not be published until the security items in `PENDING-WORK.md` are fixed**
   (cross-clinic id lookups, gateway authentication, session revocation). The post describes
   principles and past, fixed mistakes; publishing it while known gaps remain open would invite
   exactly the scrutiny those gaps can't survive.
2. Re-check each post's "where ClinuxFlow is today" section against SPEC-00.
3. Re-check dated facts (model releases, regulation timelines, library versions).
4. Have a clinician read BLOG-02, BLOG-06 and BLOG-07.

## What changed from the first drafts

The first drafts (2025) described an architecture that was never built: a Smile CDR back end,
JWE payload encryption, ABAC policy enforcement, Patroni failover, a MedCAT voice pipeline and a
"self-evolving" reinforcement-learning loop. They also contained leftover chatbot text ("let me
know if you'd like…"). Every post was rewritten on 2026-09-26 against the actual system.
