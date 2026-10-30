# CI/CD integration examples

The detector returns the same exit codes everywhere — `0` clean, `1` high-risk, `2` medium-risk, `3` per-project scan errored (bulk mode only) — so any CI/CD platform that respects exit codes can gate on it. Examples below.

## GitHub Actions (single-project)

```yaml
- name: Shai-Hulud scan
  run: |
    chmod +x ./shai-hulud-detector.sh
    ./shai-hulud-detector.sh --save-log shai-hulud-report.log .
  # Job fails on exit 1 (high) or 2 (medium)
- uses: actions/upload-artifact@v4
  if: always()
  with:
    name: shai-hulud-report
    path: shai-hulud-report.log
```

## GitHub Actions (reusable workflow)

```yaml
jobs:
  supply-chain:
    uses: infinite-solutions-group/shai-hulud-detect/.github/workflows/scan.reusable.yml@<SHA>
    # scanner_ref duplicates the uses: SHA because github.job_workflow_sha is empty on cross-repository calls
    with: { scanner_ref: <SHA> }
    permissions: { contents: read }
  deep:
    if: github.event_name == 'schedule'
    uses: infinite-solutions-group/shai-hulud-detect/.github/workflows/scan-deep.reusable.yml@<SHA>
    with: { scanner_ref: <SHA> }
    permissions: { contents: read }
```

Why `scanner_ref`: `github.job_workflow_sha` is empty on cross-repository reusable-workflow calls, so without it the scanner would be checked out from its default branch instead of the pinned commit. It must be a full 40-hex SHA. Bump `uses:` and `scanner_ref` together: a mismatch is only detectable when `github.job_workflow_sha` is present (same-repository calls).

## GitHub Actions (bulk scan of many repos)

```yaml
- name: Bulk Shai-Hulud scan
  run: |
    chmod +x ./shai-hulud-detector.sh
    ./shai-hulud-detector.sh --bulk --bulk-output bulk-report .
- uses: actions/upload-artifact@v4
  if: always()
  with:
    name: shai-hulud-bulk-report
    path: bulk-report/
```

## GitLab CI

```yaml
security_scan:
  script:
    - chmod +x ./shai-hulud-detector.sh
    - ./shai-hulud-detector.sh --save-log shai-hulud-report.log .
  artifacts:
    when: always
    paths:
      - shai-hulud-report.log
```

## Jenkins (declarative pipeline)

```groovy
stage('Shai-Hulud scan') {
  steps {
    sh '''
      chmod +x ./shai-hulud-detector.sh
      ./shai-hulud-detector.sh --save-log shai-hulud-report.log .
    '''
    archiveArtifacts artifacts: 'shai-hulud-report.log', allowEmptyArchive: true
  }
}
```

## Custom exit-code handling

If you want to treat medium-risk as a warning rather than a failure:

```bash
#!/usr/bin/env bash
./shai-hulud-detector.sh .
case $? in
  0)
    echo "Clean — no compromised packages detected."
    ;;
  1)
    echo "HIGH RISK — blocking deployment."
    exit 1
    ;;
  2)
    echo "MEDIUM RISK — review the findings."
    # Continue with the pipeline instead of failing.
    ;;
  3)
    echo "BULK SCAN ERROR — at least one per-project scan crashed."
    exit 3
    ;;
  *)
    echo "Detector exited unexpectedly with code $?"
    exit 1
    ;;
esac
```

## Pre-commit hook

```bash
# .git/hooks/pre-commit
#!/usr/bin/env bash
./shai-hulud-detector.sh . || {
  echo "Shai-Hulud scan failed — commit aborted."
  exit 1
}
```
