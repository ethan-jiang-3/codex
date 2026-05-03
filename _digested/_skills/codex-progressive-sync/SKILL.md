---
name: codex-progressive-sync
description: Use when continuing Codex upstream sync work for `_digested` docs, release triage, or progressive owner-doc alignment. Prefer repo-native truth sources: real source/tests, owner-doc front matter (`owns`, `update_when`, `out_of_scope`, `sync_status`), and dynamically discovered `_digested/_meta/git-tracking/upstream-sync/*.md` logs for the current round.
---

# Codex Progressive Sync

Use this skill when the task is to continue the repo's staged upstream-sync work for `_digested` documentation.

## Scope

This skill is for:

- aligning `_digested` owner docs to real source/tests
- continuing an existing upstream-sync chain already recorded under `_digested/_meta/git-tracking/upstream-sync/`
- checking release-note clues when they help locate a slice
- appending factual progress to the currently active upstream-sync log for this round

This skill is not for:

- rewriting unrelated docs
- building coverage maps into owner docs
- broad UI polish passes with no owner-doc impact
- "cleaning up" unrelated dirty worktree changes

## Split of responsibility

This skill is intentionally split into two layers:

1. Stable layer: the generic upstream-sync workflow, truth order, front matter rules, and validation contract.
2. Round-specific layer: the active sync log for the current round under `_digested/_meta/git-tracking/upstream-sync/*.md`.

Round-specific progress must live in the active sync log, not in this skill. The skill stays generic; the log rotates round by round.

## Canonical truth order

Prefer sources in this order:

1. Real source and real tests for the slice (Rust code in `codex-rs/`, TypeScript in `codex-cli/`).
2. The target owner doc itself:
   - front matter `owns` tells you whether this page really owns the fact
   - `update_when` helps confirm the page should move for this change
   - `out_of_scope` tells you what not to mix in
   - `sync_status` is page-level status only, not detailed evidence
3. `_digested/README.md` for `_digested` maintenance rules and `sync_status` contract.
4. `_digested/_meta/git-tracking/git-branch-and-upstream-tracking.md` for the current baseline ref (`origin/main` in this clone unless that file says otherwise).
5. The active upstream-sync log under `_digested/_meta/git-tracking/upstream-sync/*.md` for this round.
6. Older upstream-sync logs in the same directory for history, prior slice patterns, and baseline continuity.
7. Release notes or other ad hoc side materials only as supporting clues after the repo-native sources above.

Do not treat side notes as owner truth. Do not treat `ethan` diff status as sync truth.

## Active round discovery

Before choosing the next slice, determine the active round dynamically.

### 1. Find the candidate sync logs

Look in:

- `_digested/_meta/git-tracking/upstream-sync/`

Ignore non-round files:

- `README.md`
- `TEMPLATE.md`

Candidate round logs are the dated files that match the directory naming contract, for example:

- `YYYY-MM-DD-upstream-sync.md`
- `YYYY-MM-DD-upstream-sync-2.md`

### 2. Pick the active sync log

Prefer this order:

1. A dated sync log whose front matter `status` is `in_progress`.
2. If multiple logs are `in_progress`, use the newest dated file as the default active log and explicitly note the ambiguity.
3. If no log is `in_progress` but the user is clearly continuing the latest recorded round, use the newest dated sync log and say that you inferred continuity from the latest log.
4. If the user is starting a fresh sync round, create a new dated log from `TEMPLATE.md` and make that the active log.

### 3. Derive the next slice from the active log

Once the active log is identified:

- read its latest completed phase
- read its latest "未覆盖风险 / 下一步" (uncovered risks / next steps)
- read its front matter status and purpose
- keep new append-only progress in that same active log unless you are intentionally starting a new round

### 4. Derive release-note clues dynamically

If release notes are relevant for the round, discover them from the repo state rather than from a hardcoded scratch file:

- look for root-level release notes or changelogs
- treat them as direction-finding indexes only
- return to source/tests before writing owner docs

## Required workflow

1. Read the real source files and tests first (Rust: `codex-rs/**/*.rs`, TS: `codex-cli/**/*.ts`).
2. Re-read the relevant `_digested` owner doc before editing it, including its front matter.
3. Use `owns`, `update_when`, and `out_of_scope` to confirm you are editing the right page.
4. Keep the slice narrow. Prefer one runtime surface or one crate group at a time.
5. Update only the direct owner doc(s) for the facts you verified.
6. If the page was formally reviewed, update or add `sync_status` using the current baseline contract.
7. Append a new phase entry to the active upstream-sync log for the current round.
8. Run `git diff --check -- <touched files>`.
9. Run relevant tests when possible (e.g., `cargo test -p codex-core`).
10. If tests are blocked by environment, record that honestly instead of claiming success.

