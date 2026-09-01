# Changelog

All notable changes to these skills are recorded here. Versions follow
[Semantic Versioning](https://semver.org/), read against the skills as a
published set rather than against any single file.

## 2.4.0 — 2026-09-01

### Added

- **Twenty-six environment traps, every one met in real work rather than imagined.** They
  span content assembly (`A6`—`A8`), long-running work (`B11`—`B13`), imports and
  subprocess (`C4`—`C6`), test scope (`D7`—`D9`), git (`E11`—`E20`), line endings
  under hash pinning (`F6`—`F7`), a detector that chooses its own destination (`G3`), and
  a new section `J` on toolchain availability.

  The largest group is git, and the two worth reading first are `E19` and `E20`. `E19` is
  the INBOUND direction of a hazard usually documented only outbound: you correctly decline
  to commit a shared file, another process commits it first and takes your change with it,
  and your working tree goes clean without you having committed anything. `E20` is the
  reassurance problem: a check proving a deployed control matches its tracked source says
  nothing about whether anything loads it, and a control can be tracked, deployed,
  byte-identical, covered by passing tests and referenced by nothing.

  `J1` is the one most likely to change a decision: an absent command-line client says
  nothing about whether the capability exists, and the credential is frequently already on
  the machine.

### Fixed

- **Thirty-eight em dashes in the newly added entries had arrived as U+FFFD REPLACEMENT
  CHARACTER, and the cause was not where it looked.** The natural suspicion is the write,
  the read, or a line-ending pass. Measured: the source scripts already carried all
  thirty-eight and carried zero surviving em dashes, so the corruption happened when those
  scripts were AUTHORED and the write copied faithfully what it was handed.

  It presented as success from both ends — the run exited zero with the right count, and
  the output looked correct in a terminal that cannot render an em dash anyway. It was
  visible only in a renderer, which is to say only to a person reading the published
  result. That is now `A8`, together with the two defences that follow from it: prefer ASCII
  in generated content, and where a non-ASCII character is genuinely required build it from
  its codepoint so it never travels through the authoring path at all.

## 2.3.0 — 2026-08-27

### Added

- **`resume` step 4 now sweeps for abandoned worktrees, and `E10` records why.** A run that creates a temporary linked worktree is expected to remove it in finalization; a run that dies or skips that path leaves a full checkout and a registration that nothing retires. Measured: nine registered worktrees on one machine, three of them detached validation checkouts pinned at commits two and three days old, found only because the user asked why their disk was busy.

  The removal criterion is stated so it can be applied without judgement, and rests on one fact worth stating outright: **removing a worktree deletes a checkout and never a branch or a commit**, so the only thing at risk is uncommitted content -- which is why the status check must count untracked paths, the case that has already stopped a removal for real. The skill also requires reporting what was kept, not only what was removed, because a sweep reporting only deletions cannot be checked.

## 2.2.1 — 2026-08-26

### Fixed

- **`B10` contradicted itself, and the wrong half was the published half.** Its opening
  states the rule correctly — an injection is recorded only when it arrives at a turn
  boundary, so a message to a *running* agent is not. Its closing then generalised that
  away by message class, claiming terminal task-notifications and coordinator messages are
  *likewise recorded*, making the transcript a complete inbox for three classes and not the
  fourth. **That is false.** The discriminator is the recipient's state and nothing else:
  any class delivered to a running agent goes unrecorded.

  Measured by execution: an agent received its own child's terminal notification while it
  was running and its transcript held 129 entries and **no origin entries at all**, while
  across 252 per-agent transcripts every one of the 46 origin entries was a resume-boundary
  entry.

- **The census behind the original claim was misread, not wrong.** The entries it counted as
  terminal notifications are about background shell commands rather than agents, so they
  were never evidence for the claim they were cited for. The `Detect` section now says so,
  because a kind breakdown reads like evidence and is not one until each entry's subject is
  checked.

## 2.2.0 — 2026-08-25

> **Superseded in part by 2.2.1.** The sentence below about *three classes of injection*
> is false; the discriminator is the recipient's running/stopped state, not the message
> class.

### Added

- **`B10` — an agent's transcript records what was injected into it, except while it is
  running.** A runtime that writes a per-agent transcript records injected messages
  only when they arrive at a **turn boundary**. A message delivered to an agent that is
  already running is injected into its context and never written to its transcript at
  all; the same message sent once it has stopped resumes it and *is* recorded, with a
  structured origin object carrying the sender's unique agent id. So the transcript is
  a complete inbox record for three classes of injection and silently is not one for
  the fourth — and the gap falls exactly on messages sent to a **busy** agent, which is
  when supervision matters.

  **Both directions of the error are live.** A supervisor reconstructing what an agent
  knew gets a confident wrong answer; and two mid-run sends returning nothing are
  enough to conclude the transcript is not an inbox at all, discarding a sender-identity
  record the runtime genuinely does write. Established by execution — a census of 237
  per-agent transcripts in one session, and a probe agent sent four messages across both
  delivery boundaries, which recorded only the resume-boundary pair.

  Filed in section B rather than appended at the end, which is where a naive append put
  it: it belongs with the other traps about whether delegated work is alive and who is
  telling you.

## 2.1.0 — 2026-08-25

### Fixed

- **The location note at the end of every skill was wrong, in the direction that
  costs work.** It said the file must live under the primary working directory's
  `.claude/skills/`, that additional working directories were not scanned, and
  that anyone using these from more than one repository should therefore keep a
  copy in each and update every copy whenever a file here changed. Reported by an
  adopter whose user-level install registered perfectly well, and then checked
  against the current Claude Code documentation rather than corrected from the
  report alone.

  A personal skill at `~/.claude/skills/<name>/SKILL.md` is available in every
  project on the machine. Precedence runs enterprise, then personal, then
  project, so a personal copy also wins over a project one. An entry may be a
  symlink to a directory elsewhere on disk, and the same target reachable from
  more than one location loads once. A `.claude/skills/` inside a directory added
  with `--add-dir` is loaded, and watched. **There is no fan-out to maintain.**

  **The README's install section has described the locations correctly since
  2.0.0**, so this repository was contradicting itself in two places at once —
  the correct answer and the expensive one, shipped side by side.

- **An edit does not need a fresh session, and the note claimed it did.** The
  report did not raise this one; the same documentation check turned it up.
  Claude Code watches these directories and picks up an added, edited or removed
  skill within the current session. Only creating a *top-level skills directory
  that did not exist when the session started* needs a restart, because nothing
  was there to watch.

- **The one real limit on a personal install is now stated rather than left to be
  discovered.** Cowork sessions, cloud sessions and routines do not read
  `~/.claude/skills/` from the machine, so a project install — and
  `sync_into_repo.py` — keep their purpose.

### Added

- **`prepare-compact` now sweeps for warnings the session worked around rather
  than resolved** — a check that keeps failing and keeps getting bypassed, a test
  red or flaky since early in the session, a validator whose output has been read
  past more than once. This is a different shape from the five things the sweep
  already looked for, every one of which was *said once and lost*. Here the
  tooling said it **repeatedly**, and each individual decision to move on was
  defensible by itself; what does not survive a compaction is the **pattern**,
  because no single instance ever looked worth recording. A deliberate deferral
  is a legitimate answer provided it reaches a file — the failure is leaving it
  unwritten.

## 2.0.0 — 2026-08-18

The first tagged release. Everything before this point was published only as a
branch tip, so this entry is written as the baseline rather than as a diff
against a version nobody could pin.

### Added

- **`wrap-up`** — a third skill, and the one that makes the other two honest.
  `prepare-compact` describes itself as a bookkeeping pass rather than an audit,
  which only holds if the state it transcribes has actually settled. Run into a
  session with agents still mid-mutation, it either understates what is live or
  stalls attempting audit-grade work it was never built for. `wrap-up` brings
  every live agent to a deliberate, self-reported stopping point first. It is a
  **coordination pass, not a shutdown**: an agent partway through a multi-file
  edit has not reached a logical pause point merely because it stopped emitting
  output, and forcing one is worse than waiting.

- **`resume` now starts by confirming it is in the right project.** An agent
  environment routinely has several repositories open at once, and the one the
  harness calls "primary" reflects whichever folder a window happened to have
  open. A resumption that grounds itself in the wrong one can find only template
  scaffolding, conclude there is no work to resume, and report that — while the
  real work sits in a sibling repository the same environment already had open.

- **`prepare-compact` now sweeps the conversation itself**, not just the work.
  What escapes a transcription pass is smaller and less visible than work: an
  authorization granted in passing, a routing decision announced and never
  written down, a correction that falsifies something a durable record still
  asserts, a delegated agent's finding that reached the orchestrator in prose
  and nowhere else, a stated limit that stops a later claim from overreaching.
  None of these has a dirty file or a running process to hang from, which is
  exactly why a pass organised around work steps over them.

### Changed

- **`prepare-compact` replaces its record sections in place and never prepends
  to them.** Prepending disguised as rewriting is how a record regrows to its
  pre-pruning size within days of being pruned, one dated block at a time, with
  nobody removing the block underneath. Content is either in force — in which
  case it is replaced where it stands — or it is not, in which case it is
  archived. One destination, one operation, no judgement call between two
  answers.

- **A durable record cannot truthfully describe its own run's publication
  state.** The field is written before the commit that carries it, so it sits
  inside the commit it would have to describe; moving the write later makes it
  unpublished instead of stale, and no ordering resolves that. The skill now
  states the stable facts and hands the reader the commands that resolve the
  volatile ones, rather than recording a verdict that was false by construction.

- **An instruction an agent cannot satisfy is more dangerous than one it
  skips.** A skipped step is visibly incomplete. An agent that complies by
  inventing the structure the instruction names produces something
  indistinguishable from success — and, where the invented structure is one that
  was deliberately retired, quietly reintroduces it. Both skills now say so
  where it matters.

- **Empty and unobserved are different facts and are never collapsed.** A roster
  that was checked and came back empty is not the same as a roster nobody could
  look at, and reporting the second as the first is how "nobody looked" comes to
  read as "nothing is running".

- **A promise made only in prose is invisible to every tracker.** `resume` now
  treats an obligation stated in a record but attached to no trackable item as a
  finding to allocate or retract, not a note to come back to.

### Notes

- The skills remain project-agnostic: no product names, no governance-framework
  names, no work-item identifiers, no local filesystem paths, and no counted
  facts about the estate they were extracted from. The only environment-specific
  content is each skill's own install location, which a skill needs in order to
  be installable at all.
- `sync_into_repo.py` enumerates skills from this directory rather than from a
  hardcoded list, so the third skill required no change there to be published.
