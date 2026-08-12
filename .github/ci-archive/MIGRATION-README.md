# Jenkins to GitHub Actions Migration Report

## Summary

This repository contained four Jenkins pipeline definitions used as test cases for
CI/CD migration tooling. All four have been converted to equivalent GitHub Actions
workflows. The original `Jenkinsfile`s have been archived (not deleted) under
`.github/ci-archive/` for reference.

| # | Original Jenkinsfile | New Workflow | Pipeline Type |
|---|-----------------------|--------------|----------------|
| 1 | `Jenkinsfile` | `.github/workflows/jenkins-plugin-build.yml` | Shared-library (`buildPlugin`) matrix build |
| 2 | `javanexustomcat/Jenkinsfile` | `.github/workflows/java-nexus-tomcat.yml` | Declarative |
| 3 | `dockermaven/Jenkinsfile` | `.github/workflows/docker-maven.yml` | Declarative (Docker agent) |
| 4 | `laravelgithubapi/Jenkinsfile` | `.github/workflows/laravel-github-api.yml` | Declarative + scripted (`script {}`) blocks |

## 1. `Jenkinsfile` → `jenkins-plugin-build.yml`

**Original:** Used the `buildPlugin()` step from the
[jenkins-infra/pipeline-library](https://github.com/jenkins-infra/pipeline-library/)
shared library with two configurations: `linux`/JDK 21 and `windows`/JDK 17.

**Conversion:**
- The shared library call was expanded inline into a GitHub Actions build matrix.
- `agent { label 'linux' }` / `label 'windows'` → `runs-on: ubuntu-latest` /
  `runs-on: windows-latest`.
- JDK setup performed via `actions/setup-java`.
- The library's internal `mvn -B clean verify` build step was reproduced directly.

**Triggers:** `push` to `main`, `pull_request` to `main`, manual `workflow_dispatch`.

**Secrets / variables required:** None.

## 2. `javanexustomcat/Jenkinsfile` → `java-nexus-tomcat.yml`

**Original:** Declarative pipeline running on a Windows agent (`bat` steps), building
with Maven, running SonarQube analysis, uploading the built WAR to Nexus, downloading
it back, and deploying to Tomcat.

**Conversion:**
- `agent any` with `bat` steps → `runs-on: windows-latest` with `shell: cmd` / `shell: pwsh` steps.
- `tools { maven; jdk 'jdk8' }` → `actions/setup-java` with `java-version: '8'` (Maven ships with the runner).
- `nexusArtifactUploader` plugin step → replaced with direct `curl` upload calls to the
  Nexus REST API (no verified marketplace action exists for this plugin).
- `withCredentials([[$class: 'UsernamePasswordMultiBinding', ...]])` → `secrets.NEXUS_USERNAME` / `secrets.NEXUS_PASSWORD` env vars.
- `deploy adapters: [tomcat9(...)]` (Deploy to Container plugin) → replaced with a
  `curl` call to the Tomcat Manager text API, using `secrets.TOMCAT_USERNAME` / `secrets.TOMCAT_PASSWORD`.

**Credential / secret mapping:**

| Jenkins credential | GitHub Actions secret |
|---|---|
| `sonarqube-token` | `SONARQUBE_TOKEN` |
| `tomcat-user` | `TOMCAT_USERNAME`, `TOMCAT_PASSWORD` |
| `nexus-user` | `NEXUS_USERNAME`, `NEXUS_PASSWORD` |

**Repository variables (optional overrides):** `NEXUS_URL`, `NEXUS_REPOSITORY`,
`NEXUS_PROTOCOL`, `NEXUS_VERSION`.

**Triggers:** `push` to `main`, `pull_request` to `main`, manual `workflow_dispatch`.

## 3. `dockermaven/Jenkinsfile` → `docker-maven.yml`

**Original:** Declarative pipeline with a Docker agent (`maven:3.5.0-jdk-8`), building,
testing (with JUnit result publishing), and packaging a Maven project.

**Conversion:**
- `agent { docker { image 'maven:3.5.0-jdk-8'; args '-v /tmp:/tmp -p 8000:8000' } }` →
  `jobs.build.container` with the same image and the `/tmp` volume mount (the port
  mapping is not needed for a build job and was dropped).
- `environment { MAVEN_OPTS = '-Xmx2048m' }` → `env.MAVEN_OPTS` at job level.
- `publishTestResults` (JUnit plugin) → `dorny/test-reporter` action reading the same
  `target/surefire-reports/*.xml` pattern, run with `if: always()` to mirror `post { always { ... } }`.
- `archiveArtifacts artifacts: 'target/*.jar', fingerprint: true` → `actions/upload-artifact`.

**Secrets / variables required:** None.

**Triggers:** `push` to `main`, `pull_request` to `main`, manual `workflow_dispatch`.

## 4. `laravelgithubapi/Jenkinsfile` → `laravel-github-api.yml`

**Original:** Declarative pipeline with `script {}` blocks: checks out code, runs
Composer, sets up a unique per-build MySQL database, runs the Laravel test suite, and
in `post { always { ... } }` updates the GitHub commit status and closes the PR on
failure via raw `curl` calls to the GitHub REST API, finally dropping the test database.

**Conversion:**
- `checkout scm` → `actions/checkout`.
- `agent any` + shell steps → `runs-on: ubuntu-latest` with a MySQL `services:` container
  replacing the Jenkins agent's local MySQL install.
- `env.BUILD_NUMBER` (used to build a unique DB name) → `github.run_number`.
- `withCredentials([usernamePassword(credentialsId: MYSQL_CREDENTIALS_ID, ...)])` →
  `secrets.DB_USER` / `secrets.DB_PASS` env vars.
- Manual `curl`-based GitHub status/PR-closing logic (using `GITHUB_CREDENTIALS_ID`
  personal credentials) → replaced with the GitHub CLI (`gh api` / `gh pr close`)
  using the built-in `secrets.GITHUB_TOKEN`, removing the need for a personal
  access token credential and the manual PR-number lookup fallback.
- `post { always { ... drop database ... } }` → a dedicated `if: always()` step.

**Credential / secret mapping:**

| Jenkins credential | GitHub Actions equivalent |
|---|---|
| `mysql_credentials_id` | `DB_USER`, `DB_PASS` secrets |
| `faveobot` (GitHub PAT) | Built-in `GITHUB_TOKEN` (requires `statuses: write`, `pull-requests: write` permissions, already granted in the workflow) |

**Triggers:** `pull_request` to `main`, manual `workflow_dispatch` (the original only
ran for pull request builds via `CHANGE_ID`/`BRANCH_NAME`).

## Validation

- All four workflow files were validated with [`actionlint`](https://github.com/rhysd/actionlint) v1.7.12 with no errors or warnings.

## Actions Used

| Action | Version (pinned to SHA) |
|---|---|
| `actions/checkout` | v4 (`11d5960a326750d5838078e36cf38b85af677262`) |
| `actions/setup-java` | v4 (`cf277c60eb25467037889841efdb72551f06f6c3`) |
| `actions/upload-artifact` | v4 (`ea165f8d65b6e75b540449e92b4886f43607fa02`) |
| `shivammathur/setup-php` | v2 (`b604ade2a87db23f8871b7182e69ec5e75effb45`) |
| `dorny/test-reporter` | v1 (`3eeb9fc888e82e8be2fb356bbeec2750231672bc`) |

## Follow-up Items

- Configure the secrets/variables listed above in the repository settings before
  these workflows can run successfully end-to-end.
- The Nexus upload/download steps and Tomcat deploy step use direct REST/`curl`
  calls because no verified marketplace actions exist for the
  `nexusArtifactUploader` and `deploy adapters: [tomcat9(...)]` Jenkins plugins;
  review and adjust the endpoint URLs/paths for your actual Nexus/Tomcat instances.
- Original Jenkinsfiles are preserved under `.github/ci-archive/` for reference and
  can be removed once the new workflows are confirmed working in production.
