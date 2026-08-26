# Agent environment traps

Environment- and tool-level failure modes, each one met in real agentic coding work rather than imagined. Observed on Windows, with an agent runtime driving both PowerShell and a POSIX-style Bash tool, Git for version control, and Python with pytest as the toolchain. Most of them are not specific to that combination.

## The one property worth reading twice

A large share of these traps **present as SUCCESS** — a green exit code, a passing test, a "consistent" reconciliation report, a clean commit — while doing something silently wrong. That is what makes them expensive: a trap that fails loudly teaches itself, and one that reports success teaches you the wrong lesson instead. Every entry where it applies says so in bold.

## A. Shell quoting and content assembly

Every entry here is one failure: **content corrupted as it crosses a shell boundary.** Two are Bash, two are PowerShell, and one of those two is specifically about confusing the two shells for each other — the mixed environment is the point, not an accident of where these were found. The rule they all reduce to is stated at A3: content that will be stored should never be assembled by the shell that runs the command. For PowerShell's *language* semantics — collections, parameter binding, scoping, cmdlet behaviour — see section I.

### A1. Heredoc backslashes silently collapse

**What breaks:** Some shell/heredoc environments collapse doubled backslashes inside heredocs. Every escaped sequence (e.g. `\\n`) you write inside a heredoc can arrive at the receiving program as a real, unescaped character, so a find-and-replace script hunting for the escaped sequence matches zero occurrences.

**Presents as: SUCCESS.** The script exits 0 and reports a completed edit — and the file is unchanged. The failure compounds if verification runs inside the *same* command: a test then exercises the unedited file, and its green result reads as proof the edit worked. Two independent mechanisms can each silently report health.

**Detect:** Grep the target file for the new content *after* the command returns, in a separate invocation. Never accept an edit script's exit code as evidence.

**Do instead:** Pass the pattern through a file, a command-line argument, or a verified raw-string literal, rather than embedding it in a heredoc. Prefer a dedicated file-editing tool over a shell-mediated edit script entirely. Assert the precondition *before* writing, not only after — an edit script whose asserts gate the write converts silent corruption into a visible no-op.

**Remedy:** Guard the write with an assert on the match before writing, then re-read the file in a *separate* command. Never verify inside the command that performed the write.

### A2. A PowerShell here-string is not a Bash heredoc

**What breaks:** PowerShell here-string syntax (`cmd -m @'...'@`) is not heredoc syntax in a POSIX/Bash shell. Bash reads the `@` as a literal one-character argument and the remaining lines as separate arguments or as nothing at all.

**Presents as: SUCCESS.** The command exits 0, and only the *content* is wrong — e.g. a commit's subject becomes a bare `@`, with the real message demoted into the body. Nothing in the exit code or immediate output reveals it.

**Detect:** Read the result back in a *separate* command (e.g. `git log -1 --format=%s` after a commit). A one-character or empty result is the signature.

