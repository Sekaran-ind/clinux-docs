# AI in the Clinical Loop: Where Models Belong and Where They Don't

*Building Clinical Software That Deserves Trust, part 2*

**For:** CTOs, clinical leads and AI engineers adding language models to clinical workflows.

---

Every clinical software company is being asked the same question: where is your AI? The honest
answer for most products should start with a different question: what happens when the model is
wrong, who notices, and who is accountable?

A language model that writes a pleasant discharge summary is impressive. The same model silently
putting "chest pain" into a structured field after the patient said "no chest pain" is a patient
safety incident that looks, in the database, exactly like a correct entry. This post sets out the
rules we use to decide where models are allowed to act in ClinuxFlow, and how we check them. It
also describes, plainly, how little of our AI is sophisticated today, and why that's deliberate.

## A risk ladder for AI in clinical software

Not all AI features carry the same risk. We sort them into five rungs and apply stricter controls
as we climb.

| Rung | What the model does | Example | Main risk | Control |
|---|---|---|---|---|
| 1 | Understands a command | "Switch to cardiology" | Wrong screen | Reversible; just show what happened |
| 2 | Proposes values for structured fields | "BP 130/85" → systolic and diastolic | Plausible wrong value saved as fact | Visible suggestion; human accepts per field |
| 3 | Drafts narrative | A SOAP note draft | Fabricated or omitted findings | Clinician edits and signs; draft is labeled |
| 4 | Supports decisions | "This drug conflicts with a recorded allergy" | False reassurance or alert fatigue | Evidence shown; tuned thresholds; regulatory review |
| 5 | Acts autonomously | Places an order or sends a prescription | Harm with no human in the loop | Not allowed |

Most "AI-powered" products blur rungs 2 and 3 with rung 5 in their marketing. In the product they
must be kept apart.

## Six rules

### 1. Output structure comes from the form, never from the prompt

If a model fills a form, the set of fields it may write, their types and their allowed values
should be generated from the form definition itself. In ClinuxFlow that's the compiled FHIR
`Questionnaire`. A prompt that says "return JSON with these fields" is a request; a schema or
grammar enforced at decode time is a guarantee, at least about shape. (Part 7 of this series goes
deep on grammar-constrained decoding.) Shape is not truth, but getting shape right removes a whole
class of failure: invented fields, wrong types, values outside a code list.

### 2. Suggest, then confirm, and never accept raw text for a coded field

A model's output is a proposal. In the UI it should look like one: highlighted, attributed, and
accepted field by field. For any field with a closed vocabulary (a diagnosis code, a district
code, a drug), the model's text must resolve through the same picker a human would use. It is
never stored as free text that merely looks like a code.

We learned the underlying lesson from a bug that had nothing to do with AI. Our form renderer
discarded a value typed into a coded field unless the user clicked a suggestion. A model writing
into that field would have failed the same way, silently. Coded fields need an explicit
confirmation step whoever is typing.

### 3. Language has negation, uncertainty, time and attribution

"Denies chest pain." "Rule out pneumonia." "Mother had diabetes." "Chest pain last year, none now."
Each contains a clinical term that a keyword matcher, and a careless model, will turn into a
positive finding about the patient, now. Clinical NLP has decades of work on this (the NegEx and
ConText algorithms, and tools such as medspaCy built on them). Any slot-filling pipeline needs an
explicit answer to four questions for every extracted concept: is it asserted or negated? certain
or hypothetical? current or historical? about the patient or someone else? If your pipeline
can't say, it shouldn't write to the problem list.

### 4. Record provenance for everything a model touches

For every value a model proposed, you should be able to answer: which model and version, which
prompt template, what input, who accepted it, and when. FHIR has a resource for this
(`Provenance`), and it deserves to be used. Without provenance you can't investigate an incident,
can't measure a model after an upgrade, and can't answer a regulator. ClinuxFlow does not yet
record model provenance, and that is a gap we're closing before AI output reaches clinical
records beyond drafts.

### 5. Evaluate like a clinical instrument, not a demo

