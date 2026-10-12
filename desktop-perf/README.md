# Hermes Desktop performance picks (upstream open PRs)

A local series of six **open upstream PRs** carried on `local/patches`. Together they
fix slow transcript scrolling in long chats, a sluggish sidebar when there are many
sessions, and wasted background re-renders. None of these are my own changes: each
patch is the PR author's diff, picked unchanged as one commit, with the upstream PR
and author named in the subject line.

| # | Patch | Upstream PR | Author | PR head when picked | What it fixes |
|---|---|---|---|---|---|
| 1 | `0001-…sticky-prompt-clip…` | [#133314](https://github.com/NousResearch/hermes-agent/pull/133314) | UE2020 | `58fbd5ecb6` | Registers `--sticky-prompt-clip` as a non-inherited `@property`. Before this, every scroll frame restyled the whole subtree of each clipped turn. This is the main fix for long-chat scrolling. [#120598](https://github.com/NousResearch/hermes-agent/pull/120598) is the same CSS without tests. |
| 2 | `0002-…virtual-session-list…` | [#134041](https://github.com/NousResearch/hermes-agent/pull/134041) | OutThisLife | `03ae76e0b5` | Gives the sidebar `VirtualSessionList` stable TanStack callbacks and memoized rows. Before this, every store tick rebuilt the size/position cache for every row and re-rendered every visible row. This is the main sidebar fix. [#133953](https://github.com/NousResearch/hermes-agent/pull/133953) is a near-duplicate. |
| 3 | `0003-…idle-countdown-clocks…` | [#122446](https://github.com/NousResearch/hermes-agent/pull/122446) | whyyagswhy | `67bed908f5` | Idle countdown clocks share one 1 s ticker that runs only while they are visible. |
| 4 | `0004-…session-tiles…` | [#106604](https://github.com/NousResearch/hermes-agent/pull/106604) | kshitijk4poor | `988b12a102` | Session tiles no longer re-render their chat shell on unrelated `sessions.changed` ticks. |
| 5 | `0005-…pet-cron-polls…` | [#128216](https://github.com/NousResearch/hermes-agent/pull/128216) | kshitijk4poor | `7e932ed4d4` | Pet and cron polls that return unchanged data no longer republish their atoms, so they cause no re-render. |
| 6 | `0006-…sidebar-wheel-scrolling…` | [#88195](https://github.com/NousResearch/hermes-agent/pull/88195) | royalaid | `259eb3c593` | Makes the sidebar section stack the single scroll owner, so the mouse wheel no longer gets stuck over headers or at list edges. This is a correctness fix, not a speed fix. |

Apply them **in this order**: 6 depends on the sidebar files as left by 2.

Deliberately **not** included: [#98644](https://github.com/NousResearch/hermes-agent/pull/98644)
(caps animated sidebar rings). It conflicts with `main` and makes no measured claim.

## Base revisions and verification

- Picked onto `local/patches` at `8f5ac4207d`, which is upstream `71d34bad10` plus the
  `json-subagent-output` patch.
- `git am --3way` applies all six cleanly onto upstream `main` at `7cc3fb8cb5` (2026-10-12).
- Desktop checks on the picked tree: `npm run typecheck` clean, eslint clean on the
  changed files, and the vitest `ui` project for the touched directories. The new
  tests from #133314 and #134041 are included; with patches 1–2 alone, the sidebar and
  thread tests gave 47 files / 362 tests passing.

SHA-256:

```text
abeb4e62e89f103a1cfeb280290b195e240c7fb31ef85fd8bbffb6c1722fc31c  0001-perf-desktop-register-sticky-prompt-clip-as-non-inhe.patch
b05f53b15a981864db5500ac924e5de313a76f5e81bcf2ae800a9c041ef2dd63  0002-fix-desktop-stable-virtual-session-list-callbacks-me.patch
d7269fb47a897aa4327773af1ada0782ca0fb55a27163e453ed0f50f8a6aa2fb  0003-fix-desktop-gate-idle-countdown-clocks-on-one-shared.patch
78cb414fd5dca0e06e9c308b58dfbccd87a20b10f78571b0707b8b018f9cb8ee  0004-fix-desktop-session-tiles-stop-re-rendering-their-ch.patch
d4efebaa4708082d594bdc0772bb5898f6ac7c64e22c39fcaf0a60ede9f425c8  0005-perf-desktop-unchanged-pet-cron-polls-stop-republish.patch
63b6bdb2b5680ed2a1627215a1c1c1e7db2e6a6d7b13daf8d7aa43a16153699d  0006-fix-desktop-restore-sidebar-wheel-scrolling-upstream.patch
```

## Applying

The install runs from `local/patches` with
`updates.parked_branch_strategy=update_in_place`, so `hermes update` merges
`origin/main` into the branch and these commits are kept. Reapply them only if the
branch is rebuilt from scratch:

```bash
cd ~/.hermes/hermes-agent
git switch local/patches
git am --3way /path/to/hermes-agent-patches/desktop-perf/*.patch
hermes desktop --build-only    # stage-and-swap rebuild of apps/desktop/release/linux-unpacked
# then quit and relaunch Hermes Desktop
```

`hermes desktop --build-only` builds into a staging directory and swaps it in only
after verification, so it is safe to run while the app is open. The running app
keeps the old build until it restarts.

## Instructions for the next agent (after an upstream update)

For each PR in the table:

1. **Has it merged upstream?**
   `gh pr view <N> -R NousResearch/hermes-agent --json state,mergedAt,mergeCommit`.
   If `MERGED` and `git merge-base --is-ancestor <mergeCommit> origin/main`, drop the
   local commit. After the merge, Git usually treats the identical diff as a no-op. If it
   conflicts, keep upstream's version.
2. **Closed without merging?** Read the closing comment. If the fix landed under another
   PR (a "salvage" or "duplicate"), find it with `git log origin/main --grep` and
   drop the local commit. Otherwise keep carrying the patch file from this directory.
3. **Still open but force-pushed** (the head is no longer the SHA above)? Compare
   `gh pr diff <N>` with the patch file. Take the newer diff only if it still applies
   and the PR's tests pass, then update the patch file, the head SHA and the checksum here.
4. **Conflict on `git am --3way`:** resolve it in favor of upstream's structure,
   keeping the PR's intent. For the sidebar (patches 2 and 6), check that
   `virtual-session-list.tsx` still uses `useCallback`-stable
   `estimateSize`/`getItemKey`/`getScrollElement` and memoized row shells. For the
   transcript (patch 1), check that `@property --sticky-prompt-clip { inherits: false }`
   is still present in `apps/desktop/src/styles.css`.
5. **Verify** from `apps/desktop`: `npm run typecheck`,
   `npx vitest run --project ui src/app/chat/sidebar src/components/assistant-ui/thread src/hooks src/store src/components/pet --maxWorkers=4`
   (use bounded workers; this machine is shared). Then `hermes desktop --build-only`.
6. Regenerate the patches from the carried commits with `git format-patch`, update
   the table, the base revisions and the checksums, and commit them to this repository.
