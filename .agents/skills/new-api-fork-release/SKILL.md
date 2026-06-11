---
name: new-api-fork-release
description: >-
  Release and deploy this project's forked new-api workflow across
  `D:\code\new-api` and sibling `..\new-api-ops`. Use when Codex needs to
  sync upstream into the fork, keep fork-only changes thin, push `main` and a
  fork tag to `origin`, wait for the fork GHCR image to publish, deploy a tag
  to `rockapi` with `..\new-api-ops\scripts\deploy-rockapi.ps1`, or
  troubleshoot this release/deploy path.
---

# new-api Fork Release

## Overview

- Treat `D:\code\new-api` as the source fork: code, CI, upstream sync, and `docs/fork-delta.md`.
- Treat `D:\code\new-api-ops` as the runtime repo: production compose, env files, Caddy config, and deploy script.
- Keep project-specific operational truth in repo files first; use this skill to remember the workflow and decision points.

## Workflow

### 1. Classify the task

- **Upstream sync**: work mainly in `D:\code\new-api`; use `sync/upstream-*` branches and record lasting fork-only changes in `docs/fork-delta.md`.
- **Fork release**: coordinate both repos; publish code from `D:\code\new-api`, deploy from `D:\code\new-api-ops`.
- **Server update / rollback**: work mainly in `D:\code\new-api-ops`; use the deploy script with a known image tag.

### 2. Run preflight checks

- Confirm both sibling repos exist and are on the expected branch.
- Run `git status --short --branch` in both repos before editing or tagging.
- Do not revert unrelated user changes. Temporary local files may exist; inspect before deleting.
- For branding-only releases, check the active frontend theme from `/api/status` before deciding which static assets to update:
  - `theme=default` => `web/default/public/logo.png` and `web/default/public/favicon.ico`
  - `theme=classic` => `web/classic/public/logo.png` and `web/classic/public/favicon.ico`
  - If you want both themes visually aligned for future switches, update both sets.
- Never rely on interactive `git credential-manager` login for this workflow. It can block unattended runs while waiting for UI interaction.
- Use `D:\code\new-api-ops\env\prod.env` as the non-interactive credential source:
  - prefer `GITHUB_USERNAME` / `GITHUB_TOKEN` when present;
  - otherwise fall back to `GHCR_USERNAME` / `GHCR_TOKEN`.
- Ensure `D:\code\new-api-ops\env\prod.env` exists before production deploys.
- If `ghcr.io/leivii/new-api` is private, ensure `GHCR_USERNAME` and `GHCR_TOKEN` are populated in `env/prod.env`.

### 3. Publish a fork release

- Preferred one-shot entrypoint: `D:\code\new-api-ops\scripts\release-rockapi.ps1 -ImageTag <tag>`
- That script performs non-interactive `push main`, `push tag`, GHCR polling, and deployment using credentials from `env/prod.env`.
- Push validated source changes from `main`.
- Create tags as `v<upstream-version>-rockapi.<n>`, for example `v1.0.0-rc.10-rockapi.1`.
- Keep the current image repository as `ghcr.io/leivii/new-api` until the GitHub owner/repo changes; only the tag suffix changes for branding.
- Push the tag and wait for GitHub Actions to publish:
  - GitHub Release artifacts
  - `ghcr.io/leivii/new-api:<tag>`
- Prefer checking GHCR readiness with `docker manifest inspect ghcr.io/leivii/new-api:<tag>`.
- If the manifest is missing, read [references/release-runbook.md](references/release-runbook.md) for the two common causes.
- The `Build and Test` workflow on `main` is currently not a clean release gate. Recent Linux failures reproduced outside GitHub Actions include:
  - `controller.TestListModelsTokenLimitIncludesTieredBillingModel`
  - `relay/channel/claude.TestRequestOpenAI2ClaudeMessage_IgnoresUnsupportedFileContent`
  - `relay/channel/claude.TestRequestOpenAI2ClaudeMessage_SupportsPDFFileContent`
  - `relay/channel/claude.TestRequestOpenAI2ClaudeMessage_ConvertsTextFileContentToText`
- Treat those as upstream test noise unless you are explicitly fixing them; for fork branding or ops-only releases, let tag image publication plus runtime health decide deploy readiness.
- When you need GitHub Actions or release diagnostics, use the same token from `env/prod.env` for authenticated GitHub API requests instead of anonymous API calls.

### 4. Deploy to production

- Production host alias is `rockapi`.
- Production runtime directory is `/opt/llm-hub`.
- Canonical deployment entrypoint is `D:\code\new-api-ops\scripts\deploy-rockapi.ps1`.
- Deploy by tag; do not deploy `latest` manually.
- The deploy script uploads compose/Caddy/env files, keeps timestamped backups under `/opt/llm-hub/backups`, logs in to GHCR when needed, and runs `docker compose up -d`.

### 5. Verify actual runtime state

- Treat container health and runtime logs as authoritative:
  - `ssh rockapi "docker ps --format '{{.Names}}\t{{.Image}}\t{{.Status}}'"`
  - `ssh rockapi "docker inspect llm-hub-new-api-1 --format '{{.Config.Image}}'"`
  - `ssh rockapi "docker logs --tail 100 llm-hub-new-api-1"`
- A post-deploy probe may fail even when the rollout succeeded. Confirm the service state before declaring failure or rolling back.
- If needed, inspect `new-api` logs for `New API <tag> started` and live request handling.

### 6. Roll back safely

- Redeploy the last known-good tag with the same deploy script.
- If a release changed schema in a backward-incompatible way, restore DB/data backups before starting the older image.

## References

- Read [references/release-runbook.md](references/release-runbook.md) for the exact command sequence, current topology, and known failure modes.
- Read `D:\code\new-api-ops\README.md` when you need the current server layout or release contract.
