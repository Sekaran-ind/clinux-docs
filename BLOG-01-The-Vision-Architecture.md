# FHIR as the Data Model, Not the Export Format

*Building Clinical Software That Deserves Trust, part 1*

**For:** CTOs, architects and clinical informatics leads deciding how their product should hold
clinical data.

---

Most clinical software treats interoperability as a translation problem. Data lives in whatever
tables the first engineer designed, and an export layer maps those tables to FHIR when a
regulator, a partner or a national network asks for it. It works until the day the export layer
has to answer questions its source tables were never designed for: which of these three phone
numbers is the mobile, which practitioner signed this note, whether this address belongs to the
branch or to the parent organization.

ClinuxFlow took the other road. FHIR R4 is the data model from the first form definition onward,
not a format we convert into at the edge. This post explains what that means in practice, what
it costs, and the mistakes we made along the way. The mistakes are the useful part.

## Why this matters more in India than most places

India's national digital health programme, the Ayushman Bharat Digital Mission (ABDM), is built
on FHIR. Health records exchanged between facilities travel as FHIR bundles conforming to profiles
published by NRCeS, the national standards body. Facility and professional registries (HFR and
HPR) and the patient identifier (ABHA) all assume your system can speak those structures fluently.

For a small clinic this is not an abstract concern. A clinic that registers with HFR takes on a
standing obligation to answer future requests for its patients' records. If the clinic's software
stores data in a shape that has to be reverse-engineered into FHIR every time, each request is a
fresh chance to lose meaning. If the software stores FHIR-shaped data from the start, answering
is a lookup.

## What "FHIR-native" actually means here

It is easy to say "FHIR-native" and mean "we have a FHIR API". We mean four concrete things.

### 1. Forms are compiled from FHIR paths

Every field in every ClinuxFlow form names the FHIR element it populates. A clinic's form author
writes something like:

```yaml
- id: staff_email
  path: Practitioner.telecom.value
  label: Email
  uiComponent: TextInput
```

That YAML is compiled into a standard FHIR `Questionnaire`. The compiler refuses any path that is
not a real R4 element. We don't maintain the list of valid paths by hand: a build step walks the
FHIR R4 TypeScript type definitions and generates a path graph for each of 31 resource types, so
the compiler cannot drift from the specification.

The practical effect is that a typo like `Practitioner.telcom.value` fails at authoring time, with
the path named, instead of producing a form that silently stores data nowhere.

### 2. Capture produces standard QuestionnaireResponses

Whether a form is rendered by LHC-Forms (the National Library of Medicine's open-source form
renderer, which we use for clinical rooms) or by one of our hand-written registration screens,
what gets saved is a standard `QuestionnaireResponse`. That matters for two reasons. The captured
record is portable and inspectable by any FHIR tool. And it separates "what the user answered"
from "what those answers mean as clinical resources", which turns out to be a very useful seam.

### 3. Resources are extracted, not hand-mapped

FHIR's Structured Data Capture (SDC) specification defines how a completed questionnaire turns
into real resources: a Patient, an Observation, a Practitioner and their role at an Organization.
We implement definition-based extraction: each item carries a `definition` pointing at its FHIR
element, and the extractor walks the response and assembles linked resources, including the
references between them.

### 4. Conformance is checked against real profiles

A profile (a FHIR `StructureDefinition`) states what a conformant resource must look like:
cardinality, fixed values, value-set bindings, extensions, and slicing, meaning which element of
a repeating list plays which role. ClinuxFlow ships seven profiles, for facilities, practitioners,
practitioner roles, affiliate practitioners, affiliate organizations, patients and workflow tasks.
The facility, practitioner and patient profiles were written against ABDM's own HFR, HPR and ABHA
API specifications. A validator checks extracted resources against them and returns specific,
field-level errors.

## The costs nobody mentions

FHIR-native is not free. These are the costs we actually paid.

**Cardinality is unforgiving.** In FHIR, `telecom`, `identifier`, `address` and many other
elements are arrays even when you only ever have one value. Our first extractor wrote them as
single objects. It looked fine in our own UI and would have been rejected by any real FHIR
server. Worse, two fields that mapped to the same element (phone and email both going to
`telecom.value`) overwrote each other, and only the last one survived. We fixed both with an
explicit table of which paths are arrays, and every field now claims its own slot.

