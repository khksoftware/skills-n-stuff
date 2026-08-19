---
name: prepare-compact
description: Prepare this chat session for a context compaction (/compact) so no in-flight work, agreed plan, or supervision state is lost. Use immediately before compacting a long-running session, especially one that has delegated background agents, uncommitted work-in-progress, or a freshly agreed multi-step plan not yet executed.
---

# Prepare for compaction

A context compaction has the same practical effect on this agent as starting a fresh chat: everything not written to a durable artifact is reduced to a condensed summary, and any live handle to a background process is gone unless it is independently re-derivable from repository state. Losing track of a delegated background agent this way is a known, generally-observed failure mode of compaction, not a hypothetical edge case — a task can keep running with no one checking on it until the user has to ask directly. Treat this skill as the standing mitigation for that risk, not a one-off cleanup.

**This is a bookkeeping pass, not an audit.** Its job is to transcribe state the session already established — its own commits, the reports it already received from anything it delegated — into durable files, plus a couple of cheap freshness checks (live agents, dirty tree). It is not a review of the work itself: don't re-run test suites, re-verify commit contents against claims, or re-derive conclusions the session already reached. That kind of verification belongs in a dedicated review task, not here. If you find yourself dispatching an agent to re-check the session's own prior work in the course of running this skill, stop — that is scope creep on what this skill is for.

Do not run this skill's steps from memory of what it says below once invoked in a future session — re-read this file fresh each time, and re-derive the governing requirements from this project's own actual current source files rather than assuming they match what's written here.

## 1. Confirm it is actually safe to prepare now

- Check whether any background agents/tasks are currently running (probe the task registry or equivalent mechanism this environment provides for listing in-flight delegated work). Distinguish a roster you actually verified as empty from one you simply couldn't check — those are different states, and treating an unchecked roster as equivalent to a confirmed-empty one is itself a failure mode. If any agents are running:
  - Either wait for them to finish before proceeding, or
  - If the user wants to proceed anyway, explicitly record their exact identity, assigned task, and expected next checkpoint in the handover (step 4) — do not compact with a live, undocumented agent in flight.
- Run a full repository-wide `git status --short` (not scoped to files you remember touching) to get a complete, current accounting of dirty state. Compare it against what you expect; investigate and explain anything unexpected before proceeding — git status should always be checked before any operation that could discard uncommitted work.

## 2. Read the governing requirements fresh, don't assume them

- Look for this project's own documented protocol for session records and handovers — a governing doc that specifies what a session-state file or handover artifact must contain and when it must be updated, if this project has formalized one. If it exists, read it fresh rather than assuming it matches a prior session's understanding of it. This skill's remaining steps operationalize that kind of protocol for a same-chat compaction, not only whatever cross-chat "switching sessions" trigger it may name explicitly — the practical memory-loss risk is the same in both cases, so the same rigor applies. If no such protocol doc exists in this project, proceed with steps 3 onward using ordinary judgment about what state matters.
- Find this project's active-session or engineering-state record — a living document tracking current work-in-progress, if this project keeps one — and its handover file(s), if any, and read what they currently claim before rewriting either. A stale prior state is itself information (it tells you what's actually changed since it was last updated). If this project has no such records at all, note that explicitly and skip ahead to producing one from scratch in step 4, or to skipping steps 4-5 entirely if a durable record genuinely isn't this project's convention.

## 3. Account for every piece of in-flight state

Before writing anything, make sure you can answer each of these from actual current evidence (re-check live files/commits, don't recall from earlier in the conversation):

- What has genuinely completed and been committed/pushed this session (condensed to pointers — commit hashes, file paths — not exhaustive prose)?
- Is there any uncommitted work sitting in the working tree? For each dirty file or group of files: why is it there, is it safe to commit, safe to discard, or must it be left exactly as-is pending a specific next action? Never silently commit or discard working-tree state you can't explain.
- Is there an explicit plan or decision the user just gave that has not yet been executed? Capture its exact intent and ordering, not a paraphrase that loses the sequencing or the reasoning behind it.
- Are there standing risks, stop-on conditions, or authority boundaries already recorded that remain valid? Carry them forward — do not drop a still-true boundary just because it wasn't touched this session.
- Are there decisions still pending the user's (not this agent's) judgment? Name them explicitly as open rather than letting them disappear into a summary.

### Sweep the conversation itself, not just its outputs

The bullets above are answered from what the session already produced — its commits, its dirty files, the plan just given. This pass is different, and it is worth running as an explicit step rather than trusting that you would have remembered: **re-read the conversation from its start and look specifically for material that only ever existed in the chat.**

What escapes a pass built around outputs is smaller and less visible than work product: an authorization the user gave in passing, a decision the agent itself announced out loud and never wrote down, a correction to a claim some durable record still asserts, a finding a delegated agent reported in prose that nobody persisted to a file, or an explicit limit ("this doesn't cover X," "this was only checked against Y") that exists precisely to stop a later reader from overclaiming. None of these leaves a dirty file or a running process behind, which is exactly why a pass that only looks at outputs steps over them.

Look for at least:

- **Authorizations the user granted in conversation.** These are the most costly to lose — losing one doesn't just forget work, it sends a resumed session back to ask for permission it already has, which reads to the user as not having listened.
- **Decisions or rulings the agent itself announced.** A routing call, a scope ruling, an accepted recommendation. An announced decision feels settled precisely because it was said out loud, and that feeling is the failure mode.
- **Corrections that falsify something a durable record still asserts.** If a number, a state, or a present-tense claim was corrected in conversation, some file probably still carries the old version.
- **Reports from anything delegated.** A subordinate agent's findings reach the orchestrator and nowhere else, unless the agent itself wrote them to a file.
- **Explicit non-claims and stated limits.** These are what stop a later closure from overclaiming, and they are the first thing a summary drops.

