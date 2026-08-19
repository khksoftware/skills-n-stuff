# Changelog

All notable changes to these skills are recorded here. Versions follow
[Semantic Versioning](https://semver.org/), read against the skills as a
published set rather than against any single file.

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
