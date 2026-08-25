---
name: wrap-up
description: Bring a chat session's live delegated agents and in-flight task tracking to a genuine logical pause point before preparing for a context compaction, so no agent is cut off mid-mutation and no task's in-flight/done boundary becomes ambiguous. Use whenever the session has delegated background agents or active task tracking and a compaction may be coming soon — run this before `prepare-compact`, not instead of it.
---

# Wrap up

`prepare-compact` describes itself as a bookkeeping pass, not an audit — its job is to transcribe state the session already established into durable files. That description only holds if the state is actually settled by the time it runs: no agent still mid-mutation, no task whose in-flight/done boundary is ambiguous. A session running several delegated background agents at once, each carrying its own in-progress reasoning and uncommitted intent, breaks that assumption — a transcription pass has no safe way to summarize work that hasn't reached a stopping point yet. Running `prepare-compact` straight into that state means it either silently understates what's actually live, or stalls trying to do audit-grade work it isn't built for. `wrap-up` exists to get a session from "agents actively working" to the quiescent state `prepare-compact` is entitled to assume. It is preparatory to that skill, not a substitute for any of its steps.

Do not run this skill's steps from memory of what it says below once invoked in a future session — re-read this file fresh each time.

**This is a coordination pass, not a shutdown.** Its job is to get every live agent to a deliberate, self-reported stopping point, and to leave task tracking in an unambiguous paused state — not to kill processes, cancel runs in progress, or force anything to stop mid-step. A delegated agent has its own in-progress reasoning and state; ending its turn abruptly is a rug-pull, not a pause. If a live agent can't reach a clean stopping point quickly, the right move is below (step 3) — never to force one anyway.

## 1. Identify every currently-outstanding agent and in-flight task

