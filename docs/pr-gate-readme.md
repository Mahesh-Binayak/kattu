# PR Gate Workflow

A single, consolidated **reusable** PR gate (`workflow_call`) that MOSIP services call from their
own repositories. It replaces per-repo copies of secret scanning, dependency review, workflow
linting, Trivy scanning and Codecov coverage with one pinned, centrally-maintained gate.

The gate has two tiers:

- **Tier 1 — Security (always runs, in parallel):** secret scan (gitleaks), dependency CVE review,
  actionlint, Trivy filesystem scan (sticky PR comment + SARIF upload) and Trivy config/IaC scan.
- **Tier 2 — Coverage & supply-chain (conditional):** a `detect` job checks for stack marker files
  at the locations you declare, then only the relevant jobs run — Java / Go / npm / Android / Flutter
  coverage (uploaded to Codecov), plus a non-blocking Maven dependency **PGP signature check**
  (`pgpverify`) whenever a `pom.xml` is present.

> **This document lists everything a caller must set up.** If you only remember three things:
> 1. Trigger the caller on `pull_request`.
> 2. Grant the token `pull-requests: write` and `security-events: write`.
> 3. Make secrets reachable (`secrets: inherit`) and point each `*_SERVICE_LOCATION` at the right folder.

---

## 1. Quick start

Single-stack repo (everything at the repo root):

```yaml
name: PR Gate
on:
  pull_request:
    types: [opened, reopened, synchronize]

permissions:
  contents: read
  pull-requests: write
  security-events: write

jobs:
  pr-gate:
    uses: mosip/kattu/.github/workflows/pr-gate.yml@master   # pin to a tag/SHA for production
    secrets: inherit
```

With `SERVICE_LOCATION` defaulting to `.`, the gate auto-detects whichever stacks live at the repo
root and runs only those coverage jobs.

---

## 2. Inputs

All inputs are **optional** — sensible defaults are applied. Set only what your repo needs.

| Input | Type | Default | What it does |
|---|---|---|---|
| `SERVICE_LOCATION` | string | `.` | Base path used by every stack **unless** a type-specific override below is set. |
| `JAVA_SERVICE_LOCATION` | string | `''` -> `SERVICE_LOCATION` | Folder containing `pom.xml`. |
| `GO_SERVICE_LOCATION` | string | `''` -> `SERVICE_LOCATION` | Folder containing `go.mod`. |
| `NPM_SERVICE_LOCATION` | string | `''` -> `SERVICE_LOCATION` | Folder containing `package.json`. |
| `ANDROID_SERVICE_LOCATION` | string | `''` -> `SERVICE_LOCATION` | Folder containing `settings.gradle` / `settings.gradle.kts`. |
| `FLUTTER_SERVICE_LOCATION` | string | `''` -> `SERVICE_LOCATION` | Folder containing `pubspec.yaml`. |
| `JAVA_VERSION` | number | `21` | Temurin JDK version for the Java **and** Android coverage jobs. |
| `NODE_VERSION` | string | `'20'` | Node.js version for the npm coverage job. |
| `FLUTTER_VERSION` | string | `''` (latest stable) | Flutter SDK version; empty = latest of the `stable` channel. |
| `SCAN_SEVERITY` | string | `'HIGH,CRITICAL'` | Trivy severities that are reported and gated on. |
| `trivy_comment_mode` | string | `'full'` | `full` = comment all current HIGH/CRITICAL; `diff` = comment only findings **newly introduced** by the PR. (The **gate** still fails on any current HIGH/CRITICAL in both modes — mode only affects the comment.) |
| `NPM_COVERAGE_COMMAND` | string | `'npm test -- --coverage'` | Command that runs tests and emits coverage. Override for non-Jest runners, e.g. `npx vitest run --coverage`. |

### How location resolution works

For each stack: **use the type-specific input if it is non-empty, otherwise fall back to
`SERVICE_LOCATION`.** A stack's Tier-2 job runs **only if its marker file exists** at the resolved
location:

| Stack | Marker file checked | Coverage job |
|---|---|---|
| Java | `<loc>/pom.xml` | `coverage-java` + `verify-maven-signatures` (pgpverify) |
| Go | `<loc>/go.mod` | `coverage-go` |
| npm | `<loc>/package.json` | `coverage-npm` |
| Android | `<loc>/settings.gradle` or `settings.gradle.kts` | `coverage-android` |
| Flutter | `<loc>/pubspec.yaml` | `coverage-flutter` |

If the marker isn't found, that job is skipped silently — a repo only pays for the stacks it has.

---

## 3. Secrets

All secrets are declared `required: false`, but several are needed **in practice** for the relevant
jobs to succeed. In a `workflow_call`, secrets do **not** flow automatically — use `secrets: inherit`
(recommended) or map each one explicitly.

| Secret | Needed by | Notes |
|---|---|---|
| `CODECOV_LICENSE` | **All coverage jobs** | Codecov v4+ requires a token even for public repos. The jobs use `fail_ci_if_error: true`, so a **missing token makes any coverage job go red.** Must exist for any repo with a detected stack. |
| `GITLEAKS_LICENSE` | `secret-scan` | `gitleaks-action` requires a license for **organization-owned** repos. Paid org product; reaches the workflow only via `secrets: inherit` (or explicit mapping). |
| `OSSRH_USER` / `OSSRH_SECRET` | `coverage-java`, `verify-maven-signatures` | Needed only if the Maven build resolves dependencies from OSSRH (e.g. SNAPSHOT/staging). Injected into a generated `settings.xml` under the `ossrh` server id. |

---

## 4. Caller-side requirements (checklist)

These are things the **calling workflow / caller repo** must provide — the gate cannot set them for you.

