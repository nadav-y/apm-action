# APM Action

A GitHub Action that installs [APM (Agent Package Manager)](https://github.com/microsoft/apm) and deploys agent primitives (instructions, prompts, skills, agents) into your CI workflows. One line. Zero config.

📖 [APM Documentation](https://microsoft.github.io/apm) · [Security Model](https://microsoft.github.io/apm/enterprise/security/) · [CI/CD Guide](https://microsoft.github.io/apm/integrations/ci-cd/)

## Usage

```yaml
- uses: microsoft/apm-action@v1
```

This installs the APM CLI, reads your `apm.yml`, and runs `apm install`.

### With options

```yaml
- uses: microsoft/apm-action@v1
  with:
    compile: 'true'                    # generate AGENTS.md after install
    apm-version: '0.7.0'              # pin a specific APM version
    working-directory: './my-project'  # custom working directory
```

### Isolated mode (inline dependencies, no apm.yml needed)

```yaml
- uses: microsoft/apm-action@v1
  with:
    isolated: 'true'
    dependencies: |
      - microsoft/apm-sample-package
```

### Setup-only mode (install CLI, then exit)

Just install the APM CLI and put it on `PATH`, like `actions/setup-node`. Run any `apm` command yourself in subsequent steps. No `apm.yml` required, no install step runs.

```yaml
- uses: microsoft/apm-action@v1
  id: apm
  with:
    setup-only: 'true'
    apm-version: '0.11.0'

- run: apm --version
- run: apm pack -o build --format plugin
```

`setup-only: true` is mutually exclusive with `pack`, `bundle`, and `bundles-file`. The action will not read `apm.yml`, run `apm install`, or deploy primitives. Sets the `apm-version` and `apm-path` outputs so downstream steps can branch on the resolved CLI.

### Bundle format (`apm` vs `plugin`)

`apm pack` supports two layouts:

- `bundle-format: apm` (default) -- produces an APM bundle containing `apm.lock.yaml` and a `.github/` (or `.claude/`) tree. Restorable by this action via `bundle:` / `bundles-file:`. **Use this when the consumer is another `microsoft/apm-action` step.**
- `bundle-format: plugin` -- produces a Claude Code plugin bundle with `plugin.json` at the root and flat primitive directories (`agents/`, `skills/`, ...). **Use this when publishing to a Claude Code marketplace.** Plugin bundles are not restorable by this action; restore them with your plugin tooling.

```yaml
- uses: microsoft/apm-action@v1
  with:
    pack: 'true'
    bundle-format: 'plugin'    # opt-in; default is 'apm'
```

The `bundle-format` output reflects the format of the produced or restored bundle.

### Pack mode (produce a bundle)

Install dependencies, scan for hidden Unicode threats, and pack into a self-contained `.tar.gz` archive. Add `audit-report` to generate a SARIF report alongside the bundle:

```yaml
- uses: microsoft/apm-action@v1
  id: pack
  with:
    pack: 'true'
    target: 'copilot'
    audit-report: true

- uses: github/codeql-action/upload-sarif@v3
  if: always() && steps.pack.outputs.audit-report-path
  with:
    sarif_file: ${{ steps.pack.outputs.audit-report-path }}
    category: apm-audit

- uses: actions/upload-artifact@v4
  with:
    name: agent-bundle
    path: ${{ steps.pack.outputs.bundle-path }}
```

This works with all modes — `isolated`, inline `dependencies`, or from `apm.yml`.

### Pack with marketplace artifacts (publishing flow)

When `apm.yml` declares an `outputs:` map (vendor-format marketplace files), forward the pack-time controls so CI emits exactly the right files for your release:

```yaml
- uses: microsoft/apm-action@v1
  id: pack
  with:
    pack: 'true'
    archive: 'true'
    marketplace: 'claude,codex'      # which formats to emit (default: all from outputs:)
    json-output: 'pack.json'         # capture --json report for downstream steps
    offline: 'true'                  # hermetic build using apm.lock.yaml
    include-prerelease: 'false'      # (default) skip pre-release tags

- name: Stage marketplace artifacts for the release
  run: |
    cat ${{ steps.pack.outputs.pack-json }}
    # bundle-path is empty for marketplace-only projects; use pack-json
    # to enumerate bundles + marketplace files + sidecars uniformly.
```

`marketplace-path` overrides where each format file is written, useful when you need a vendor-expected filename in the release artifact set:

```yaml
- uses: microsoft/apm-action@v1
  with:
    pack: 'true'
    marketplace-path: |
      claude=marketplace.json
      codex=plugins.toml
```

**Vendor-neutral by design.** This action does not assume which downstream CLI consumes the marketplace files. It produces the artifacts your `apm.yml` `outputs:` map declares; how consumers install them is a separate concern. See the `apm marketplace init` scaffold for guidance on which formats to declare for which consumer ecosystems.

<a id="release-mode"></a>
### Release mode (one-step tag publish)

Collapse the canonical release pipeline -- gate, matrix-pack, sha256 sidecars, marketplace.json drift detection, `gh release create` -- into a single step. Triggered by tag pushes; vendor-neutral underneath (the CLI primitives work identically in GitLab CI, Jenkins, ADO).

```yaml
on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: microsoft/apm-action@v1
        with:
          mode: release
          # release-tag defaults to GITHUB_REF_NAME
          # release-prerelease: auto  (detects -rc/-alpha/-beta/-pre suffix)
```

What it does, in order:

1. `apm pack --check-versions --check-clean --json` -- fails the job on misaligned per-package versions or uncommitted marketplace.json drift.
2. Detect repo shape (`aggregator` if `plugins/<name>/apm.yml` files exist, otherwise `single-plugin`).
3. Matrix-pack every package with `apm pack --offline --archive` -> tarballs in `dist/`.
4. Write `<tarball>.sha256` sidecars next to each tarball.
5. _(Optional)_ Publish each package to an APM registry via `apm publish` (see below).
6. Stage `marketplace-<version>.json` for aggregator shapes.
7. Render a GitHub Step Summary table of the release contents.
8. `gh release create <tag> <files...>` (skipped if `release-skip-publish: true`).

Outputs: `packages` (JSON), `marketplace-drift`, `release-url`, `release-tag`, `registry-publish-results` (JSON).

For the non-GitHub-Actions equivalent (the same primitives wrapped in `.gitlab-ci.yml`, `Jenkinsfile`, ADO `azure-pipelines.yml`), see [`producer/releasing-from-any-ci.md`](https://microsoft.github.io/apm/producer/releasing-from-any-ci/).

#### Registry publish (optional, experimental)

Set `release-registry-publish: true` to also push each package to an APM registry after packing. This is **additive** — the GitHub Release is still created as normal.

```yaml
      - uses: microsoft/apm-action@v1
        with:
          mode: release
          release-registry-publish: true
          release-registry-name: corp-main        # optional; required only when multiple registries are configured
          release-registry-package: acme/web-skills  # OWNER/REPO identity (single-plugin)
        env:
          APM_REGISTRY_TOKEN_CORP_MAIN: ${{ secrets.APM_REGISTRY_TOKEN }}
```

For aggregator repos (multiple plugins under `plugins/`), each plugin's `apm.yml` `name` is used as the package identifier automatically — omit `release-registry-package`. For single-plugin repos, `release-registry-package` is required.

For authentication details (env-var naming, Bearer vs Basic, `~/.apm/config.json`) see the [APM registries guide](https://microsoft.github.io/apm/guides/registries/).

> **Feature gate:** the `registries` experimental feature is enabled automatically when `release-registry-publish: true`. No manual `apm experimental enable registries` step needed.

Use `release-registry-dry-run: true` to validate the publish without uploading (useful on PR branches).

### Restore mode (verified extraction)

Restore primitives from a bundle. The action installs APM (cached across runs) and uses `apm unpack` for integrity verification — no Python, minimal network. Only files listed in the bundle's lockfile (`deployed_files`) are written to `working-directory`; the lockfile and `apm.yml` themselves are not, so the workspace stays clean for downstream steps such as `git checkout`.

```yaml
- uses: actions/download-artifact@v4
  with:
    name: agent-bundle

- uses: microsoft/apm-action@v1
  with:
    bundle: './*.tar.gz'
```

<a id="multi-bundle-restore"></a>
### Multi-bundle restore (multi-org / multi-app)

**Why:** when you fan out a `pack` job across N GitHub Apps (or N orgs, or N teams) you end up with N separate bundle artifacts. Without `bundles-file`, the consumer job has to call `microsoft/apm-action@v1` N times in sequence, which adds latency and obscures which install came from which source. `bundles-file` lets a single restore step merge all N bundles into one workspace in caller-specified order. See [issue #29](https://github.com/microsoft/apm-action/issues/29) for the full rationale and diagrams.

**Backward compatibility:** existing single-`bundle` callers are unaffected. `bundles-file` is a new opt-in input; `pack`, `bundle`, and `bundles-file` are mutually exclusive (the action errors if more than one is set).

```yaml
# In a downstream job that consumes all bundles:
- uses: actions/download-artifact@v4
  with:
    pattern: apm-*
    path: /tmp/bundles

- run: find /tmp/bundles -name '*.tar.gz' | sort > /tmp/bundle-list.txt

- uses: microsoft/apm-action@v1
  id: restore
  with:
    bundles-file: /tmp/bundle-list.txt
    working-directory: /tmp/agent-workspace

- run: echo "Merged ${{ steps.restore.outputs.bundles-restored }} bundles into the workspace"
```

The `bundles-restored` output reports the integer count of bundles successfully merged, which is convenient for assertions and logging in downstream steps.

**Collision policy:** bundles are applied in list order; on file conflicts, later bundles overwrite earlier bundles. The action logs an explicit warning naming the bundle count before the restore loop begins, so the policy is never silent. Per-file SHA-aware collision detection is planned for v1.6.0.

### Cross-job artifact workflow

Pack once, restore everywhere — identical primitives across all consumer jobs.

```yaml
jobs:
  agent-config:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: microsoft/apm-action@v1
        id: pack
        with:
          pack: 'true'
          target: 'copilot'
      - uses: actions/upload-artifact@v4
        with:
          name: agent-bundle
          path: ${{ steps.pack.outputs.bundle-path }}

  lint:
    needs: agent-config
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: agent-bundle
      - uses: microsoft/apm-action@v1
        with:
          bundle: './*.tar.gz'
      # .github/ is ready — primitives deployed

  deploy:
    needs: agent-config
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: agent-bundle
      - uses: microsoft/apm-action@v1
        with:
          bundle: './*.tar.gz'
      # Same primitives, different job. Byte-identical.
```

### Security scanning

`apm install` automatically blocks packages with critical hidden-character findings — no configuration needed. Add `audit-report` for visibility: a SARIF report for [Code Scanning](https://docs.github.com/en/code-security/code-scanning) annotations and a markdown summary in `$GITHUB_STEP_SUMMARY`. See the [APM security model](https://microsoft.github.io/apm/enterprise/security/) for details.

```yaml
- uses: microsoft/apm-action@v1
  id: apm
  with:
    audit-report: true
- uses: github/codeql-action/upload-sarif@v3
  if: always() && steps.apm.outputs.audit-report-path
  with:
    sarif_file: ${{ steps.apm.outputs.audit-report-path }}
    category: apm-audit
```

## Private repo authentication

By default, `github-token` (which defaults to `${{ github.token }}`) is automatically forwarded to APM as `GITHUB_APM_PAT`. This means same-org private repos work with zero config.

```yaml
# Same-org private repos: zero config
- uses: microsoft/apm-action@v1
```

For cross-org private repos, pass a PAT with broader scope via the `github-token` input:

```yaml
# Cross-org private repos: pass a broader-scoped PAT
- uses: microsoft/apm-action@v1
  with:
    github-token: ${{ secrets.APM_PAT }}
```

For multi-org or multi-platform scenarios, use the `env:` block for full control. An explicit `GITHUB_APM_PAT` in `env:` always wins over the auto-forwarded value. (For the matrix-based fan-out pattern that pairs one App per matrix replica with [`bundles-file:`](#multi-bundle-restore), see [issue #29](https://github.com/microsoft/apm-action/issues/29).)

```yaml
# Multi-org / multi-platform: full control via env block
- uses: microsoft/apm-action@v1
  env:
    GITHUB_APM_PAT: ${{ secrets.APM_PAT }}
    GITHUB_APM_PAT_CONTOSO: ${{ secrets.APM_PAT_CONTOSO }}
    ADO_APM_PAT: ${{ secrets.ADO_PAT }}
    ARTIFACTORY_APM_TOKEN: ${{ secrets.ARTIFACTORY_TOKEN }}
```

> **Note:** GitHub Actions forbids secrets named with the `GITHUB_` prefix, so you cannot create a secret called `GITHUB_APM_PAT` directly. The auto-forward from `github-token` covers the common case. For cross-org tokens, name your secret something like `APM_PAT` and pass it via `github-token` or `env: GITHUB_APM_PAT`.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `working-directory` | No | `.` | Working directory for execution. Must exist in non-isolated mode (with your `apm.yml`). In `isolated`, `pack`, or `bundle` modes the directory is created automatically. |
| `apm-version` | No | `latest` | APM version to install |
| `github-token` | No | `${{ github.token }}` | GitHub token for API calls. Auto-forwarded as `GITHUB_APM_PAT` so same-org private repos work with zero config. Pass a broader-scoped PAT for cross-org access. |
| `script` | No | | APM script to run after install |
| `dependencies` | No | | YAML array of extra dependencies to install (additive to apm.yml) |
| `isolated` | No | `false` | Ignore apm.yml and clear pre-existing primitive dirs — install only inline dependencies |
| `compile` | No | `false` | Run `apm compile` after install to generate AGENTS.md |
| `pack` | No | `false` | Pack a bundle after install (produces `.tar.gz` by default) |
| `bundle-format` | No | `apm` | Bundle layout when `pack: true`. `apm` produces an APM bundle (with `apm.lock.yaml` and a `.github/` tree, restorable by this action). `plugin` produces a Claude Code plugin bundle (with `plugin.json` at the root, intended for marketplace consumption). |
| `setup-only` | No | `false` | Install the APM CLI and exit. No `apm.yml` is read, no `apm install` runs, no primitives are deployed. Mutually exclusive with `pack`, `bundle`, and `bundles-file`. |
| `bundle` | No | | Restore from a bundle (local path or glob). Installs APM and unpacks via `apm unpack` (verified). |
| `bundles-file` | No | | Path to a UTF-8 text file with one bundle path per line. Restores N bundles into a single workspace in caller-specified order (last wins on collisions). Mutually exclusive with `pack` and `bundle`. |
| `target` | No | | Bundle target: `copilot`, `vscode`, `claude`, or `all` (used with `pack: true`) |
| `archive` | No | `true` | Produce `.tar.gz` instead of directory (used with `pack: true`) |
| `marketplace` | No | | Forwarded to `apm pack --marketplace=<value>` (used with `pack: true`). Comma-separated format list (`claude,codex`), `all`, or `none`. Defaults to whatever is configured in `apm.yml`'s `outputs:` map. |
| `marketplace-path` | No | | Forwarded to `apm pack --marketplace-path FORMAT=PATH` (used with `pack: true`). Repeatable: one `FORMAT=PATH` override per line. Newline is the only separator -- `,` is a legal filename character. Overrides where each marketplace format file is written. |
| `json-output` | No | | Forwarded to `apm pack --json` (used with `pack: true`). When set, the action passes `--json` to the CLI and captures the JSON report to this path. Consume from downstream steps via the `pack-json` output. |
| `offline` | No | `false` | Forwarded to `apm pack --offline` (used with `pack: true`). Skips network resolution of marketplace dependency refs. Useful in hermetic CI where versions are pinned in `apm.lock.yaml`. |
| `include-prerelease` | No | `false` | Forwarded to `apm pack --include-prerelease` (used with `pack: true`). Considers pre-release version tags when resolving marketplace dependency refs. |
| `audit-report` | No | | Generate a SARIF audit report (hidden Unicode scanning). `apm install` already blocks critical findings; this adds reporting for Code Scanning and a markdown summary in `$GITHUB_STEP_SUMMARY`. Set to `true` for default path, or provide a custom path. |
| `mode` | No | | High-level orchestration mode. `release` runs the full release pipeline (gate → pack → registry publish → marketplace stage → GH Release). Mutually exclusive with `pack`, `bundle`, `bundles-file`, `setup-only`. |
| `release-tag` | No | `$GITHUB_REF_NAME` | Tag for the GitHub Release (e.g. `v1.2.3`). Defaults to `GITHUB_REF_NAME` when triggered by a tag push. |
| `release-name` | No | | Display name for the GitHub Release. Defaults to `release-tag`. |
| `release-notes` | No | | Body for the GitHub Release. Auto-generated (package table + sha256 verify hint) when empty. |
| `release-draft` | No | `false` | Create the GitHub Release as a draft. |
| `release-prerelease` | No | `auto` | `true`, `false`, or `auto`. `auto` marks the release as a prerelease when the tag contains `-` (e.g. `v1.2.3-rc.1`). |
| `release-skip-publish` | No | `false` | Run every release step except `gh release create`. Useful for dry-runs and PR builds. |
| `release-registry-publish` | No | `false` | Publish each packed package to an APM registry via `apm publish` after the matrix-pack step. Enables the `registries` experimental feature automatically. See [Registry publish](#registry-publish-optional-experimental). |
| `release-registry-name` | No | | Registry name to publish to. Required when multiple registries are configured (in `apm.yml` or global `~/.apm/config.json`); auto-selected when exactly one is configured. |
| `release-registry-package` | No | | Registry package identity in `OWNER/REPO` form (e.g. `acme/web-skills`). Required for single-plugin repos. For aggregator repos, each plugin's `apm.yml` `name` is used automatically; setting this is an error when more than one plugin is present. |
| `release-registry-dry-run` | No | `false` | Pass `--dry-run` to `apm publish`: validates the operation without uploading. |

## Outputs

| Output | Description |
|---|---|
| `success` | Whether the action succeeded (`true`/`false`) |
| `apm-version` | Resolved APM CLI version (e.g. `0.11.0`). Always set. |
| `apm-path` | Absolute path to the resolved `apm` binary. Resolved via tool-cache when the action installed APM, or via `which apm` when reusing a pre-existing CLI on `PATH`. |
| `bundle-format` | Format of the produced or restored bundle (`apm` or `plugin`). Set in pack and single-bundle restore modes. |
| `primitives-path` | Path where agent primitives were deployed (`.github`) |
| `bundle-path` | Path to the packed bundle (only set in pack mode). **Now empty** for marketplace-only projects (no `dependencies:` block in `apm.yml`) — consume `pack-json` to discover what was emitted. Pre-existing `if: steps.pack.outputs.bundle-path != ''` guards continue to work and will correctly skip downstream upload steps on marketplace-only projects. |
| `pack-json` | Path to the captured `apm pack --json` report. Set when the `json-output` input was provided. Source of truth for downstream steps that need to enumerate every artifact (bundles, marketplace files, sidecars) without globbing `build/`. |
| `audit-report-path` | Path to the generated SARIF audit report (if `audit-report` was set) |
| `bundles-restored` | Number of bundles successfully restored (multi-bundle mode only) |
| `packages` | `mode: release` — JSON array of packed artifacts. Each element: `{name, version, bundle, sha256, sha256_path}`. Always `[]` outside release mode. |
| `marketplace-drift` | `mode: release` — `true` when `apm pack --check-clean` detected uncommitted marketplace drift (the run also fails). |
| `release-url` | `mode: release` — URL of the created GitHub Release. Empty when `release-skip-publish: true`. |
| `release-tag` | `mode: release` — Resolved release tag (`release-tag` input or `GITHUB_REF_NAME`). |
| `registry-publish-results` | `mode: release` — JSON array of per-package registry publish results when `release-registry-publish: true`. Each element: `{name, version, registry}`. Always `[]` when registry publish is not enabled. |

## Third-Party Dependencies

This action bundles the following open-source packages (see `dist/licenses.txt` for full license texts):

- [@actions/core](https://github.com/actions/toolkit) — GitHub Actions toolkit (MIT)
- [@actions/exec](https://github.com/actions/toolkit) — GitHub Actions exec helpers (MIT)
- [@actions/io](https://github.com/actions/toolkit) — GitHub Actions I/O helpers (MIT)
- [js-yaml](https://github.com/nodeca/js-yaml) — YAML parser (MIT)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft
trademarks or logos is subject to and must follow
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
