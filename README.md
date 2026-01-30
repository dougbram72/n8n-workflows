# n8n-workflows

Central repository for my n8n workflows (automations for Gmail, weather, notifications, etc.).

## Structure

- `workflows/` – Exported workflow JSON from n8n (`*.json`). Each file is a single workflow export.
- `docs/` – Notes, diagrams, or usage docs for complex workflows.

## Usage

### Exporting from n8n

In the n8n UI:

1. Open a workflow.
2. Use **⋯ → Export → Download** (or equivalent in your version).
3. Save the file into this repo under `workflows/`, e.g.:
   - `workflows/gmail-archive-old-noise.json`

### Importing into n8n

In n8n:

1. Create a new workflow.
2. Use **Import from File / JSON**.
3. Select the JSON file from `workflows/`.

### Conventions

- Filenames should be kebab-case with a short description, e.g. `gmail-archive-old-noise.json`.
- Keep credentials out of the JSON where possible; use n8n credentials by name.

