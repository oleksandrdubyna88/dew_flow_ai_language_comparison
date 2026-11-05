# Task spec and sample format

Format version: 2
Gate verdict: `proceed` · 2026-11-05
Freeze tag: none yet

This document fixes the shape of every task spec in `todo/tasks/` and of every visible sample beside it: the file and its
header, the requirement ids, the sentences every spec carries word for word, the sample files of each kind, and the
scenario grammar of the daemon and web samples with its comparison rules. It is stack-neutral: nothing in it favours or
names an implementation language.

Sections 1 to 6 are the format; section 7 is its changelog. **Sections 3 to 5 are copied verbatim into every workspace as
`runner/GRAMMAR.md`**, beside the runner that implements them, so they are written to be read on their own. The rules for
checking a spec against this format and for amending it are kept in a separate maintainers' document that no workspace
and no run base carries.

**Section classes.** Every section below has one class, or, where a section mixes kinds of task, every subsection has
one: `common` (it serves olympiad tasks and another kind, or every kind), `olympiad`, `daemon`, `web`, or `daemon-web` (it
serves daemon and web tasks and no olympiad task). The lead text of a section, before its first subsection, is `common`. In
2.1's table, a row whose sentence stands in a `daemon-web` subsection has that subsection's class. After the freeze of the
first group of tasks, the `common` and `olympiad` parts do not change and keep their numbers; a `daemon`, `web` or
`daemon-web` part may change before the freeze of a later group, as a new dated version that names the tasks it touches, and
such a change touches no olympiad task. The changelog only grows.

| Section | Class |
|---|---|
| 1. The spec file | common |
| 2.1 How they are used | common |
| 2.2 Every kind | common |
| 2.3 Olympiad | olympiad |
| 2.4 Daemon and web | daemon-web |
| 2.5 The timing contract of daemon and web tasks | daemon-web |
| 2.6 Olympiad output: the closed list of normalisations | olympiad |
| 3.1 Files and ids | common |
| 3.2 Olympiad samples | olympiad |
| 3.3 Daemon and web samples: the scenario pair | daemon-web |
| 3.4 Showing samples in the spec | common |
| 4. The scenario script (`.in`) | daemon-web |
| 5. The expected transcript (`.out`) | daemon-web |
| 6.1 Olympiad — one sample of the pilot spec | olympiad |
| 6.2 Daemon — a toy store over an invented framed protocol | daemon-web |
| 6.3 Web — two clients racing on one idempotency key | daemon-web |
| 7. Changelog | common |

## 1. The spec file

### 1.1 Name and title

A spec is `todo/tasks/NN-slug.md`: `NN` is the task number with two digits (`01` … `10`), `slug` is lower-case ASCII
letters and digits in words joined by single hyphens (pattern `^[0-9]{2}-[a-z0-9]+(-[a-z0-9]+)*\.md$`). This file,
`FORMAT.md`, is not a spec. The first line of a spec is its title, `# Task NN — <Title>`, with the same `NN` and an em
dash (U+2014). The workspace's `STATEMENT.md` is generated from the spec; the spec is written so that it reads correctly
there.

### 1.2 The header table

Directly under the title, a two-column table with an empty header row and exactly these five rows, in this order:

| Row | Value before the gate and the freeze | Value afterwards |
|---|---|---|
| `Kind` | the kind phrase of the task (below) | unchanged |
| `Gate verdict` | `not yet gated` | `` `<verdict>` · <YYYY-MM-DD> ``, the verdict being `proceed`, `good_enough` or `continue_anyway` |
| `Freeze tag` | `none yet` | `` `spec-frozen-<group>-<yyyymmdd>` ``, `<group>` being `A` or `B`, the group of tasks frozen together |
| `Freeze rule` | `this text changes only by a dated amendment from its authors` | unchanged |
| `Amendments` | `none yet` | `A1 (section 8)`, or `A1–A<k> (section 8)` once there are k |

The kind phrases form a closed list; the phrase is the whole value of the `Kind` row:

| Kind | Phrase |
|---|---|
| olympiad | `a program that reads standard input once and writes standard output once` |
| daemon | `a long-running program that serves until it is stopped by SIGTERM` |
| web | `a long-running HTTP server that serves until it is stopped by SIGTERM` |

The kind decides the required sections (1.4), the fixed sentences (section 2) and how the samples are read (section 3).

### 1.3 The requirement sentences (R-ids)

Every requirement opens with its id in bold, `**R<n>.**`, followed by one or more sentences. The ids are `R1` … `Rn`:
unique, contiguous (no id missing between 1 and n) and, until the freeze, in ascending order through the document.
Held-out cases cite these ids; nothing outside them is tested.

**A requirement's span.** A requirement runs from its `**R<n>.**` to the next requirement or the next heading of any
level. Every paragraph, list, table and fenced code block in that span belongs to it: a table or a code block (a
grammar, say) is part of the requirement it follows, and so is a lead-in line before it (`Type gauge:`) or a sentence
after it. Outside every span stand only F-REQ (before section 1), the lead of section 6 with the samples and the prose
that explains how a sample is built, and section 8. A table, a code block or a normative sentence anywhere else is a
defect of the spec; prose outside a span explains and never adds a requirement.

**Stable after the freeze: append only.** An id is never renumbered and never reused. An amendment that adds a
requirement gives it the next free id, `R(n+1)`, placed in the section it belongs to, so after an amendment the ids may
stand out of order in the document. An amendment that withdraws a requirement keeps its paragraph as
`**R<n>.** Withdrawn by amendment A<k>.`

### 1.4 The sections, in order

Each spec has exactly these eight `##` sections, numbered and titled as shown, in this order, and no other `##` section.
Sub-sections (`###`) are free inside sections 1 to 3; section 6 of a daemon or web spec has one per sample (3.4).

| # | olympiad | daemon and web |
|---|---|---|
| 1 | `## 1. What the program does` | `## 1. What the program does` |
| 2 | `## 2. Input` | `## 2. Start, readiness and stop` |
| 3 | `## 3. Output` | `## 3. Interface` |
| 4 | `## 4. Limits` | `## 4. Limits` |
| 5 | `## 5. Build and run — the workspace contract` | `## 5. Build and run — the workspace contract` |
| 6 | `## 6. Visible samples` | `## 6. Visible samples` |
| 7 | `## 7. Hidden tests` | `## 7. Hidden tests` |
| 8 | `## 8. Amendments` | `## 8. Amendments` |

Section 8 holds `None yet.` until the first amendment, then one list item per amendment, oldest first:
`- A<k> · <YYYY-MM-DD> · <R-ids added, changed or withdrawn> — <what changed and why>`.

### 1.5 What a spec never contains

- An identifier of the bench's plan: no story, epic, decision, gate or test id.
- For a task that paraphrases a known problem: its id, its title, its site or source, a link, or any mention that the task
  is known. Its credit lives only in the attribution section of the repository README, which no workspace and no run base
  carries; there is no credit file beside the task.
- A link of any kind.
- A trap, a hint at one, or a language, runtime, framework or library specific. The spec names no implementation
  language; the stack sentence (and, for a web task, the framework sentence) is added only when `STATEMENT.md` is
  generated.
- Held-out content, and the exact registered limit values (they go to the test runner, never into a spec).
- The `⏎` glyph anywhere but section 6.

## 2. The fixed sentences

### 2.1 How they are used

The sentences below are said identically by every spec of the kinds named. Each is written once here; a spec copies it
word for word into the section given, inside a requirement's span wherever the table says "in an R". A conformance check
compares them after collapsing every run of spaces and line breaks to one space, so a spec may wrap them at any width. A
slot in braces is the only part that varies, and only with the values the slot allows.

**`{ref}` slots** take one R-id (`R5`), or `any of` followed by a range or a list: a range is `R5–R7` (with an en
dash), and a list holds ids and ranges of any length, separated by commas with `and` before the last. So `R5`,
`any of R5–R7`, `any of R5 and R7`, `any of R8, R11, R14, R17 and R32`, `any of R5–R7, R9 and R12`. `{eref}` slots take
the same forms without `any of`.

