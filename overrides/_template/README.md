# New Overrides Template

Steps to onboard a new overrides' override layer:

1. Copy this folder:
   ```
   cp -r overrides/_template overrides/<your-org-name>
   ```
2. Rename `manifest.yaml.template` to `manifest.yaml` inside your new folder
   and fill it in.
3. Delete this `README.md` from your new folder (it's only onboarding
   instructions, not part of the guideline itself) — or replace it with a
   short org-specific README if useful.
4. Add override files at the same relative path they have under
   [`../../base/`](../../base) — e.g. to override the backend API design
   guideline, create `backend/api-design.md` inside your org folder.
5. Open a PR. See [`../README.md`](../README.md) for the full resolution
   model and review expectations.
