---
marp: true
theme: default
paginate: true
---

# Dependency Management in Enterprise Projects
## Dependabot Setup & Comparison with Renovate

Engineering Fundamentals — Part F

---

## 1. Why Automated Dependency Management?

In enterprise environments with multiple developers and repositories:
- **Security**: Rapidly patch known CVEs in direct and transitive dependencies.
- **Stability**: Avoid "big bang" upgrades by applying small, frequent updates.
- **Consistency**: Keep `npm` packages, Docker base images, and CI actions aligned.
- **Developer Productivity**: Eliminate manual `npm outdated` checks and PR creation.

---

## 2. Ecosystems in This Repository

Our repository has **three** dependency surfaces that must stay up to date:

1. **npm** (`package.json` & `package-lock.json`)
   - Runtime (`react`, `react-dom`) and dev tools (`vite`, `typescript`, `jest`, `eslint`).
2. **Docker** (`Dockerfile`)
   - Base image `node:24-alpine` used in both `build` and `serve` stages.
3. **GitHub Actions** (`.github/workflows/*.yml`)
   - Actions such as `actions/checkout`, `actions/setup-node`, `dependabot/fetch-metadata`.

---

## 3. Dependabot vs. Renovate

| Dimension | **Dependabot** (GitHub-native) | **Renovate** (Mend / Open Source) |
| :--- | :--- | :--- |
| **Setup & Hosting** | Built into GitHub; single `.github/dependabot.yml` file. | GitHub App, self-hosted runner, or scheduled GitHub Action. |
| **Multi-Repo Enterprise Config** | Per-repo `dependabot.yml` (org-level security defaults available). | **Shareable org presets** (`extends: ["github>org/renovate-config"]`). |
| **Grouping & Noise Control** | Supports `groups` (e.g., minor/patch devDependencies) & PR limits. | Highly granular grouping, custom schedules, lockfile maintenance. |
| **Supply-Chain Cooldown** | Limited cooldown options. | Built-in `minimumReleaseAge` (wait N days before adopting new releases). |
| **Custom Managers** | Built-in ecosystems (`npm`, `docker`, `github-actions`, etc.). | Built-in ecosystems + **custom regex managers** for arbitrary files. |
| **Token & Auth Model** | Managed `dependabot[bot]` identity; no PATs to manage. | Requires GitHub App token (recommended) or PAT for self-hosted runs. |

---

## 4. Why Dependabot Here (and When to Pick Renovate)

**Why we chose Dependabot for this project:**
- Zero external infrastructure or token rotation overhead.
- Native integration with GitHub Security Advisories & Dependabot Alerts.
- Covers all 3 ecosystems (`npm`, `docker`, `github-actions`) cleanly in one file.

**When an enterprise should consider Renovate instead:**
- Managing 50+ microservices that need **one centralized preset**.
- Requiring `minimumReleaseAge` (e.g., 3–7 day cooldown) to mitigate newly published malicious packages.
- Updating non-standard version pins (e.g., tool versions in shell scripts or Helm/Terraform regexes).

---

## 5. Our Dependabot Setup (`.github/dependabot.yml`)

Configured with `version: 2` to monitor all three ecosystems every **Monday at 06:00 (`Europe/Zurich`)**:

- **`npm`** (`package.json` & `package-lock.json`):
  - Max 5 open PRs, labeled `dependencies` + `npm`, commit prefix `chore(deps)`.
  - Groups `minor`/`patch` updates separately for **dev** and **production** dependencies to cut PR noise; **major** updates stay individual for careful review.
- **`docker`** (`Dockerfile`):
  - Tracks `node:24-alpine` base image updates (max 3 open PRs, prefix `chore(docker)`).
- **`github-actions`** (`.github/workflows/`):
  - Groups all action updates into a single PR (max 3 open PRs, prefix `chore(ci)`).

---

## 6. CI Verification Workflow (`.github/workflows/ci.yml`)

Every `push` and `pull_request` to `main` (including Dependabot PRs) runs the `Build, Lint & Test` job:

- **Least-Privilege Token**: Top-level `permissions: contents: read` only.
- **Hardened Install**: Runs `npm ci --ignore-scripts` on Node 24 to prevent arbitrary lifecycle scripts from executing in CI.
- **Quality Gates**: Executes `npm run lint`, `npm test`, and `npm run build` before any PR can be merged.

---

## 7. Auto-Merge Workflow (`.github/workflows/dependabot-auto-merge.yml`)

Dependabot PRs receive a **read-only token** on `pull_request`, so auto-merge runs safely via `workflow_run`:

- **Triggered After CI**: Runs only when the `CI` workflow succeeds for a `dependabot[bot]` PR.
- **Minimal Privilege & Isolation**: Grants write permissions only to the merge job and never checks out PR code.
- **Selective Auto-Merge**: Auto-merges `minor`/`patch` updates; `major` updates still require human approval.

---

## 8. Required GitHub Repository Settings

Three repository settings are needed for CI gating and auto-merge to work:

1. **Allow Auto-Merge** — Enable **Allow auto-merge** (and optional auto-delete of head branches) in repository Pull Request settings.
2. **Protect `main` Branch** — Require pull requests and passing **`Build, Lint & Test`** status checks before merging.
3. **Enable Dependabot Security** — Turn on **Dependabot alerts** and **Dependabot security updates** in Code Security settings.

---

## 9. How Dependabot Works Day-to-Day

1. **Weekly Scan (Mondays)** — Checks `npm`, Docker, and GitHub Actions for updates.
2. **Grouped PRs** — Opens batched PRs for minor/patch updates; keeps major updates separate.
3. **CI Validation** — Automatically runs lint, tests, and build on every PR.
4. **Merge Decision** — Auto-merges green minor/patch PRs; holds major updates for human review.
5. **Security Alerts** — Opens CVE fix PRs immediately without waiting for the weekly schedule.

---

## 10. Under the Hood: How Dependabot Knows What's New & Safe

- **Finding Updates**: Parses your manifests/lockfiles and queries upstream registries (npm, Docker registries, GitHub releases) in an isolated runner to compute valid lockfile updates.
- **Detecting Vulnerabilities**: Matches your dependency graph against the **GitHub Advisory Database** (CVEs) to prioritize patched versions.
- **Predicting Breakage**: Shows a **Compatibility Score** based on anonymized CI pass/fail rates for that same update across public GitHub repositories.
- **Final Safety Gate**: Dependabot does not guarantee runtime safety on its own—**your CI checks** and **SemVer policies** (e.g., holding major bumps for review) are what validate the update.

---

## 11. Security & Supply-Chain Hygiene

Key guardrails for enterprise dependency automation:

- **Least-Privilege Permissions**: Grant read-only access by default; scope write permissions strictly to the merge job.
- **Secrets Isolation**: Never check out untrusted PR code in privileged workflows (e.g., avoid `pull_request_target` + checkout).
- **Managed Identities**: Rely on Dependabot's built-in bot token (or a GitHub App for Renovate) instead of long-lived PATs.
- **Supply-Chain Hardening**: Block install scripts in CI, pin third-party Actions to commit SHAs, and enforce branch protection on `main`.
