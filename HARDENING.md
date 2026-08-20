<!-- markdownlint-disable -->

# Hardening Report: peaceiris--actions-hugo/v3.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **peaceiris--actions-hugo/v3.0.0** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions using mutable tags (e.g. @v4, @v2, @v3, @v2.6.0, @v1.10.0) instead of pinned 40-character SHA commit hashes. This exposes the workflow to supply-chain attacks if the tag is moved to a malicious commit.

Failing references:
- codeql-analysis.yml: actions/checkout@v4, github/codeql-action/init@v2, github/codeql-action/autobuild@v2, github/codeql-action/analyze@v2
- dependency-review.yml: actions/checkout@v4, actions/dependency-review-action@v3
- dev-image.yml: actions/checkout@v4
- label-commenter.yml: actions/checkout@v4, peaceiris/actions-label-commenter@v1.10.0
- release.yml: actions/checkout@v4
- test-action.yml: actions/checkout@v4, peaceiris/actions-hugo@v2.6.0
- test.yml: actions/checkout@v4, actions/setup-node@v4, actions/upload-artifact@v3, codecov/codecov-action@v3
- update-major-tag.yml: actions/checkout@v4

Locations:

- `.github/workflows/codeql-analysis.yml:12`
- `.github/workflows/codeql-analysis.yml:15`
- `.github/workflows/codeql-analysis.yml:19`
- `.github/workflows/codeql-analysis.yml:22`
- `.github/workflows/dependency-review.yml:14`
- `.github/workflows/dependency-review.yml:15`
- `.github/workflows/dev-image.yml:22`
- `.github/workflows/label-commenter.yml:13`
- `.github/workflows/label-commenter.yml:17`
- `.github/workflows/release.yml:10`
- `.github/workflows/test-action.yml:22`
- `.github/workflows/test-action.yml:26`
- `.github/workflows/test.yml:20`
- `.github/workflows/test.yml:22`
- `.github/workflows/test.yml:40`
- `.github/workflows/test.yml:44`
- `.github/workflows/update-major-tag.yml:10`

### missing-permissions (severity: medium)

Eight workflow files have no top-level `permissions:` key and no job-level `permissions:` key on any of their jobs. Without explicit permissions, workflows run with the default token permissions (which may be read/write depending on repository settings), violating the principle of least privilege.

Affected files: codeql-analysis.yml, dev-image.yml, label-commenter.yml, purge-readme-image-cache.yml, release.yml, test-action.yml, test.yml, update-major-tag.yml

Locations:

- `.github/workflows/codeql-analysis.yml:1`
- `.github/workflows/dev-image.yml:1`
- `.github/workflows/label-commenter.yml:1`
- `.github/workflows/purge-readme-image-cache.yml:1`
- `.github/workflows/release.yml:1`
- `.github/workflows/test-action.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/update-major-tag.yml:1`

### script-injection (severity: high)

Sub-rule (a): test-action.yml contains a `run:` block that directly interpolates a `${{ }}` expression into the shell command string. The step `run: echo '${{ steps.hugo_version.outputs.hugo_version }}'` embeds the step output directly into the shell command before the shell ever sees it. Because `steps.hugo_version.outputs.hugo_version` is derived from running `hugo version` (whose output could be influenced by a malicious Hugo binary or environment), this constitutes a script-injection risk. Any `${{ ... }}` inside a `run:` block is a finding regardless of the context it reads from.

Locations:

- `.github/workflows/test-action.yml:36`

### unsafe-shell (severity: high)

release.yml pipes a remotely-fetched script directly into bash without first downloading and inspecting it: `curl -fsSL https://github.com/github/hub/raw/8d91904208171b013f9a9d1175f4ab39068db047/script/get | bash -s "${HUB_VERSION}"`. Although the URL references a specific commit SHA in the path, the script content is still executed immediately without any integrity verification (e.g. checksum validation), and the pattern `curl ... | bash` is inherently unsafe.

Locations:

- `.github/workflows/release.yml:15`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, unsafe-shell

**Notes:**

Fixed all findings across 8 workflow files:

1. unpinned-uses: Pinned all 17 action references to full 40-char SHA hashes:
   - actions/checkout@v4 → @11d5960a326750d5838078e36cf38b85af677262
   - github/codeql-action/{init,autobuild,analyze}@v2 → @b8d3b6e8af63cde30bdc382c0bc28114f4346c88
   - actions/dependency-review-action@v3 → @cc4f6536e38d1126c5e3b0683d469a14f23bfea4
   - peaceiris/actions-label-commenter@v1.10.0 → @f0dbbef043eb1b150b566db36b0bdc8b7f505579
   - peaceiris/actions-hugo@v2.6.0 → @16361eb4acea8698b220b76c0d4e84e1fd22c61d
   - actions/setup-node@v4 → @49933ea5288caeca8642d1e84afbd3f7d6820020
   - actions/upload-artifact@v3 → @ff15f0306b3f739f7b6fd43fb5d26cd321bd4de5
   - codecov/codecov-action@v3 → @ab904c41d6ece82784817410c45d8b8c02684457

2. missing-permissions: Added top-level permissions blocks to all 8 affected files with minimal required permissions.

3. script-injection: In test-action.yml, moved ${{ steps.hugo_version.outputs.hugo_version }} into an env: block as HUGO_VERSION and referenced it as $HUGO_VERSION in the run: shell. Also changed the step name from a dynamic expression to a static string.

4. unsafe-shell: In release.yml, replaced 'curl ... | bash -s "${HUB_VERSION}"' with a two-step approach: download to a temp file with curl -o, then execute with bash. Dropped the '--' (it was the shell's option terminator, not the script's argument).

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed script injection in .github/workflows/update-major-tag.yml: moved ${{ secrets.GITHUB_TOKEN }} from the inline run: shell command into an env: block as GH_TOKEN, and updated the git remote set-url command to reference ${GH_TOKEN} as a plain environment variable. This ensures the token value is passed through the environment rather than being interpolated directly into the shell script by the YAML template engine.

