# ADR-0001: Replace CI PAT with GitHub App installation token

**Date:** 2026-04-16
**Status:** Proposed
**Author:** lucos-architect[bot]
**Related issues:** lucas42/lucos_deploy_orb#83 (security finding), lucas42/lucos_deploy_orb#82 (rate limit incident)

## Context

The CI pipeline (CircleCI, via the `lucos/deploy` orb) requires a GitHub token to:

1. **Push git tags** when a new version is calculated (`calc-version` command, line 39–41)
2. **Create GitHub Releases** via the REST API (`calc-version` command, line 44–52)
3. **Upload release assets** (APK files) to GitHub Releases (`publish-apk` command, lines 25–31)

All three operations currently use a single `GITHUB_TOKEN` environment variable, which is a **personal access token (PAT)** belonging to lucas42. The PAT has `repo` scope, which grants read/write access to all repositories on the account — far exceeding what any individual CI job needs.

This token is stored in lucos_creds under the `lucos_deploy_orb/publish` credential path and is fetched at the start of every build job via `fetch-publish-creds`.

### Problems with the current approach

**Security (blast radius):** A leaked PAT with `repo` scope gives an attacker read/write access to every repository on the account. CI build logs are a known leakage vector — CircleCI's secret masking is imperfect for partial values, base64-encoded variants, and secrets appearing in error stack traces. The blast radius of a single leaked token is the entire repository estate.

**Rate limits:** PAT rate limits are per-user (5,000 REST requests/hour, 5,000 GraphQL points/hour), shared across all API calls made with that user's credentials. When 35+ CI pipelines run concurrently — as happened in the 2026-04-16 incident — they compete for the same rate limit budget. The previous `semantic-release`-based `calc-version` was particularly expensive (multiple GraphQL calls per run), but even the current REST-based version makes 2 API calls per pipeline. At 57 concurrent pipelines, that's 114 calls in seconds, which is fine for REST but leaves no margin for coincident use of the same PAT elsewhere.

**Bus factor:** The token is tied to a personal account. If that account is suspended, has its PAT revoked, or changes ownership, all CI pipelines break estate-wide.

### Existing infrastructure

The lucos agent ecosystem already uses GitHub App installation tokens extensively. The `lucos_agent` repository contains:

- A `get-token` script that generates short-lived installation tokens from a GitHub App's PEM private key
- A `personas.json` registry of 10 GitHub Apps, each with App ID, Installation ID, and PEM variable name
- PEM keys stored in lucos_creds with newline-to-space flattening (a solved problem)

The token generation flow is: parse PEM from `.env` → build JWT (`openssl dgst -sha256 -sign`) → exchange JWT for installation token via `POST /app/installations/{id}/access_tokens` → token valid for 1 hour.

## Decision

Create a new GitHub App (`lucos-ci` or similar) dedicated to CI pipeline operations, and replace the PAT-based `GITHUB_TOKEN` with a short-lived installation token generated at the start of each build job.

### Architecture

```
lucos_creds
  └── lucos_deploy_orb/publish/.env
        ├── DOCKERHUB_USERNAME=...
        ├── DOCKERHUB_ACCESS_TOKEN=...
        ├── NPM_TOKEN=...
        ├── LUCOS_CI_APP_ID=...           (new)
        ├── LUCOS_CI_INSTALLATION_ID=...  (new)
        └── LUCOS_CI_PEM="..."            (new, replaces GITHUB_TOKEN)
```

A new orb command (`generate-github-token`) runs after `fetch-publish-creds` and before `calc-version`:

1. Reads `LUCOS_CI_APP_ID`, `LUCOS_CI_INSTALLATION_ID`, and `LUCOS_CI_PEM` from the environment (already loaded by `fetch-publish-creds`)
2. Restores the PEM key's newlines (same technique as `lucos_agent/get-token`)
3. Builds a JWT signed with the PEM key
4. Exchanges the JWT for a short-lived installation token via one REST API call, **scoped to the current repository** by passing `{"repositories": ["$CIRCLE_PROJECT_REPONAME"]}` in the token request body
5. Exports the result as `GITHUB_TOKEN` into `$BASH_ENV`