**Do instead:** Write the content to a file and pass it by path (a `-F <file>` flag, or your tool's equivalent file-input form) rather than assembling it inline in a shell string. A mixed environment's two shells take different syntax, and a command valid in one can be silently wrong in the other.

**Remedy:** `git commit -F <path-to-message-file>` (or the equivalent file-input form for whatever command you're running).

### A3. Backticks inside a double-quoted Bash string run as commands

**What breaks:** Prose containing backtick-quoted tokens (ticket numbers, code symbols, command names written in Markdown style) passed through a double-quoted `bash -c "..."` string — or any double-quoted shell context — makes the shell evaluate every backticked span as command substitution. Each one runs, fails with "command not found," and is silently replaced with the *empty string*. The surrounding text is untouched, so the sentence still reads as a sentence with the tokens deleted out of it.

**Presents as: SUCCESS, with wrong content**, and the diagnostic is easy to discard. The write exits 0, and downstream validation (schema checks, round-trips) passes, because a missing token is rarely itself a schema violation. The only signal is "command not found" lines on stderr, which read as unrelated shell noise next to a success message.

**Detect:** Re-read the *written* value in a separate command and assert the expected tokens are present — never the source you intended to write, and never the writer's exit code. Also read stderr, not just the exit status: a "command not found" naming one of your own tokens is this trap and nothing else.

**Do instead:** Never pass content containing backticks through a quoted shell string. Write the body to a file directly, then have your program read the file. Assert the tokens are present in the *source* before writing, so a mangled input fails closed rather than landing. A single-quoted heredoc also suppresses substitution for genuinely static content, but doesn't compose with content assembled elsewhere.

*General takeaway, shared with A1 and A2: content that will be stored should never be assembled by the shell that runs the command.*

### A4. A double quote inside a PowerShell here-string still splits a native executable's argument

**What breaks:** A single-quoted PowerShell here-string (`@'...'@`) is literal to PowerShell — it correctly resists `$` expansion and backtick substitution — but the resulting string is re-parsed on the way to a *native* executable. A double-quote character anywhere inside the here-string ends the argument there, and every following word arrives as a separate argument. The here-string syntax itself is correct; the defect is entirely in the native-argument handoff, so the usual fix for A2's trap does not fix this one.

**Presents as:** Usually loudly — a wall of "pathspec did not match" style errors, one per stray word, which is the good case. The bad case is silent: if any stray word happens to match something real (a file path, an option), the command succeeds having consumed a completely different, truncated argument list. Careful quoting doesn't protect you, because the argument list itself is what gets corrupted.

**Detect:** Before running the command, look for a literal `"` anywhere in the here-string body. After running a command whose payload contained one, verify both the effect and the full content survived intact.

**Do instead:** Write the content to a file and pass it by path/flag rather than inline. This removes the content from argument parsing entirely, so quotes, backticks and dollar signs are all inert. Rewording to avoid quotes works too, and is worse — it lets the shell edit your content.

**Remedy:** Pass the message/content via an external file argument, never inline in a here-string, whenever a native (non-shell-builtin) executable is the target.

---

### A5. `printf` reinterprets backslash sequences, so what lands is not what you wrote

**What breaks:** `printf` interprets backslash escape sequences in its argument. A Windows-style path written through it is silently transformed -- a backslash followed by certain letters becomes a control character and the rest of the sequence disappears. This bites hardest where the whole point of the file is to hold a literal path, which is exactly the case when building a fixture for a path-detecting check.

**Presents as: SUCCESS, with a clean exit code and a plausible file.** The content is corrupted on write, the file exists, the subsequent commit returns 0, and the committed blob is wrong. Caught only by reading the blob back.

**Detect:** Read the written content back in a *separate* command and compare it byte for byte against what you intended, whenever the content contains backslashes. An exit code proves the command ran, never that it wrote what you meant.

**Do instead:** Write the file with your agent's file-write tool rather than a shell `printf`/`echo`. Where a shell is unavoidable, a single-quoted heredoc passes the bytes through unchanged. The general rule: shell text-emitting builtins reinterpret their input, and every trap in section A shares one remedy -- do not assemble literal content in a shell at all.

## B. Long-running work: whether it is alive, whether it finished, and who is telling you

### B1. Piping a long run through `tail`/`head` loses it

**What breaks:** Piping a long-running command through `tail` (or `head`, or a pager) buffers the whole stream until the process exits, and destroys the result if the run is killed or backgrounded first.

**Presents as:** No output, a truncated run, or an apparently hung command — even when the underlying run was entirely healthy. What's lost is the *evidence*, not the work.

**Detect:** Any pipeline of the shape `long-command | tail`, `| head`, or a pager, on a run lasting more than a few seconds.

**Do instead / remedy:** Redirect to a file, then read the file: `long-command > out.txt 2>&1`, then read `out.txt` separately. This trap is easy to know about and easy to commit anyway under time pressure — it has been observed to catch even someone actively warning others about it, within the same hour.

### B2. Redirected stdout block-buffers and looks stalled

**What breaks:** An interpreter (e.g. Python) block-buffers stdout when it's redirected to a file rather than attached to a terminal, so a long run's progress doesn't reach the file as it's produced. Measured directly: a script writing steadily to a redirected file sat at zero bytes for several seconds, then delivered everything at once at process exit; the same script with unbuffered output advanced continuously.

**Presents as: A HANG.** The output file sits at the same byte count across repeated reads, and the natural conclusion is that the run has stopped or deadlocked — the opposite symptom to B1 (which *loses* output rather than delaying it), though the underlying cause is the same, so the two are worth knowing together.

**Detect:** Check the OS process, not the output file — compare accumulated CPU time across two observations minutes apart. A frozen file has two possible causes (buffering, or genuinely slow work in the current section), and only running with unbuffered output separates them: if the file still doesn't advance with buffering disabled, it's slow work, not buffering.

**Do instead / remedy:** Force unbuffered output whenever a redirected run's progress must be readable while it executes (for Python, the `-u` flag). Where a run is already in flight without it, read the process rather than the file. When waiting on such a run, make the wait exit on *every* terminal state, not only a success marker — a watcher that greps only for a summary line is silent through a crash, and silence is indistinguishable from still-running.

### B3. Process/port listings go stale during rapid restarts

**What breaks:** Process and network-connection listings (e.g. `Get-Process`, `Get-NetTCPConnection` on Windows) report stale, misleading state during rapid kill/relaunch cycles: a killed process still listed, a released port still shown bound, a new listener not yet visible.

**Presents as:** A confident diagnosis of the *wrong* process — "the fix didn't take" and "the old process is still answering" look identical from a stale listing.

**Detect:** Check the listing twice, seconds apart, and treat disagreement between the two observations as the actual answer rather than as noise.

**Do instead / remedy:** Prefer a full clean restart over continued diagnosis of an ambiguous listing, and re-verify any claimed-fixed live bug against the actually-running application rather than against a process/port listing.

### B4. Process display names vary by installed version

**What breaks:** The display name a process shows up under in a process listing can vary by installed version.

**Presents as:** A watchdog or monitoring check matching nothing at all, or matching the wrong process.

**Detect:** Compare against your own declared list of expected process names, and confirm by process tree/parentage rather than by name alone.

**Do instead:** Identify processes you own structurally (parent/child relationship, working directory, command line), never by a hardcoded name string alone; never terminate a process you can't positively identify as your own.

### B5. Process-liveness readings disagree, and acting on the wrong one corrupts a live run

**What breaks:** On Windows, different process-liveness mechanisms (e.g. a POSIX-style `ps` inside a Bash layer, versus native WMI/CIM queries) can disagree about whether a given process is still alive — and the "dead" answer can be the wrong one. Concluding a long run has died and then deleting its working directory doesn't stop the run: it keeps executing against a tree being removed underneath it, and finishes with corrupted results rather than an error.

**Presents as:** A large, plausible-looking failure count — which is the real danger. Hundreds of failures that look like a catastrophic regression can, on inspection, be entirely artifacts of the tree being deleted out from under a run that was never actually dead.

**Detect:** Before deleting or reusing a directory a long run might still be using, prove the run is finished by its *own output* (a completed run writes a final summary line to its redirected file) rather than by any process table. Absence of a process in one listing is not proof of absence. Afterwards, treat any result from a run whose working directory was touched mid-flight as void — especially if something already proven to work in isolation now shows as failed for a reason (like a missing fixture) that points at the deletion.

**Do instead:** Leave the working directory alone until the run's own output file shows a completion line. If a run must genuinely be abandoned, abandon its *result* too rather than salvaging a tally from it — a contaminated number that looks like real signal is worse than an honestly absent one.

### B6. A task/agent registry is not authoritative about what's actually running

**What breaks:** A task or job registry can report a completed process as still running, and can report an empty roster while something is genuinely live. Both directions have been observed within minutes of each other.

**Presents as:** A confident, wrong answer in either direction — a phantom live job, or an invisible real one.

**Detect:** Corroborate the registry against the observable footprint: working-tree state, actual OS processes, lock files.

**Do instead:** Treat the registry as one input among several, never as the answer by itself.

---

### B7. A backgrounded process's completion notification never arrives

**What breaks:** A delegated agent (or any process) that backgrounds a long-running job and then ends its turn/session waiting to be woken by a completion notification can simply never receive one — including when the backgrounding is wrapped in what looks like a proper monitor, not just a raw background process.

**Presents as:** A silently stalled agent that looks like a slow run rather than a stopped one.

**Detect:** Actively probe for the real process/job at the start of any turn where a background job might be outstanding, and corroborate against the actual OS-level process — don't trust a "still running" status alone.

**Do instead:** Poll actively for the process within the same turn/session rather than yielding and waiting to be woken. If something has already stalled this way, locate the real underlying process from the orchestrating context, wait on it there, and feed the result back in manually.

### B8. A subagent's plain-text output never reaches its invoker

**What breaks:** In many agent-orchestration setups, a subagent's plain prose final answer is visible only within its own transcript — it does not automatically propagate to whatever invoked it. Only an explicit, structured "send this message/result back" call actually delivers anything. An instruction to "report back" whose fulfillment the invoker can't observe is effectively no instruction at all.

**Presents as: SUCCESS at both ends.** The subagent writes a careful, complete report and considers its job done; the invoker sees nothing at all and has no way to distinguish a conforming agent from a silent one. Neither side notices the gap on its own.

**Detect:** Ask what durable *artifact* carries the claimed result. If the honest answer is "the agent said so in its own final message" and nothing else, nothing durable actually exists yet.

**Do instead / remedy:** Have agents commit findings to a durable location (a file, a shared record) *as they're produced*, not only at the very end, and have the final message name/point to that location rather than restating the content as the deliverable itself. If a session or turn limit can kill agents abruptly, the ones that had already written to disk keep their findings; the ones holding everything in the final message lose it entirely.

### B9. A delegated agent that yields waiting for a wake-up notification is simply terminated

**What breaks:** A delegated agent that launches a long detached run and then ends its own turn, intending to be resumed later by a monitor or completion notification, is *terminated* at that point in some agent runtimes. Nothing resumes it automatically. The background run may finish perfectly, and the result reaches nobody, because the only party that knew where its output lived has already stopped existing. Writing findings to disk incrementally as they're produced is the standing defense and genuinely helps — but it doesn't cover the very last step: a report built up incrementally still needs one final "commit/finalize" action, and an agent that yields before that leaves even a near-complete report stranded.

**Presents as:** Exactly like an in-progress status update, which is precisely why it's easy to miss — a terminated agent's final recorded message often reads as a plan stated in the present tense: *"I've launched the run and I'm waiting on a monitor that will notify me when it finishes; I'll write up the final report once that arrives."* That is a description of intent from an agent that no longer exists to carry it out. This kind of failure can recur multiple times in a single working session, including with substantial completed-but-uncommitted work left behind in a checkout other work was actively relying on. Observed five times across two consecutive days, twice from a single dispatch of two agents working on unrelated tasks — roughly half an hour of runtime apiece, returning no findings at all. The orchestrator's own dispatch-tracking list named both as still outstanding while one had already stopped, and it was wrong in that same direction both times; only a live roster probe was right.

**Detect:** Read an agent's final message specifically for a future-tense commitment — "I'll finalize," "once that arrives," "waiting on." A terminal message describing what the agent is *about to* do next is a stopped agent, not a running one. Confirm against the live roster of actually-running agents, then check for uncommitted work or orphaned resources it may have left behind.

**Do instead / remedy:** Write the task so the agent never yields mid-task waiting on an external wake-up — have it poll/read its own run's output directly and finish within the turn it started, rather than blocking on being told. Where a long run is genuinely unavoidable, have the agent report an explicit partial result with its own denominator rather than blocking for a complete one. If an agent has already stopped this way, most runtimes let you resume it directly by its own identity, restoring its full prior context rather than starting over — that recovery is usually cheap, which is the real reason it's worth checking for.

**Do not carry this rule in the per-task brief alone.** The two same-day instances above both happened under briefs that pointed their agents at a written list of environment traps containing this exact entry, without reproducing the prohibition itself — so the rule existed, was cited, and still did not arrive. A rule that must survive every dispatch belongs in the standing agent definition, where no individual brief can omit it; a brief that merely points at where the rule lives is not the same thing as stating it. And forbid waiting on a *concurrent* worker or stream for the same reason: work you do not own has no completion you control, so a contended resource is a finding to report, not a condition to block on.

---

### B10. An agent's transcript records what was injected into it — except while it is running

**What breaks:** A runtime that writes a per-agent transcript records both what the agent emitted and what was injected into it — but only for injections that arrive at a **turn boundary**. A message delivered to an agent that is *already running* is injected into its context and is never written to its transcript at all. The same message sent to that same agent once it has stopped instead **resumes** it, and *is* recorded, as an entry carrying a structured origin object: kind, sender name, the sender's unique agent id, and the body. **The discriminator is the recipient's state, not the message class.** Any injection delivered to a *running* agent goes unrecorded — terminal notifications and coordinator messages included. Measured: an agent received its own child's terminal notification while it was running, and its transcript held 129 entries and **no origin entries at all**; across 252 per-agent transcripts, every one of the 46 origin entries was a resume-boundary entry, without exception.

**Presents as: SUCCESS.** The transcript parses cleanly, is complete for everything else, and is appended live — measured about five seconds behind a running agent's own file writes. Nothing signals that an entire injection class is missing. A supervisor reconstructing *what did this agent know* gets a confident, wrong answer, and the gap falls precisely on messages sent to a **busy** agent, which is exactly when supervision matters. **The inverse error is equally available:** two mid-run sends, neither recorded, are enough to conclude the transcript is not an inbox at all — a generalisation that would discard the sender-identity record the runtime genuinely does write.

**Detect:** The sender can tell which class it sent, at send time, from the tool's own response string — a *queued for delivery at its next tool round* means mid-turn and will **not** appear on the recipient side, while a *resuming agent* means a turn boundary and will. On the recipient side, scan its transcript for entries carrying an origin object and group them by kind. Measured in one session: a census of 237 per-agent transcripts found 42 injected entries across 24 agents, plus over a thousand more in the main session transcript; a probe agent sent four messages — two mid-turn, two at a resume boundary — recorded only the resume-boundary pair. **Correction, measured later:** the entries that census counted as terminal notifications turned out to be about background shell commands rather than agents. Do not read a kind breakdown as evidence about agent notifications without checking each entry's subject.

**Do instead:** Never treat a recipient's transcript as the complete record of what it was told. Cross-check against the **sender's** own transcript, which records every send as a tool call carrying the full recipient and message — that leg is complete for both classes.

**Remedy:** Where sender identity must be adjudicated rather than trusted, read the sender's unique agent id from a resume-boundary entry's origin object. The sender does not control it, and an attempt to forge a second wrapper from inside the message body is escaped by the runtime, so the origin fields stay authoritative. **Residual worth knowing:** the escaped text still renders close enough to the real thing that a recipient reading rendered prose reported two apparent senders on both spoof attempts. Adjudicate from the record, never from what the recipient believes it saw.

## C. Python and subprocess

### C1. A stale or wrong virtual environment produces a wave of fictitious failures

**What breaks:** An old or incompletely-maintained virtual environment left inside a repository can be missing dependencies the current code needs. Running a test suite under it produces a large number of failures that have nothing to do with the code.

**Presents as:** A large, alarming, entirely fictitious red — it looks like a broken repository rather than a broken interpreter/environment choice.

**Detect:** Import errors on a dependency that "should" be installed, or a suspiciously large, round-ish failure count that doesn't correspond to any real change you made.

**Do instead:** Use the interpreter/environment you know is correctly provisioned (a system installation, a freshly-built venv, a CI image) — never a stale one sitting in the repo out of convenience. Record which interpreter/environment produced any validation result you report.

### C2. Running a script by file path instead of as a module

**What breaks:** Running a repository script by file path (`python path/to/script.py`) raises `ModuleNotFoundError` naming the top-level package, when that script does internal imports assuming it was launched as part of the package.

**Presents as:** A hard, loud failure — but one commonly misread as a broken script or broken environment rather than as a wrong invocation form.

**Detect:** `ModuleNotFoundError` naming a top-level package that clearly does exist in the repo.

**Do instead:** Invoke as a module from the repository root (`python -m package.path.to.script`) rather than by file path. This is also frequently the only invocation form a project's own launcher scripts and tooling actually validate.

### C3. `subprocess` text mode silently mis-decodes non-ASCII output on Windows

**What breaks:** `subprocess.run(..., text=True)` on Windows decodes the child process's output using the system's legacy ANSI codepage, not UTF-8. Any non-ASCII byte a tool emits is silently mangled or replaced rather than raising an error.

**Presents as: SUCCESS, with wrong content.** A genuinely correct operation (e.g. a file migration) can report phantom content differences purely because the comparison ran over mis-decoded text — exit code 0, every line individually looking plausible. Any comparison, hash, or diff built on that text is wrong in a way no error surfaces.

**Detect:** Compare a byte-level read of the same output against the decoded string, or assert that a known non-ASCII character round-trips correctly. A test fixture that's ASCII-only cannot catch this.

**Do instead / remedy:** Pass `encoding='utf-8'` explicitly alongside `text=True` (or capture bytes and decode yourself). Never rely on the platform default codepage for content you're going to compare, hash, or diff.

---

## D. pytest and test collection scope

### D1. pytest cache residue lands inside a directory governed by an exact-file allowlist

**What breaks:** Running pytest with its working directory inside a tree that's governed by an exact-file allowlist (a manifest, a "these are the only files that may exist here" check) writes `.pytest_cache/` and `__pycache__/` into that tree, which then fails the allowlist check.

**Presents as:** Tests failing on an allowlist/manifest mismatch — "unexpected file present" — with nothing wrong in the code under test. This has been severe enough in practice to **mask real, unrelated failures** behind a residue-caused error.

**Detect:** Check the tree's file/version-control status after any run, and read the allowlist-mismatch message for a `.pytest_cache` or `__pycache__` path. If tests governed by such an allowlist fail on an "unexpected file" message, suspect residue before suspecting the code.

**Do instead / remedy:** Pass `-p no:cacheprovider` on any pytest invocation whose working directory is inside such a governed tree. Prefer running from outside the tree with an explicit test path rather than changing directory into it. If residue has already landed, remove it before re-running rather than interpreting the resulting failure as real.

### D2. `unittest discover` silently collects only a fraction of a mixed test suite

**What breaks:** `python -m unittest discover` collects only class-based `TestCase`s. In a codebase with hundreds of bare pytest-style test functions, it takes none of them, and reports zero import errors while silently skipping modules it can't import.

**Presents as:** A green run over a fraction of the codebase (measured as low as ~57% in one real case), indistinguishable from a green run over all of it. An entire module holding every known failure can be wholly invisible to this discovery mode — the failures don't fail, they vanish.

**Detect:** Compare the collected node/test count against `pytest --collect-only -q` (or your runner's equivalent enumeration). Any gap is the trap.

**Do instead:** Use pytest (or whatever runner actually collects your full test style) for estate-wide validation. Reserve `unittest discover` only for the narrow cases that specifically require that exact invocation.

### D3. Two same-named top-level packages in different trees shadow each other under one test run

**What breaks:** If two separate directory trees in a repository each contain a top-level package with the *same* name, naming both trees on one test-runner command line binds that package name to whichever tree is imported first. Every module that meant to import the *other* tree's same-named package then fails at import — and this is exactly the case a validation run spanning both trees legitimately needs to do.

**Presents as:** Loud collection errors, but pointing at the *wrong* place: the error names the package and the tree it resolved to, while the actually-failing file has nothing to do with that tree. The real damage is silent shrinkage of what got tested: one measured case collected 394 nodes running the trees together versus 176+368=544 running them separately — **150 test nodes vanished**, and the smaller number looked entirely plausible on its own. Under a "continue past collection errors" flag, or for anyone reading only the pass count, this loss is completely silent.

**Detect:** Sum the per-tree collected-test counts from separate `--collect-only` runs and compare against the combined invocation's count. A gap is the trap. Also read the actual collection-error block, not just the summary line — a runner can exit non-zero for a reason no failure count shows.

**Do instead / remedy:** Run the two trees as *separate* test-runner invocations rather than naming both on one command line. Report the result as two runs with two denominators, not one combined figure. Do not reach for a "continue past collection errors" flag here — it converts a loud, obvious failure into exactly the silent partial run this trap is about.

### D4. A test that re-reads a live, shared file changes verdict with no code change

**What breaks:** A test that binds its subject at import time but re-reads that same file/state at call time will pass or fail depending purely on whether something else has written to that file in the meantime — nothing to do with the code under test. In a codebase touched by many concurrent agents or processes, this can affect a large share of the suite.

**Presents as:** Both outcomes look like real verdicts about the code. The same command against the same tree can genuinely report different failure counts run to run, and neither number is actually about the code.

**Detect:** If a test reads a live, shared file (rather than an isolated fixture) at call time, its verdict depends on what else is running concurrently. Re-run it with nothing else in flight and see if the verdict changes.

**Do instead / remedy:** Run any suite that reads a live/shared tree in a detached, pinned copy of that tree (e.g. a detached git worktree at a fixed commit) rather than against a tree other processes might still be writing to. A result taken against a moving tree is not reproducible and shouldn't be reported as one.

### D5. Tightening a shared contract can abort test collection instead of failing a test

**What breaks:** If test modules validate or render a live, shared artifact at *module import* time (code that runs as soon as the test file is imported, not inside a test function), then tightening a schema or contract that artifact must satisfy doesn't produce a failing test — it raises during collection and the test runner abandons the *entire* run.

**Presents as:** A run that finishes in seconds with a single "error during collection" and *zero* tests executed — reading as a broken import in one module, rather than as a shared contract the artifact can no longer satisfy. If checked only via a background task's completion notification, this can additionally read as a fast success (see D6), because the notification only reports that the wrapper process exited, not what it actually did.

**Detect:** After any change to a shared contract (a schema, a required field, a renderer), run the owning test suite and read the *collected* count, not just the failure count. A collected count far below normal, or the word "Interrupted," is the signature.

**Do instead:** Sequence a tightened contract behind the content it constrains — land only the relaxing half that existing content can still satisfy, and keep the strict half pending until the artifact itself has actually moved to comply.

### D6. A background task's completion notification reports the wrapper's exit, not the run's result

**What breaks:** A background task's completion notification carries the exit status of the *wrapper* process that hosted the command, not of the command itself. A long test run that dies partway (its working directory disappears underneath it, or it's hard-stopped) can be announced as "completed, exit code 0," with no summary line and no indication the run was truncated.

**Presents as: SUCCESS,** over a denominator that never actually ran. Taken at face value, the notification reads as a full suite finishing cleanly — in real cases this would have made thousands of tests that never executed look green. There's no partial-result marker; the only difference between a truncated run and a complete one is the absence of a summary line, and absence is exactly what a reader tends not to notice.

**Detect:** Read the actual exit/status marker out of the run's *own* output file, not the notification's status field. Confirm a genuine summary line is present. Confirm the process ended for the reason claimed. Any one check alone can be fooled by a truncated run; use all three.

**Do instead / remedy:** Never state a test-suite result from a completion notification alone. State it from the run's own summary line; where no summary line exists, report the partial denominator explicitly rather than the total that was merely collected. Redirect long runs to a file with unbuffered output, poll the file for the summary line, and treat the notification only as a signal that something ended — never as evidence of what it ended as.

---

## E. Git: history, worktrees, hooks, staging

### E1. `git rev-list --all` under-reports a repository whose history was rewritten

**What breaks:** `git rev-list --all` misses commits that were force-pushed away. In one measured case it returned 183 commits and missed 10 that were provably still reachable and served by the remote. Force-pushed-away commits stay reachable by SHA and can still be served by a remote's web UI/API even though `--all` no longer sees them.

**Presents as:** A clean all-clear on an exposure/secret-leak sweep. The naive sweep completes, finds nothing, and closes the question in the operator's mind while the actual exposure continues.

**Detect:** Compare `git rev-list --all` against the ancestor closure of every tip found in the reflog (`git reflog show <remote>/<branch>`) and against `git rev-list --all --reflog`. Any divergence between these is the trap firing.

**Do instead:** Use the reflog-derived closure as the "ever public" set, not `--all` alone. Treat any surviving pre-rewrite objects as *evidence*, not garbage: running `git gc --prune=now`, expiring the reflog, or deleting `refs/original/*` destroys your only local proof while changing nothing about what the remote still serves.

### E2. Git hooks are shared by every linked worktree of a repository

**What breaks:** When a repository checkout is one of several linked worktrees sharing one common `.git` directory, hooks resolve to that *common* directory, not to any individual worktree — so a hook installed from any one worktree fires on every commit in *all* of them. Hooks are also normally untracked, so nothing in the repository itself records that a hook exists, and a fresh clone silently has none.

**Presents as:** Two opposite, both-quiet failure modes. Installing a hook reads as a purely local act and isn't — a gate meant to protect one worktree's work can start refusing another, unrelated worktree's commits, invisibly, until a commit fails for a reason that worktree has no context for. And a genuinely-installed, working hook leaves no tracked evidence at all, so its *absence* in a different checkout looks identical to its presence.

**Detect:** `git worktree list` before installing anything hook-related, and compare `git rev-parse --git-common-dir` against `git rev-parse --git-dir` — when they differ, you're in a linked worktree and the hooks directory isn't yours alone. For the untracked half: only a test that actually *drives* the hook end-to-end (not just checks the tracked files exist) can distinguish an armed gate from a disarmed one.

**Do instead / remedy:** Arm hooks per-worktree rather than per-repository — have the shared hook script read a marker file from `git rev-parse --git-dir` (which resolves separately per worktree) and exit cleanly when that marker is absent. Refuse to overwrite a pre-existing hook rather than silently replacing it — another workstream may own it. Remember a hook is bypassable (`--no-verify`, repointing the hooks path, deleting the marker) — it raises the cost of a defect reaching a commit, never a hard guarantee.

### E3. A coordination lease taken inside a linked worktree coordinates with nothing

**What breaks:** If a coordination/locking mechanism resolves "the repository" to the current *checkout path* rather than to the shared git directory, then every linked worktree of the same repository gets its own private, disconnected coordination state. A lease/lock taken in one worktree is invisible to every other worktree, including the primary checkout.

**Presents as: SUCCESS.** The lease is granted, recorded, and the acquiring process proceeds believing it holds exclusivity nothing else can see. Two concurrent processes can each believe they hold "the" lease at the same time.

**Detect:** Compare `git rev-parse --git-dir` with `git rev-parse --git-common-dir`. If they differ, you're in a linked worktree, and any lock/lease taken there may be local to it rather than shared.

**Do instead:** Acquire coordination locks only from the primary checkout (or from a mechanism explicitly rooted at the common git directory), and never treat a gate as satisfied by a lease taken inside a linked worktree.

### E4. Agent-runtime worktree isolation can target the wrong repository

**What breaks:** An agent-orchestration tool's "run this in an isolated worktree" option can isolate the *orchestrating session's own default working directory*, rather than whichever repository the subagent's task actually names as its target — and an explicit "enter this worktree" call can be refused outright once a subagent already has a working-directory override.

**Presents as:** Apparent isolation that isn't. One dispatch can land inside a genuinely isolated worktree — of the *wrong* repository; another dispatch, configured identically, can land directly in the target repository's primary checkout with no isolation at all.

**Detect:** Run `git worktree list` inside the *actual* target repository before trusting any isolation claim made by the tooling.

**Do instead:** Have the subagent create its own worktree with an ordinary shell command inside the real target repository (`git worktree add <path> -b <branch>`), and operate through explicit full paths under that directory, rather than relying on a harness-level isolation flag.

### E5. A regenerating write to a shared file erases another process's in-flight work

**What breaks:** Staging/committing by exact file path protects boundaries *between* files, not *within* one. When multiple processes edit the same shared file concurrently, a write that regenerates the *whole* file (rather than patching it) can silently erase another process's in-flight entries — and a consistency check that only verifies internal agreement (e.g. two derived views of the same file agree with each other) will certify the result as healthy, because both views were regenerated together and agree by construction.

**Presents as: CONSISTENT.** A reconciliation or consistency check can report a shared file as fully consistent throughout, while it is actually missing entries another process had just added — the check compares the file against itself, not against independent evidence of what should be there.

**Detect:** Compare the file you're about to commit against its prior committed state on *both* sides of your intended change, and prove everything outside your own edit is byte-for-byte identical. Verifying your own content is present is not the same as verifying nothing else is absent. Where possible, check against something that exists independently of the file itself (a separate log, a second data source), not just an internal round-trip.

**Do instead:** Stage exact paths only; never stage everything indiscriminately, never commit blindly. Don't remove a git index lock file to "unblock" yourself — wait, re-verify state, then retry; forcing past it can destroy a concurrent process's in-flight commit. Avoid stashing when other processes are concurrently live. For work that must touch a genuinely shared, non-isolated file concurrently, prefer building and applying a targeted single-hunk patch over regenerating the whole file, and verify the diff before committing.

### E6. A shared git index holds a derived value another process has already moved past

**What breaks:** When multiple processes share *one* git index (staging area) on the same checkout, a blob you stage is a snapshot. If another process then edits the *same* file and recomputes some derived value inside it (a hash, a restamped digest, a generated count), your already-staged blob keeps the old derived value while the working tree has moved on — and nothing about your staged content is individually wrong, it's just stale.

**Presents as: CONSISTENT,** from two directions at once. Your staged blob is internally coherent and contains everyone's content, so nothing looks missing; and a consistency check run against the current working tree also passes, because it's comparing the working tree to itself. The staleness lives only in the gap between the index and the working tree, which neither check inspects. (Distinct from E5: nothing is erased here — the derived value is just behind.)

**Detect:** `git diff --name-only -- <path>` (working tree against the index) must be empty for any shared file you have staged. Compare any derived/generated value between the staged blob (`git show :<path>`) and the same value in the working tree.

**Do instead / remedy:** Don't leave shared files staged if you're not about to hand them over — staged state sitting in a shared index across live concurrent processes *is* the exposure. Re-run whatever regenerates the derived value, re-stage the exact path, and confirm the working-tree/index diff is empty before committing. Commit using a form that takes content from the *working tree* for the paths you name (see E7 for why this is a double-edged tool, not a full remedy on its own). Never re-stage someone else's in-flight content to "fix" this for them — that silently claims their work as yours.

### E7. Committing "by exact path" still takes whatever is in the working tree, not what you staged

**What breaks:** A commit form that restricts itself to named paths (e.g. `git commit --only <paths>`) commits the *current working-tree content* of those paths — not your own edits, and not what you staged. On a checkout shared by multiple concurrent processes, if another process has uncommitted work-in-progress on a path you name, that work gets swept into your commit, under your message, without you doing anything obviously wrong.

**Presents as: SUCCESS.** The commit summary reports a plausible-looking line count, and if you don't check it carefully it just reads as "my diff was bigger than I thought." One real case: a commit intended to touch a handful of lines for one purpose ended up carrying hundreds of added lines and a dozen new top-level names belonging to an entirely different, in-flight module migration by another process — discovered only because that other process happened to notice afterward.

**Detect:** `git diff HEAD -- <path>` immediately before committing, and confirm every hunk shown is actually yours. After committing, compare each file's changed-line count in the commit against what you actually believe you edited — a file you touched in one small place showing hundreds of changed lines is the signal.

**Do instead:** Name only paths you're confident no other live process holds, and re-check the working status for that exact path immediately before committing. Where a file is genuinely being co-edited by multiple processes, coordinate the handover explicitly rather than relying on path-scoped staging/committing to protect you — restricting the paths you name only stops interference *across* files; it cannot stop interference *within* one, because this commit form deliberately reads from the working tree by design.

### E8. Pushing an explicit ref doesn't fence out another process's commits

**What breaks:** `git push origin <sha>:<branch>` publishes that commit *and every ancestor of it*. Naming an explicit SHA only protects against publishing commits *ahead* of yours (ones you haven't pulled in); it does nothing about commits another process landed *behind* yours on the same branch in the meantime — those are ancestors of your SHA by the time you push, whether you intended to publish them or not.

**Presents as: SUCCESS,** and specifically success at a precaution you thought you'd taken. The push reports exactly the ref you named, so the output confirms your *intent* rather than the actual *effect* — leaving you more confident, not less, than someone who never tried to fence anything out at all.

**Detect:** Before pushing, enumerate `git log --oneline <remote>/<branch>..<your-sha>` — this lists everything the push will actually publish. If it names a commit you didn't author, naming the specific ref did not fence it out.

**Do instead / remedy:** Enumerate the ancestry first and read it. If foreign commits appear that must not be published yet, don't push that ref — resequence your own commits instead. Treat every push to a shared branch as publishing the whole ancestry, and verify the ancestry, not the ref you typed.

---

### E9. A test identity left in local git config silently authors every later commit, in every worktree

**What breaks:** `git config --local user.name/user.email`, set to prove some identity-sensitive behaviour, persists after the proof is over — and `--local` is per-*repository*, so it authors every subsequent commit in every linked worktree sharing that repository, including commits by other processes that never touched the setting. A correct global identity does not help; local wins.

**Presents as: SUCCESS.** Every commit succeeds and nothing warns. The author field is read by no gate, no test, and no human in the ordinary course, so the only place it shows is `git log`'s author column. Measured: **twelve consecutive commits across two independent streams** authored under a synthetic `@example.invalid` proof identity, one of them already pushed to the remote before anyone noticed. It surfaced as an incidental aside in an unrelated report, not from any check.

**Detect:** `git config --local --get user.email` before your first commit of a session, and `git log --format='%an <%ae>' -5` when resuming into a repository you did not just configure. A synthetic address — `example.invalid`, `example.com`, `localhost` — in recent history is this, not a contributor.

**Do instead:** Never write an identity into config to run one proof. Scope it to the single invocation: `git -c user.name=... -c user.email=... commit ...` overrides for that command only and writes nothing to `.git/config`, so there is nothing to forget to undo. If a durable override is genuinely needed, unset it in the same change that finishes the work needing it, and re-read the effective identity afterwards in a separate command.

## F. Content-hash pinning and line endings

### F1. A fresh clone is not a byte-faithful copy when hash pins, CRLF, and path limits are involved

**What breaks:** Two independent ways a freshly cloned repository can differ from the one you meant to copy. (1) If your machine's global git config doesn't set `core.autocrlf`, a new clone defaults to auto-converting line endings, while the original checkout may carry an explicit `core.autocrlf=false`. Every text file governed by a byte-level hash or manifest binding then silently gains different line endings, and that binding no longer matches. (2) A repository that vendors a complete nested copy of another project (a plugin bundle, a vendored dependency tree) can produce a path deep enough that, combined with a sufficiently deep clone target directory, it exceeds the operating system's path-length limit, and the checkout dies partway through.

**Presents as:** Neither symptom names its actual cause. (1) presents as a hash/manifest-binding validation error pointing you to inspect the manifest and the binding — both of which are actually correct in the source repository and wrong only in your freshly cloned working copy; nothing in the error mentions line endings. (2) presents as a clone command reporting *partial success* ("clone succeeded, but checkout failed") — a few specific files are named as missing, drawing your eye to those files rather than to path length as the actual cause.

**Detect:** Before trusting a clone of a hash/manifest-governed repository, hash the relevant file yourself and compare against the recorded value; a mismatch combined with the file containing CRLF bytes is the line-ending half. For the path-length half, read the clone command's own stderr rather than trusting its exit status, and check the file count under the deepest vendored directory against the source.

**Do instead / remedy:** Clone with long-path support enabled and explicitly force the correct line-ending config, then re-materialize the working tree (force line endings, drop the index, and hard-reset) — simply flipping the config *after* an initial checkout is not enough, because git considers already-converted files unchanged. Don't diagnose a hash-binding failure by re-reading the binding itself; and don't treat a zero exit code from `git clone` as proof the checkout is complete.

### F2. A byte-pinned file without a matching line-ending pin checks out differently in different working copies

**What breaks:** The moment anything binds a file's raw byte length or SHA-256 hash (a signed seal, a frozen-content check, a containment proof), that binding is only as reliable as the file's line endings being forced consistent (a `.gitattributes` `eol=lf` rule or equivalent). Without that pin, the *same* commit can check out with CRLF in one working copy and LF in another — even with the relevant git config nominally identical in both — because nothing connects the hash binding to the line-ending rule. One measured case: a sealed file was 9,725 bytes in one checkout and 9,843 bytes in a second worktree of the exact same commit.

**Presents as:** A governance/integrity check refusing on a raw byte count, which reads as corruption or tampering rather than as a line-ending difference. The tempting "fix" — re-pin the check to whatever length you're currently observing — actually writes a *false claim* into the integrity mechanism and makes the underlying defect permanent. Worse, the exposure is largest exactly where discipline is strongest: a clean, detached, freshly-checked-out copy (the thing you are told to use for reliable validation) is precisely what surfaces the wrong bytes.

**Detect:** Compare the file's byte length between two independent working copies of the *same* commit (e.g. your normal checkout and a fresh detached worktree of the same SHA). A difference proves a line-ending divergence and rules out actual corruption. Check whether the file has an explicit end-of-line attribute set at all.

**Do instead / remedy:** Add an explicit `eol=lf` (or equivalent forced-line-ending) rule for any byte-pinned or hash-pinned path, then re-check it out. The missing pin is the fix; the integrity check itself is correct and should not be adjusted to match the wrong bytes. When you first bind a file's length or hash anywhere, pin its line-ending behavior in the *same* action — treat them as one decision, not two.

### F3. A byte-pinned or sealed file is not safe to touch with an automated cleanup pass

**What breaks:** Any file whose exact bytes are pinned somewhere else (a recorded hash, a signed seal, a fixture asserting exact content) is not safe to touch with a bulk automated pass — a formatter, a line-ending normalizer, a "fix this class of issue everywhere" codemod — even when the specific fix being applied is obviously correct in isolation. Editing it in place breaks whatever binding made it trustworthy as evidence.

**Presents as:** A clean, obviously-correct automated repair, followed later — sometimes much later — by an unrelated-looking integrity/seal verification failure that has nothing to do with the content itself.

**Detect:** Before running any bulk edit, formatter, or codemod across a tree, check whether any file it will touch has its exact bytes or hash referenced elsewhere (a manifest, a lockfile, a test asserting exact content, a signature).

**Do instead:** Treat a genuine problem inside a byte-pinned file as something requiring a deliberate, explicit re-issuing of the pin (a conscious "yes, re-seal this") — never an in-place repair folded into an unrelated bulk pass, and never while anything is actively relying on the current hash.

### F4. A signing or stamping step rewrites the artifact, invalidating every hash taken before it

**What breaks:** Code signing (Authenticode, and equally any step that embeds a version resource, a build id, or a notarization ticket) rewrites the file in place. A checksum manifest, catalog, or digest computed *before* that step describes bytes that no longer exist. No single step is wrong — only the order is — and each one passes its own local check.

**Presents as:** The artifact fails its *own* integrity check at its destination, which reads as corruption in transit, a broken verifier, or a bad download — anywhere but the build. It is worse than a plain mismatch, because the manifest and the files are both authentic; they are simply from two different moments. And where the receiving end treats "signature present but does not verify" as more serious than "unsigned", a correctly signed artifact can be refused harder than one that was never signed at all.

**Detect:** Hash one signed file at the end of the build and compare it against that file's entry in the shipped manifest. A build that never makes this comparison cannot tell the two cases apart. Verifying the signature alone does not catch it — the signature is valid; it is the manifest that is stale.

**Do instead:** Fix the order once, and write it down where the build cannot drift from it: **sign, then checksum, then catalog, then sign the catalog.** Each step covers everything produced before it, and nothing after it rewrites anything it covered. Where some artifact types cannot carry a signature at all (plain `.sql` or `.csv` files shipped alongside signed scripts), a signed catalog over the whole tree — including the checksum file — is what covers them; without it, "signed package" means "signed runner, plus whatever files happened to be sitting next to it".

---

### F5. `git add` can normalise line endings into the committed blob, so an archived copy is not the source

**What breaks:** Under git's default `text=auto`, `git add` may rewrite CRLF to LF *into the committed blob*. For content being archived or preserved -- where the bytes **are** the artifact, and often the only copy -- the committed copy is then not the source. A copy-time verification cannot see this, because comparing the two on-disk files happens before git touches anything.

**Presents as: SUCCESS at the copy step, from a check that genuinely compared the files.** Measured while preserving 211 files that existed in exactly one place on disk: the copy's own self-check reported **zero** mismatches, and a fresh checkout from the pushed branch found **9 of 211 not byte-identical**. The entire gap lies inside `git add`, after every check the copying process was able to run.

**Detect:** Verify **from the branch** -- a fresh worktree checked out from the pushed tip, hashed against the original source. Not from the copy operation's own comparison, not from its exit code, and not from the working tree it just wrote. A check that runs before git has not checked git.

**Do instead:** Put a directory-scoped `.gitattributes` carrying `* -text` over the preserved tree, which forces every path under it to be treated as binary -- no normalisation on add or on checkout, on any machine. Add it **before** the first `git add` of that content; afterwards the blob is already normalised and must be re-staged from a source copy that still holds the original bytes. Then verify from the branch anyway.

## G. Automated pattern detectors

### G1. A document describing a bad pattern trips the detector built to find that pattern

**What breaks:** A document that *quotes* a pattern in order to discuss, forbid, or record the retirement of it will itself match an automated detector built to find that same pattern — a rule about not hardcoding secrets gets flagged for its own example of what a hardcoded secret looks like; a linter for a deprecated API name matches the changelog entry announcing its deprecation.

**Presents as:** A detector's single largest source of hits being its own documentation about itself, which reads like a wave of real violations and buries the genuine ones among the noise.

**Detect:** Read every finding at its cited location before acting on the count, not just the count itself. It's realistic for the same false-positive class to trip more than once across different checks in a short period.

**Do instead:** Fix the detector, never the prose describing what it's for: anchor the match more precisely, structurally exempt the genre of document that legitimately quotes the pattern, and never reword a real, live instruction just to dodge a check. If you're writing about redacting real values, avoid using an actual real value as your example in the first place.

---

### G2. A validator that re-runs a generator uses the generator in your working tree, not the one in the commit

**What breaks:** A check that proves a generated artifact matches its source by *re-generating and comparing* imports the generator from the working tree, not from the commit under judgement. So a commit whose generated member was produced by an **uncommitted** generator passes, and the committed pair is self-inconsistent: the checked-in tooling cannot reproduce the checked-in output. In a shared working copy this needs no mistake by the committing process -- another process's dirty generator is simply on disk, and any write imports it.

**Presents as: SUCCESS**, and specifically as a clean `valid: true` from a check whose entire purpose is to prove the two agree. Measured: at one commit the committed generated file carried **18** instances of a marker while the generator committed *at that same commit* contained **none** of the logic that emits it. The validation returned valid then and returns valid now, correctly by its own terms.

**Detect:** Compare the two blobs **at the commit** rather than in the working tree -- search the committed output for a token only the new generator emits, then search the committed generator for the code that emits it. Disagreement is the defect. A green validation is not evidence either way, because it is the thing that cannot see this.

**Do instead:** Commit a generated artifact and any generator change **together**, so the pair stays reproducible from its own commit. Before trusting such a validation in a shared working copy, check whether the generator is dirty; a dirty generator belonging to another process is imported silently by your write. Where the two must land separately, regenerate the output from the committed generator after the generator lands.

## H. Browser and UI testing

### H1. jsdom-green does not mean browser-green

**What breaks:** A full test suite passing under a DOM-emulation library like jsdom — even backed by mutation testing — does not establish that UI behavior actually works in a real browser. jsdom models the DOM API surface, not a real engine's selection, layout, or clipboard behavior.

**Presents as: SUCCESS at full apparent strength** — a green suite plus a mutation score, which looks like *more* evidence than a typical passing run, not less.

**Detect:** Ask whether any assertion in the suite actually ran inside a real browser engine. If every one ran under a DOM emulator, real browser behavior is completely unmeasured regardless of how large or green the suite is.

**Do instead:** Treat a real-browser pass as non-optional for UI-affecting work. Launch an actual browser engine explicitly and verify the behavior there, using a dedicated profile/user-data directory rather than the machine's default browser session.

### H2. `Selection.toString()` output differs by rendering engine

**What breaks:** `Selection.toString()` returns different text depending on the rendering engine. As one concrete example, Chrome excludes elements styled `user-select: none` from the result, while a Chromium-based Edge/WebView2 engine includes them in the Selection object even though the OS clipboard strips them back out on copy. A desktop app packaged with an embedded web view often runs a different engine than the browser used for day-to-day development.

**Presents as:** A defect that reproduces *only* in the shipped/packaged artifact, never in the development browser — or, from the other direction, a defensive workaround that looks completely redundant in every test (because the dev browser never needed it) and gets deleted as dead code.

**Detect:** Verify selection-dependent behavior specifically in the engine the product actually ships on, not the one used during development.

**Do instead:** Keep any defensive code that compensates for this kind of engine difference; don't remove it as apparently-redundant based on observations made only in the development browser.

### H3. Real-browser verification needs a dedicated profile and careful process cleanup

**What breaks:** Real-browser verification of UI behavior (related to H1 and H2 above) requires care about which browser and profile you use, and how you clean up afterward. A machine's default browser (unconfigured) or an incognito/private window can attach to a real person's actual running session rather than giving you an isolated one, contaminating results or interfering with someone's real usage.

**Presents as:** A full green test suite (jsdom plus mutation testing) coexisting with a real, shipped UI bug — because the suite never actually exercised a real engine at all.

**Detect:** Confirm explicitly which browser, profile, and process launched the verification — never assume "the browser that opened" was the isolated one you intended.

**Do instead:** Launch a real browser engine explicitly with a dedicated, disposable profile/user-data directory — never the machine's default browser, and never incognito/private mode, which can attach to a real ongoing session. When cleaning up browser processes afterward, identify and terminate them by process tree/lineage, *never* by matching on the executable's image name alone — a shared machine can easily have dozens of unrelated browser processes carrying someone else's real, unrelated work, and killing by name risks taking those down too.

---

## I. PowerShell language and cmdlet semantics

These are language- and cmdlet-level rather than environment-level, and they are here for the same reason as the rest of this file: an agent writing PowerShell meets them repeatedly, and nearly every one of them **presents as success**. All were met in real work on a production PowerShell codebase; several cost a debugging session apiece, and I1 alone accounted for four separate defects.

This section is to PowerShell what C is to Python: the language and its runtime, not the boundary around them. The neighbouring problem — content mangled on its way *through* a shell, including two PowerShell entries — is section A.

### I1. A single-element collection decays to a scalar on return

**What breaks:** A function returning `@(...)` returns a *scalar* whenever the collection happens to hold exactly one element — the pipeline unwraps the array on the way out. The documented fix, the leading-comma idiom `return , @(...)`, then introduces two follow-on traps of its own: a caller that wraps the call in `@()` now nests one array inside another, and the comma idiom throws `ArgumentException: Argument types do not match` when applied to a generic `List` rather than a real array.

**Presents as: SUCCESS, with wrong content, and only at a specific input size.** Zero elements and two elements both behave; one element does not. A parser returning a list of SQL batches returned a bare string for single-batch input, and the consumer that iterated it either iterated its *characters* or concatenated every batch into one command — visible only once a fixture had more than one batch in it. Small test fixtures are overwhelmingly single-element, so the suite is the least likely place this surfaces.

**Detect:** Exercise every collection-returning function at zero, one, *and* two elements, and assert the returned **type** (`-is [array]`) as well as the count. A test that only asserts on content passes in the broken case.

**Do instead:** `return , @($items)` for arrays, `return , $list.ToArray()` for generic lists — and then delete any `@()` the callers wrap around the call, because the two fixes do not compose. Whichever convention you pick, apply it at the boundary and state it once, rather than per call site.

### I2. `@($null)` has a Count of 1

**What breaks:** Wrapping a possibly-absent value in `@(...)` — the standard "make sure this is a collection" idiom — yields a one-element array *containing null* when the value is absent, not an empty array. `@($null).Count` is 1.

**Presents as: SUCCESS.** A guard of the form `if (@($x).Count -gt 0)` passes when there is nothing there. The code behind it then renders a section header with a count of one and no rows beneath it, or loops once over null. The rendered count is the tell, and it looks like an off-by-one in the renderer rather than a truth about the wrapper.

**Detect:** Any `@(...)` around a value that can legitimately be absent — a hashtable key that may not exist, an optional parameter, a property off a partially-populated object.

**Do instead:** Filter as well as wrap: `@($x | Where-Object { $_ })`. Never treat `@(...)` alone as a null-to-empty conversion; it is a scalar-to-collection conversion, and null is a scalar.

### I3. A local variable differing from a parameter only in case *is* that parameter

**What breaks:** PowerShell variable names are case-insensitive, so a local initialized as `$expectedDigest = ''` at the top of a function whose parameter is `-ExpectedDigest` does not shadow the parameter — it overwrites it. The argument the caller passed is gone before the first line of real work.

**Presents as: SUCCESS, and specifically as a caller error.** Every later `if ($ExpectedDigest)` is quietly false, so the function behaves exactly as it should when the optional argument was genuinely not supplied. Nothing warns. In the measured case the only failing test was the *happy path*; every negative test passed while the feature was completely inert, because "does nothing" is the correct answer when the argument really is absent. A suite weighted toward negative cases actively conceals this.

**Detect:** Compare each function's local declarations against its parameter names case-insensitively — a linter rule, not an eyeball pass, because the two names look different on the page. Where a feature "behaves as if the argument was never passed", suspect this before suspecting the caller.

**Do instead:** Give locals names that differ from parameters by more than capitalisation.

### I4. `?` and `*` are wildcards in `-like`, so a containment test matches everything

**What breaks:** `-like` is a wildcard match, not a substring test. `$s -like '*?*'` asks for "any single character with anything either side" and is therefore true for every non-empty string. The same applies to a literal `*`, and to `[` ranges appearing inside what was meant as plain text.

**Presents as: SUCCESS, on the wrong branch every single time** rather than intermittently — which perversely makes it harder to spot, because there is no working case to compare against. A URL builder testing `-like '*?*'` to choose between appending `?` and `&` appended `&` to every URL it ever built, and every request came back HTTP 400.

**Detect:** Grep for `-like` whose pattern contains `?`, `*`, or `[` in a position meant literally. A `-like` test that is never false against real data is this trap.

**Do instead:** Use `.Contains('?')` for containment. Where a wildcard match is genuinely wanted but part of the pattern is literal, escape that part rather than hoping it contains no metacharacters.

### I5. A variable followed by a colon inside a double-quoted string is parsed as a scope qualifier

**What breaks:** A colon immediately following a variable name inside a double-quoted string makes PowerShell read the name as a *scope or drive qualifier* — the same syntax as `$env:PATH` or `$script:x`. Interpolating a host and port that way is not the interpolation you wrote.

**Presents as:** Loudly, at parse time — which is the good case, and the reason this one is cheap. The whole file fails to load, and the error names the variable rather than the colon, so it reads as an undefined-variable problem.

**Do instead:** Brace the name, in the `"${server}:$port"` form.

### I6. `[AllowEmptyCollection()]` on an untyped parameter refuses to bind at all

**What breaks:** The attribute needs a collection-typed parameter to attach to. Put it on an untyped parameter and binding fails with `Argument types do not match` on *every* call, not only the empty ones the attribute was added for.

**Presents as:** A loud binding error complaining about a type mismatch on a parameter that has no declared type — which reads as nonsense, and sends you looking at the argument rather than at the attribute.

**Do instead:** Type the parameter `[object[]]`.

### I7. An unbound typed parameter is null, and forwarding it throws

**What breaks:** A declared-but-not-supplied `[datetime]` (or any non-nullable value type) parameter holds `$null`, not the type's default. Forwarding it unconditionally to another typed parameter throws at the *second* call, not the first.

**Presents as:** A cast error naming a type neither the caller nor the immediate function mentions, one frame further down than where the omission actually happened.

**Do instead:** Splat only what `$PSBoundParameters` actually contains, rather than forwarding every parameter unconditionally.

### I8. `continue` is dynamically scoped and escapes into the caller's loop

**What breaks:** `continue` binds to the nearest enclosing loop **at run time**, not lexically. Module code re-entered through an injected scriptblock, callback, or event handler can therefore `continue` a loop belonging to a completely different caller, several frames up.

**Presents as: SUCCESS, with silently skipped iterations** in a loop whose author has never seen the `continue` that is skipping them. No error, no warning, and a result set quietly short by however many iterations were jumped.

**Detect:** Any `continue` or `break` on a code path reachable from a caller-supplied scriptblock.

**Do instead:** Restructure as nested `if` on those paths. Reserve loop-control keywords for code that certainly owns the loop it is controlling.

### I9. A verification cmdlet's skip list matches file *names*, not paths

**What breaks:** `Test-FileCatalog -FilesToSkip`, and exclusion parameters on several other cmdlets, match against the leaf file name rather than the relative path. A folder-shaped pattern therefore excludes nothing, while a `*.log`-shaped one works. There is no path-shaped exclusion at all, so an entire folder cannot be excluded by any pattern.

**Presents as:** A verification failure listing files you believed you had excluded, which reads as the exclusion parameter being ignored — and the natural next move is to fight the pattern syntax rather than to conclude the parameter cannot express what you need.

**Detect:** Test the exclusion against a file whose *name* is distinctive and whose *folder* is the thing you meant to exclude. If a same-named file in a different folder is also skipped, it is matching on the name.

**Do instead:** Where the exclusion has to be path-shaped, read the catalog's contents directly and do the comparison yourself, rather than expressing the intent in a parameter that cannot carry it.

### I10. A verification cmdlet hashes the whole target tree, including files the current process holds open

**What breaks:** `Test-FileCatalog -Path` does not merely read the catalog; it walks the path given, hashes every file it finds there, and throws outright if any one of them is locked. Pointed at a directory containing a log or transcript that the *running process* has open, it fails on a file with nothing to do with what was being verified.

**Presents as:** A hard failure naming a locked file rather than a verification result — and one that is easy to mistranslate one level up. In the measured case it turned every real deployment into "this package has no catalog", because the deployment run's own open transcript lived inside the tree being verified. Verification never got far enough to report on the artifacts at all.

**Detect:** Any verification call whose target is a live working directory rather than a quiesced artifact. Note specifically whether the process performing the verification writes anything into that tree.

**Do instead:** When you only want to *read* a catalog's contents, point the cmdlet at an empty temporary directory and do the real comparison yourself. More generally: never verify a tree that the verifying process is still writing into.

### I11. Pester: a `BeforeAll` mock is not yet in force, and its loop-control error is usually a misreport

**What breaks:** Two separate things, both about trusting the framework's own account of events. (1) A mock declared in `BeforeAll` is not in effect while that same `BeforeAll` block is still executing, so setup code in the block calls the real implementation. (2) Pester's *"break or continue statement escaped"* error is frequently a **misreported parameter-binding failure** — the real exception is wrapped, and the message you are shown describes a control-flow problem that does not exist.

**Presents as:** (1) as real side effects during setup — a genuine network call, a real file written — while every test in the block passes, because by the time they run the mock is live. (2) as a hunt through the test body for a stray loop-control keyword that is not there.

**Detect:** For (2), read `$_.ErrorRecord.Exception.Message` before believing the surface message. For (1), assert inside `BeforeAll` that the mock is active, or move the setup that depends on it into the block that follows.

**Do instead:** Put anything that must observe a mock into `BeforeEach`, or after the mock declaration in a separate block — never in the same `BeforeAll` that declares it.