| Id | Kinds | Where | Slots |
|---|---|---|---|
| F-REQ | all | before section 1, its own paragraph | `{n}`: the highest R-id number |
| F-INVALID | all, unless the task defines errors for every invalid input | section 2 (olympiad) or 3 (daemon, web), in an R | `{ref}` |
| F-ERRORS | all, instead of F-INVALID when the task defines errors for every invalid input | as F-INVALID | `{ref}`, `{eref}` |
| F-BYTES-O | olympiad, the default | section 3, in an R | none |
| F-BYTES-O-N | olympiad, instead of F-BYTES-O when the output is normalised | section 3, in an R | `{normalisations}`: from the closed list of 2.6 |
| F-STDERR | all | section 3, in an R | none |
| F-EXIT-O | olympiad | section 3, in an R | none |
| F-PORT | daemon and web that listen on a port | section 2, in an R | none |
| F-READY | daemon, web | section 2, in an R | `{readiness}`: one of two phrases (2.4); `{T}`: whole seconds (2.5) |
| F-STOP | daemon, web | section 2, in an R | `{which}` (2.4); `{G}`: whole seconds (2.5) |
| F-EOF | daemon, web whose spec defines an end of the test at the end of its input | section 2, in an R | `{eref}`: what the program does at the end of its input; `{G}` as in F-STOP |
| F-TIMING | daemon, web whose T or G is not the series default | section 2, in the same R as F-READY or F-STOP | `{what}`, `{D}`, `{V}`, `{reason}` (2.5) |
| F-STDOUT-D | daemon, web | section 3, in an R | none |
| F-BYTES-D | daemon | section 3, in an R | none |
| F-BYTES-W | web | section 3, in an R | none |
| F-JSON | daemon, web whose replies, or some of them, are compared as JSON values | section 3, in an R | `{subject}` (2.4) |
| F-LIMITS-O | olympiad | section 4, in an R | none |
| F-LIMITS-D | daemon, web | section 4, in an R | none |
| F-LOAD | daemon, web | section 4, in the same R as F-LIMITS-D | `{size}`, `{duration}` (2.4) |
| F-MEMORY | all | section 4, in the same R as F-LIMITS-O or F-LIMITS-D | none |
| F-PROCS | all | section 4, in the same R as F-MEMORY, after it | none |
| F-DEADLINES | daemon, web | section 4, in an R | none |
| F-FROZEN | all | section 5, in an R | none |
| F-BUILD | all | section 5, in an R | none |
| F-RUN-O | olympiad | section 5, in an R | none |
| F-RUN-D | daemon, web | section 5, in an R | none |
| F-ENV | all | section 5, in the same R as F-RUN-O or F-RUN-D | none |
| F-OFFLINE | all | section 5, in an R | none |
| F-SAMPLES-O | olympiad | section 6, first paragraph | `{k}`: the number of samples, in words or digits |
| F-SAMPLES-D | daemon, web | section 6, first paragraph | `{k}` as above |
| F-HIDDEN | all | section 7, in an R: the first sentence opens the R, the second closes it | none |

### 2.2 Every kind

F-REQ:

```text
Every numbered sentence (**R1** … **R{n}**) is a requirement. Hidden tests refer to these numbers; nothing outside them is tested.
```

F-INVALID (olympiad):

```text
Input that violates {ref} does not occur in any visible or hidden test. What the program does with such input is unspecified and is never tested.
```

F-INVALID (daemon, web):

```text
A request or input that violates {ref} does not occur in any visible or hidden test. What the program does with it is unspecified and is never tested.
```

A task that defines error behaviour for some invalid input states that behaviour in its own requirements and uses
F-INVALID for the rest, `{ref}` naming what stays undefined. A task that defines error behaviour for every invalid
input carries F-ERRORS instead:

```text
Every input that violates {ref} is answered as {eref} says, and hidden tests check it.
```

F-STDERR:

```text
Standard error is not compared and may hold anything; it is recorded, not judged. Only its size is bounded: a run that writes more than 256 MiB to standard error is stopped, and that test fails.
```

F-MEMORY, memory as the program's own control group counts it:

```text
Memory is measured as the peak memory of the program's own control group (cgroup `memory.peak`), which counts the program and every process it starts, including the page cache and kernel memory charged to them, with no swap. The program's memory reaches the limit when the system kills a process of the program's control group at the limit, or when the program's own runtime stops it for lack of memory; either way the run is stopped and fails that test. The test machine caps each runtime's own heap at 90 % of the limit, by that runtime's own setting. The tests run only on a machine that can read this peak, and no other measurement of memory passes or fails a test.
```

F-PROCS, the process bound (it follows F-MEMORY in the same requirement):

```text
The program and every process it starts may hold at most 512 processes and threads together. Beyond that the system refuses to create another, and a run that needs more fails that test.
```

F-FROZEN, the frozen list:

```text
The workspace holds `STATEMENT.md`, `samples/`, `runner/`, `build.sh`, `run.sh`, `PACKAGES.md` and the toolchain pin files named in `build.sh`. These are frozen inputs: do not change, rename or remove them. The program's sources go where `build.sh` and `run.sh` expect them; the two scripts say where.
```

F-BUILD, the build bound:

```text
`bash build.sh`, run in the workspace directory, builds the program with no network access and with no package outside the set listed in `PACKAGES.md`. It must exit 0 within 600 seconds. It runs once before the tests; a build that does not exit 0 fails every test.
```

F-ENV, the base environment (it follows F-RUN-O or F-RUN-D in the same requirement):

```text
Every test gives the program the same base environment, identical for both stacks apart from a few documented runtime settings of each stack: in it `LANG` and `LC_ALL` are `de_DE.UTF-8`, nothing else selects a locale, and `TZ` is `UTC`.
```

F-OFFLINE:

```text
The tests run offline. They use a clean copy of the workspace as you left it: build output and installed packages are removed, and the frozen inputs are put back as they were given. Every other file is yours, including the project files that declare or lock your packages, and is built as you left it, by `bash build.sh`. The program must not depend on the network, on files outside the workspace, or on anything written during an earlier run.
```

F-HIDDEN — the first sentence opens the requirement, the task's own sentences about what the hidden tests cover stand
between, and the second closes it:

```text
Hidden tests exist.
```

```text
The visible samples are a small part of the same suite; a program that passes them has not thereby passed the hidden tests.
```

### 2.3 Olympiad

F-LIMITS-O, the limits as floors and the registration rule (F-MEMORY and F-PROCS follow it in the same requirement):

```text
Each test runs the program once, in a fresh process, under a limit on wall-clock time and a limit on memory. The exact values are registered when the hidden suite is calibrated against reference solutions in both stacks, before any session, and are given to the test runner, not to the program's author. The registered values are never below these floors: **1 second** of wall-clock time per run and **256 MiB** of memory. (The registration rule: at least three times the wall time of the slower reference, and at least twice the peak memory of the larger reference.)
```

F-BYTES-O, byte-exact output, the default:

```text
The output is compared with the expected output byte for byte. Nothing is normalized: a missing or extra line feed, a trailing space or a carriage return makes the test fail.
```

F-EXIT-O:

```text
The process exits with status 0 after writing its output. Any other exit status fails the test, whatever was written.
```

F-RUN-O (F-ENV follows it in the same requirement):

```text
`bash run.sh`, run in the workspace directory with no arguments, starts the built program once: the test's input is its standard input, its standard output is collected whole, its exit status is read. One fresh process per test; nothing carries over from one run to the next. `run.sh` ends by replacing itself with the program (`exec`), so the process the tests start, signal and measure is the program itself.
```

F-SAMPLES-O:

```text
The same {k} cases are in `samples/`, one input file and one expected-output file per case (`samples/<id>.in`, `samples/<id>.out`). The bytes below are exact; `⏎` marks the single line feed that ends each line. `python3 -I runner/run_samples.py samples/` builds the program and runs them.
```

### 2.4 Daemon and web

F-PORT:

```text
The environment variable `PORT` holds a TCP port number; the program listens on it, and the tests connect to it at 127.0.0.1.
```

F-READY. `{readiness}` is one of `it accepts a TCP connection on that port` or `it has written the line {line} to
standard output`, where `{line}` is the line in a code span; `{T}` is whole seconds (2.5).

```text
The program is ready when {readiness}. A test waits at most {T} seconds for readiness; a program that is not ready by then fails that test.
```

F-STOP. `{which}` is `A test`, or, in a task that also carries F-EOF, `A test that does not end at the end of its
input`; `{G}` is the grace period in whole seconds (2.5).

```text
{which} ends by sending the signal SIGTERM to the program. The program must then exit with status 0 within {G} seconds, leaving no process it started still running. A program still running after {G} seconds is killed and fails that test, and so does one that exits with another status or leaves a process running.
```

F-EOF, for a program that reads standard input and whose spec defines what it does when that input ends. `{eref}` names
the requirements that say what the program does then; `{G}` is the same grace period as F-STOP's.

```text
A test may instead end at the end of its input: it closes the program's standard input, and the program then does what {eref} says and must exit with status 0 within {G} seconds of the end of its input, leaving no process it started still running. A program still running after {G} seconds is killed and fails that test, and so does one that exits with another status or leaves a process running.
```

F-STDOUT-D:

```text
Standard output is compared only where a requirement of this spec says what it holds; otherwise it is recorded, not judged.
```

F-BYTES-D:

```text
Every reply is compared with the expected reply byte for byte, except a field that a requirement of this spec says is compared otherwise. Nothing else is normalized.
```

F-BYTES-W:

```text
Every reply is compared on its status code, on each response header that a requirement of this spec names, and on its body; other headers are not compared. The body is compared byte for byte, except where a requirement of this spec names another comparison. Nothing else is normalized.
```

F-JSON, the one other comparison a spec may name. `{subject}` says what it covers: for a web task `The body of` and the
replies in words (`The body of every reply`); for a daemon the replies or output lines in words (`Each line of standard
output that R12 describes`).

