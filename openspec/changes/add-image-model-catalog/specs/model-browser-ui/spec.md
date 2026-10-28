# Spec Delta

## Purpose

Lets users browse, filter, and select fal image models from the local catalog and have those selections drive the Generate form's model choices.

## ADDED Requirements

### Requirement: Two-tab layout
The application SHALL present a Generate tab containing the existing generation form and a Models tab containing model discovery. Switching tabs MUST preserve unsaved form values and the current selection.

#### Scenario: Switch away and back
- **WHEN** a user types a prompt, switches to Models, and returns to Generate
- **THEN** the prompt and all other form values are unchanged

### Requirement: Refresh control reports progress and outcome
The Models tab SHALL provide a refresh action that indicates work in progress while importing, reports added and updated counts on success, and surfaces failures without emptying the visible model list.

#### Scenario: Successful refresh
- **WHEN** the user activates refresh and the import completes
- **THEN** the control leaves its loading state and displays how many models were added and updated

#### Scenario: Failed refresh
- **WHEN** the import fails or is rate limited
- **THEN** an error message is shown and previously listed models remain visible

### Requirement: Model cards show decision-relevant metadata
Each model card SHALL display the model's thumbnail, title, category, and price, and SHALL display lab or family when the catalog provides it. A model with no known price MUST show an explicit unknown indicator rather than a blank price.

#### Scenario: Card with structured price
- **WHEN** a model has a known price
- **THEN** its card shows the formatted price and unit

#### Scenario: Card without price
- **WHEN** a model has no known price
- **THEN** its card shows an unknown-price indicator

### Requirement: Filtering and search
The Models tab SHALL filter the visible cards by category and by free-text query over model title and endpoint ID, and SHALL show the number of matching models. When no model matches, it MUST show an empty state explaining that no results match the current filters.

#### Scenario: Filter by category
- **WHEN** the user chooses a category
- **THEN** only models in that category are listed and the result count reflects the filter

#### Scenario: No matches
- **WHEN** the search text matches no stored model
- **THEN** an empty state is displayed instead of an empty grid

### Requirement: Checkbox selection is immediate and reversible
Each model card SHALL provide a checkbox that reflects the model's persisted selection state. Toggling it SHALL persist the change immediately, and toggling it again SHALL restore the previous state.

#### Scenario: Select and deselect
- **WHEN** the user checks a model's checkbox and then unchecks it
- **THEN** the model is persisted as selected and then as unselected

### Requirement: Generate drop-down reflects selected models
The Generate tab's model drop-down SHALL contain exactly the models selected in the Models tab and MUST update without a page reload when selection changes. If no model is selected, the form MUST indicate that a model must be selected before generating and MUST prevent submission.

#### Scenario: Selection updates the drop-down
- **WHEN** the user selects a model in the Models tab and returns to Generate
- **THEN** the drop-down includes that model and its endpoint ID is used for generation

#### Scenario: No models selected
- **WHEN** the user deselects the last selected model
- **THEN** the Generate form shows a prompt to select a model and disables generation

### Requirement: Large catalogs remain usable
The Models tab SHALL keep the interface responsive when the full image catalog (hundreds of models) is stored, by rendering the list incrementally or virtually and loading thumbnails only for visible cards.

#### Scenario: Full catalog browsed
- **WHEN** the catalog contains several hundred models
- **THEN** the tab remains scrollable and interactive without loading every thumbnail at once