Accuracy on a slide means little. What matters:
- **Per-field precision and recall**, weighted by severity. A wrong allergy is worse than a
  misspelled city.
- **Out-of-scope behavior.** When we tested our own intent classifier, which was trained on only
  two commands, the sentence "what is the weather" was classified as "generate the SOAP note" with
  full confidence. A classifier with no concept of "none of the above" will always confidently pick
  something. Test with irrelevant input, not just relevant input.
- **Behavior under paraphrase and code-switching.** Indian clinical speech mixes English with
  Hindi, Tamil and other languages mid-sentence. An evaluation set in textbook English measures the
  wrong thing.
- **Regression on every model or prompt change**, against a fixed, consented, de-identified test
  set.
- **Acceptance and correction rates in production**: how often clinicians accept, edit or reject
  suggestions, per field. A field that is edited 40% of the time is telling you something.

### 6. Know where the text goes

Clinical conversation is sensitive personal data. Sending it to a third-party model API is a data
transfer that needs a lawful basis, a stated purpose, a retention answer and a contract. India's
Digital Personal Data Protection Act, 2023, and its Rules (notified in November 2025, with
obligations phasing in over the following months) make purpose limitation and notice concrete
obligations. Some deployments will want inference to stay on premises entirely; part 5 of this
series looks at what that takes.

## The regulatory frame, briefly

This is not legal advice, but every team should know the landscape:
- **Medical device regulation.** Under India's Medical Devices Rules, 2017, software intended for
  diagnosis or treatment decisions can itself be a regulated medical device. Rung 4 features should
  be designed with that in mind from day one, not retrofitted.
- **Telemedicine Practice Guidelines (2020)** allow practitioners to use AI tools to assist them,
  but state that such tools cannot themselves counsel patients or prescribe.
- **ICMR's Ethical Guidelines for the Application of AI in Biomedical Research and Healthcare
  (2023)** set expectations on accountability, transparency, data provenance and bias that map
  directly onto rules 4 and 5 above.
- **Consent for ambient capture.** Recording a consultation to generate notes needs the patient's
  informed consent, and the clinician's too.

## What ClinuxFlow's AI actually is today

We would rather under-claim. As of this writing:

| Capability | Implementation | Rung |
|---|---|---|
| Command recognition | A small intent classifier (`node-nlp`) trained on two commands, with a confidence threshold | 1 |
| Field suggestions from chat | Keyword and entity matching against labels harvested from our forms, shown as highlights the user accepts | 2 |
| Deterministic shortcuts | `/field value` commands for exact entry | 1 |
| SOAP drafting | One call to an open-weight model (Llama 3.3 70B on Cloudflare Workers AI), paid tier, draft only | 3 |
| Vocabulary enrichment | Wikidata synonyms for form fields, looked up at design time and confirmed by a human | Design-time |

There is no voice pipeline, no clinical NLP with negation handling, no decision support and no
autonomous action. Earlier drafts of this blog described a voice-to-FHIR pipeline built on MedCAT
and medspaCy. We designed it; we have not built it.

That modesty is partly timing and partly principle. The rules above are expensive to meet, and we
would rather ship a narrow feature that meets them than a broad one that doesn't. Two directions
are next:
- **Grammar-constrained local extraction** (part 7): output structure generated from our
  Questionnaires, enforced at decode time, on models we host.
- **Terminology grounding**: role-scoped SNOMED CT subsets (licensed in India through NRCeS) so
  that suggestions resolve to real concepts, not strings.

## A checklist before you ship an AI feature in a clinical product

1. Which rung is it on, and who agreed?
2. Is output structure generated from the form or schema, and enforced?
3. Do coded fields require explicit confirmation?
4. How are negation, uncertainty, time and attribution handled?
5. What provenance is recorded, and where?
6. What is the evaluation set, and does it include irrelevant and code-switched input?
7. Where does the patient's text go, under what lawful basis, and for how long?
8. What is the regulatory position of this feature, in writing?
9. How will a clinician report a bad suggestion, and who reads those reports?

If any answer is "we'll figure it out after launch", the feature isn't ready.