**Repeating groups are where data goes missing.** A clinic with three doctors fills in the "care
team" section three times. Our extractor cached resources by type, so three practitioners went in
and one came out. No error, no warning, two doctors gone. We only caught it because a test
extracted two named practitioners and counted the results. The fix distinguishes "this repeating
group produces separate resources" from "this repeating group fills an array inside one resource",
and detects which from the structure of the paths, not from new syntax.

**Silent drops are the default failure mode.** An answer whose `definition` is missing or
malformed used to vanish without a trace. It is now collected as a warning. If you build an
extractor, make every dropped answer loud.

**Identity is harder than it looks.** Extraction mints a new resource id every time it runs, which
is fine for a one-off and wrong for a record that is edited and saved ten times. Our generic
save endpoint requires the client's own stable record id for exactly this reason. For
practitioners we derive identity from name plus facility. That is weak, since names are not
unique, and the upgrade path is to anchor on the HPR ID once one exists.

**Slicing needs data you might not capture.** Our patient profile says one of the patient's
contact points must be a mobile phone (`telecom` sliced by `system = phone`). Our capture forms
record the number but never record that it is a phone. So no real patient can currently pass that
rule. The profile is right; the capture is incomplete. This is the kind of gap a profile exposes
and a hand-written export layer would hide forever.

**Standards have sharp edges in the details.** SDC expects an item's `definition` to be a
canonical StructureDefinition URL (`http://hl7.org/fhir/StructureDefinition/Patient#Patient.name`).
Ours is a shortened form. Our own extractor accepts it, but a third-party SDC engine would not.
It's a one-line fix, and exactly the kind of thing to find before a partner does.

## Two decisions we would make again

**Generate the dictionary; never hand-curate it.** Every hand-maintained list of "valid fields"
eventually diverges from reality. Generating it from the standard's own type definitions means
adding a resource type is a configuration change and a rebuild.

**Keep capture and meaning separate.** Because every screen saves a QuestionnaireResponse and
extraction happens afterwards, we could replace our entire registration UI (twice) without
touching extraction, validation or document assembly. The response shape is the contract.

## One decision we reversed

We began by rendering every form, including registration, through LHC-Forms. For clinic-authored
clinical forms that is the right tool: arbitrary structure is the point. For registration, where
the schema is fixed by ABDM and every field is known in advance, a generic renderer caused a
real data-loss bug (typed values in autocomplete fields discarded unless a suggestion was clicked)
and a cramped, generic experience. We first built a second generic renderer, which repeated the
mistake. What worked was hand-written screens for registration, a shared navigation shell, and
validation against profiles. The general lesson: **use a generic form engine where the structure
is genuinely unknown, and purpose-built screens where it is fixed.**

## What an assembled record looks like

The last stage turns extracted resources into a FHIR document: a `Bundle` of type `document` with
a `Composition` first and every resource it references included, so the document is
self-contained. We default its status to `preliminary`. Nothing is `final` until a responsible
practitioner attests it, and we don't pretend otherwise. We also refuse to invent a coding for the
document type when no standard code fits; honest plain text is better than an authoritative-looking
code the document doesn't really conform to.

## Questions to ask of any clinical platform, including ours

1. Where is the list of valid fields defined, and who keeps it in sync with the standard?
2. If a user answers a question and the mapping is broken, does anything tell you?
3. What happens to repeating data (three doctors, two phone numbers) on the way to FHIR?
4. How does a record keep the same identity across a hundred edits?
5. Can you show a profile, and a validation result against it, for your core resources?
6. When a document is signed, what exactly is frozen, and how is an amendment represented?

## Where ClinuxFlow is today

- Built: the generated dictionary and compiler, SDC-style extraction with the fixes described,
  seven profiles and a validator, graph-based "what to capture next" suggestions, and document
  assembly.
- Not built yet: document attestation (signing), sending documents to an external FHIR server or
  to ABDM's exchange, a published CapabilityStatement, and the two fixes named above (canonical
  `definition` URLs and capturing `ContactPoint.system`).

The next post looks at where AI belongs in this pipeline, and more importantly where it does not.
