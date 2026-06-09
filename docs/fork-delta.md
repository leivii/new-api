# Fork Delta Record

This file keeps the fork intentionally thin. Every source-level change that is not a straight upstream sync should be recorded here before release.

## Rules

- Prefer configuration and reverse-proxy changes over source edits.
- Prefer upstream contribution for generic fixes that are not business-specific.
- Sync upstream by tag or release branch, not by arbitrary `HEAD`.
- Keep production deployment files out of this repo. Use the separate `new-api-ops` repo.

## Current Delta

| Date | Area | Files | Why it exists | Upstream candidate |
| --- | --- | --- | --- | --- |
| 2026-06-09 | CI/CD | `.github/workflows/docker-build.yml`, `.github/workflows/docker-image-alpha.yml`, `.github/workflows/docker-image-nightly.yml`, `.github/workflows/build-and-test.yml`, `.github/workflows/release.yml` | Publish images to the fork-owned GHCR repository, add a real build/test gate, and consolidate tagged releases into a single generated GitHub Release with all platform artifacts attached. | No |

## Template For Future Entries

| Date | Area | Files | Why it exists | Upstream candidate |
| --- | --- | --- | --- | --- |
| YYYY-MM-DD | frontend/backend/deploy | `path/to/file` | Short explanation of the business or operational reason. | Yes/No |
