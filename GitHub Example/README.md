# GitHub Actions example

This folder mirrors the ADO lifecycle using GitHub Actions building blocks.

The GitHub Actions example is shown here as a caller workflow plus reusable pieces so it’s easy to compare with the ADO example.

The lifecycle maps the same way:

- a calling workflow, similar to `azure-pipelines.yml`
- a reusable workflow, called by the app team’s workflow
- a composite action, called by the reusable workflow
- dev then prod sequencing with `needs`
- JSON-driven dev/prod object input instead of ADO-style object parameters

## Files

- `someapp-repo/azure-pipelines.yml` — **consumer workflow template** in an app repo (named to mirror the ADO example)
- `.github/workflows/deploy-appservices.yml` — reusable workflow hosted in the shared workflows repo
- `.github/actions/deploy-appservices/action.yml` — composite action used by the reusable workflow

## Notes

This mirrors the ADO model closely:

- ADO app repo: `azure-pipelines.yml` references shared templates via `resources.repositories`
- GitHub app repo: `someapp-repo/azure-pipelines.yml` (example file name for parity) references shared workflows via `uses: org/repo/.github/workflows/file.yml@ref`

How teams use this:

1. Keep the reusable workflow + composite action in a shared workflows repo.
2. Copy `someapp-repo/azure-pipelines.yml` into each app repo (or rename as desired).
3. Update only:
	- `uses:` repository and ref (branch/tag)
	- environment/app-service JSON values
	- secrets configuration
