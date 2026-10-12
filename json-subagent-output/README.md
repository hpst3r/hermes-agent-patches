# Hermes JSON-schema subagent-output patch

Makes `delegate_task(output_schema=...)` a strict JSON contract:

- **Parent prompt:** a static "Structured subagent output" block, added only when
  `delegate_task` is loaded. It is static so the parent's prompt cache stays valid.
- **Child prompt:** the OUTPUT CONTRACT is the final system-prompt block, and the
  prose-summary instructions are left out when a schema is set, for both leaf and
  orchestrator children. Children without a schema get exactly the upstream prompt.
- **Validation:** the whole response must parse as JSON, so fenced or
  prose-wrapped JSON is a contract miss. The single bounded correction retry is kept.
- **Result budget:** a schema-valid result is never truncated, so the parent always
  gets parseable JSON.

The patch is a `git format-patch` mbox. `git am` keeps the commit message and
author, and `git apply` also works.

## Base revision

Rebased onto NousResearch/hermes-agent `main` at:

```text
7cc3fb8cb5  fix(kanban): respawn guards compare mixed int/ISO times by value  (2026-10-12)
```

It applies cleanly there. On the previous base `71d34bad10`, `git apply --3way`
leaves one trivial conflict in the `agent/system_prompt.py` import list, because
upstream `3bf97898ac` removed `ASYNC_HANDOFF_GUIDANCE`.

SHA-256: `1399692473c3d803d448b45dde94076f9f925a284be2b8df111f1113cbf60d4f`

Files touched:

- `agent/prompt_builder.py`, `agent/system_prompt.py`
- `tools/delegate_tool.py`, `tools/delegate_tool_progress.py`,
  `tools/delegate_tool_results.py`, `tools/delegation_output_schema.py`
- `tests/tools/test_delegate_output_schema.py`,
  `tests/tools/test_delegate_output_schema_prompt.py` (new),
  `tests/tools/test_delegate_summary_budget.py`, `tests/agent/test_system_prompt.py`
- `website/docs/user-guide/features/delegation.md`

## Verification at the base revision

- 13 of the patch's tests fail on unpatched upstream and pass with the patch.
- 66 delegation and prompt test files, including the prompt-size tests:
  785 passed, 3 failed, 1 skipped. Those 3 (`test_delegate_control_actions` ×2 and
  `test_delegate_kanban_isolation` ×1) fail the same way on unpatched upstream.
- `ruff check` is clean on the changed Python files.

## Recommended: carry it as a commit on a local branch

```bash
cd ~/.hermes/hermes-agent
git switch -c local/patches            # once
git am /path/to/0001-enforce-json-schema-subagent-output.patch
hermes config set updates.parked_branch_strategy update_in_place
```

With `update_in_place`, `hermes update` merges `origin/main` into
`local/patches` instead of switching away from it. If that merge conflicts, the
update stops without changing anything; resolve it with
`git merge origin/main` and re-run the update.

## Alternative: apply as an uncommitted patch

```bash
cd ~/.hermes/hermes-agent
git apply --check /path/to/0001-enforce-json-schema-subagent-output.patch
git apply --3way  /path/to/0001-enforce-json-schema-subagent-output.patch
```

To check whether the exact patch is present before an update:
`git apply --reverse --check <patch>`. Exit status 0 means it is present.

## Testing

Upstream now requires Python 3.14 and its PM environment manager:

```bash
python3.14 -m pm.build_env --source . --out ~/hermes-testenv --group dev --group test
HERMES_PYTHON=~/hermes-testenv/bin/python scripts/run_tests.sh \
  tests/tools/test_delegate_output_schema.py tests/tools/test_delegate_output_schema_prompt.py \
  tests/tools/test_delegate_summary_budget.py tests/agent/test_system_prompt.py -q
```

Run tests from a checkout whose `.git` is not under `~/.hermes`. Otherwise the
suite's home-I/O guard reports dozens of false failures.

After applying, restart the gateway and Desktop and start a new conversation.
Running sessions keep their cached system prompt and the modules they already loaded.

## History

The previous version of this file targeted v0.20.5 (`534e12eb23`), before
upstream split `delegate_tool.py` into sibling modules. It no longer applies;
see this repository's Git history.
