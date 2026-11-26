# DevSecOps Security Review

**Date:** 26 November 2026 ? **Tools:** Snyk Web UI and Trivy 0.76.2

Forked and cloned [snyk/snyk-apps-demo](https://github.com/snyk/snyk-apps-demo).
The current revision had no Critical findings in Trivy, so this exercise uses the
historical upstream commit [273f07a](https://github.com/snyk/snyk-apps-demo/commit/273f07abee422f44d730638f6c78499b224d5969)
on the clearly labeled `training-baseline` branch. The `main` branch retains the
current upstream version for comparison.

## 1. Code and dependency scanning

Imported the fork into the **Snyk online scanner** and verified that the target
reference was `training-baseline`. Snyk analyzed `package.json` and
`package-lock.json`, reporting **4 Critical, 26 High, 36 Medium, and 5 Low issues**.
[Open the scanned project](https://app.snyk.io/org/tsurts/project/30eb2b24-3be2-460a-b7fb-f09a59973826)
(Snyk sign-in required).

### One critical vulnerability

| Item | Finding |
| --- | --- |
| Package | `form-data@4.0.0`, a direct dependency |
| Issue | Predictable Value Range from Previous Values |
| Severity | **Critical ? CVSS v4.0 9.4** |
| Identifiers | **CVE-2025-7783**, **SNYK-JS-FORMDATA-10841150** |
| Location | `package.json` / resolved version in `package-lock.json` |
| Minimum fix for this issue | Upgrade the 4.x dependency to **4.0.4 or later** |

The vulnerable package uses predictable randomness for multipart request
boundaries. Under the conditions described by the advisory, an attacker who
controls request content and can predict the boundary may inject additional
parameters into an outgoing request. This is a dependency finding; exploitation
of this application was not tested.

**How to fix:** update `form-data` to at least **4.0.6** on the 4.x line, regenerate
the lockfile, test the app, and rescan. Version 4.0.4 fixes this specific Critical
issue, but the same Snyk scan recommends 4.0.6 for a separate CRLF issue.
Also update the parent dependencies that bring in vulnerable transitive copies.
[Official Snyk advisory](https://security.snyk.io/vuln/SNYK-JS-FORMDATA-10841150).

The following screenshot shows the actual Snyk finding, affected version,
dependency path, severity, and fix:

![Critical form-data vulnerability in Snyk](docs/devsecops-assignment/evidence/snyk-critical.png)

[Scan overview screenshot](docs/devsecops-assignment/evidence/snyk-overview.png)
? [Exported online results](docs/devsecops-assignment/evidence/snyk-online-result.txt)

Snyk successfully created the dependency project. Its separate import attempt
for `tsconfig.json` reported a JSON parsing error and was not counted as a
successful scan.

### Trivy results

Ran `trivy fs --include-dev-deps --scanners vuln,secret` against both revisions.

| Revision | Critical | High | Medium | Low | Secret findings |
| --- | ---: | ---: | ---: | ---: | ---: |
| Historical training baseline `273f07a` | 6 | 52 | 41 | 15 | 0 |
| Current upstream `aaf1b69` | 0 | 15 | 8 | 3 | 0 |

Trivy counts are package-version/advisory findings, not distinct CVEs.
Snyk and Trivy use different vulnerability data, severity assessments, and
dependency coverage, so their totals are recorded separately.

Evidence: [baseline report](docs/devsecops-assignment/evidence/trivy-baseline.txt),
[current report](docs/devsecops-assignment/evidence/trivy-current.txt),
and [exact revisions, commands, and scan times](docs/devsecops-assignment/evidence/scan-metadata.json).
Machine-readable JSON reports are in the same evidence folder.

## 2. Secrets hygiene

Manual review found credential values in **historical `.env.local` files**:

| Location | Exposed configuration |
| --- | --- |
| Commit `14a8456fb802d3682ed457ac6b8e6a3fc5aebc0d`, `.env.local:3` and `:6` | `CLIENT_SECRET` and `ENCRYPTION_SECRET` |
| Commit `875a53ff6c7624afc83b9ef2b2c8b8b98324a7ce`, `.env.local:3` and `:6` | `CLIENT_SECRET` and `ENCRYPTION_SECRET` |

The current upstream [.gitleaksignore](https://github.com/snyk/snyk-apps-demo/blob/aaf1b69929e66b881dbb013896e0cb28ec43017c/.gitleaksignore)
describes these old local-development values as expired and unusable.
Their validity was not tested. Values are omitted from this submission;
[redacted evidence](docs/devsecops-assignment/evidence/secrets-redacted.json)
records the keys, paths, and commit IDs.

The current `.env.example` has empty placeholders, and the test string
`'secret'` is a fixture, not evidence of a live credential. Trivy found no
secrets in the checked-out files; that filesystem result does not cover all Git history.

**Alternatives:** inject credentials through environment variables backed by
a secret manager or CI secret store. Keep local `.env` files out of Git,
commit only empty examples, and restrict access to local secret files.
Rotate/revoke exposed live credentials, review their use, and add secret scanning
before commits and in CI. Deleting a file from the latest commit does not remove
it from earlier commits.
[Reference: Twelve-Factor configuration](https://12factor.net/config).

## 3. Compliance awareness ? CIS Docker

**Rule: run the container application as a non-root user.**

Create a dedicated user and select it with the Dockerfile `USER` instruction.<br>
Give that user only the file permissions the application needs.<br>
This limits the impact of a compromised application process.<br>
Check runtime overrides too, so deployment does not switch it back to root.

**Pipeline enforcement:** for a containerized deployment, run Trivy configuration
scanning and fail the build when the non-root check
[AVD-DS-0002](https://avd.aquasec.com/misconfig/dockerfile/general/avd-ds-0002/) fails.
Additionally inspect the built image's effective user and reject root at runtime.
This is a proposed enforcement method; no container compliance scan was claimed
for this repository.

Sources: [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
and [Docker Bench for Security, check 4.1](https://github.com/docker/docker-bench-security/blob/master/tests/4_container_images.sh).

## 4. Reflection

The most insightful part was seeing that security depends on both the current
code and its history. The current checkout had no Critical Trivy findings,
while an older revision still contained several, and old credential values
remained in Git history after the environment file was removed. Snyk and Trivy
also reported different totals, which showed why scan scope and severity
sources matter. Updating dependencies, protecting secrets, and checking
configuration need to be continuous habits rather than one-time checks.

---

[Original demo README](docs/devsecops-assignment/upstream-README.md)
