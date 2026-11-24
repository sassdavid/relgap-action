# relgap-action

A composite GitHub Action that installs a verified [relgap](https://github.com/sassdavid/relgap)
release on Linux, macOS and Windows runners. In report mode it runs `relgap outdated`,
`relgap drift` or `relgap audit` and sends the findings to code scanning. In comment mode it
digests the tools a pull request upgrades, or audits the pinned versions, and keeps one comment
on the pull request.

The action is on the [GitHub Marketplace](https://github.com/marketplace/actions/relgap).

## Usage

Pin the action by commit SHA. Each `vX.Y.Z` release is immutable. The `v1` tag moves to the
newest 1.x release and has no release of its own.

### Install

```yaml
steps:
  - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
  - uses: sassdavid/relgap-action@518ce61cef05d84d2bd980c553bd24fdca9ed0c7 # v1.2.0
  - run: relgap outdated
```

### Report

One `relgap outdated` (the default), `relgap drift` or `relgap audit` run writes JSON, Markdown,
SARIF and annotations. The action uploads the SARIF to code scanning and appends the Markdown to
the step summary. A `fail-on` match exits 3 and fails the step after the upload.

```yaml
on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: "23 6 * * 1"

permissions: {}

jobs:
  outdated:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write # the SARIF upload
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: sassdavid/relgap-action@518ce61cef05d84d2bd980c553bd24fdca9ed0c7 # v1.2.0
        with:
          mode: report
          fail-on: eol, vuln
```

The alerts are under Security > Code scanning, in the category `relgap-outdated` (or
`relgap-drift`, `relgap-audit`). With a `working-directory` other than the repository root, the
category ends with `/<directory>`. SARIF locations are relative to `working-directory`, so code scanning links
to the right file only when that directory is the repository root.

### Audit

`command: audit` lists the known vulnerabilities of every pinned tool version, with the version
that fixes each, and the pins OSV can't check with the reason. It needs relgap 0.13.0 or later,
the version this action installs by default. Unlike `outdated` and `drift`, `audit` gates `vuln`
when `fail-on` is empty, as audit tools do, so a finding fails the step. Name the gates in
`fail-on` (`vuln`, `unchecked`, `kev`) to change them, or narrow `vuln` in `args`:

```yaml
      - uses: sassdavid/relgap-action@518ce61cef05d84d2bd980c553bd24fdca9ed0c7 # v1.2.0
        with:
          mode: report
          command: audit
          args: |
            --vuln-severity
            high
```

To report the findings without failing the step, turn the gate off with an empty `--fail-on`:

```yaml
      - uses: sassdavid/relgap-action@518ce61cef05d84d2bd980c553bd24fdca9ed0c7 # v1.2.0
        with:
          mode: report
          command: audit
          args: --fail-on=
```

In comment mode, `command: audit` keeps one comment with the audit on the pull request. It opens
when the audit has a finding or an error, or when a pin is unchecked because OSV failed or the
name resolves to no tool. Images, charts and other pins without an advisory source don't open it.

### Comment

On a `pull_request` event, `relgap digest --pull-request` digests each tool whose pinned version
the pull request changes, from the base version to the head version. On a Renovate pull request it
digests only the dependency the branch names. `relgap notify` keeps one comment with the digest:
a re-run edits it, and only when the digest changed.

A pull request from a fork or from Dependabot gets a read-only `GITHUB_TOKEN`. The action then
skips the comment, says why in a notice, and writes the digest to the step summary.

```yaml
on:
  pull_request:
    paths: ["mise.toml", ".tool-versions", "**/*.tf", "**/*.hcl", "**/Dockerfile"]

permissions: {}

jobs:
  digest:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write # the comment
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          fetch-depth: 0 # relgap reads the pins at the base commit
          persist-credentials: false
      - uses: sassdavid/relgap-action@518ce61cef05d84d2bd980c553bd24fdca9ed0c7 # v1.2.0
        with:
          mode: comment
          fail-on: breaking
```

## Inputs

| Input | Default | Meaning |
|---|---|---|
| `version` | the pin in `scripts/tool.env` | relgap version: `X.Y.Z`, `vX.Y.Z` or `latest`; comment mode needs 0.13.1 or later |
| `verify` | `attestation` | the check after sha256: `attestation`, `cosign` or `checksum` |
| `mode` | `install` | `install`, `report` or `comment` |
| `command` | `outdated` in report mode, `digest` in comment mode | report takes `outdated`, `drift` or `audit`, comment takes `digest` or `audit` |
| `args` | | more arguments for the command, one per line |
| `from` | | not used by relgap; a value exits 2 |
| `fail-on` | | `--fail-on` values, separated by commas or newlines |
| `sarif` | `true` | upload the SARIF to code scanning |
| `step-summary` | `true` | append the Markdown to the step summary |
| `github-token` | `${{ github.token }}` | token for `gh attestation verify`, relgap's GitHub API calls, the SARIF upload and the comment |
| `working-directory` | `.` | directory the command runs in |

A bad input exits 2 before the install: an unknown mode or command, `command`, `args`, `from`
or `fail-on` in install mode, or comment mode with a `version` below 0.13.1. Older relgap
releases can notify upstream authors from the comment: 0.4.0 and older quote release notes with
their user mentions and issue references, and 0.5.0 to 0.13.0 miss some of them, such as one
after a non-ASCII letter or written as a character reference.

## Outputs

| Output | Meaning |
|---|---|
| `version` | the installed version, without a leading `v` |
| `path` | absolute path of the relgap binary |
| `exit-code` | the command's exit code: 0, 1 (a source failed), 2 (usage), 3 (`fail-on` matched) or 4 (auth) |
| `json-file` | absolute path of the JSON report |
| `sarif-file` | absolute path of the SARIF file (`outdated`, `drift` and `audit` only) |
| `md-file` | absolute path of the Markdown report |

The step fails with the command's exit code. In comment mode a refused comment fails it with
notify's code when the command passed.

## Verification

The action checks every archive against the release's `checksums.txt` (sha256), then runs the
check that `verify` names. There is no option to skip verification.

- `attestation` runs `gh attestation verify` with `--signer-workflow
  sassdavid/relgap/.github/workflows/release.yml` and `--deny-self-hosted-runners`. GitHub-hosted
  runners have `gh`.
- `cosign` runs `cosign verify-blob` on `checksums.txt` with the release's
  `checksums.txt.sigstore.json`, for a certificate from relgap's release workflow. It uses
  `cosign` from PATH. Without it, mise installs the cosign version that `scripts/install.sh`
  pins. Without either, the step fails.
- `checksum` stops after sha256. It catches a broken download, not a replaced release.

## Development

```sh
mise run check          # jactionlint, shellcheck, prose lint, install tests (network and gh), run tests
mise run sibling:diff   # compare the files shared with tfcensus-action and relgap
```

CI runs the install and run tests and the action itself on ubuntu, macos and windows, and zizmor in
`security.yml`. `renovate.yml` runs Renovate weekly as the `sassdavid-renovate` GitHub App, and
`renovate-config.yml` validates `renovate.json` on pull requests. `scripts/test-run.sh` runs
`scripts/run.sh` with a stub binary: the inputs, the arguments each mode builds, the outputs, the
exit codes, and the comment rules for forks and Dependabot. Report and comment mode run for real on
[sassdavid/action-test](https://github.com/sassdavid/action-test).

## License

Apache-2.0. See [LICENSE](LICENSE).
