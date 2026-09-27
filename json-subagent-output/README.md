# Hermes JSON-schema subagent-output patch

This bundle preserves a local Hermes Agent customization that makes delegated
`output_schema` responses strict JSON contracts.

It adds parent-system-prompt guidance, appends the exact schema contract last in
the child system prompt, rejects Markdown/prose wrappers, retains the existing
one-retry correction path, removes conflicting prose-summary instructions from
schema-bound children, and prevents validated JSON from being mangled by the
normal summary-truncation budget.

## Base revision

Generated from NousResearch/hermes-agent commit:

```text
534e12eb23cb58c3d4b3ddc91ade60d8f989cd5c
```

The patch touches only:

- `agent/prompt_builder.py`
- `agent/system_prompt.py`
- `tools/delegate_tool.py`
- `tools/delegation_output_schema.py`
- `tests/tools/test_delegate_output_schema.py`
- `tests/tools/test_delegate_summary_budget.py`
- `tests/run_agent/test_run_agent.py`

It intentionally excludes unrelated local changes in the email adapter and its
tests.

## Apply after installing or updating Hermes

```bash
cd ~/.hermes/hermes-agent
git apply --check ~/hermes-agent-patches/json-subagent-output/0001-enforce-json-schema-subagent-output.patch
git apply --3way ~/hermes-agent-patches/json-subagent-output/0001-enforce-json-schema-subagent-output.patch
pytest -q tests/tools/test_delegate*.py
pytest -q tests/run_agent/test_run_agent.py -k 'BuildSystemPrompt or ToolUseEnforcement or ExecutionGuidance or TaskCompletionGuidance'
```

Restart Hermes and start a new conversation afterward; live sessions retain
their already-cached system prompts and loaded Python modules.

## Before a Hermes update

Keep this bundle in a separate GitHub repository. Before updating, check whether
the exact patch is currently present:

```bash
cd ~/.hermes/hermes-agent
git apply --reverse --check ~/hermes-agent-patches/json-subagent-output/0001-enforce-json-schema-subagent-output.patch
```

Exit status 0 means it is present and can be removed with:

```bash
git apply --reverse ~/hermes-agent-patches/json-subagent-output/0001-enforce-json-schema-subagent-output.patch
```

Do not discard unrelated local changes. After `hermes update`, apply and test the
patch again. If `git apply --check` fails, upstream changed or incorporated one
of the touched sections; review/rebase rather than forcing it.

## Suggested private GitHub repository

```bash
cd ~/hermes-agent-patches
git init
git add json-subagent-output
git commit -m "Add Hermes strict JSON subagent-output patch"
gh repo create hermes-agent-patches --private --source . --push
```

If this directory later belongs to another repository, commit the updated patch
and README normally instead of re-running `git init`.