- [ ] **Trigger on `pull_request`.** The Trivy comment, pgpverify comment and dependency-review
      comment all depend on `github.event_name == 'pull_request'` and PR context. (You may add `push`
      too, but PR is required for the comment features.)
- [ ] **Grant token permissions.** A reusable workflow's token can never exceed the caller's, so set
      at the caller (workflow- or job-level):
      ```yaml
      permissions:
        contents: read           # checkout
        pull-requests: write     # dependency-review + Trivy + pgpverify PR comments
        security-events: write   # Trivy SARIF upload to the Security tab
      ```
      If your org defaults the `GITHUB_TOKEN` to read-only, these are **mandatory** or the comment /
      SARIF steps fail.
- [ ] **Make secrets reachable** — `secrets: inherit`, and ensure `CODECOV_LICENSE`,
      `GITLEAKS_LICENSE` and (for Maven) `OSSRH_USER` / `OSSRH_SECRET` exist as org/repo secrets.
- [ ] **Declare each stack's location** via the matching `*_SERVICE_LOCATION` input (see §2).
- [ ] **Satisfy the per-stack project prerequisites** below.

---

## 5. Per-stack project prerequisites

What each detected stack needs to exist **in your repo** for its coverage job to succeed:

### Java (`coverage-java`)
- `pom.xml` at the Java location.
- **JaCoCo must be configured** so `mvn verify` produces `**/target/site/jacoco/jacoco.xml` (this is
  what gets uploaded). No JaCoCo report -> nothing to upload -> `fail_ci_if_error` fails the job.
- `OSSRH_USER` / `OSSRH_SECRET` if the build pulls from OSSRH.

### Maven signature check (`verify-maven-signatures` / pgpverify)
- Runs automatically whenever a `pom.xml` is present. **Non-blocking** — it never fails the PR; it
  posts a sticky `pgpverify` comment listing unsigned / invalid / untrusted / missing-key
  dependencies.
- Expect a **large first comment** unless you configure a PGP keys map, since most transitive deps
  are unsigned. That's informational; it will not red-gate the PR.

### Go (`coverage-go`)
- `go.mod` at the Go location; tests present.
- The job runs `go test -coverprofile=coverage.out -covermode=atomic ./...`.

### npm (`coverage-npm`)
- `package.json` **and** `package-lock.json` at the npm location (the lockfile is used for caching).
- Your `NPM_COVERAGE_COMMAND` must emit **`coverage/lcov.info`**.
  - Jest: the default `npm test -- --coverage` works if `lcov` is in `coverageReporters`.
  - **Vitest:** set `NPM_COVERAGE_COMMAND: 'npx vitest run --coverage'` **and** install
    `@vitest/coverage-v8` (matching your Vitest major) as a dev dependency.

### Android (`coverage-android`)
- `settings.gradle` / `settings.gradle.kts` at the Android location; a committed Gradle wrapper.
- A `jacocoTestReport` task producing `**/build/reports/jacoco/**/*.xml` (the job runs
  `./gradlew testDebugUnitTest jacocoTestReport`).

### Flutter (`coverage-flutter`)
- `pubspec.yaml` at the Flutter location; tests under `test/`.
- The job runs `flutter pub get` then `flutter test --coverage`, producing `coverage/lcov.info`.
- Optionally pin the SDK via `FLUTTER_VERSION`.

---

## 6. Behavior on fork PRs

On pull requests from forks the `GITHUB_TOKEN` is read-only, so:

- The **Trivy** and **pgpverify** sticky-comment steps **auto-skip** (they compare head/base repo
  `full_name`, not `head.repo.fork`).
- Scans and coverage jobs still run; only the write-back (comments / SARIF) is skipped.

No caller action is needed — this is handled inside the gate.

---

## 7. Example: esignet (multi-stack)

esignet ships Java, Go and an npm UI, so it sets three locations and a Vitest coverage command:

```yaml
name: PR Gate
on:
  pull_request:
    types: [opened, reopened, synchronize]

permissions:
  contents: read
  pull-requests: write
  security-events: write

jobs:
  pr-gate:
    uses: mosip/kattu/.github/workflows/pr-gate.yml@master   # (currently Mahesh-Binayak/kattu@esignet-343 for testing)
    with:
      SERVICE_LOCATION: '.'
      JAVA_SERVICE_LOCATION: '<folder-containing-pom.xml>'    # set to the module root; '.' if root pom
      GO_SERVICE_LOCATION: 'esignet-service'
      NPM_SERVICE_LOCATION: 'oidc-ui'
      NPM_COVERAGE_COMMAND: 'npx vitest run --coverage'
    secrets: inherit
```

esignet-side prerequisites that are **not** kattu changes:
- `oidc-ui` uses Vitest (`^4.1.5`) — its coverage needs `@vitest/coverage-v8@^4.1.5` installed.
- The Java module needs JaCoCo wired so `mvn verify` emits `jacoco.xml`.
- `CODECOV_LICENSE`, `GITLEAKS_LICENSE`, `OSSRH_USER`/`OSSRH_SECRET` must be available as org secrets
  (they are, via `secrets: inherit`).

> Note: with `SERVICE_LOCATION: '.'`, if a `pom.xml` exists at the repo root the Java jobs run there
> even without `JAVA_SERVICE_LOCATION`. Set `JAVA_SERVICE_LOCATION` explicitly when the POM you want
> covered is in a subfolder.

---

## 8. What this replaces

Repos adopting the gate can retire standalone copies of: secret scanning, Trivy fs/config scanning,
dependency review, actionlint, and per-stack Codecov upload workflows — the gate covers all of them
behind one pinned entry point.