**Critical: per-repo token scoping.** The app is installed on all repositories, but the installation token must be scoped to the single repo being built. Without the `repositories` parameter in the `POST /app/installations/{id}/access_tokens` request, the returned token has access to every repo the app is installed on — which would replicate the PAT's blast radius problem. The `CIRCLE_PROJECT_REPONAME` environment variable (provided by CircleCI) identifies the repo being built and must be passed in every token request.

All downstream commands (`calc-version`, `publish-apk`) continue to use `$GITHUB_TOKEN` unchanged. The migration is transparent to consuming repos — they get it automatically on the next orb version bump.

### Why a new GitHub App (not an existing agent app)

The existing agent apps (`lucos-agent`, `lucos-developer`, etc.) are designed for interactive agent operations and have permissions tailored to that use case. CI is a different trust boundary:

- CI runs in CircleCI's infrastructure, not in the agent sandbox
- CI credentials are stored in a different lucos_creds path (`lucos_deploy_orb/publish` vs `lucos_agent/development`)
- CI operations are mechanical (tag, release, upload) — they don't need issue/PR permissions
- Separation means revoking the CI app's key doesn't affect agent operations, and vice versa

### Required GitHub App permissions

The new app needs exactly:

| Permission | Level | Used by |
|---|---|---|
| `Contents` | Write | `calc-version` (push tags via git), `publish-apk` (read release ID) |

That's it. `Contents: write` includes the ability to create releases and upload release assets. No other permissions are required.

The app should be installed on **all repositories** in the lucas42 account. The blast-radius reduction comes not from the installation scope but from **per-repo token scoping at generation time** — see the `repositories` parameter in the architecture section above.

### Rate limit improvement

GitHub App installation tokens get **5,000 REST requests/hour per installation** (not per-app — per repo-specific token). Each concurrent pipeline generates its own installation token scoped to its own repo, so 57 concurrent pipelines each have their own independent 5,000-request budget. The shared rate limit problem is eliminated entirely.

Note: this is the per-installation limit for Apps installed on user accounts. Organisation installations get 15,000/hour. The 5,000 limit is still a massive improvement over the shared PAT budget, because the budget is now per-repo rather than shared.

### Implementation in the orb

The `generate-github-token` command will be self-contained — approximately 40 lines of shell using `openssl` and `curl`, both available in all `cimg/*` images used by the orb. No external dependencies.

The command will be inserted into each build/release job between `fetch-publish-creds` and `calc-version`. Since it's an orb command, this change is made once in the orb and propagated to all consuming repos via the normal orb update mechanism.

### Migration plan

The migration can be done with zero downtime and no changes to consuming repos:

1. **lucas42 creates the GitHub App and stores credentials** (see "What lucas42 needs to do" below)
2. **Orb change:** Add `generate-github-token` command; insert it into all jobs that call `calc-version` or `publish-apk`; update `fetch-publish-creds` docs
3. **Transition period:** Both `GITHUB_TOKEN` (old PAT) and the new PEM variables coexist in `lucos_deploy_orb/publish/.env`. The `generate-github-token` command generates a fresh token and overwrites `GITHUB_TOKEN` in `$BASH_ENV`. If the new variables are missing, the command emits a visible warning ("GITHUB_TOKEN is a PAT — migrate to GitHub App token per ADR-0001") and falls back to the existing PAT. The warning ensures the fallback is noticed rather than silently persisting
4. **Verification:** Manually trigger a build on a test repo, confirm tagging and release creation work
5. **Cleanup:** Remove the old `GITHUB_TOKEN` PAT from lucos_creds; revoke the PAT in GitHub

### What lucas42 needs to do

The following steps require lucas42's direct involvement — they cannot be performed by agents.

#### Step 1: Create the GitHub App

1. Go to **GitHub → Settings → Developer settings → GitHub Apps → New GitHub App**
2. Set:
   - **App name:** `lucos-ci` (or similar — must be globally unique on GitHub)
   - **Homepage URL:** `https://github.com/lucas42` (any valid URL)
   - **Webhook:** Uncheck "Active" — this app doesn't need webhooks
   - **Permissions → Repository permissions:**
     - **Contents:** Read and write
     - All others: No access
   - **Where can this GitHub App be installed?** Only on this account
3. Click **Create GitHub App**
4. Note the **App ID** displayed on the app's settings page

#### Step 2: Install the app

