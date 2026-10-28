# Spec Delta

## Purpose

Renders a generation form that matches the selected model's input contract instead of a fixed Flux parameter set.

## ADDED Requirements

### Requirement: Controls are chosen from schema types
The form SHALL render a select for enum fields, a checkbox for booleans, a numeric input for integer and number fields, a single-line text input for short strings, and a multi-line input for prompt-like strings. Each control MUST display the schema's title or description when provided.

#### Scenario: Enum field
- **WHEN** a model's `output_format` is declared as an enum
- **THEN** the form shows a select containing only the declared values

#### Scenario: Bounded numeric field
- **WHEN** a numeric field declares minimum and maximum
- **THEN** the input enforces those bounds and surfaces an error when the value is outside them

### Requirement: Defaults are prefilled
When a model schema declares a default, the form SHALL initialize that field with the default value and submit it unless the user changes it.

#### Scenario: Default applied
- **WHEN** a model declares `num_images` default `1`
- **THEN** the field shows `1` and the submitted input contains `1` without user interaction

### Requirement: Required fields are validated before submission
The form MUST mark required fields and MUST prevent submission while any required field is empty or invalid, showing which fields need attention.

#### Scenario: Missing required prompt
- **WHEN** the model requires `prompt` and the user leaves it empty
- **THEN** submission is blocked and the prompt field is identified as required

### Requirement: Optional fields are grouped as advanced
Optional parameters SHALL be grouped under a collapsed "Advanced" section while required and commonly used fields remain visible by default.

#### Scenario: Advanced defaults remain available
- **WHEN** the user never expands Advanced
- **THEN** optional defaults are still included in the submitted input

### Requirement: LoRA-style arrays keep a dedicated editor
When a schema declares an array of objects with `path` and `scale` (LoRA inputs), the form SHALL render repeatable rows with add/remove controls instead of a generic array control.

#### Scenario: Add and remove LoRA rows
- **WHEN** the user adds two LoRA rows and removes one
- **THEN** the submitted array contains the remaining row with its path and scale

### Requirement: Unsupported schemas fall back to raw JSON
If a schema cannot be represented by the generated controls, the form SHALL offer a raw JSON editor for the full input and validate that the JSON parses before submission.

#### Scenario: Unsupported construct
- **WHEN** a schema uses a construct the renderer does not support
- **THEN** the user can switch to raw JSON, edit the complete input, and submit once it parses

### Requirement: Values survive model switching where compatible
Switching models SHALL preserve values whose field name and type exist in both schemas, and MUST drop values that the new model does not accept.

#### Scenario: Switch between Flux variants
- **WHEN** the user types a prompt and switches from one Flux model to another
- **THEN** the prompt is preserved and parameters unique to the first model are removed

### Requirement: Schema loading is visible
While a model's schema is loading, the form SHALL show a loading state and MUST NOT submit using the previous model's schema.

#### Scenario: Model change while loading
- **WHEN** the user selects a model whose schema is still loading
- **THEN** the form indicates loading and submission stays disabled until the new schema or fallback is ready