**Confirm each recovered item against what is actually committed, not against memory of having written it.** An item drafted into a file but never committed is in the same position as one never written at all.

## 4. Update the active-session record — by replacing in place, never by prepending

If this project keeps an active-session or engineering-state record, update it now to reflect everything gathered above.

**Replace outdated content in place; do not prepend a new dated or session-end block in front of what's already there.** This shape has a measured failure history: an instruction to "rewrite" a section still tends to produce a new block stacked in front of the old one, because prepending feels safer than deciding what to overwrite, and once that pattern starts, periodic pruning barely dents it — a record pruned hard can climb back to nearly its pre-prune size within days, purely from new blocks accumulating in front of old ones nobody ever removes. For each piece of content, decide whether it is still in force (in which case it stays, edited in place) or no longer in force (in which case it belongs in whatever this project uses as an archive for retired record content, not in a second "history" section inside the live record itself — a tier nothing ever reads is functionally the same as deleting the content, minus the honesty of calling it deleted).

**If this project's active-session record follows a schema — specific fields, headings, or anchors this skill or the project's own protocol expects you to update — and one of them is missing, renamed, or restructured since you last touched it, that is a finding to report, not an instruction to satisfy anyway.** An agent that can't find the field it's told to update and simply skips the edit is at least visibly incomplete; something downstream can notice the edit never landed. An agent that instead *invents* the missing structure so the instruction can be followed literally — recreating a heading that was deliberately retired, or fabricating a section that merely looks plausible — produces a result indistinguishable from success, and that is worse: it silently reintroduces exactly the shape the project had moved away from. When the record's actual schema doesn't match what you expected, stop and say so rather than papering over the gap.

**Before moving or archiving any section that claims to be a complete, deduplicated summary of something** — a standing-boundaries list, an open-risks list — **re-derive that summary first.** Content matching what the section claims to hold can accumulate elsewhere in the interim, in a different section or in conversation, and moving the section as found silently destroys anything that never made it into the summary. Re-derive the union, then move.

**If any part of the record you're about to write would have to describe this run's own publication** — the exact commit hash this update ships in, or a flat claim like "nothing pushed yet" — recognize that no ordering fixes this: such a field is necessarily written before the commit that would make it true, so committing it makes it stale the instant it lands, and writing it after the commit leaves it uncommitted. Don't resolve this by asserting a verdict. State what's already fixed — prior commits this session produced, what's still open or blocked — describe this run's own publication in the forward tense as a remaining act rather than a completed fact, and point the reader at how to resolve it themselves (their own status check, their own diff against the remote) rather than asserting an answer you cannot actually have at the moment of writing. A claim made without access to its own subject is false by construction even on the runs where it happens to land right.

Prefer pointers to authoritative files (a work order, a ledger, a specific commit) over restating their content. State the exact next action to take on resumption plainly enough that a fresh read of only this file (no conversation memory) is sufficient to resume correctly.

## 5. Regenerate the handover artifact through its real generator, if one exists

If this project has a dedicated handover-generation script or tool, never hand-edit the handover file(s) directly — use the generator, following this rough shape:

1. Read the current handover artifact's fields describing the latest completed and currently-executing work to find the correct arguments to pass the generator (reuse them unchanged unless the underlying work identities have actually changed this session). Check the generator's help output or source for its actual argument names rather than guessing.
2. Run the generator with those arguments.
3. If the generator supports a validate-only or dry-run mode, run it and confirm the result is valid before proceeding. If it fails, fix the actual cause — a stale reference, a contradictory pair of fields, a missing input — rather than working around the validator.

If no such generator exists in this project, hand-editing the handover/session file directly is the only option — in that case, be extra careful to preserve its existing schema, format, and structure exactly, changing only the content that has genuinely changed.

The same rule can apply one level up, to the active-session record itself: if this project maintains it as a generated render of a separate canonical source, rather than as freeform prose edited directly, treat the canonical source as what step 4 above actually edits, and regenerate the render from it — never hand-edit the render. A rendered copy that was hand-edited once is easy to overwrite silently on the next regeneration, and drifts unnoticed in the meantime.

## 6. Commit exactly what changed, nothing else

`git status --short` again and confirm only the session/handover files (and anything else this skill's run genuinely produced, e.g. a new backlog entry the user asked to log first) are staged — never a broad `git add`. Commit with an explicit pathspec and push. Leave any other in-flight/blocked work (per step 3) exactly as it was found.

## 7. Report readiness plainly

Tell the user, in plain terms: what was saved and where, what (if anything) is still uncommitted and why that's correct, and the exact next action recorded for resumption.

If this project designates certain content as requiring separate authorization beyond ordinary task approval — a shared framework, governing rules, foundational configuration, however this project draws that boundary — state explicitly whether anything under it changed this session and whether that was authorized, even when the answer is plainly no. This is easy to omit precisely because the answer usually is no, and an omitted disclosure reads no differently from an unreported yes.

Only after this should compaction actually proceed.

## Note on location

This skill file must live under the **primary working directory's** `.claude/skills/` to be invocable as `/prepare-compact` — Claude Code does not scan `.claude/skills/` in additional working directories for slash-command registration. If you use this skill from multiple repositories or working-directory setups, keep a copy in each one's own `.claude/skills/prepare-compact/SKILL.md` and update all copies when this file changes. A newly added or edited skill file also typically requires a fresh session before Claude Code picks it up.
