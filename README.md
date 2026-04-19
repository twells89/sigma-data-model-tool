# Sigma Data Models - Repository Template

This folder contains everything you need to set up a GitHub repository for managing Sigma data models as code. Every pull request automatically runs [Sigma Sentinel](https://github.com/twells89/sigma-ci) validation and posts the results as a PR comment before merge.

## Quick Setup

1. **Create a new private GitHub repository**

2. **Copy these files to your repository:**
   ```
   your-repo/
   ├── .github/
   │   └── workflows/
   │       ├── sync-to-sigma.yml      # Validates PRs and syncs changes to Sigma on merge
   │       └── pull-from-sigma.yml    # Pulls changes from Sigma (scheduled)
   ├── scripts/
   │   ├── sync_to_sigma.py           # Push to Sigma
   │   ├── pull_from_sigma.py         # Pull from Sigma
   │   ├── get_model_ids.py           # Resolves model IDs for changed files
   │   └── generate_diff_report.py    # PR diff comments
   ├── data-models/
   │   └── _template.json             # Example data model template
   ├── config.yml                     # Configuration file
   └── .gitignore
   ```

3. **Configure GitHub Secrets** (Settings → Secrets and variables → Actions → Secrets):
   - `SIGMA_CLIENT_ID` - Your Sigma API Client ID
   - `SIGMA_SECRET` - Your Sigma API Client Secret

4. **Configure GitHub Variables** (Settings → Secrets and variables → Actions → Variables):
   - `SIGMA_CLOUD` - Your cloud: `aws`, `azure`, or `gcp`
   - `SIGMA_FOLDER_ID` - Folder ID where new data models will be created

5. **Enable Workflow Permissions** (Settings → Actions → General):
   - Select "Read and write permissions"
   - Check "Allow GitHub Actions to create and approve pull requests"

6. **Use the [Data Model Manager Tool](https://twells89.github.io/sigma-data-model-tool/)** to edit and commit changes!

## How It Works

```
┌─────────────────┐         ┌─────────────────────────────────┐         ┌──────────────┐
│   Web Tool or   │  ──PR──▶│            GitHub               │──Merge─▶│    Sigma     │
│   Direct Edit   │         │  ┌─────────────────────────┐    │         │ Auto-updated │
└─────────────────┘         │  │   Sigma Sentinel checks: │    │         └──────────────┘
                            │  │  • Schema drift          │    │
                            │  │  • Blast radius          │    │
                            │  │  • Formula integrity     │    │
                            │  └─────────────────────────┘    │
                            │  Results posted as PR comment    │
                            └─────────────────────────────────┘
```

1. Edit data models in the web tool or directly in JSON
2. Open a Pull Request — Sigma Sentinel automatically validates the changed models
3. Review the validation comment (diff report + drift/formula check results)
4. Merge to deploy to Sigma automatically

## PR Validation (Sigma Sentinel)

The `sync-to-sigma.yml` workflow runs [Sigma Sentinel](https://github.com/twells89/sigma-ci) on every PR that touches `data-models/*.json` files. It checks only the models that changed in the PR.

### Checks Run

| Check | What it catches | Blocks merge? |
|-------|----------------|---------------|
| **Schema Drift** | Warehouse columns referenced by the model that no longer exist in the warehouse | ✅ Yes (by default) |
| **Blast Radius** | Downstream workbooks affected by changes to this model | No (informational) |
| **Formula Integrity** | Column formulas referencing columns that don't exist in the element | No (warning) |

### PR Comment Examples

**All clear:**
```
## ✅ Sigma Sentinel — Validation Passed
### ✅ No Schema Drift
All referenced warehouse columns exist.
### ✅ No Broken Formula References
All formula references resolve correctly.
```

**Drift detected (merge blocked):**
```
## ❌ Sigma Sentinel — Drift Detected (merge blocked)
### ⚠️ Schema Drift Detected
Warehouse columns referenced in the model no longer exist:
**My Model**
- `ORDERS`: `LEGACY_COLUMN`, `OLD_FIELD`
```

**Formula errors detected (warning, does not block merge):**
```
## ⚠️ Sigma Sentinel — Formula Errors Detected
### 🔗 Broken Formula References
**My Model** — 1 broken reference(s)
- _Order Analysis_ / **Revenue YoY**: `[Prior Year Revenue]` → `Prior Revenue`
```

### Configuring Merge Blocking

The action inputs `fail-on-drift` and `fail-on-formula-errors` control which checks block the PR. To also block on formula errors, update the step in `sync-to-sigma.yml`:

```yaml
- uses: twells89/sigma-ci@main
  with:
    sigma-client-id: ${{ secrets.SIGMA_CLIENT_ID }}
    sigma-client-secret: ${{ secrets.SIGMA_SECRET }}
    model-ids: ${{ steps.models.outputs.model_ids }}
    fail-on-drift: 'true'
    fail-on-formula-errors: 'true'   # change to block on formula errors too
```

### Action Outputs

| Output | Type | Description |
|--------|------|-------------|
| `has-drift` | boolean string | `"true"` if schema drift was detected |
| `has-formula-errors` | boolean string | `"true"` if broken formula references were detected |
| `report-markdown` | string | Full validation summary as GitHub-flavoured markdown |
| `report-json` | string | Complete report as a JSON string for custom processing |

## Files Reference

| File | Purpose |
|------|---------|
| `sync-to-sigma.yml` | Validates PRs (Sigma Sentinel) and syncs data models to Sigma on merge |
| `pull-from-sigma.yml` | Pulls changes from Sigma (runs daily + on demand) |
| `sync_to_sigma.py` | Python script to push data model JSON to Sigma API |
| `pull_from_sigma.py` | Python script to pull data model specs from Sigma |
| `get_model_ids.py` | Resolves Sigma model IDs from filenames via `config.yml` |
| `generate_diff_report.py` | Posts a spec diff comment on PRs showing field-level changes |
| `config.yml` | Stores filename → Sigma model ID mappings |
| `_template.json` | Example data model JSON structure |