Use whatever mechanism this environment provides for listing genuinely running delegated work (many such mechanisms let you probe a non-existent task ID and read the resulting error, which lists what's actually live). Do this fresh — don't rely on this conversation's memory of what it dispatched, since a delegated task can outlive the turn that launched it.

Cross-reference the result against whatever task or todo tracking is currently active in this session, so that every tracked in-progress item is either accounted for by a live agent above, or flagged as in-progress with no corresponding live process. That mismatch is itself worth surfacing before proceeding, not silently carried forward.

## 2. Bring each live agent to a genuine logical pause point

For each agent identified in step 1, message it directly — don't simply stop waiting on it — with an explicit instruction to:

- Finish its current atomic unit of work (the file edit already underway, the command already running, the reasoning step already committed to) but not start a new one.
- Durably record, commit, or otherwise save anything it has genuinely finished, exactly as it would if reporting a normal completion — nothing it considers done should be left sitting only in its own working state.
- Report back a clean, explicit stopping point: what it finished, what it deliberately left undone and why, and what the next step would be on resumption.
- State explicitly whether it touched anything this project treats as requiring separate authorization beyond ordinary task scope — a shared framework, governing rules, foundational configuration, however this project draws that boundary — yes or no, even when the answer is obviously no. This is not housekeeping: a delegated agent is the one actor that can change something in a protected area without whoever dispatched it ever seeing it happen, which makes this the one disclosure nothing else in the cycle can observe on its own.

Record each agent's report as a structured entry — agent identity, assigned task, and the authorization disclosure — rather than one flat sentence per agent folded into prose. The shape is worth keeping deliberately narrow: a structured entry stays extensible (a later need can add a field to an existing entry instead of inventing a second, parallel record beside it) and stays checkable (a missing field is visibly missing, where a missing clause buried in a paragraph is not).

This is a deliberate handoff, not merely the absence of output. An agent partway through a multi-file edit, a multi-step operation, or any sequence where a partial application would leave the project or a durable record in a broken intermediate state has not reached a logical pause point just because it stopped producing output — it has reached one only once it has itself confirmed the four things above.

## 3. Handle agents that can't pause quickly, explicitly — don't paper over this case

Some agents won't be able to reach a clean pause point on short notice: mid-way through an external retrieval, a long-running command, or any other bounded unit with no safe interruption point inside it. For those:

- Don't force an interruption at an unsafe point merely to finish this skill faster. A forced stop mid-mutation is exactly the failure this step exists to avoid, and it's worse than waiting.
- Let the agent finish its current bounded unit of work, then have it report the same clean stopping point step 2 asks for.
- Record explicitly, in whatever you carry forward, which agent this applied to and why — a real, known-slow case, not a silent exception. If it's genuinely blocking (no foreseeable end, or an unreasonable wait), say so to the user plainly and let them decide whether to wait or accept the risk of proceeding with that one agent still live.

## 4. Pause, don't abandon, task tracking

Once every live agent has either reached a reported pause point (step 2) or is a documented exception (step 3), update whatever tracks in-progress work in this session to reflect reality, not aspiration:

- Anything an agent reported as genuinely finished and durably recorded may be marked done.
- Anything an agent reported as deliberately left mid-flight should be marked paused/in-progress, carrying the agent's own account of what's done versus what's left — never silently reset to "not started," never left claiming a state that's no longer accurate.
- The result should be resumable later without ambiguity: a fresh reader of the tracking state alone, with no memory of this conversation, should be able to tell exactly what finished, what paused and where, and what the very next action is for each paused item.

## 5. Verify it is actually safe before handing off to `prepare-compact`

Before invoking `prepare-compact`, confirm:

- No agent identified in step 1 is still mid-mutation — each is either fully stopped at a reported pause point, or is a step-3 exception whose bounded unit has since completed.
- No dangling lease, lock, or held resource remains that an abrupt stop would have left corrupted or contended.
- The task tracking updated in step 4 accurately reflects what every agent actually reported, not a guess at what it probably did.
- Every agent's authorization disclosure from step 2 is explicit, not assumed — an agent that never stated one hasn't been asked, so go ask, rather than recording an assumed "no." If any answer is yes, carry that forward plainly, since `prepare-compact`'s own reporting step depends on it.
- If this project distinguishes a verified-empty roster of live agents from one that simply couldn't be checked, don't collapse the two into each other — "nobody looked" is not the same claim as "nothing is running," and treating them as interchangeable is the exact false comfort this step exists to prevent.

If any of these isn't true yet, this skill isn't done — wait, re-message the outstanding agent, or surface the blocker to the user per step 3, rather than proceeding anyway.

## 6. Invoke `prepare-compact`

Once step 5 is confirmed, invoke the `prepare-compact` skill. Its own first step re-checks for live agents and dirty state as a cheap freshness check — that's expected and correct, a fast confirmation that this skill's work actually held, not a duplicate audit. `wrap-up`'s job ends here: don't perform any of `prepare-compact`'s own steps from within this skill.

**Where the line falls between the two skills.** This skill's write scope is narrow: if this project keeps a durable record of delegated-agent activity as part of its session state, step 2 above is what produces those entries, and nothing downstream reconstructs them if they go missing. Everything else in that session record — the narrative of current state, the next-action pointer, standing boundaries, an agreed plan — belongs to `prepare-compact`. Write the delegated-agent entries; leave the rest to the skill that owns it.

## Note on location

Install this skill in **one** of two places:

- **Personal — `~/.claude/skills/wrap-up/SKILL.md`.** Available in every project on the machine, so a single install covers all of them: nothing to copy, nothing to keep in step. The entry may be a symlink to a directory elsewhere on disk — Claude Code follows it, and loads the skill once even when the same target is reachable from more than one location.
- **Project — `<repo>/.claude/skills/wrap-up/SKILL.md`.** Scoped to that repository and committable with it, which is what you want when the skill should travel with the project rather than with the person. Where both exist the personal one wins: precedence runs enterprise, then personal, then project.

**Corrected 2026-08-25, and stated plainly because the previous version of this note cost adopters real work.** It said the file *must* live under the primary working directory's `.claude/skills/`, that additional working directories were not scanned, and that you should therefore keep a copy in every repository and update them all whenever this file changed. **A personal install covers every project; a `.claude/skills/` inside a directory added with `--add-dir` _is_ loaded; and there is no fan-out to maintain.**

**An edit does not need a fresh session either.** Claude Code watches these directories and picks up an added, edited or removed skill within the current session. The one case that genuinely needs a restart is creating a *top-level skills directory that did not exist when the session started* — there was nothing there to watch. This note previously claimed every edit required one.

**The one real limit on the personal install:** Cowork sessions, cloud sessions and routines do not read `~/.claude/skills/` from your machine. If the skill has to work in those, commit it to the repository's `.claude/skills/` or ship it in a plugin.

Codex discovers the same file under `.agents/skills/wrap-up/SKILL.md`, plus a personal `$HOME/.agents/skills/`, by the same walk-up convention. Its reload behaviour is not covered by any of the above — assume a fresh session there unless you have checked.

If a given project also maintains its own project-specific, non-generic variant of this skill (naming its own concrete files, governance documents, and incident history), that variant is the one to actually invoke and keep current for that project — this generic version is the portable baseline to adapt from, not a replacement for a project's own tailored copy where one already exists.