## Working rules

- `_digested/README.md` is the contract for how owner docs should read and what `sync_status` means.
- Owner-doc body text should reflect verified runtime facts from source/tests, not guesses from release notes alone.
- Do not mix "which files were reviewed / not reviewed" inventory work into owner-doc body text.
- If you need a coverage map or reviewed/unreviewed inventory, keep it in `_tmp_tracking/_digested_isuues/` or a dedicated side document, not in owner-doc body text.
- Real evidence may live in Rust (`codex-rs/`), TypeScript (`codex-cli/`), YAML configs, or Bazel BUILD files. Do not assume sync work is Rust-only.
- Respect a dirty worktree. Never revert unrelated user changes.
- Prefer `rg` for search.
- For edits, use targeted patches and keep prose compact.

## Front matter rules

When touching a target owner doc, use its front matter deliberately:

- `owns`: primary signal for whether this doc is the right owner.
- `update_when`: secondary signal that a newly observed behavior belongs here.
- `out_of_scope`: stop sign against scope creep.
- `purpose` and `audience`: keep the rewrite at the page's intended abstraction level.
- `updated`: bump only when you actually change the page.
- `sync_status`: use only for formally reviewed sync state.

`sync_status` contract:

```yaml
sync_status:
  source: "upstream-sync"
  baseline_ref: "origin/main"
  baseline_before: "<short hash>"
  baseline_after: "<short hash>"
  state: "updated"
```

Rules:

- `baseline_ref` should come from `_digested/_meta/git-tracking/git-branch-and-upstream-tracking.md`, not from guesswork.
- Keep `baseline_ref`, `baseline_before`, and `baseline_after` internally consistent.
- Allowed states are `updated`, `reviewed_no_change`, `unknown`.
- Missing `sync_status` does not automatically mean `unknown`.
- Only write `reviewed_no_change` when the sync log explicitly records a formal review with no owner-text change.
- Do not infer review state from "git diff is empty" or "this page was not edited today".

## Typical loop

1. Identify the active sync log for the current round.
2. Identify the next smallest unresolved slice from that log, especially the latest "未覆盖风险 / 下一步".
3. Gather evidence from source and tests.
4. Re-open the candidate owner doc(s) and confirm ownership from front matter.
5. Decide the minimum owner-doc set that truly owns the changed behavior.
6. Patch owner doc(s).
7. Append tracking with:
   - reviewed source paths
   - reviewed tests
   - factual behavior changes written back
   - touched docs
   - residual risks / next step

## Minimum discovery checklist

At the start of each use of this skill, confirm:

- which dated sync log is active for this round
- which baseline ref / before / after hashes the round is aligned to
- which owner doc really owns the slice you are about to change
- which tests best verify that slice
- whether this is a continued round append or the start of a new round log

## Validation notes

- For Rust tests: prefer `cargo test -p <crate-name>` for the relevant crate.
- For Bazel: use `bazel test //path/to:target` when appropriate.
- If no build tooling is set up, report the failure exactly.
- `git diff --check` should pass for the files you touched.

## Codex-specific crate → owner doc mapping

When determining which owner doc owns a source change, use this quick reference:

| Source path | Primary owner doc |
|---|---|
| `codex-rs/core/**` | `03-agent-core/` |
| `codex-rs/tools/**`, `codex-rs/exec/**`, `codex-rs/sandboxing/**` | `04-tools-execution-and-sandboxing/` |
| `codex-rs/skills/**`, `codex-rs/plugin/**`, `codex-rs/core-skills/**`, `codex-rs/core-plugins/**` | `05-skills-and-plugins/` |
| `codex-rs/model-provider/**`, `codex-rs/models-manager/**`, `codex-rs/lmstudio/**`, `codex-rs/ollama/**` | `06-models-and-providers/` |
| `codex-rs/state/**`, `codex-rs/memories/**`, `codex-rs/thread-store/**` | `07-state-memory-and-sessions/` |
| `codex-rs/tui/**`, `codex-rs/cli/**`, `codex-rs/app-server/**`, `codex-rs/codex-mcp/**`, `codex-cli/**` | `08-tui-cli-and-integration/` |
| `codex-rs/secrets/**`, `codex-rs/execpolicy/**`, `codex-rs/keyring-store/**` | `04-tools-execution-and-sandboxing/` (security aspects) |
| `codex-rs/config/**`, `codex-rs/login/**` | `02-system-architecture/` or `01-beginner/` |

For the full mapping, refer to the latest owner map in `_tmp_tracking/_digested_isuues/`.
