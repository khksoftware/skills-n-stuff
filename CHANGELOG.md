# Changelog

All notable changes to these skills are recorded here. Versions follow
[Semantic Versioning](https://semver.org/), read against the skills as a
published set rather than against any single file.

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
