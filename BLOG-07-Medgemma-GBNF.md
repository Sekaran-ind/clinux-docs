# Making Local Models Emit Valid Clinical Data: Grammar-Constrained Extraction with MedGemma

*Building Clinical Software That Deserves Trust, part 7*

**For:** AI engineers building structured extraction from clinical text with open-weight models.

---

Ask a language model to "return JSON with these fields" and most of the time you get JSON with
those fields. The rest of the time you get a missing brace, a field you didn't ask for, a
diagnosis where a number should be, or an enum value that is almost right. In a demo that's an
edge case. In a clinic doing a hundred consultations a day, "most of the time" means several
malformed records a day, and malformed is the *good* failure: at least it's visible.

This post describes how we plan to extract structured clinical data from text with a local model
(Google's MedGemma, running on owned hardware; see part 5) using grammar-constrained decoding, and
why the grammar is the easy part. It is a design write-up; the pipeline is not yet built.

## Constrained decoding in one paragraph

A language model generates one token at a time, picking from a probability distribution over its
vocabulary. Constrained decoding removes, at each step, every token that would make the output
violate a formal grammar, then samples from what's left. The output is guaranteed to parse.
llama.cpp expresses these grammars in a format called GBNF and can convert a JSON Schema into one;
other engines (Outlines, XGrammar, llguidance) offer the same idea with different trade-offs.

## Derive the grammar from the form, don't write it by hand

ClinuxFlow's forms are compiled FHIR `Questionnaire`s (part 1). Every item already declares its
type, its allowed answers, whether it repeats, and how it nests. That's everything a grammar
needs, so the grammar should be generated from the Questionnaire:

| Questionnaire item | Grammar |
|---|---|
| `decimal`, `integer` | A number pattern, or `null` |
| `date` | `YYYY-MM-DD`, or `null` |
| `boolean` | `true`, `false` or `null` |
| `choice` with `answerOption` | An enum of exactly those codes, or `null` |
| `open-choice` with a large ValueSet | A text span, resolved afterwards by terminology lookup and human confirmation (never an enum of 50,000 codes) |
| `repeats: true` | An array with a sensible maximum length |
| `group` | A nested object |

Generating it means the form and the grammar can never disagree. Change the form and the grammar
changes with it.

## The most important design rule: every field can say "not mentioned"

This is the mistake almost everyone makes first. If the grammar says a field is required and must
be a number, the model **must** emit a number, even when the conversation never mentioned one. The
grammar has just made fabrication mandatory.

So in an extraction grammar:
- **Every field is nullable.** The model must be able to answer "the text didn't say".
- **Clinical findings carry an assertion status**, not just presence: `present`, `absent`,
  `historical`, `family`, `hypothetical`, `not_mentioned`. "Denies chest pain" becomes
  `{status: absent}`, not a missing field and never `present`.
- **Every value carries its evidence**: the exact span of the source text it came from. That turns
  checking into a mechanical step (below) and gives the clinician something to look at.

A simplified grammar for a vitals-and-complaints section:

```
root      ::= "{" ws "\"systolic_bp\":" ws int3 "," ws
                    "\"diastolic_bp\":" ws int3 "," ws
                    "\"chest_pain\":" ws finding ws "}"
finding   ::= "null" | "{" ws "\"status\":" ws status "," ws "\"evidence\":" ws str ws "}"
status    ::= "\"present\"" | "\"absent\"" | "\"historical\"" | "\"family\"" | "\"not_mentioned\""
int3      ::= "null" | [0-9] | [0-9][0-9] | [0-9][0-9][0-9]
str       ::= "\"" [^"\\]* "\""
ws        ::= " "?
```

Given "BP one thirty over eighty-five, no chest pain, father had a heart attack", a well-behaved
model produces (line breaks added for reading; the grammar emits it on one line):

```json
{ "systolic_bp": 130, "diastolic_bp": 85,
  "chest_pain": { "status": "absent", "evidence": "no chest pain" } }
```

Two small details matter. The whitespace rule is deliberately tight, because an unbounded
whitespace rule lets some models emit spaces until they hit the length limit. And the father's
heart attack has no field in this section, which is correct: bounded context (part 6) means the
grammar covers only the section being worked on, keeping it small and fast and keeping stray facts
out of the wrong place.

## What a grammar cannot do

A grammar guarantees shape. It says nothing about truth. Everything after decoding is about truth:

1. **Evidence check.** Does each `evidence` string actually occur in the source text? If not, the
   value is dropped or flagged. This single check catches a large share of fabrications.
2. **Plausibility.** Systolic below diastolic, a pulse of 400, a date in the future: these are range
   and cross-field rules, and many can be written as invariants on the FHIR profile the data must
   conform to anyway.
3. **Units and normalization.** "One thirty over eighty-five", "130/85" and "BP 85/130" (a
   transcription swap) all need normalizing and sanity-checking.
4. **Terminology resolution.** Free-text spans for coded fields go through a terminology lookup
   (SNOMED CT subsets, licensed in India via NRCeS) and then through the same confirmation picker a
   human would use. A model never writes a code directly into a coded field.
5. **Profile validation.** The resulting FHIR resources are validated against the same
   StructureDefinitions as manually entered data.
6. **Human acceptance.** Values appear as suggestions a clinician accepts or corrects, field by
   field, and each accepted value is recorded with provenance: model, version, grammar version,
   evidence, and who accepted it.

The pipeline:

```
transcript ─► section scope ─► grammar from Questionnaire ─► constrained decode (MedGemma)
      ─► evidence check ─► plausibility and unit rules ─► terminology resolution
      ─► profile validation ─► suggestions in the UI ─► clinician accepts ─► Provenance recorded
```

## Practical notes

- **Grammars cost speed.** Masking at every step adds overhead, and very large enums make it worse.
  Section-scoped grammars stay small.
- **Quantization changes accuracy.** A 4-bit model is not the 16-bit model. Evaluate the exact
  quantized file you will run.
- **Model size is a real trade-off on small hardware.** On a 24 GB machine, MedGemma's 4B model runs
  comfortably; the 27B model at 4-bit fits but leaves little room for context. Measure throughput on
  your real transcripts before promising response times.
- **Pin everything.** Model file hash, grammar version, prompt template version. An extraction you
  can't reproduce can't be investigated.
- **Code-switched speech is the norm.** Evaluation sets must include the way clinicians in India
  actually speak.

## Measuring it

| Metric | What it tells you |
|---|---|
| Field-level precision and recall | Basic extraction quality |
| Assertion accuracy (negated, historical, family) | Whether "no chest pain" stays absent |
| Abstention accuracy | Whether the model says "not mentioned" when it should |
| Evidence validity rate | How often the quoted span really exists in the source |
| Clinician correction rate per field | Real-world quality after deployment |

Abstention accuracy is the one most teams skip, and the one that tells you whether your grammar is
forcing fabrication.

## Where ClinuxFlow is today

Design only. Today's SOAP drafting uses a hosted open-weight model without grammar constraints, and
its output is a draft the clinician edits (part 2). An early constrained-output engine exists in our
codebase but was never connected. The pieces this design depends on are real: compiled
Questionnaires to generate grammars from, FHIR profiles to validate against, and a suggestion UI
that requires explicit acceptance. Grammar generation, the local model service and the evidence and
provenance steps come next, in that order.
