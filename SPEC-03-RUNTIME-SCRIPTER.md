# Specification 03: Runtime Inference Constraint & State Patch Engine

## 1. Objective
Build a local client runtime application script that maps natural language medical conversations into an unstructured sequence of target variables using schema-constrained model decoding, and updates active frontend form interfaces without introducing schema errors.

## 2. Architecture Logic Requirements

### Component A: Local Constraint Configuration Routine
- **Input**: The active layout file configuration compiled from your design-time pipeline.
- **Target Interface**: Local inference orchestration layers (Ollama API or local Llama.cpp wrappers).
- **Core Strategy**: 
  - Dynamically isolates all nodes categorized as active fields for real-time transcription tracking.
  - Constructs a structured parameter mask to control token probabilities during inference.
  - Forces the inference engine to return a flat JSON structure that precisely follows an array configuration protocol matching target properties to raw strings or number values.

### Component B: State Synchronization & Patch Controller
- **Input**: The verified json dataset output from Component A alongside the master layout configuration dictionary.
- **Operation Blueprint**:
  1. Receives data segments from the voice transcription pipeline.
  2. Iterates through the collection of captured values.
  3. Maps variables directly to target fields inside the client-side state machine.
  4. Automatically injects predetermined values into fields flagged as administrative system variables without triggering unnecessary visual screen redraws.
  5. Leverages a local validation runner or custom expression interpreter to process dependent value fields (e.g., automatically calculating an abnormality alert if a captured numerical value crosses a specific safety threshold).
