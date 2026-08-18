![KHK Software logo](assets/khk-logo.png)

# Generic Compaction-Handling Skills and Other Useful Stuff

Two portable Claude Code skills that protect a long-running chat session against the memory loss caused by context compaction: **`prepare-compact`** (run before compacting) and **`resume`** (run after resuming, whether from a compaction or a fresh session picking up prior work).

They are written to be project-agnostic — no hardcoded file paths, governance frameworks, or product names — so they can be dropped into any repository's `.claude/skills/` and adapt to whatever session-tracking conventions (or lack of them) that project already has.

## Why these exist

A context compaction has the same practical effect on an agent as starting a fresh chat: everything not written to a durable artifact collapses into a condensed summary, and any live handle to a background process is gone unless it can be independently re-derived from repository state. Two distinct failure modes follow from this, confirmed by real incidents rather than treated as hypothetical:

- **On the way into compaction**: a delegated background agent can keep running with nothing left to check on it, because the session's own memory of having dispatched it doesn't survive compaction.
- **On the way out of compaction**: the auto-generated summary can drop a detail a durable record actually preserved correctly — and a resumed session that trusts the summary (or a stale tracked-status field) over the durable record itself can redo or even duplicate work that was already done.

`prepare-compact` and `resume` are the standing mitigations for these two failure modes, one on each side of the compaction boundary.

## What each skill does

### `prepare-compact/SKILL.md`

Run immediately before compacting a session that has anything genuinely at stake — delegated background agents, uncommitted work-in-progress, or a freshly agreed plan not yet executed. It walks through:

1. Confirming it's actually safe to prepare (no undocumented live agents, no unexplained dirty working tree).
2. Reading this project's own governing/session-record conventions fresh, rather than assuming they match a prior session's understanding.
3. Accounting for every piece of in-flight state: what's genuinely done, what's uncommitted and why, what plan is still pending, what standing risks/boundaries still hold, what decisions remain the user's to make.
4. Updating the project's active-session record (if it has one) so a fresh read of that file alone is enough to resume correctly.
5. Regenerating the handover artifact through its real generator, if one exists — never hand-editing a generated file.
6. Committing exactly what changed, nothing else.
7. Reporting readiness in plain terms before compaction actually proceeds.

Every step degrades gracefully when a project doesn't have some piece of this machinery (no session record, no handover generator) rather than assuming it must exist.

### `resume/SKILL.md`

The symmetric counterpart, run right after a compaction or at the start of a session continuing prior work. It is a **state-recovery pass, not a fresh investigation** — its job is to reconstruct accurate context from durable artifacts that already exist, not to re-derive conclusions they already recorded or re-audit already-verified work:

1. Re-grounding in the project's own governing framework in full — following its designated entry point/reading sequence if one exists, not substituting a single top-level rules file for the whole thing.
2. Reading the project's designated bootstrap artifact, if one exists.
3. Reading the active-session/engineering-state record start to finish — the whole file, fresh, every time.
4. Verifying live execution state directly: probing for live background agents, fetching before comparing local/remote state, reconciling any unexplained dirty path against whatever ownership mechanism the project uses for concurrent streams.
5. Cross-checking any claimed-complete work against real repository history before acting near it or dispatching further work against it — a status field can lag genuinely completed work, and lagging status is not itself evidence the work remains undone.
6. Reporting plainly what the durable records actually show, then proceeding directly into the recorded next action if the summary and the records agree, or surfacing the discrepancy plainly if they don't.

If a project maintains its own project-specific, non-generic variant of either skill (naming its own concrete governance documents, file paths, and incident history), that tailored variant is the one to actually invoke and keep current — these generic versions are the portable baseline to adapt from, not a replacement for it.

## Installing into a project

This folder — wherever you clone or check it out to — is the single git-tracked master copy of these skills. A consuming repository never edits its own copy directly; it receives a synced copy (or symlink) generated from here, and stays current by re-running the sync script after something changes in this folder.

Two tools discover skills this way, using the same walk-up convention on different directory names:

- **Claude Code** looks for `<name>/SKILL.md` under `.claude/skills/`, searched from the current working directory up to the repository root, plus a personal `~/.claude/skills/`.
- **Codex** looks for `<name>/SKILL.md` under `.agents/skills/`, searched the same way — from the current working directory up to the repository root, plus a personal `$HOME/.agents/skills/` (it also checks an admin-installed system location and its own built-in skills, neither relevant here).

### Using `sync_into_repo.py`

Run the sync script included in this folder against any target repository:

```text
python <path-to-this-folder>/sync_into_repo.py --target <repo-root>
```

This regenerates, under `<repo-root>`:

```text
<repo-root>/.claude/skills/prepare-compact/
<repo-root>/.claude/skills/resume/
<repo-root>/.agents/skills/prepare-compact/
<repo-root>/.agents/skills/resume/
```

Pass `--target` more than once to sync several repositories in one run, and add `--check` to preview what would change without writing anything — useful before a real run, or in CI to catch drift.

For each skill and each destination, the script first attempts a real OS symlink back to this folder's own copy, so an edit made here is reflected in the target immediately with nothing further to run. If the operating system or filesystem doesn't permit symlinks (for example, Windows without administrator rights or Developer Mode enabled), the script transparently falls back to a recursive copy instead. A copy is a snapshot, not a live view — it will not pick up later edits on its own, so re-run the script after changing `prepare-compact/SKILL.md` or `resume/SKILL.md` here to refresh any copy-mode destinations. The script's report states which mode (`symlink` or `copy`) was actually used for each entry, a run with nothing left to do writes nothing further, and any skill this folder used to publish but no longer does gets cleaned up from a target's `.claude/skills/` and `.agents/skills/` on the next real run, without touching anything else already present there.

`.claude/` and `.agents/` are commonly gitignored in a consuming repository, so synced copies placed there are not necessarily durable or version-controlled on their own — this folder remains the durable, version-controlled source of truth; treat everything `sync_into_repo.py` writes elsewhere as disposable, regenerable output.

A newly synced or edited skill file also typically requires a fresh Claude Code (or Codex) session before it's picked up.