```text
{subject} is compared as a JSON value: the members of an object may come in any order, numbers are compared by their exact decimal value with no rounding (so `2` equals `2.0` and `-0` equals `0`, and two numbers that differ in any digit differ), and whitespace outside strings is ignored; arrays keep their order unless a requirement says otherwise.
```

**F-BYTES-W against F-JSON.** For every reply that F-JSON's `{subject}` covers, F-JSON's comparison replaces the
byte-for-byte body comparison of F-BYTES-W (or of F-BYTES-D); the status and the named headers are still compared as
F-BYTES-W says. Every other reply's body stays byte for byte. In the scenario files the covered HTTP replies carry
`body json`, every other reply `body <str>` or `body empty`; a covered daemon reply or output line is the entry
`recv <conn> json <JSON text>`, every other one `recv <conn> <str>`: the transcript's form is the comparison (5.2).

A field whose value the program chooses (an id, a token, a timestamp) is named in a requirement with the form it must
have — for example "the `id` member is a UUID; it is compared by its form, and every later reply that names the same
order carries the same value". The scenario files express that with the normalised fields of 5.3, and relations between
such values (pairwise distinct, equal to a count) with the checks of 5.5.

F-LIMITS-D (F-LOAD, F-MEMORY and F-PROCS follow it in the same requirement):

```text
Each test starts the program once, in a fresh process, and drives one scenario against it under a bound on the program's memory. The exact bound is registered when the hidden suite is calibrated against reference solutions in both stacks, before any session, and is given to the test runner, not to the program's author. The registered bound is never below a floor of **256 MiB**. (The registration rule: at least twice the peak memory of the larger reference over all the task's load scenarios.)
```

F-LOAD. `{size}` says in words how large a load scenario gets — counts of connections, requests or bytes, or the R-ids
of the bounds it reaches (`the bounds of R32 and R33`); `{duration}` is whole seconds, and the whole scenario, its start and
its stop included, stays inside the test bound of 4.6 (600 seconds).

```text
The load scenarios are hidden scenarios of up to {size}, each lasting at most {duration} seconds; the bound on memory holds in every test, whether it is a load scenario or not.
```

F-DEADLINES:

```text
Every step of a scenario that waits — for readiness, for a reply, for the exit — has a deadline, and a step that does not complete by its deadline fails the test. The visible samples state their deadlines in their files; the deadlines of the hidden scenarios are fixed when the suite is calibrated, so that the reference solutions in both stacks meet each of them with margin.
```

F-RUN-D (F-ENV follows it in the same requirement):

```text
`bash run.sh`, run in the workspace directory with no arguments, starts the built program once per test, with the environment section 2 describes. One fresh process per test; nothing carries over from one test to the next. `run.sh` ends by replacing itself with the program (`exec`), so the process the tests start, signal and measure is the program itself.
```

F-SAMPLES-D:

```text
The same {k} scenarios are in `samples/`, one scenario file and one expected-transcript file per scenario (`samples/<id>.in`, `samples/<id>.out`), shown below exactly as stored. `runner/GRAMMAR.md` describes their format, and `python3 -I runner/run_samples.py samples/` builds the program and runs them.
```

### 2.5 The timing contract of daemon and web tasks

- **T**, the readiness deadline, and **G**, the grace period after SIGTERM, have series defaults: **T = 10 seconds** and
  **G = 5 seconds**. A spec states its own T in F-READY and its own G in F-STOP, always, also when they are the defaults.
- A task that needs another value states it there and adds F-TIMING, once per overridden value, in the same requirement.

F-TIMING:

```text
The series default for {what} is {D} seconds; this task uses {V} seconds because {reason}.
```

`{what}` is `readiness` or `the grace period`; `{D}` is `10` or `5`; `{V}` is the task's value; `{reason}` is a clause
that names the task's own cause, never a stack.

- **Where the runner reads them.** Every scenario of the task, visible or hidden, writes T as the `within` of its `start`
  and G as the `within` of the first `wait-exit` after its end trigger — `signal TERM`, or `close stdin` in a task that
  carries F-EOF (4.4); that deadline is measured from the trigger, not from the start of the `wait-exit` step. The runner
  takes them from there; when
  it is also given the task's registered T and G, a scenario whose values differ is refused as a format error and never
  re-timed. A conformance check requires every visible scenario's values to equal the spec's.
- Every other deadline is a step deadline: the step's own `within`, or the file's `default within` (4.3).

### 2.6 Olympiad output: the closed list of normalisations

Byte-exact output (F-BYTES-O) is the default. A task whose output is compared otherwise carries F-BYTES-O-N instead,
naming one or more items of this closed list, joined by `; ` in the order of the list:

| Item | The phrase in `{normalisations}` |
|---|---|
| final line feed | `a missing final line feed is ignored` |
| carriage returns | `a carriage return directly before a line feed is ignored` |
| line-end spaces | `spaces at the end of a line are ignored` |
| tokens | `the two are compared as sequences of tokens separated by runs of spaces and line feeds` |
| numeric tolerance (with tokens only) | `a token that is a decimal number matches an expected decimal number that differs from it by at most {a}`, `{a}` a decimal |

```text
The output is compared with the expected output byte for byte, except that {normalisations}. Nothing else is normalized.
```

The runner applies the named items to both outputs in the order of the list. In the numeric tolerance, a decimal number is an
optional `-`, digits, and optionally `.` and digits; a token with an exponent is not one.

No other normalisation exists. The test runner takes the task's normalisation from its registered entry, which is
copied from this sentence; the sample files stay exact bytes either way.

## 3. Visible samples

### 3.1 Files and ids

A task's visible samples are pairs of files with the same id, `samples/<id>.in` and `samples/<id>.out`, in the
workspace's read-only `samples/` directory, identical in both stacks' workspaces. Visible ids are `s01` … `s99`, numbered
from `s01` without gaps; `samples/` holds these pairs and nothing else. Samples are small and clean examples of the
spec's behaviour.

The hidden tests use the same pairs, with ids `h001` … `h999`, plus one manifest, `cases.txt`: one line per case,
`<id> <comparison> <R-id> [<R-id> …]`, naming how the case is compared and every requirement it asserts (comment lines
start with `#`). `<comparison>` is one of `bytes` (byte for byte throughout), `normalised` (a normalisation of the spec
or a normalised field applies), `json` (a JSON body is compared as a JSON value), `relational` (the transcript has
checks, 5.5), or several of the last three joined by `+` (`json+relational`). The visible samples need no manifest.

The runner, `runner/run_samples.py` (Python 3, standard library only), reads these files. `python3 -I
runner/run_samples.py samples/`, run in the workspace, builds the program with `bash build.sh`, then runs every pair and
prints one `PASS` or `FAIL` line per case (5.6). It reads the task's kind from the `Kind` row of `STATEMENT.md`. Without
the registered limits, which only the hidden tests have, it judges an olympiad case's time against the floor of the
spec's section 4 and judges no memory.

### 3.2 Olympiad samples

`<id>.in` is the program's standard input, byte for byte; `<id>.out` is its expected standard output, byte for byte.
Neither file has a header, a trailer or an encoding of its own: they are the raw bytes. Either may be empty. The
comparison is the spec's: byte for byte by default, or with the normalisation the spec names in its section 3.

### 3.3 Daemon and web samples: the scenario pair

`<id>.in` is a **scenario script** (section 4): what the test runner does — start the program, open connections, send,
read, wait, stop. `<id>.out` is the **expected transcript** (section 5): what the runner must have observed, one entry per
observation, with the comparison rules of section 5. The runner executes the script, writes the actual transcript in the
same syntax, and compares the two. Both files are UTF-8 text with line feeds; the program's bytes inside them are written
as strings with escapes, so any byte can be expressed.

### 3.4 Showing samples in the spec

Section 6 of a spec shows every visible sample, and only those, with the same bytes as the files.