1. On the app's settings page, click **Install App** in the left sidebar
2. Install on the **lucas42** account
3. Select **All repositories**
4. Note the **Installation ID** from the URL after installation (the numeric ID in `https://github.com/settings/installations/{id}`)

#### Step 3: Generate a private key

1. On the app's settings page, scroll to **Private keys**
2. Click **Generate a private key**
3. A `.pem` file will be downloaded

#### Step 4: Store credentials in lucos_creds

Add three values to the `lucos_deploy_orb/publish` credential set:

| Variable | Value | Notes |
|---|---|---|
| `LUCOS_CI_APP_ID` | The App ID from Step 1 | Numeric, e.g. `3045678` |
| `LUCOS_CI_INSTALLATION_ID` | The Installation ID from Step 2 | Numeric, e.g. `114900000` |
| `LUCOS_CI_PEM` | The contents of the `.pem` file | lucos_creds will flatten newlines to spaces — this is expected and handled by the orb |

After storing, verify the `.env` file contains the new variables:
```bash
scp -P 2202 "creds.l42.eu:lucos_deploy_orb/publish/.env" /dev/stdout | grep LUCOS_CI
```

#### Step 5: Securely delete the .pem file

Once stored in lucos_creds, delete the downloaded `.pem` file from your local machine. The private key should only exist in lucos_creds.

#### After these steps

Notify the team that the credentials are in place. The orb implementation work (adding `generate-github-token`, updating jobs) can then proceed as a normal agent-implemented issue. No further manual steps are required until the final cleanup (revoking the old PAT), which will be coordinated explicitly.

## Consequences

### Positive

- **Reduced blast radius:** Each installation token is scoped to the single repository being built (via the `repositories` parameter at generation time), so a leaked token cannot access any other repo
- **Independent rate limits:** Each pipeline gets its own per-installation rate budget — concurrent estate-wide builds no longer compete
- **No personal account dependency:** The App is owned by the account but not tied to a personal PAT that can be independently revoked or expire
- **Transparent to consumers:** Orb changes propagate automatically; no per-repo config changes needed
- **Short-lived tokens:** Installation tokens expire after 1 hour, vs PATs which are long-lived
- **Reuses proven patterns:** The JWT generation and PEM handling are already battle-tested in `lucos_agent/get-token`

### Negative

- **One additional API call per pipeline:** The JWT → token exchange adds one REST call before the build starts. This is negligible (< 1 second) but is a new network dependency at the start of every job
- **lucas42 setup required:** Creating the App, installing it, and storing credentials requires manual work that only the account owner can do
- **PEM key management:** The private key must be stored in lucos_creds and handled correctly (newline flattening). This is a solved problem but adds one more secret to manage
- **New orb command to maintain:** `generate-github-token` is ~40 lines of shell, but it's security-critical code that must be reviewed carefully

### Risks

- **`openssl` availability:** The command requires `openssl` for JWT signing. All current `cimg/*` images include it, but a future image change could break this. Mitigated by the fact that `openssl` is a fundamental system tool unlikely to be removed.
- **GitHub App API changes:** The installation token exchange endpoint is stable and well-documented, but any GitHub API change could affect it. This risk is equivalent to the current dependency on the REST releases API.

## Alternatives considered

### Keep the PAT, add retry-with-backoff

Addresses the rate limit issue (#82) but not the security issue (#83) or the bus factor. The blast radius of a leaked token remains the entire account. This is a necessary operational improvement regardless, but it's not sufficient as the sole mitigation.

### Use CircleCI's OIDC tokens with GitHub

CircleCI supports OIDC identity tokens that can be exchanged for short-lived GitHub credentials. This would eliminate stored secrets entirely. However, GitHub's OIDC trust for non-Actions CI is not well-documented, the setup is significantly more complex, and the lucos ecosystem has no existing OIDC infrastructure to build on. This is architecturally cleaner in theory but practically riskier. Worth revisiting if GitHub improves OIDC support for third-party CI.

### Use a fine-grained PAT instead of a classic PAT

GitHub fine-grained PATs can be scoped to specific repositories and permissions. This addresses the blast radius issue but not the rate limit issue (still a single shared budget) or the bus factor (still tied to a personal account). It's a partial improvement that doesn't justify the migration effort when GitHub App tokens solve all three problems.
