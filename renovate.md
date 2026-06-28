# Renovate

How we keep dependencies current across our repos. One `renovate.json` at the repo root; the bot is already running (self-hosted at [renovate.oglimmer.com](https://renovate.oglimmer.com)). These are defaults with rationale — add repo-specific `packageRules` only when you hit a real constraint.

Generate or review configs interactively at [renovate.oglimmer.com](https://renovate.oglimmer.com). Per-repo adoption: [assessments/renovate.md](assessments/renovate.md).

## Philosophy

- **Automerge the boring stuff.** Minor, patch, pin, and digest updates merge themselves once CI passes.
- **Human review for majors.** Major bumps require Dependency Dashboard approval — never automerge.
- **Group by manager.** One PR per package manager per week beats twenty individual patch PRs.
- **Soak new releases.** `minimumReleaseAge: 7 days` on normal updates; security fixes bypass the wait.
- **Pin immutably, float intentionally.** Docker digests and GitHub Action SHAs get pinned; `:latest` tags stay unpinned.
- **Never automerge 0.x.** Pre-1.0 semver allows breaking changes in minor/patch releases.
- **Describe every rule.** Every `packageRules` entry has a `description` so the Dependency Dashboard and PR bodies stay readable.

## File location

```
renovate.json    # repo root — always this name
```

- Always include `$schema` for editor autocomplete.
- Pick one config file per repo. Mixing locations (`renovate.json5`, `.github/renovate.json`, …) is undefined behaviour — don't.
- For non-trivial overrides, `renovate.json5` (JSON5 with comments) is fine. Most repos use plain `renovate.json`.

## Standard config

Copy this baseline and add repo-specific rules at the end of `packageRules`.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended",
    ":enableVulnerabilityAlerts",
    ":dependencyDashboard",
    ":semanticCommits",
    "docker:pinDigests",
    "helpers:pinGitHubActionDigests",
    ":pinDevDependencies",
    "abandonments:recommended"
  ],
  "configMigration": true,
  "timezone": "Europe/Berlin",
  "prHourlyLimit": 10,
  "prConcurrentLimit": 20,
  "rebaseWhen": "behind-base-branch",
  "rangeStrategy": "bump",
  "automergeType": "branch",
  "packageRules": [ "..." ],
  "minimumReleaseAge": "7 days",
  "internalChecksFilter": "strict",
  "lockFileMaintenance": {
    "enabled": true,
    "schedule": ["before 5am on monday"],
    "automerge": true
  },
  "vulnerabilityAlerts": {
    "minimumReleaseAge": "0 days",
    "labels": ["security"],
    "schedule": ["at any time"],
    "automerge": true
  }
}
```

### `extends` presets

| Preset | What it gives us |
|--------|------------------|
| `config:recommended` | Sensible defaults for any language |
| `:enableVulnerabilityAlerts` | Security-fix PRs from the platform advisory feed |
| `:dependencyDashboard` | Tracking issue listing all pending updates |
| `:semanticCommits` | `chore(deps): …` commit messages |
| `docker:pinDigests` | Pin Docker image references to `@sha256:…` |
| `helpers:pinGitHubActionDigests` | Pin `uses:` actions to full commit SHAs |
| `:pinDevDependencies` | Exact versions for dev deps in manifests |
| `abandonments:recommended` | Flag unmaintained packages |

Keep `configMigration: true` so Renovate opens PRs when option names change upstream.

### Global settings

| Key | Our value | Why |
|-----|-----------|-----|
| `timezone` | `Europe/Berlin` | Pairs with scheduled jobs (lockfile maintenance) |
| `prHourlyLimit` | `10` | Enough throughput without flooding CI |
| `prConcurrentLimit` | `20` | Backpressure on concurrent open PRs |
| `rebaseWhen` | `behind-base-branch` | Rebase when base moved; don't rebase on every upstream touch |
| `rangeStrategy` | `bump` | Raise lower bound in ranges (`^1.2.0` → `^1.3.0`) — right for apps |
| `automergeType` | `branch` | Automerged updates land directly on the branch (no PR audit trail for trivial bumps) |
| `minimumReleaseAge` | `7 days` | Soak period before non-security updates are offered |
| `internalChecksFilter` | `strict` | Don't open PRs until release-age and other internal checks pass |

We intentionally omit a global `schedule` — non-security PRs can open any time. Lockfile maintenance and vulnerability alerts carry their own schedules.

**Automerge requires CI.** Branch protection with required status checks must pass before automerge fires. Repos without CI should not enable automerge.

## Standard `packageRules`

These seven rules are the same in every repo unless noted. Order matters — broad rules first, specific overrides last.

### 1. Group non-major updates per manager

```json
{
  "description": "Group each manager's non-major updates into one PR per manager",
  "matchUpdateTypes": ["minor", "patch", "pin", "pinDigest", "digest"],
  "groupName": "{{manager}} non-major dependencies"
}
```

`{{manager}}` expands to `npm`, `gomod`, `dockerfile`, `github-actions`, etc. One grouped PR per manager per cycle.

Only include managers the repo actually uses. If a repo has no Maven, the rule just never matches `maven` — no harm. For repos that predate the `{{manager}}` template, per-manager rules (`matchManagers: ["npm"]`, `groupName: "npm dependencies"`) work the same way.

### 2. Automerge non-major updates

```json
{
  "description": "Automerge minor, patch, pin, pinDigest, digest updates (low-risk, non-major)",
  "matchUpdateTypes": ["minor", "patch", "pin", "pinDigest", "digest"],
  "automerge": true
}
```

### 3. Automerge devDependencies

```json
{
  "description": "Automerge all devDependencies (build/test tooling, not shipped to consumers)",
  "matchDepTypes": ["devDependencies"],
  "automerge": true
}
```

Build tooling can move faster than production deps.

### 4. Automerge wrapper updates

```json
{
  "description": "Automerge Maven/Gradle wrapper updates",
  "matchManagers": ["maven-wrapper", "gradle-wrapper"],
  "automerge": true
}
```

### 5. Never automerge 0.x

```json
{
  "description": "Never automerge 0.x releases — semver allows breaking changes before 1.0",
  "matchCurrentVersion": "/^0\\./",
  "automerge": false
}
```

### 6. Keep `:latest` unpinned

```json
{
  "description": "Keep Docker :latest live — never pin a moving tag to a digest",
  "matchDatasources": ["docker"],
  "matchCurrentValue": "/^latest$/",
  "pinDigests": false
}
```

Pinning `:latest` to a digest gives a false sense of immutability — the tag keeps moving.

### 7. No soak on digest-only Docker updates

```json
{
  "description": "No release-age delay on Docker digest pins — a pin records which image a tag resolves to, not a new version (version bumps keep their soak)",
  "matchDatasources": ["docker"],
  "matchUpdateTypes": ["pin", "pinDigest", "digest"],
  "minimumReleaseAge": "0 days"
}
```

### 8. Majors need dashboard approval

```json
{
  "description": "Isolate major updates behind Dependency Dashboard approval",
  "matchUpdateTypes": ["major"],
  "dependencyDashboardApproval": true
}
```

Approve majors from the Dependency Dashboard issue when ready to take the migration cost.

## Lockfile maintenance

```json
"lockFileMaintenance": {
  "enabled": true,
  "schedule": ["before 5am on monday"],
  "automerge": true
}
```

Runs weekly before 5am Monday (Berlin time). Refreshes `package-lock.json`, `go.sum`, etc. without a version bump in the manifest. Automerges when CI passes.

## Vulnerability alerts

```json
"vulnerabilityAlerts": {
  "minimumReleaseAge": "0 days",
  "labels": ["security"],
  "schedule": ["at any time"],
  "automerge": true
}
```

Security fixes bypass the 7-day soak and any global schedule. Requires the platform dependency graph enabled (GitHub: Settings → Security → Dependabot alerts).

## Repo-specific overrides

Add these **after** the standard rules. Document the reason in `description` — future you needs to know why a dep is frozen.

### Bundled Postgres — disable major bumps

When the app ships an in-cluster Postgres whose data directory is tied to a major version, never let Renovate offer a major image bump. A major upgrade needs a deliberate `pg_upgrade` or dump+restore.

```json
{
  "description": "Never offer a Postgres major bump — the bundled DB's data dir is tied to its major version, so a major upgrade needs a deliberate pg_upgrade / dump+restore, not an image swap (see helm/<chart>/values.yaml postgres.image.tag)",
  "matchDatasources": ["docker"],
  "matchPackageNames": ["postgres"],
  "matchUpdateTypes": ["major"],
  "enabled": false
}
```



### Other common overrides

| Situation | Rule shape |
|-----------|------------|
| Monorepo sub-package | `matchFileNames: ["packages/api/**"]` + custom `groupName` |
| Pin a specific lib | `matchPackageNames: ["lodash"]`, `rangeStrategy: "pin"` |
| Ignore a generated path | top-level `ignorePaths: ["vendor/**"]` |
| Slow-moving infra dep | `matchPackageNames: ["terraform"]`, `minimumReleaseAge: "30 days"` |
| Internal/private package | `matchPackageNames: ["@myorg/*"]`, custom registry rules |

Use `matchPackageNames` for both exact names and regex (`"/^@angular//"`). Deprecated matchers (`matchPackagePatterns`, `matchPackagePrefixes`) are rewritten by `configMigration` — don't use them in new rules.

## Detecting managers in a new repo

Before writing `packageRules`, scan for manifests:

| File | Renovate manager |
|------|------------------|
| `package.json` | `npm` |
| `go.mod` | `gomod` |
| `pom.xml` | `maven` |
| `build.gradle` | `gradle` |
| `Dockerfile`, `*.Dockerfile` | `dockerfile` |
| `compose.yml` | `docker-compose` |
| `.github/workflows/*.yml` | `github-actions` |
| `helm/**/Chart.yaml` | `helm-values`, `helm-requirements` |
| `*.tf` | `terraform` |
| `pyproject.toml` | `pep621` |
| `requirements.txt` | `pip_requirements` |

The standard `{{manager}}` grouping handles all of these without per-manager entries.

## Generating a config

Three ways, same output:

1. **Interactive UI** — [renovate.oglimmer.com](https://renovate.oglimmer.com): pick options, preview live, download `renovate.json`.
2. **Generate API** — deterministic, no rate limit:
   ```bash
   curl 'https://renovate.oglimmer.com/api/generate?prLimitStrategy=active&groupAllNonMajor=true' \
     -o renovate.json
   ```
   Use `prLimitStrategy=active` to match our `prHourlyLimit: 10` / `prConcurrentLimit: 20`. See [Developer API](https://renovate.oglimmer.com/developers) for all query parameters.
3. **Copy from a sibling repo** in the org with the same package managers, or generate via [renovate.oglimmer.com](https://renovate.oglimmer.com).

For AI review of an existing config: `POST https://renovate.oglimmer.com/api/feedback` (rate-limited to 1 req/min).

## Validation

Always validate before committing:

```bash
npx --yes --package renovate -- renovate-config-validator renovate.json
```

Success prints `Config validated successfully`.

**Gotcha:** `npx renovate-config-validator` 404s — the binary ships inside the `renovate` package, so `--package renovate` is required.

Wire validation into CI if the repo has a lint workflow — a broken config should fail the build, not silently confuse the bot.

## New-repo checklist

1. Scan manifests (see table above).
2. Copy the [standard config](#standard-config) into `renovate.json`.
3. Append repo-specific `packageRules` (Postgres freeze, monorepo paths, etc.).
4. Run `renovate-config-validator`.
5. Commit and push. Renovate opens an onboarding PR on first scan.
6. Confirm branch protection requires CI status checks (automerge depends on this).
7. Watch the Dependency Dashboard issue for the first grouped PRs.

## Relationship to GitHub Actions

| Concern | How they interact |
|---------|-------------------|
| Action SHA pins | `helpers:pinGitHubActionDigests` keeps `.github/workflows/*.yml` SHAs current — see [github-actions.md](github-actions.md) |
| CI must pass | Automerge waits for required checks from `ci.yml` |
| Docker digests in workflows | `docker:pinDigests` + the digest-pin `packageRule` keep `services:` image pins updated |
| Registry cleanup | `cleanup-images.yml` prunes old digests that pinned `:latest` builds leave behind |

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| Hand-roll a 200-line config from a blog post | Extend the standard presets; add rules when you hit a real problem |
| Automerge majors | `dependencyDashboardApproval: true` on majors |
| Automerge 0.x deps | `matchCurrentVersion: "/^0\\./"`, `automerge: false` |
| Pin `:latest` to a digest | `pinDigests: false` when `matchCurrentValue: "/^latest$/"` |
| Set `minimumReleaseAge` on security fixes | Override to `"0 days"` in `vulnerabilityAlerts` |
| Use `npm` presets (`renovate-config-*`) | JSON `extends` presets only — npm presets are deprecated |
| Skip validation | `npx --yes --package renovate -- renovate-config-validator` |
| Enable automerge without CI | Add `ci.yml` first, then enable automerge |
| Use deprecated `matchPackagePatterns` | `matchPackageNames` with regex (`"/^foo/"`) |

## Further reading

- In-repo deep dive: `oglimmer/start-renovate/renovate-json-guide.md`
- Renovate docs: [configuration options](https://docs.renovatebot.com/configuration-options/), [presets](https://docs.renovatebot.com/presets-config/), [upgrade best practices](https://docs.renovatebot.com/upgrade-best-practices/)
- `/renovate-config` skill — generate and validate a config tailored to a repo's package managers