- **Olympiad:** the lead sentence, then a table `| Id | Standard input | Expected standard output |` with one row per
  sample, each cell one code span. Bytes are shown as they are, except that each line feed is shown as `⏎`; a cell may
  show only printable ASCII and the space. A cell is a code span of one backtick when its bytes hold no backtick and do
  not both begin and end with a space; otherwise it is a code span of two backticks with one space added inside each
  end (``` `` a`b `` ```). In either form a `|` is written `\|`, which the table reads as `|`. A sample that cannot be shown
  these ways (bytes that are only spaces or hold two backticks in a row, another byte, or more than 200 bytes) shows
  `<size> bytes, see samples/<id>.in` (or `.out`) in its cell instead. Prose after the table may explain how a sample is
  built.
- **Daemon and web:** the lead sentence, then for each sample a `### <id>` sub-section with two fenced code blocks, the
  first headed by the line `` `samples/<id>.in` `` and the second by `` `samples/<id>.out` ``, each holding the file's
  text exactly (the file's final line feed is the line feed before the closing fence). Prose after a sample may explain
  how it is built. The grammar itself is not repeated in the spec: `runner/GRAMMAR.md` carries it.

## 4. The scenario script (`.in`)

The grammar is small on purpose: a runner written with a language's standard library alone implements it, with no
parser library. (The runner is Python 3 with its standard library only; the regular expressions of `capture … regex` use
the syntax of Python's `re` module on bytes.)

### 4.1 Lexical rules

- **Lines.** UTF-8, no byte-order mark, every line ended by a line feed, no tab outside a string. A line is blank, a
  comment, or one statement. A comment line's first non-space character is `#`; there are no comments at the end of a
  statement. Leading spaces are ignored: indentation is for the reader and means nothing.
- **Tokens** are separated by one or more spaces: keywords and names (`[a-z][a-z0-9_-]*`), the signal name `TERM`, an
  HTTP method (`[A-Z]+`), integers (`[0-9]+`), durations, strings, a JSON path, and JSON text.
- **Durations** are an integer and a unit, `250ms` or `2s`; at most `600s`.
- **Names** of connections and variables match `[a-z][a-z0-9_]*`. Two connection names are built in: `stdin` (the
  program's standard input; write only) and `stdout` (the program's standard output; read only). Environment variable
  names match `[A-Z_][A-Z0-9_]*`. No name is a keyword of 4.5 or 5.1.
- **Strings** are written in double quotes. Inside them every character stands for its UTF-8 bytes, except the escapes
  `\\` `\"` `\$` `\r` `\n` `\t` `\0` and `\xHH` (two hex digits, any case, one byte). A raw control character inside the
  quotes is an error.
- **String expressions.** Wherever the grammar says `<str>`, one or more strings stand separated by spaces and are joined
  into one byte sequence; a string followed directly by `*<count>` is repeated count times (1 ≤ count ≤ 10 000 000):
  `"0512|" "v"*512`. The **joined bytes** are the result: after joining, repetition and variable
  substitution. Every byte count and offset the grammar names (`split`, `bytes`, a length) counts the joined bytes, and an
  escape such as `\r` or `\xHH` is one byte.
- **Variables.** Inside a string, `${name}` is replaced by the bytes of the variable `name`, which a `file`, a `repeat` or
  a `capture` bound earlier (4.3, 4.4). A `$` not followed by `{` is an ordinary byte; `\$` is always a literal `$`, so
  `\${` is the three bytes `${`. In a transcript `${:` opens a normalised field (5.3) instead. A variable is bound once;
  binding a name twice is an error, except a `repeat` variable, which takes a new value each round.
- **A JSON path** is `$` followed by any number of `.member` (member `[A-Za-z_][A-Za-z0-9_]*`) and `[index]` (decimal)
  steps: `$.id`, `$.orders[0].id`.
- **JSON text** occupies the rest of a `body json` line, after `json`, and is one JSON value on that line. (After
  `capture <name> json` comes a JSON path instead.)

### 4.2 The file's head

```text
scenario 2
default within <duration>
[env <NAME> <str> | file <name> <str>]…
start ready port within <T>          (or)
start ready stdout <str> within <T>
```

- `scenario 2` is the first statement: the grammar version. This grammar is version 2; a file of another version is a
  format error, named with its file and line, and nothing is run.
- `default within <duration>` is the second: the deadline of every waiting step that names none.
- `env <NAME> <str>` sets one environment variable of the program: a variable the base environment (F-ENV) does not
  have, or `TZ`. `PORT` is reserved: the runner chooses a free port and sets it. No other variable of the base
  environment, in any spelling of its case, and not the per-test setting by which the test machine caps a runtime's heap,
  may be set: a scenario that sets one is refused and runs nothing.
- `file <name> <str>` writes the bytes into a new file inside `.scenario/` of the test's copy of the workspace and binds
  the variable `name` to its absolute path, for use in a later `env` (a fixture the program reads). `env` and `file`
  lines may come in any order, each variable bound before its use.
- Exactly one `start`, after these: it runs `bash run.sh` in the workspace copy and waits at most T for readiness —
  `ready port`: a TCP connection to 127.0.0.1 on `PORT` is accepted (the runner retries every 50 ms and closes that probe
  connection without sending a byte); `ready stdout <str>`: the program writes that line (the bytes of `<str>`, then a
  line feed) to standard output, every earlier line being discarded. It records the entry `ready`. Its `within` is the
  task's readiness deadline T and is always written.

**The program's environment** is the runner's own environment, plus `PORT`, plus the scenario's `env` lines; the runner
adds nothing else. The test machine gives the runner the series' base environment: the same for every task and for both
stacks apart from a few documented runtime settings of each stack, with `LANG` and `LC_ALL` set to `de_DE.UTF-8`.

The program's standard input is a pipe that stays open and receives only what the scenario sends to `stdin`. The runner
reads the program's standard output and standard error continuously, so the program never blocks on a full pipe. When a
step of the scenario reads `stdout`, **every byte of standard output is kept, in order**, from the start (after the
readiness line when `ready stdout` is used): a `recv stdout` reads the bytes no earlier step has read, whenever the
program wrote them, also after the program has exited. When no step reads `stdout`, standard output, like standard error,
is only kept with the test's record (bounds in 4.6).

### 4.3 Steps

Options after a step's fixed words may come in any order. `[within <duration>]` overrides the default deadline of a
waiting step, measured from the moment the step starts. The *entry* column names what the step adds to the transcript.

| Step | What the runner does | Entry |
|---|---|---|
| `connect <conn> tcp\|http [rcvbuf <bytes>] [within <d>]` | opens a TCP connection to 127.0.0.1 on `PORT` and names it, retrying a refused attempt every 50 ms until its deadline — its `within`, else the scenario's T (the `within` of `start`); `http` makes it an HTTP/1.1 client connection; `rcvbuf` sets the client's receive buffer (a slow reader's backpressure) | none |
| `send <conn> <str> [split <o1>[,<o2>…] pause <d>] [within <d>]` | writes the joined bytes in one write; with `split`, writes them in pieces cut at the given offsets into the joined bytes (strictly increasing, each 0 < o < length), sleeping `pause` between pieces; completes when every byte is written | none |
| `request <conn> <METHOD> <str>` | starts an HTTP request on an `http` connection, the string being the target (it begins with `/`); the `header <str> <str>` lines after it add or replace headers (a header's name is exactly one plain string; its value is any string), and at most one `body <str>` or `body json <JSON text>` line gives the body; the request is written when the next other statement is reached | none |
| `recv <conn> <mode> [chunk <bytes> pause <d>] [capture …]… [within <d>]` | reads one reply (modes below); with `chunk`, reads at most that many bytes per read and sleeps `pause` between reads | one `recv` or `reply` entry; `lines <k>` adds k entries |
| `expect <conn> quiet for <d>` | checks that during that time there is no unread byte on the connection (one that waits from earlier counts, as well as one that arrives) and the program does not close the connection; allowed on `stdout` too | `quiet <conn>` |
| `expect <conn> closed [within <d>]` | checks that the program closes the connection: end of stream with no byte before it; on `stdout`, the end of standard output | `closed <conn>` |
| `close <conn>` | closes the connection; `close stdin` ends the program's standard input | none |
| `shutdown <conn>` | closes the sending half of a TCP connection; reading goes on | none |
| `sleep <d>` | waits; it has no deadline and takes no `within` (one is a format error); its time counts toward the test bound of 4.6 | none |
| `signal TERM` | sends SIGTERM to the program | none |
| `wait-exit within <d>` | waits for the program to exit, or records the exit it has already made; at the deadline the runner kills it and its process group | `exit <status>` |

**Request serialisation and default headers.** The runner writes `<METHOD> <target> HTTP/1.1`, then its two default
headers — `Host: 127.0.0.1:<PORT>`, and `Content-Length: <n>` when there is a body — then the `header` lines in order as
`<name>: <value>`, then an empty line and the body; every line ends with CR LF. It writes no other header: no
`User-Agent`, no `Accept`, no `Connection` (so the connection is persistent, as HTTP/1.1 makes it by default). A `header`
line whose name equals a default header's, ASCII letters compared without regard to case, replaces that default at the
default's position instead of adding a second one. Two `header` lines with one name, when it is not a default's, are
written as two lines, in order. A request that must lack a default header, or break HTTP's syntax, is
written as raw bytes with `send` on the same `http` connection. `body json <JSON text>` sends the JSON text's bytes as
written, after variable substitution, and adds no `content-type`; a spec that requires one lists it with `header`.

**A closed HTTP connection.** When a request on an `http` connection meets a connection the program has closed before a
single byte of the response arrived, the runner opens a new connection under the same name and writes the request once
more, within the same step deadline. It reconnects at most once per request: a second close is the actual entry
`eof <conn> ""`. A program that must keep connections open is tested with `expect <conn> quiet`, which fails when the
program closes the connection.

**`recv` modes.** On a `tcp` connection or `stdout`, the mode says where the reply ends:

| Mode | The reply is |
|---|---|
| `bytes <n>` | exactly the next n bytes |
| `until <str> [count <k>]` | every byte up to and including the k-th next occurrence of the delimiter (k = 1 by default); occurrences do not overlap, so the count goes on after the end of the one before |
| `lines <k>` | the next k lines, each ending with a line feed; each line, its line feed included, is its own entry |
| `eof` | every byte up to the end of the stream |

A reply's bytes are every byte its mode delimits, the delimiter and the final line feed included. Bytes that arrived
beyond the reply stay buffered for the next `recv` on that connection, so pipelined replies are read
one step at a time. On an `http` connection `recv` takes no mode: the reply is one whole HTTP response — status line,
headers, and a body delimited by `Content-Length`, by chunked transfer coding (which the runner decodes) or by the end of
the connection; an interim `1xx` response is skipped.

**Captures** bind a value of the reply just read to a variable for later steps and for the transcript:

- `capture <name> json <path> [as <type>]` — only on an `http` reply (elsewhere it is a format error), parses the body as JSON (numbers as exact decimals,
  5.2) and takes the value at the path;
- `capture <name> regex <str> [as <type>]` — searches the reply's bytes (on `http`: its body) with the regular expression
  and takes group 1, or the whole match when the expression has no group.

A capture reads the whole reply just read: with `lines <k>`, all k lines together. On `http` a capture reads the body only; no capture reads a header. A header's value is compared in the transcript, where a
binding field (5.3) can bind it for the transcript's checks (5.5); a later step of the script cannot use it.

`as <type>` checks the value against a type of 5.3. A capture that finds nothing, or a value of another form, ends the
scenario with the entry `capture-failed <name>`. A value captured from JSON keeps its JSON type: substituted into a
string, a JSON string gives its UTF-8 bytes and a number gives its digits as written in the reply.

**Failure.** A waiting step that misses its deadline records what it saw (5.1) and the scenario goes on, so the report
shows every later entry too. These failures end the scenario at once: the program is not ready in time, a connection is
still refused at its deadline, the program exits early (4.4), a capture fails, a buffer overflows (4.6), and the test
bound is reached (4.6). The runner then stops the program (SIGTERM, then SIGKILL to its process group after the
deadline of the last `wait-exit`, or 5 s when there is none).

### 4.4 Groups and the end

```text
together
  <send or request steps, one per connection, two or more>
end

repeat <count> <name>
  <steps>
end
```

- `together … end` is a **start-together group**, the barrier of a race: the runner prepares every write of the group in
  full, waits until each one's writer is ready, and releases them at the same instant. The group completes when every
  write has completed, within the default deadline. Only `send` and `request` (with its `header` and `body` lines) may
  stand inside, each on a different connection; the replies are read by `recv` steps after the group.
- `repeat <count> <name> … end` runs its steps count times (1 ≤ count ≤ 100 000) with the variable `name` bound to `0`,
  `1`, … in decimal. Repeats may nest, and may hold a `together` group; a `together` group holds no `repeat`.

**The end.** A scenario has exactly one `wait-exit`, always with its `within`. Before it stands the scenario's **end
trigger**: `signal TERM`, or `close stdin` for a program that ends at the end of its input (F-EOF) — whichever of the two
comes first. The `within` of the `wait-exit` is the task's grace period G, measured from the trigger. The runner keeps this deadline on its
own clock: at the trigger plus G it kills the program, whichever step is running then.

- **Between the trigger and `wait-exit`**, and after `wait-exit`, only `recv`, `expect`, `sleep` and `close` may stand:
  the steps that read what the program wrote (on `stdout` above all: the lines it writes when it stops) or wait. Nothing
  more is sent to the program.
- **An exit after the trigger is not early.** The runner records it, keeps every byte the program wrote before it, and
  lets the later steps read those bytes in order; a `recv` that needs more than the program wrote fails with
  `eof <conn> "…"`. The `wait-exit` then records the exit already made.
- **An exit before the trigger is early**: the scenario ends with `early-exit <status>`.
- **Where these stand.** The end trigger and `wait-exit` stand at the top level of the script, never inside a group. A
  `capture` does not stand inside a `repeat`, because it would be bound every round.

### 4.5 Keywords

The complete list of the script's words: `scenario`, `default`, `within`, `env`, `file`, `start`, `ready`, `port`,
`stdout`, `stdin`, `connect`, `tcp`, `http`, `rcvbuf`, `send`, `split`, `pause`, `request`, `header`, `body`, `json`,
`recv`, `bytes`, `until`, `count`, `lines`, `eof`, `chunk`, `capture`, `regex`, `as`, `expect`, `quiet`, `for`, `closed`,
`close`, `shutdown`, `sleep`, `together`, `repeat`, `end`, `signal`, `TERM`, `wait-exit`.

### 4.6 The runner's bounds

No wait and no buffer is without a ceiling:

- **One test lasts at most 600 seconds** from the start of the program, sleeps included. At that bound the runner stops
  the program and the case fails with `test bound: 600 s reached`.
- The program's standard output (when no step reads it) and its standard error are kept up to 16 MiB each; what is cut
  is recorded as cut. A program that writes more than 256 MiB to standard error is stopped, and the test fails with
  `standard error overflow` (F-STDERR).
- A connection's bytes received and not yet read by a step are held up to 256 MiB; beyond that the scenario ends with the
  actual entry `overflow <conn>`. Standard output counts as such a connection when a step reads it (`overflow stdout`).
- A regular-expression capture searches at most the first 16 MiB of a reply.
- A script expands to at most 1,000,000 steps in all, each `repeat` counted every round; a longer one is a format error.
- A string expression expands to at most 200 parts, and a longer one is a format error. A JSON value nested deeper than
  200 levels is read as invalid JSON: the entry does not match, and the report says `bad-json`. Matching the fields of one entry stops after 500,000 steps, and the entry then does not match (`the match was abandoned as
  too complex`). One step is one try to match the rest of the entry from one position; a position that failed is remembered and
  not tried again.

## 5. The expected transcript (`.out`)

### 5.1 Entries

The transcript's first statement is `transcript 2`, the grammar version. Then, one per line (an HTTP reply on several),
the entries in the order the script produces them, and after them any `check` statements (5.5); blank lines and `#`
comment lines are allowed, indentation means nothing, and `repeat <count> <name> … end` stands for its body written
count times, as in the script.

| Entry | Produced by | Holds |
|---|---|---|
| `ready` | `start` | — |
| `recv <conn> <str>` | `recv` on `tcp` or `stdout` | the reply's bytes, compared as bytes |
| `recv <conn> json <JSON text>` | `recv` on `tcp` or `stdout` | the reply's bytes, compared as one JSON value (5.2) |
| `reply <conn> <status>` followed by `header <str> <str>` lines and exactly one `body …` line | `recv` on `http` | the status code, the headers to compare (a header's name is exactly one plain string), the body: `body <str>`, `body json <JSON text>` or `body empty` |
| `quiet <conn>` | `expect … quiet` | — |
| `closed <conn>` | `expect … closed` | — |
| `exit <status>` | `wait-exit` | the exit status |

The actual transcript, written by the runner, uses the same entries and, where a step failed, one of these, which never
match an expected entry:

| Actual-only entry | Meaning |
|---|---|
| `not-ready` | the program was not ready by the deadline |
| `refused <conn>` | the connection was still refused at its deadline |
| `timeout <conn> <str>` | the reply was not complete by the deadline; the bytes received so far |
| `eof <conn> <str>` | the stream ended before the reply was complete, or an HTTP connection closed twice; the bytes received so far |
| `noise <conn> <str>` | bytes arrived where `quiet` or `closed` was expected |
| `open <conn>` | the connection was still open at the deadline of `closed` |
| `bad-reply <conn> <str>` | the bytes are not an HTTP response |
| `overflow <conn>` | the connection's unread bytes passed the bound of 4.6 |
| `early-exit <status>` | the program exited before the scenario's end trigger (4.4) |
| `exit-timeout` | the program was still running at the deadline of `wait-exit` and was killed |
| `exit signal <NAME>` | the program ended by a signal |
| `capture-failed <name>` | a capture found nothing of its form |

Where `quiet` was expected and the program closed the connection instead, the actual entry is `eof <conn> ""`.

The runner writes strings canonically: printable ASCII as it is, `"` and `\` escaped, `$` escaped where `{` follows
it, CR, LF, tab and NUL as `\r` `\n` `\t` `\0`, every other byte as `\xHH` (lower case). Long runs are written as they
are; the runner never shortens a transcript it keeps, and abbreviates only in the report (5.6).

The complete list of the transcript's words: `transcript`, `ready`, `recv`, `reply`, `header`, `body`, `json`, `empty`,
`quiet`, `closed`, `exit`, `repeat`, `unordered`, `end`, `check`, `distinct`, `equal`, and, in actual transcripts only,
`not-ready`, `refused`, `timeout`, `eof`, `noise`, `open`, `bad-reply`, `overflow`, `early-exit`, `exit-timeout`,
`signal` and `capture-failed`.

### 5.2 Comparison

The actual transcript, expanded, must hold the same number of entries as the expected one, expanded, and each actual
entry must match the expected entry at its position (or its group, 5.4); then every check must hold (5.5).

- **Byte-exact by default.** A `recv <conn> <str>` entry matches when its bytes — every byte the `recv` mode delimited,
  the delimiter and the final line feed included — equal the expected bytes, every variable substituted, every normalised
  field (5.3) matched by its form. `exit`, `ready`, `quiet` and `closed` match exactly.
- **A JSON `recv` entry**, `recv <conn> json <JSON text>`, matches when the reply's bytes, read as UTF-8, are one JSON
  value — whitespace around it, the final line feed included, means nothing — that matches the expected JSON value by the
  rules below. It is for a program whose replies or output lines a requirement compares as JSON values (F-JSON);
  byte-exact stays the default.
- **An HTTP `reply`** matches when the status code is equal; when each `header` line's header is present (names compared
  without regard to case) and its value — the values of every occurrence joined by `, ` in order, each without
  surrounding spaces and tabs — equals the expected bytes; and when the body matches. Headers that the entry does not
  list are not compared. **The body form decides the body comparison:** `body <str>` compares the decoded body byte for
  byte; `body empty` requires a body of zero bytes; `body json` compares JSON values. There is no other body comparison.
- **JSON values** (`body json`, `recv … json`): the actual body or reply must be one valid JSON value, with no member name repeated in one object
  and no `NaN` or `Infinity`. **Every JSON number is read as an exact decimal from the digits written** — an integer as an
  integer of any size, any other number as an exact decimal fraction, never as a binary floating-point number — and two
  numbers match when their exact values are equal: `2` matches `2.0` and `2.00`, `-0` matches `0`, `1e2` matches `100`,
  `22.0` does not match `22.01`, and `9007199254740993` does not match `9007199254740992`. Objects match when they have
  the same member names and each member's value matches, in any order; arrays match element by element, in order;
  strings match by their characters; `true`, `false` and `null` match themselves; whitespace outside strings means
  nothing. Inside the expected JSON, a string whose whole content is `${name}` matches a value equal to the variable's (a
  JSON value keeps its type; a captured byte string matches a JSON string with those characters), and a string whose
  whole content is a normalised field matches by that field's rule (a number field such as `${:int}` then matches a JSON
  number). **A string that holds fields among literal text** — a fixed prefix and a field, `"eyJhbGciOi….${:token}"`, or
  several of each — matches a JSON string whose characters match it the way a byte entry does: the literal text exactly,
  each field by its form, the shortest run first. A literal `${` cannot be expected inside `body json`; use `body <str>`.

### 5.3 Normalised fields

A normalised field stands, inside an expected string or JSON string, for a value the program chooses. The list is closed:

| Field | Matches |
|---|---|
| `${name}` | the value of a variable bound in the script, exactly |
| `${:int}` | `-?(0\|[1-9][0-9]*)`; in JSON, a number whose value is an integer |
| `${:uint}` | `0\|[1-9][0-9]*`; in JSON, a non-negative integer number |
| `${:hex}` | `[0-9a-f]+` |
| `${:uuid}` | 8, 4, 4, 4 and 12 hex digits joined by hyphens, either case |
| `${:token}` | `[A-Za-z0-9._~-]+` |
| `${:ts}` | a UTC time `YYYY-MM-DDThh:mm:ss`, optionally `.` and 1 to 9 digits, then `Z`; it must be a real calendar time |
| `${:ts.<d>}` | a `${:ts}` with exactly d fraction digits, d from 0 to 9 (`${:ts.0}`: none, and no `.`; `${:ts.3}`: milliseconds) |
| `${:ts~<duration>}`, `${:ts.<d>~<duration>}` | a `${:ts}` (or `${:ts.<d>}`) within that duration of the runner's UTC clock at the moment the entry was recorded |
| `${:num <decimal> <tolerance>}` | a decimal number (`-?[0-9]+(\.[0-9]+)?`; in JSON, a number) within the tolerance of the decimal, in exact decimal arithmetic |

**A binding field**, `${:<type>=<name>}` (for example `${:int=a1}` or `${:token=r${i}}`), matches like `${:<type>}` and
binds the value it matched to `name` for the checks of 5.5. Inside a `repeat` the name may carry the repeat variable, so
each round binds its own name. A binding name is bound once, like a script variable, and never clashes with one.

Except in JSON, a field matches the shortest run of bytes that lets the rest of the entry match; the entry's own length
is fixed by the script's `recv` mode, so a field never decides how much is read. The same types serve
`capture … as <type>` (without the window, the tolerance and the binding).

### 5.4 Order-insensitive sets

Order is fixed everywhere except where the transcript marks it, and the transcript marks it only where the spec says the
order is not defined:

- **Entries.** `unordered … end` around two or more entries (at most 64): they match the same number of actual entries at
  that position in any order, when some one-to-one assignment matches every pair. Inside such a group only, an entry's
  connection may be `*`, matching any connection — for the replies of concurrent clients when the spec does not say which
  client gets which reply. Groups do not nest. An `unordered` group holds no
  `repeat`.
- **JSON arrays.** In expected JSON, the object `{"$unordered": [ … ]}` stands for an array whose elements match the
  listed ones in any order, under the same one-to-one rule — for a listing whose order the spec leaves undefined. The
  member name `$unordered` is reserved for this.

### 5.5 Relational expectations

Some requirements relate values to one another rather than fixing each one: the replies of concurrent clients are all
different, or a final value equals the number of operations sent. The transcript states such a relation with **checks**,
written after the last entry; a check produces no entry.

- `check distinct <name> [<name> …]` — the values are pairwise distinct: as bytes, or as exact JSON values. A name may
  end in `*` to stand for every binding and capture name with that prefix (`r*`).
- `check equal <name> <decimal | name | str>` — the value equals the decimal (in exact decimal arithmetic), the other
  value, or the string: a string matches a byte value with those bytes or a JSON string with those characters, never a
  number. A decimal matches a JSON number, or a byte value written as a decimal (an optional `-`, digits, and
  optionally `.` and digits); never a JSON string.

A check names binding fields of the transcript and `capture` variables of the script alike; both are bound once, in one
namespace. Checks run after every entry has matched, in file order. A check may not relate a value bound inside an
`unordered` group or a `$unordered` array to anything outside that group or array: `check distinct` names all of the
group's bindings or none of them. A check that does is a format error.
For example, three requests that must each get a fresh token:

```text
transcript 2
ready
repeat 3 i
  reply c 200
    body json {"token":"${:token=t${i}}"}
end
exit 0
check distinct t*
```

### 5.6 The verdict and the mismatch report

A scenario passes when every step ran, the two transcripts match (5.2 to 5.4), every check holds (5.5), the program left
no process running after its exit, and its memory did not reach the registered bound (F-MEMORY: no process of its
control group was killed at the bound, and its own runtime did not stop it for lack of memory).
The first three are this grammar's; the last two are the runner's own lifecycle checks, reported on their own lines.

The runner reports one line per scenario, `PASS <id> — <k> entries`, or `FAIL <id> — <reason>` followed by the first
mismatch:

```text
FAIL s02 — entry 4 of 9 differs
  step    s02.in:14   recv c1 bytes 10
  expect  s02.out:6   recv c1 "0005|mango"
  actual              recv c1 "0005|melon"
  first difference at byte 6
```

For a reply it compares the status, then the headers, then the body, and names the first part that differs (`status`,
`header location`, `body`) and, in a JSON body, the path of the first
difference (`body $.qty: expected 2, actual 3`); for a group, the expected entry no assignment could match; for a count,
`expected 9 entries, actual 6`, with the step that ended the scenario; for a check, `check: t0 and t2 are equal` or
`check: total is 63, expected 64`; for the lifecycle, `stop: still running 5 s after SIGTERM, killed`,
`leftover: 1 process still running after the exit`, `memory limit`, `standard error overflow` or
`test bound: 600 s reached`. Strings longer than 120 bytes are shown as their first 60 and last 40 bytes, with the count
left out between. The full actual transcript is kept with the test's record.

## 6. Worked examples

**These three examples are illustrative.** They show features of the grammar working end to end; they are not tasks, the
toy programs of 6.2 and 6.3 are no task's and fix nothing about any task of this repository, and nothing in them says
what any task's hidden tests hold. Each shows both files, complete.

### 6.1 Olympiad — one sample of the pilot spec

The pilot spec's third sample, n = 7. The spec's table row reads:

| Id | Standard input | Expected standard output |
|---|---|---|
| `s03` | `7⏎` | `7 22 11 34 17 52 26 13 40 20 10 5 16 8 4 2 1⏎` |

`samples/s03.in`, 2 bytes (`37 0a`):

```text
7
```

`samples/s03.out`, 45 bytes, ending in one line feed and holding no other line feed:

```text
7 22 11 34 17 52 26 13 40 20 10 5 16 8 4 2 1
```

The runner feeds the 2 bytes to `bash run.sh` and compares the program's standard output with the 45 bytes, byte for
byte: a missing final line feed or a trailing space fails the case (`first difference at byte 44` for the missing line
feed).

### 6.2 Daemon — a toy store over an invented framed protocol

The toy protocol is invented for this example. A frame is four decimal digits giving the payload's length in bytes
(`0000` to `9999`), the byte `|`, and the payload; requests and replies are frames. The toy understands four payloads:
`hi` → `ok`; `put <name> <text>` → `ok`; `take <name>` → the stored text, or `none`. The scenario shows a frame split
across two writes with a check that nothing comes back early, two requests in one write, `split` cutting the joined
bytes, and a slow reader that must not stop the toy from serving another client.

`x2.in`:

```text
scenario 2
# Illustrative only: a toy store over an invented framed protocol (four digits
# of payload length, '|', the payload). No task of this repository.
default within 2s
start ready port within 10s
connect c1 tcp

# 1. One frame in two writes: no reply may come before it is complete.
send c1 "0014|put fruit ki"
expect c1 quiet for 200ms
send c1 "wi"
recv c1 bytes 7

# 2. Two requests in one write; the replies come back in order.
send c1 "0010|take fruit" "0002|hi"
recv c1 bytes 9
recv c1 bytes 7

# 3. split cuts the joined bytes: of these 15, offset 12 falls inside the
#    second string.
send c1 "0010|" "take fruit" split 2,12 pause 50ms
recv c1 bytes 9

# 4. A slow reader: 'lazy' asks for a 9000-byte text 64 times in one write and
#    reads nothing; meanwhile c1 is still served. Then 'lazy' reads its 64
#    replies of 9005 bytes each, 4096 bytes at a time.
send c1 "9008|put big " "z"*9000
recv c1 bytes 7
connect lazy tcp rcvbuf 4096
send lazy "0008|take big"*64
sleep 200ms
send c1 "0002|hi"
recv c1 bytes 7 within 500ms
repeat 64 i
  recv lazy bytes 9005 chunk 4096 pause 2ms within 5s
end

# 5. The stop.
signal TERM
wait-exit within 5s
```

`x2.out`:

```text
transcript 2
ready
# 1.
quiet c1
recv c1 "0002|ok"
# 2.
recv c1 "0004|kiwi"
recv c1 "0002|ok"
# 3.
recv c1 "0004|kiwi"
# 4.
recv c1 "0002|ok"
recv c1 "0002|ok"
repeat 64 i
  recv lazy "9000|" "z"*9000
end
# 5.
exit 0
```

A toy that answers the first part of the split frame early fails at entry 2 with `noise c1 "…"`; one that blocks every
client while `lazy` reads nothing fails at entry 8 with `timeout c1 ""`.

### 6.3 Web — two clients racing on one idempotency key

The toy: `POST /orders` with an `Idempotency-Key` header creates one order and answers `201` with a `Location` header and
the order as JSON; a request that repeats a key gets the reply that key's first request got — the same status, the same
body — even when the two requests race; `GET /orders/<id>` answers `200` with the order; `GET /orders` lists every order
in no defined order. Ids are UUIDs; `at` is the creation time.

`x3.in`:

```text
scenario 2
# Illustrative only: a toy order endpoint with idempotency keys. No task of
# this repository.
default within 2s
start ready port within 10s
connect a http
connect b http

# 1. Two clients race on one key: both requests are released at one instant.
together
  request a POST "/orders"
    header "content-type" "application/json"
    header "idempotency-key" "toy-key-1"
    body json {"item":"tea","qty":2}
  request b POST "/orders"
    header "content-type" "application/json"
    header "idempotency-key" "toy-key-1"
    body json {"item":"tea","qty":2}
end
recv a capture order json $.id as uuid capture at json $.at as ts
recv b

# 2. The captured id is reused in a later request.
request a GET "/orders/${order}"
recv a

# 3. A second key makes a second order; the listing holds exactly two.
request b POST "/orders"
  header "content-type" "application/json"
  header "idempotency-key" "toy-key-2"
  body json {"item":"cake","qty":1}
recv b capture other json $.id as uuid
request a GET "/orders"
recv a

# 4. The stop.
signal TERM
wait-exit within 5s
```

`x3.out`:

```text
transcript 2
ready
# 1. Whichever request won the race, both replies name the same order.
reply a 201
  header "location" "/orders/${order}"
  body json {"id":"${order}","item":"tea","qty":2,"at":"${:ts~10s}"}
reply b 201
  header "location" "/orders/${order}"
  body json {"id":"${order}","item":"tea","qty":2,"at":"${at}"}
# 2.
reply a 200
  body json {"id":"${order}","item":"tea","qty":2,"at":"${at}"}
# 3.
reply b 201
  header "location" "/orders/${other}"
  body json {"id":"${other}","item":"cake","qty":1,"at":"${:ts~10s}"}
reply a 200
  body json {"orders":{"$unordered":[{"id":"${order}","item":"tea","qty":2,"at":"${at}"},{"id":"${other}","item":"cake","qty":1,"at":"${:ts~10s}"}]}}
# 4.
exit 0
```

A toy that let both racing requests create an order fails at entry 3:

```text
FAIL x3 — entry 3 of 7 differs
  step    x3.in:21   recv b
  expect  x3.out:7   reply b 201
  actual             reply b 201
  header location: expected "/orders/0b6f…", actual "/orders/5d2a…"
```

## 7. Changelog

After the freeze this document changes only by a dated amendment, which raises `Format version` by one and is listed
here with the tasks it touches.

- v1 · 2026-10-31 · draft — the first version, written from the pilot spec of task 01: the spec file and header, R-ids,
  the fixed sentences, the sample pair of each kind, the scenario grammar and its comparison rules, three illustrative
  examples, conformance and amendment rules. Not gated; superseded by v2.
- v2 · 2026-10-31 · draft — version: `Format version: 2`; the grammar version is `scenario 2` / `transcript 2`, and a file
  of another version is refused, never run (4.2, 5.1).
- v2 · 2026-10-31 · draft — exact-decimal JSON numbers: every number read as an exact integer or decimal from its digits,
  never as a binary float; `2` = `2.0`, `-0` = `0`, numbers differing in any digit differ (5.2; F-JSON's wording; JSON
  captures and `${:num}` likewise).
- v2 · 2026-10-31 · draft — F-LOAD: a fixed sentence with slots for a load scenario's size and duration, in the same
  requirement as F-LIMITS-D; F-LIMITS-D's rule now reads "over all the task's load scenarios" (2.1, 2.4).
- v2 · 2026-10-31 · draft — default request headers: `Host` and, with a body, `Content-Length`, nothing else; a `header`
  line of the same name replaces a default in place; a request without a default is written with `send` (4.3).
- v2 · 2026-10-31 · draft — `connect` retries a refused attempt every 50 ms until its deadline, its `within` or else the
  scenario's T; still refused ends the scenario with `refused` (4.3).
- v2 · 2026-10-31 · draft — a closed HTTP connection: reconnect once and rewrite the request within the same step deadline
  when nothing of the response arrived; a second close is `eof`; `expect … quiet` now also fails when the program closes
  the connection, so persistence stays testable (4.3, 5.1).
- v2 · 2026-10-31 · draft — relational expectations: binding fields `${:<type>=<name>}` and the checks
  `check distinct` and `check equal`, with a failure line in the report (5.3, 5.5, 5.6).
- v2 · 2026-10-31 · draft — `{ref}` slots take lists of any length, `R8, R11, R14, R17 and R32` (2.1).
- v2 · 2026-10-31 · draft — tables belong to their requirement: a requirement's span runs to the next requirement or
  heading, and a table or normative sentence outside every span is a defect (1.3).
- v2 · 2026-10-31 · draft — the T and G binding: series defaults T = 10 s and G = 5 s, always stated in F-READY and F-STOP,
  an override with F-TIMING and a reason, and the runner reading them from the `within` of `start` and of the
  `wait-exit` after `signal TERM`, refusing a mismatch (2.5, 4.2, 4.4).
- v2 · 2026-10-31 · draft — F-BYTES-W against F-JSON: F-JSON's comparison replaces the byte-for-byte body comparison for
  the replies it covers; the transcript's body form is the comparison (2.4, 5.2).
- v2 · 2026-10-31 · draft — `sleep` has no deadline and takes no `within`; it counts toward the 600 s test bound (4.3, 4.6).
- v2 · 2026-10-31 · draft — `split` offsets index the joined bytes of the whole string expression, after joining,
  repetition and substitution (4.1, 4.3; the daemon example's step 3).
- v2 · 2026-10-31 · draft — the daemon example now uses a protocol invented for it and names no real protocol or product
  (6.2); the examples say they illustrate the grammar and hold nothing of any task's hidden tests (6).
- v2 · 2026-10-31 · draft — the reading block removed: specs no longer copy a summary of the grammar, since every
  workspace carries `runner/run_samples.py` and `runner/GRAMMAR.md` (sections 3 to 5 of this document); F-SAMPLES-O and
  F-SAMPLES-D name the runner, and F-FROZEN lists `runner/` (2.2 to 2.4, 3.1, 3.4).
- v2 · 2026-10-31 · draft — the base environment: F-ENV and 4.2 say the same base for both stacks apart from a few
  documented runtime settings of each, with `LANG` and `LC_ALL` = `de_DE.UTF-8`; the runner adds only `PORT` and the
  scenario's `env` lines (2.2, 4.2).
- v2 · 2026-10-31 · draft — memory: F-MEMORY says memory is the peak of the program's own control group
  (cgroup `memory.peak`, page cache and kernel memory included, no swap), that tests run only where it can be read, and
  that nothing else judges memory (2.2, 5.6).
- v2 · 2026-10-31 · draft — `run.sh` ends with `exec`: F-RUN-O and F-RUN-D say so (2.3, 2.4).
- v2 · 2026-10-31 · draft — credit: a known task's credit lives only in the README's attribution section, which no
  workspace or run base carries; no credit file beside the task (1.5).
- v2 · 2026-10-31 · draft — the runner's bounds: one test at most 600 s, output and error kept up to 16 MiB each, unread
  bytes up to 256 MiB (`overflow`), regex captures over at most 16 MiB (4.6).
- v2 · 2026-10-31 · draft — the hidden manifest `cases.txt` names each case's kind of comparison beside its R-ids (3.1).
- v2 · 2026-10-31 · draft — the maintainers' sections (conformance check, amendment rules, the decisions' changelog)
  moved out of this document into a maintainers' document that no workspace or run base carries; this changelog keeps
  only what changed in the format.
- v2.1 · 2026-10-31 · draft — version: `Format version: 2.1`, still before the gate. The grammar stays version 2
  (`scenario 2` / `transcript 2`): every change below adds to it, and no v2 file reads differently.
- v2.1 · 2026-10-31 · draft — the end of a scenario: the end trigger is `signal TERM` or `close stdin`, whichever comes
  first; an exit after it is not early, every byte the program wrote before it stays readable by later steps, and
  `early-exit` applies only to an exit before the trigger (4.3, 4.4, 5.1).
- v2.1 · 2026-10-31 · draft — steps between the trigger and `wait-exit`: `recv`, `expect`, `sleep` and `close` may stand
  there and after `wait-exit`; G binds to the first `wait-exit` after the trigger and is measured from the trigger (2.5,
  4.4).
- v2.1 · 2026-10-31 · draft — F-EOF: a fixed sentence for a program whose spec defines an end at the end of its input,
  with the same G; F-STOP's opening becomes the slot `{which}`, `A test` or, beside F-EOF, `A test that does not end at
  the end of its input` (2.1, 2.4).
- v2.1 · 2026-10-31 · draft — JSON `recv` entries: `recv <conn> json <JSON text>` compares a TCP or standard-output reply
  as one JSON value; byte-exact stays the default; F-JSON's opening becomes the slot `{subject}`, so it covers output
  lines as well as HTTP bodies (2.4, 5.1, 5.2).
- v2.1 · 2026-10-31 · draft — standard output: when a step reads `stdout`, every byte is kept in order and a `recv stdout`
  reads what no earlier step read, whenever it was written (4.2).
- v2.1 · 2026-10-31 · draft — `expect stdout quiet` and `expect stdout closed` are allowed (4.3).
- v2.1 · 2026-10-31 · draft — an entry's bytes include its delimiter and its final line feed, `lines` entries too (4.3,
  5.2).
- v2.1 · 2026-10-31 · draft — `{ref}` lists and ranges read `any of …` (`violates any of R8, R11 and R32`); a single id
  stays bare (2.1).
- v2.1 · 2026-10-31 · draft — fields among literal text inside a JSON string (`"<prefix>.${:token}"`), and `${:ts.<d>}`
  for a time with exactly d fraction digits (5.2, 5.3).
- v2.1 · 2026-10-31 · draft — checks name `capture` variables as well as binding fields, in one namespace, and
  `check equal` takes a string (5.5).
- v2.1 · 2026-10-31 · draft — olympiad sample cells: `|` written `\|`, a backtick inside a two-backtick code span with
  one space inside each end, and the size fallback for what neither form shows (3.4).
- v2.1 · 2026-10-31 · draft — a fenced code block belongs to the requirement whose span holds it, like a table (1.3).
- v2.2 · 2026-11-03 · draft — memory: F-MEMORY names the two ways the program's memory reaches the limit (the system kills
  a process of its control group at the limit, or its own runtime stops it for lack of memory) and says that the test
  machine caps each runtime's own heap at 90 % of the limit by that runtime's own setting (2.2).
- v2.2 · 2026-11-03 · draft — every section has one class (`common`, `olympiad`, `daemon` or `web`) in the table at the
  top; after the first group's freeze the `common` and `olympiad` sections do not change.
- v2 · 2026-11-04 · draft — the version: the gated text says `Format version: 2`; the draft revisions 2.1 and 2.2 above are
  part of it.
- v2 · 2026-11-04 · draft — classes: the five classes `common`, `olympiad`, `daemon`, `web` and `daemon-web`, given per
  subsection where a section mixes kinds; 2.1's rows follow their sentence's class; the lead text of a section is `common`
  (the table at the top).
- v2 · 2026-11-04 · draft — standard error: F-STDERR and 4.6 say that a run writing more than 256 MiB to standard error is
  stopped and fails (`standard error overflow`); its content is still never compared (2.2, 4.6).
- v2 · 2026-11-04 · draft — the base environment: F-ENV says `TZ` is `UTC`; 4.2 says which variables an `env` line may set
  (2.2, 4.2).
- v2 · 2026-11-04 · draft — processes: the new fixed sentence F-PROCS, at most 512 processes and threads for the program and
  every process it starts, in the same requirement as F-MEMORY (2.1, 2.2, 2.3, 2.4).
- v2 · 2026-11-04 · draft — a header's name is exactly one plain string (4.3, 5.1).
- v2 · 2026-11-04 · draft — a check may not relate a value bound inside an `unordered` group or a `$unordered` array to
  anything outside it (5.5).
- v2 · 2026-11-04 · draft — the verdict: memory fails as F-MEMORY's two causes say, reported `memory limit`; a reply is
  compared status, headers, body, and the first differing part is named; 6.3's stated failure is the `location` header (5.6,
  6.3).
- v2 · 2026-11-04 · draft — layout only, no rule changed: F-TIMING's sentence in 2.5 stands outside the list, under its own
  label; a wrapped line of 5.3 no longer starts with a tag-like code span.
- v2 · 2026-11-04 · draft — layout only, no rule changed: 3.4's example of a two-backtick cell is shown in a three-backtick code
  span, so the spans after it pair as written.
- v2 · 2026-11-04 · draft — the bounds and placement rules the runner holds, now stated: at most 1,000,000 expanded steps, 200
  parts in a string expression, JSON nested at most 200 levels, 500,000 steps to match one entry's fields (4.6); the end
  trigger and `wait-exit` at the top level, no `capture` inside a `repeat` (4.4); a `${:ts}` is a real calendar time (5.3); no `repeat` in an `unordered` group (5.4).
- v2 · 2026-11-04 · draft — the examples of 4.1 and 5.6 use the invented frame of 6.2.
- v2 · 2026-11-04 · draft — from the document gate: F-LOAD's `{duration}` leaves room for the start and the stop inside the
  test bound (2.4); two `header` lines with one name are written as two lines (4.3); a capture on `http` reads the body only,
  and a header's value is bound in the transcript (4.3).
- v2 · 2026-11-04 · draft — from the consultant's reading, seven rules made single-valued: the order of the normalisations and
  what a decimal token is (2.6); `until` occurrences do not overlap, and `quiet` counts an unread byte (4.3); G is kept on the
  runner's own clock (4.4); what a matching step is (4.6); a decimal matches a byte value written as a decimal (5.5).
- v2 · 2026-11-04 · draft — from the final document round: standard output read by a step has the same 256 MiB bound and an
  `overflow stdout` entry; too deep a JSON value does not match and the report says `bad-json` (4.6); a JSON capture is only on
  `http`, and a capture reads the whole reply just read (4.3).
- v2 · 2026-11-04 · gated — the document gate's two rounds (`proceed`, all findings applied) and one consultant reading.
- v2 · 2026-11-05 · amendment before any spec's gate, `Format version: 2` kept — F-OFFLINE says what is scored: the workspace as left, with build output and installed packages removed and the frozen inputs put back, and every other file, including the project files that declare or lock the packages, built as left.
- v2 · 2026-11-05 · amendment — the `Freeze tag` value names the group of tasks frozen together, `spec-frozen-<group>-<yyyymmdd>` (1.2).
- v2 · 2026-11-05 · amendment — the `Freeze rule` row is `this text changes only by a dated amendment from its authors` (1.2).
- v2 · 2026-11-05 · amendment — F-FROZEN names the toolchain pin files by `build.sh`, so a project file that declares or locks the packages is not read as one.
- v2 · 2026-11-05 · amendment — the memory floors of F-LIMITS-O and F-LIMITS-D are written `256 MiB`, the unit the rest of this document uses.
- v2 · 2026-11-05 · gated — the amendment's document gate: three rounds, `proceed`, each with the gate's second vendor out of quota, so its reading came from the consultants; all findings applied.
