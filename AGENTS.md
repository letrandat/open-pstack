# open-pstack (letrandat fork)

This repository is letrandat's fork of `ericlitman/open-pstack`. It lands Cursor pstack syncs ahead of ericlitman and otherwise follows ericlitman `main`. Track fork work in this repository's GitHub Issues. Port defects that also exist upstream belong in `ericlitman/open-pstack` Issues.

Read `UPSTREAM.md` before changing upstream-derived content.

Cursor's `cursor/plugins/pstack` tree is the content upstream. Keep one shared skill tree for Claude Code and Codex; adapt harness primitives at the existing mapping boundaries instead of forking skills or adding compatibility layers. The parent harness resolves provider routing once. Children do not detect or reroute themselves.

Do not add an implicit runtime timeout or a weaker-model fallback.

## Remotes

- `origin` is `letrandat/open-pstack`.
- `upstream-port` is `ericlitman/open-pstack`.
- `cursor` is `cursor/plugins`; pstack lives under `pstack/`.

## Fork contract

The fork patch is `git diff "$port_base" origin/main`, where `port_base` is the `Port base` commit recorded in `UPSTREAM.md`:

```sh
port_base=$(sed -n 's/^| Port base | `\([0-9a-f]*\)` |$/\1/p' UPSTREAM.md)
```

That diff is the whole fork contract. A sync that rebuilds on a newer ericlitman `main` reapplies all of it and resolves each conflict. Where ericlitman's newer tree already carries a Cursor change, keep ericlitman's version.

These parts of the patch are easy to lose in a rebuild:

- `scripts/upstream-merge.py` copies Cursor's added skills verbatim, flags included. Recheck the frontmatter of every added skill against the `UPSTREAM.md` exclusions.
- Every added skill needs an entry in `.claude/skills/verify-open-pstack/features/registry.json`. The CI static test fails on an unmapped skill.
- Test and catalog updates that follow the skill text, such as `tests/skill-collision-repro.sh` and `docs/reference.md`.
- Release metadata: the version in the four places `tests/skill-collision-repro.sh` checks, and the `Port base` row in `UPSTREAM.md`.
- The fork's own process: this `AGENTS.md`, `.github/pull_request_template.md`, and the fork notes in `README.md`, `CHANGES.md`, and `UPSTREAM.md`.

## Versions

Fork versions are `X.Y.Z-ld.N`. Claude Code reinstalls on any change to the version string, so every landing needs a string never used before.

To pick the next version, first run `git fetch origin main --tags` and `git fetch upstream-port main`. Then take the highest semver among the version on `origin/main`, the version on `upstream-port/main`, and every `v*` tag with its `v` removed. Compare the `ld` counter as a number. If the highest is a plain release `X.Y.Z`, the next version is `X.Y.(Z+1)-ld.1`. If it is `X.Y.Z-ld.N`, the next version is `X.Y.Z-ld.(N+1)`. The version installed on any machine plays no part. A version that already has a `v*` tag is spent.

## Merge gate

ericlitman's release process does not run here. The `verify-open-pstack` skill reports to ericlitman's repository, and Mergify, Unfret, and `live-gate` are not installed. Keep the `verify-open-pstack` directory, because the CI static test runs its coverage test. A pull request merges after all three:

1. The CI `verify` job passes on the head.
2. A reviewer on a different model from the writer approves the diff.
3. A smoke test of the exact head passes. Post the head SHA, commands, and observed output as a pull request comment. Each new head needs its own run.

Run the smoke test from a clean checkout of the head with the installed pstack disabled for that one session. Set `PR` to the pull request number and `PROMPT` to a prompt that exercises the changed behavior. For a user-only skill (one with `disable-model-invocation: true`), start the prompt with its slash command, such as `/pstack:correct`. Any failed step stops the run:

```sh
(
  set -euo pipefail
  sha=$(gh pr view "${PR:?}" --repo letrandat/open-pstack --json headRefOid --jq .headRefOid)
  dir=$(mktemp -d)/open-pstack
  git clone -q https://github.com/letrandat/open-pstack.git "$dir"
  git -C "$dir" checkout -q --detach "$sha"
  test "$(git -C "$dir" rev-parse HEAD)" = "$sha"
  rc=0; git -C "$dir" grep -nE '^(<<<<<<<|=======|>>>>>>>|\|\|\|\|\|\|\|)( |$)' "$sha" -- || rc=$?
  test "$rc" -eq 1
  cd "$dir"
  claude --plugin-dir "$dir/plugins/pstack" \
    --settings '{"enabledPlugins":{"pstack@open-pstack":false}}' \
    --permission-mode "${MODE:-plan}" --output-format stream-json --verbose \
    -p "${PROMPT:?}" > "$OLDPWD/smoke-$sha.jsonl"
)
```

The stream-json trace is the evidence. A pass shows the expected `Skill` tool call or slash-command expansion in the trace, not only the model's text. Plan mode keeps the run read-only, which is enough to prove that a skill loads and routes. When the changed behavior needs edits or commands, set `MODE=acceptEdits`; the run then works inside the throwaway clone.

## Release

After the merge:

```sh
git fetch origin
sha=$(git rev-parse origin/main)
git show "$sha:plugins/pstack/.claude-plugin/plugin.json" | grep '"version"'
git tag "v<version>" "$sha" && git push origin "v<version>"
```

The tag must match the version that `plugin.json` shows. Keep `main` fast-forward only. Marketplace clones pull it, and installs pin to `v*` tags. The rollback target is the previous `v*` tag, or `ericlitman-1.5.0-6c44500` for the build before the fork:

```sh
claude plugin marketplace remove open-pstack
claude plugin marketplace add 'letrandat/open-pstack#<tag>'
claude plugin install pstack@open-pstack --scope user
```

## Sync

Syncs start by hand until a tested `scripts/fork-sync.sh` exists. Branch from `origin/main` when `upstream-port/main` is an ancestor of the `Port base` commit. Otherwise ericlitman has moved: rebuild the branch from `upstream-port/main`, reapply the fork contract, and land the result as a merge commit whose parents are `origin/main` and `upstream-port/main`.

Run the port's tooling against Cursor:

```sh
python3 scripts/upstream-audit.py --port HEAD --upstream cursor/main > "$TMPDIR/audit.json"
python3 scripts/upstream-merge.py "$TMPDIR/audit.json"
```

Read each upstream commit's intent, then resolve each conflict hunk with this table. Split a hunk that mixes kinds and decide each part.

| Part of a hunk | Decision |
| --- | --- |
| A harness translation listed in `CHANGES.md`: AskUserQuestion, `run`/`verify`, provider dispatch, fork-aware `gh api`, the Claude `/goal` substitute | Keep the port's wording |
| An exclusion or adaptation recorded in `UPSTREAM.md` or `CHANGES.md`: stuck only on affirmative failure evidence, draft until evidence, no `is_background`, no Cursor cloud agents, the autopilot lease and expected-head merge steps | Keep the port's text |
| A new upstream policy | Take upstream's policy, written in the port's Claude Code and Codex terms |
| A hunk no row decides | Stop and ask the operator |

Before opening the pull request, confirm `git grep -nE '^(<<<<<<<|=======|>>>>>>>|\|\|\|\|\|\|\|)( |$)'` prints nothing, then run the CI commands in `.github/workflows/ci.yml`.
