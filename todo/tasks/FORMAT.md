# Task spec and sample format

Format version: 1
Gate verdict: not yet gated
Freeze tag: none yet

This document fixes the shape of every task spec in `todo/tasks/` and of every visible sample beside it: the file and its
header, the requirement ids, the sentences every spec carries word for word, the sample files of each kind, and the
scenario grammar of the daemon and web samples with its comparison rules. It is stack-neutral: nothing in it favours or
names an implementation language.

Sections 1 to 6 are the format. Sections 7 to 9 are for the maintainers of this repository and the author of the test
suites: how conformance is checked, how this document and a spec are amended, and the changelog. No text of sections 7
to 9 ever enters a spec.

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
| `Freeze tag` | `none yet` | `` `spec-frozen-<yyyymmdd>` `` |
| `Freeze rule` | `frozen: changes only by a dated amendment before the v2 pre-registration` | unchanged |
| `Amendments` | `none yet` | `A1 (section 8)`, or `A1–A<k> (section 8)` once there are k |

The kind phrases form a closed list; the phrase is the whole value of the `Kind` row:

| Kind | Phrase |
|---|---|
| olympiad | `a program that reads standard input once and writes standard output once` |
| daemon | `a long-running program that serves until it is stopped by SIGTERM` |
| web | `a long-running HTTP server that serves until it is stopped by SIGTERM` |

The kind decides the required sections (1.4), the fixed sentences (section 2) and how the samples are read (section 3).

### 1.3 The requirement sentences (R-ids)

Every requirement is one paragraph that opens with its id in bold, `**R<n>.**`, followed by one or more sentences. The
ids are `R1` … `Rn`: unique, contiguous (no id missing between 1 and n) and, until the freeze, in ascending order through
the document. Held-out cases cite these ids; nothing outside them is tested. Prose outside a requirement — the samples'
lead sentence, an explanation of how a sample is built — explains and never adds a requirement.

**Stable after the freeze: append only.** An id is never renumbered and never reused. An amendment that adds a
requirement gives it the next free id, `R(n+1)`, placed in the section it belongs to, so after an amendment the ids may
stand out of order in the document. An amendment that withdraws a requirement keeps its paragraph as
`**R<n>.** Withdrawn by amendment A<k>.`

Directly before section 1 stands the fixed sentence F-REQ (2.2).

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

- A plan story id, epic id, decision id, test id or any other identifier of the bench's plan.
- For a task that paraphrases a known problem: its id, its title, its site or source, a link, or any mention that the task
  is known. Credit lives in the repository README and in `tasks/<NN-slug>/NOTES.md`.
- A link of any kind.
- A trap, a hint at one, or a language, runtime, framework or library specific. The spec names no implementation
  language; the stack sentence is added only when `STATEMENT.md` is generated.
- Held-out content, and the exact registered limit values (they go to the test runner, never into a spec).
- The `⏎` glyph anywhere but section 6.

## 2. The fixed sentences

### 2.1 How they are used

The sentences below are said identically by every spec of the kinds named. Each is written once here; a spec copies it
word for word into the section given, inside a requirement paragraph wherever the table says "in an R". A conformance
check compares them after collapsing every run of spaces and line breaks to one space, so a spec may wrap them at any
width. A slot in braces is the only part that varies, and only with the values the slot allows.

| Id | Kinds | Where | Slots |
|---|---|---|---|
| F-REQ | all | before section 1, its own paragraph | `{n}`: the highest R-id number |
| F-INVALID | all, unless the task defines errors for every invalid input | section 2 (olympiad) or 3 (daemon, web), in an R | `{ref}`: one R-id, a range `R5–R7`, or a list `R5 and R7` |
| F-ERRORS | all, instead of F-INVALID when the task defines errors for every invalid input | as F-INVALID | `{ref}`, `{eref}`: R-ids |
| F-BYTES-O | olympiad, the default | section 3, in an R | none |
| F-BYTES-O-N | olympiad, instead of F-BYTES-O when the output is normalised | section 3, in an R | `{normalisations}`: from the closed list of 2.5 |
| F-STDERR | all | section 3, in an R | none |
| F-EXIT-O | olympiad | section 3, in an R | none |
| F-PORT | daemon and web that listen on a port | section 2, in an R | none |
| F-READY | daemon, web | section 2, in an R | `{readiness}`: one of two phrases (2.4); `{T}`: whole seconds |
| F-STOP | daemon, web | section 2, in an R | `{G}`: whole seconds |
| F-STDOUT-D | daemon, web | section 3, in an R | none |
| F-BYTES-D | daemon | section 3, in an R | none |
| F-BYTES-W | web | section 3, in an R | none |
| F-JSON | daemon, web whose replies are compared as JSON values | section 3, in an R | `{what}`: the replies it covers, in words |
| F-LIMITS-O | olympiad | section 4, in an R | none |
| F-LIMITS-D | daemon, web | section 4, in an R | none |
| F-MEMORY | all | section 4, in the same R as F-LIMITS-O or F-LIMITS-D | none |
| F-DEADLINES | daemon, web | section 4, in an R | none |
| F-FROZEN | all | section 5, in an R | none |
| F-BUILD | all | section 5, in an R | none |
| F-RUN-O | olympiad | section 5, in an R | none |
| F-RUN-D | daemon, web | section 5, in an R | none |
| F-OFFLINE | all | section 5, in an R | none |
| F-SAMPLES-O | olympiad | section 6, first paragraph | `{k}`: the number of samples, in words or digits |
| F-SAMPLES-D | daemon, web | section 6, first paragraph, followed by the reading block (5.6) | `{k}` as above |
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
Standard error is not compared and may hold anything; it is recorded, not judged.
```

F-MEMORY:

```text
Memory is judged only on a test machine that can read the peak memory of the test's control group (cgroup `memory.peak`). On a machine that cannot, peak memory is recorded for information only, and no test passes or fails on memory.
```

F-FROZEN, the frozen list:

```text
The workspace holds `STATEMENT.md`, `samples/`, `build.sh`, `run.sh`, `PACKAGES.md` and the toolchain pin files of the stack. These are frozen inputs: do not change, rename or remove them. The program's sources go where `build.sh` and `run.sh` expect them; the two scripts say where.
```

F-BUILD, the build bound:

```text
`bash build.sh`, run in the workspace directory, builds the program with no network access and with no package outside the set listed in `PACKAGES.md`. It must exit 0 within 600 seconds. It runs once before the tests; a build that does not exit 0 fails every test.
```

F-OFFLINE:

```text
The tests run offline, on a pristine copy of the workspace, after the build. The program must not depend on the network, on files outside the workspace, or on anything written during an earlier run.
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

F-LIMITS-O, the limits as floors and the registration rule (F-MEMORY follows it in the same requirement):

```text
Each test runs the program once, in a fresh process, under a limit on wall-clock time and a limit on memory. The exact values are registered when the hidden suite is calibrated against reference solutions in both stacks, before any session, and are given to the test runner, not to the program's author. The registered values are never below these floors: **1 second** of wall-clock time per run and **256 MB** of memory. (The registration rule: at least three times the wall time of the slower reference, and at least twice the peak memory of the larger reference.)
```

F-BYTES-O, byte-exact output, the default:

```text
The output is compared with the expected output byte for byte. Nothing is normalized: a missing or extra line feed, a trailing space or a carriage return makes the test fail.
```

F-EXIT-O:

```text
The process exits with status 0 after writing its output. Any other exit status fails the test, whatever was written.
```

F-RUN-O:

```text
`bash run.sh`, run in the workspace directory with no arguments, starts the built program once: the test's input is its standard input, its standard output is collected whole, its exit status is read. One fresh process per test; nothing carries over from one run to the next.
```

F-SAMPLES-O:

```text
The same {k} cases are in `samples/`, one input file and one expected-output file per case (`samples/<id>.in`, `samples/<id>.out`). The bytes below are exact; `⏎` marks the single line feed that ends each line.
```

### 2.4 Daemon and web

F-PORT:

```text
The environment variable `PORT` holds a TCP port number; the program listens on it, and the tests connect to it at 127.0.0.1.
```

F-READY. `{readiness}` is one of `it accepts a TCP connection on that port` or `it has written the line {line} to
standard output`, where `{line}` is the line in a code span; `{T}` is whole seconds.

```text
The program is ready when {readiness}. A test waits at most {T} seconds for readiness; a program that is not ready by then fails that test.
```

F-STOP. `{G}` is the grace period in whole seconds.

```text
A test ends by sending the signal SIGTERM to the program. The program must then exit with status 0 within {G} seconds, leaving no process it started still running. A program still running after {G} seconds is killed and fails that test, and so does one that exits with another status or leaves a process running.
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

F-JSON, the one other body comparison a spec may name:

```text
The body of {what} is compared as a JSON value: the members of an object may come in any order, numbers are compared by their value, and whitespace outside strings is ignored; arrays keep their order unless a requirement says otherwise.
```

A field whose value the program chooses (an id, a token, a timestamp) is named in a requirement with the form it must
have — for example "the `id` member is a UUID; it is compared by its form, and every later reply that names the same
order carries the same value". The scenario files express that with the normalised fields of 5.3.

F-LIMITS-D (F-MEMORY follows it in the same requirement):

```text
Each test starts the program once, in a fresh process, and drives one scenario against it under a bound on the program's memory. The exact bound is registered when the hidden suite is calibrated against reference solutions in both stacks, before any session, and is given to the test runner, not to the program's author. The registered bound is never below a floor of **256 MB**. (The registration rule: at least twice the peak memory of the larger reference under the task's load scenario.)
```

F-DEADLINES:

```text
Every step of a scenario that waits — for readiness, for a reply, for the exit — has a deadline, and a step that does not complete by its deadline fails the test. The visible samples state their deadlines in their files; the deadlines of the hidden scenarios are fixed when the suite is calibrated, so that the reference solutions in both stacks meet each of them with margin.
```

F-RUN-D:

```text
`bash run.sh`, run in the workspace directory with no arguments, starts the built program once per test, with the environment section 2 describes. One fresh process per test; nothing carries over from one test to the next.
```

F-SAMPLES-D (the reading block of 5.6 follows it, verbatim):

```text
The same {k} scenarios are in `samples/`, one scenario file and one expected-transcript file per scenario (`samples/<id>.in`, `samples/<id>.out`), shown below exactly as stored. How to read them:
```

### 2.5 Olympiad output: the closed list of normalisations

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

No other normalisation exists. The test runner takes the task's normalisation from its registered entry, which is
copied from this sentence; the sample files stay exact bytes either way.

## 3. Visible samples

### 3.1 Files and ids

A task's visible samples live in `tasks/<NN-slug>/samples/` and are copied unchanged into `samples/` of both
workspaces. Each sample is a pair of files with the same id: `samples/<id>.in` and `samples/<id>.out`. Visible ids are
`s01` … `s99`, numbered from `s01` without gaps; `samples/` holds these pairs and nothing else. Samples are small and
clean: they reach no trap.

The held-out part of the suite uses the same pairs, with ids `h001` … `h999`, plus one manifest, `cases.txt`: one line
per case, `<id> <R-id> [<R-id> …]`, naming every requirement the case asserts (comment lines start with `#`). Every case
appears once; every R-id it names exists in the frozen spec. The visible samples need no manifest.

### 3.2 Olympiad samples

`<id>.in` is the program's standard input, byte for byte; `<id>.out` is its expected standard output, byte for byte.
Neither file has a header, a trailer or an encoding of its own: they are the raw bytes. Either may be empty. The
comparison is F-BYTES-O's, or F-BYTES-O-N's when the spec names a normalisation.

### 3.3 Daemon and web samples: the scenario pair

`<id>.in` is a **scenario script** (section 4): what the test runner does — start the program, open connections, send,
read, wait, stop. `<id>.out` is the **expected transcript** (section 5): what the runner must have observed, one entry per
observation, with the comparison rules of section 5. The runner executes the script, writes the actual transcript in the
same syntax, and compares the two. Both files are UTF-8 text with line feeds; the program's bytes inside them are written
as strings with escapes, so any byte can be expressed.

### 3.4 Showing samples in the spec

Section 6 of a spec shows every visible sample, and only those, with the same bytes as the files.

- **Olympiad:** F-SAMPLES-O, then a table `| Id | Standard input | Expected standard output |` with one row per sample,
  each cell one code span. Bytes are shown as they are, except that each line feed is shown as `⏎`; a cell may show only
  printable ASCII and the space. A sample that cannot be shown that way, or is longer than 200 bytes, shows
  `<size> bytes, see samples/<id>.in` (or `.out`) in its cell instead. Prose after the table may explain how a sample is
  built.
- **Daemon and web:** F-SAMPLES-D, the reading block of 5.6 verbatim, then for each sample a `### <id>` sub-section with
  two fenced code blocks, the first headed by the line `` `samples/<id>.in` `` and the second by `` `samples/<id>.out` ``,
  each holding the file's text exactly (the file's final line feed is the line feed before the closing fence).

## 4. The scenario script (`.in`)

The grammar is small on purpose: a runner written with a language's standard library alone implements it, with no
parser library. (The bench's runner is Python 3 with its standard library only; the regular expressions of `capture …
regex` use the syntax of Python's `re` module on bytes.)

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
  `"$16384\r\n" "v"*16384 "\r\n"`.
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
scenario 1
default within <duration>
[env <NAME> <str> | file <name> <str>]…
start ready port within <duration>          (or)
start ready stdout <str> within <duration>
```

- `scenario 1` is the first statement: the grammar version, which is this document's format version.
- `default within <duration>` is the second: the deadline of every waiting step that names none.
- `env <NAME> <str>` sets one environment variable of the program, beyond the runner's fixed base environment, which is
  the same for both stacks. `PORT` is reserved: the runner chooses a free port and sets it.
- `file <name> <str>` writes the bytes into a new file inside `.scenario/` of the test's copy of the workspace and binds
  the variable `name` to its absolute path, for use in a later `env` (a fixture the program reads). `env` and `file`
  lines may come in any order, each variable bound before its use.
- Exactly one `start`, after these: it runs `bash run.sh` in the workspace copy and waits for readiness — `ready port`: a
  TCP connection to 127.0.0.1 on `PORT` is accepted (the runner retries every 50 ms and closes that probe connection);
  `ready stdout <str>`: the program writes that line (the bytes of `<str>`, then a line feed) to standard output, every
  earlier line being discarded. It records the entry `ready`.

The program's standard input is a pipe that stays open and receives only what the scenario sends to `stdin`. Its
standard output, when no step reads `stdout`, and its standard error are drained and kept with the test's record, so the
program never blocks on a full pipe.

### 4.3 Steps

Options after a step's fixed words may come in any order. `[within <duration>]` overrides the default deadline of a
waiting step, measured from the moment the step starts. The *entry* column names what the step adds to the transcript.

| Step | What the runner does | Entry |
|---|---|---|
| `connect <conn> tcp\|http [rcvbuf <bytes>] [within <d>]` | opens a TCP connection to 127.0.0.1 on `PORT` and names it; `http` makes it an HTTP/1.1 client connection; `rcvbuf` sets the client's receive buffer (a slow reader's backpressure) | none |
| `send <conn> <str> [split <o1>[,<o2>…] pause <d>] [within <d>]` | writes the bytes in one write; with `split`, writes them in pieces cut at the given byte offsets (strictly increasing, inside the string), sleeping `pause` between pieces; completes when every byte is written | none |
| `request <conn> <METHOD> <str>` | starts an HTTP request on an `http` connection, the string being the target (it begins with `/`); the `header <str> <str>` lines after it add headers in order, and at most one `body <str>` or `body json <JSON text>` line gives the body; the request is written when the next other statement is reached | none |
| `recv <conn> <mode> [chunk <bytes> pause <d>] [capture …]… [within <d>]` | reads one reply (modes below); with `chunk`, reads at most that many bytes per read and sleeps `pause` between reads | one `recv` or `reply` entry; `lines <k>` adds k entries |
| `expect <conn> quiet for <d>` | checks that no byte arrives on the connection for that long | `quiet <conn>` |
| `expect <conn> closed [within <d>]` | checks that the program closes the connection: end of stream with no byte before it | `closed <conn>` |
| `close <conn>` | closes the connection; `close stdin` ends the program's standard input | none |
| `shutdown <conn>` | closes the sending half of a TCP connection; reading goes on | none |
| `sleep <d>` | waits | none |
| `signal TERM` | sends SIGTERM to the program | none |
| `wait-exit [within <d>]` | waits for the program to exit; at the deadline the runner kills it and its process group | `exit <status>` |

**Request serialisation.** The runner writes `<METHOD> <target> HTTP/1.1`, then `Host: 127.0.0.1:<PORT>`, then the
`header` lines in order as `<name>: <value>`, then `Content-Length: <n>` when there is a body and no `header` gives it,
then an empty line and the body; every line ends with CR LF. `body json <JSON text>` sends the JSON text's bytes as
written, after variable substitution, and adds no header. Nothing else is added.

**`recv` modes.** On a `tcp` connection or `stdout`, the mode says where the reply ends:

| Mode | The reply is |
|---|---|
| `bytes <n>` | exactly the next n bytes |
| `until <str> [count <k>]` | every byte up to and including the k-th next occurrence of the delimiter (k = 1 by default) |
| `lines <k>` | the next k lines, each ending with a line feed; each line is its own entry |
| `eof` | every byte up to the end of the stream |

Bytes that arrived beyond the reply stay buffered for the next `recv` on that connection, so pipelined replies are read
one step at a time. On an `http` connection `recv` takes no mode: the reply is one whole HTTP response — status line,
headers, and a body delimited by `Content-Length`, by chunked transfer coding (which the runner decodes) or by the end of
the connection; an interim `1xx` response is skipped.

**Captures** bind a value of the reply just read to a variable for later steps and for the transcript:

- `capture <name> json <path> [as <type>]` — on an `http` reply, parses the body as JSON and takes the value at the path;
- `capture <name> regex <str> [as <type>]` — searches the reply's bytes (on `http`: its body) with the regular expression
  and takes group 1, or the whole match when the expression has no group.

`as <type>` checks the value against a type of 5.3. A capture that finds nothing, or a value of another form, ends the
scenario with the entry `capture-failed <name>`. A value captured from JSON keeps its JSON type: substituted into a
string, a JSON string gives its UTF-8 bytes and a number gives its digits as written in the reply.

**Failure.** A waiting step that misses its deadline records what it saw (5.1) and the scenario goes on, so the report
shows every later entry too. Four failures end the scenario at once: the program is not ready in time, a connection is
refused, the program exits before the last step, and a capture fails. The runner then stops the program (SIGTERM, then
SIGKILL after the deadline of the last `wait-exit`, or 5 s when there is none).

### 4.4 Groups

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

The scenario's last statement is `wait-exit`. A scenario that tests the stop sends `signal TERM` just before it; one
where the program ends by itself (for example after `close stdin`, where the spec says so) waits without it.

### 4.5 Keywords

The complete list of the script's words: `scenario`, `default`, `within`, `env`, `file`, `start`, `ready`, `port`,
`stdout`, `stdin`, `connect`, `tcp`, `http`, `rcvbuf`, `send`, `split`, `pause`, `request`, `header`, `body`, `json`,
`recv`, `bytes`, `until`, `count`, `lines`, `eof`, `chunk`, `capture`, `regex`, `as`, `expect`, `quiet`, `for`, `closed`,
`close`, `shutdown`, `sleep`, `together`, `repeat`, `end`, `signal`, `TERM`, `wait-exit`.

## 5. The expected transcript (`.out`)

### 5.1 Entries

The transcript's first statement is `transcript 1`. Then, one per line (an HTTP reply on several), the entries in the
order the script produces them; blank lines and `#` comment lines are allowed, indentation means nothing, and `repeat
<count> <name> … end` stands for its body written count times, as in the script.

| Entry | Produced by | Holds |
|---|---|---|
| `ready` | `start` | — |
| `recv <conn> <str>` | `recv` on `tcp` or `stdout` | the reply's bytes |
| `reply <conn> <status>` followed by `header <str> <str>` lines and exactly one `body …` line | `recv` on `http` | the status code, the headers to compare, the body: `body <str>`, `body json <JSON text>` or `body empty` |
| `quiet <conn>` | `expect … quiet` | — |
| `closed <conn>` | `expect … closed` | — |
| `exit <status>` | `wait-exit` | the exit status |

The actual transcript, written by the runner, uses the same entries and, where a step failed, one of these, which never
match an expected entry:

| Actual-only entry | Meaning |
|---|---|
| `not-ready` | the program was not ready by the deadline |
| `refused <conn>` | the connection was refused |
| `timeout <conn> <str>` | the reply was not complete by the deadline; the bytes received so far |
| `eof <conn> <str>` | the stream ended before the reply was complete; the bytes received so far |
| `noise <conn> <str>` | bytes arrived where `quiet` or `closed` was expected |
| `open <conn>` | the connection was still open at the deadline of `closed` |
| `bad-reply <conn> <str>` | the bytes are not an HTTP response |
| `early-exit <status>` | the program exited before the last step |
| `exit-timeout` | the program was still running at the deadline of `wait-exit` and was killed |
| `exit signal <NAME>` | the program ended by a signal |
| `capture-failed <name>` | a capture found nothing of its form |

The runner writes strings canonically: printable ASCII as it is, `"` and `\` escaped, `$` escaped where `{` follows
it, CR, LF, tab and NUL as `\r` `\n` `\t` `\0`, every other byte as `\xHH` (lower case). Long runs are written as they are; the runner never shortens a
transcript it keeps, and abbreviates only in the report (5.5).

The complete list of the transcript's words: `transcript`, `ready`, `recv`, `reply`, `header`, `body`, `json`, `empty`,
`quiet`, `closed`, `exit`, `repeat`, `unordered`, `end`, and, in actual transcripts only, `not-ready`, `refused`,
`timeout`, `eof`, `noise`, `open`, `bad-reply`, `early-exit`, `exit-timeout`, `signal` and `capture-failed`.

### 5.2 Comparison

The actual transcript, expanded, must hold the same number of entries as the expected one, expanded, and each actual
entry must match the expected entry at its position (or its group, 5.4).

- **Byte-exact by default.** A `recv` entry matches when its bytes equal the expected bytes, every variable substituted,
  every normalised field (5.3) matched by its form. `exit`, `ready`, `quiet` and `closed` match exactly.
- **An HTTP `reply`** matches when the status code is equal; when each `header` line's header is present (names compared
  without regard to case) and its value — the values of every occurrence joined by `, ` in order, each without
  surrounding spaces and tabs — equals the expected bytes; and when the body matches. Headers that the entry does not
  list are not compared. `body <str>` compares the decoded body byte for byte; `body empty` requires a body of zero
  bytes; `body json` compares JSON values.
- **JSON values** (`body json`): the actual body must be one valid JSON value, with no member name repeated in one object.
  Objects match when they have the same member names and each member's value matches, in any order; arrays match element
  by element, in order; strings match by their characters; numbers match by their decimal value, so `2` matches `2.0`;
  `true`, `false` and `null` match themselves; whitespace outside strings means nothing. Inside the expected JSON, a
  string whose whole content is `${name}` matches a value equal to the variable's (a JSON value keeps its type; a
  captured byte string matches a JSON string with those characters), and a string whose whole content is a normalised
  field matches by that field's rule. A literal string of exactly that shape cannot be expected with `body json`; use
  `body <str>`.

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
| `${:ts}` | a UTC time `YYYY-MM-DDThh:mm:ss`, optionally `.` and 1 to 9 digits, then `Z` |
| `${:ts~<duration>}` | a `${:ts}` within that duration of the runner's UTC clock at the moment the entry was recorded |
| `${:num <decimal> <tolerance>}` | a decimal number (`-?[0-9]+(\.[0-9]+)?`; in JSON, a number) within the tolerance of the decimal |

Except in JSON, a field matches the shortest run of bytes that lets the rest of the entry match; the entry's own length
is fixed by the script's `recv` mode, so a field never decides how much is read. The same types serve `capture … as
<type>` (without the window and the tolerance).

### 5.4 Order-insensitive sets

Order is fixed everywhere except where the transcript marks it, and the transcript marks it only where the spec says the
order is not defined:

- **Entries.** `unordered … end` around two or more entries: they match the same number of actual entries at that
  position in any order, when some one-to-one assignment matches every pair. Inside such a group only, an entry's
  connection may be `*`, matching any connection — for the replies of concurrent clients when the spec does not say which
  client gets which reply. Groups do not nest.
- **JSON arrays.** In expected JSON, the object `{"$unordered": [ … ]}` stands for an array whose elements match the
  listed ones in any order, under the same one-to-one rule — for a listing whose order the spec leaves undefined. The
  member name `$unordered` is reserved for this.

### 5.5 The verdict and the mismatch report

A scenario passes when every step ran, the two transcripts match (5.2 to 5.4), the program left no process running after
its exit, and its memory stayed within the registered bound where the machine certifies memory (F-MEMORY). The first two
are this format's; the last two are the runner's own lifecycle checks, reported on their own lines.

The runner reports one line per scenario, `PASS <id> — <k> entries`, or `FAIL <id> — <reason>` followed by the first
mismatch:

```text
FAIL s02 — entry 4 of 9 differs
  step    s02.in:14   recv c1 until "\r\n" count 2
  expect  s02.out:6   recv c1 "$5\r\nmango\r\n"
  actual              recv c1 "$5\r\nmelon\r\n"
  first difference at byte 5
```

For a reply it names the part (`status`, `header location`, `body`) and, in a JSON body, the path of the first
difference (`body $.qty: expected 2, actual 3`); for a group, the expected entry no assignment could match; for a count,
`expected 9 entries, actual 6`, with the step that ended the scenario; for the lifecycle, `stop: still running 5 s after
SIGTERM, killed`, `leftover: 1 process still running after the exit` or `memory: peak above the registered bound`.
Strings longer than 120 bytes are shown as their first 60 and last 40 bytes, with the count left out between. The full
actual transcript is kept with the test's record.

### 5.6 The scenario reading block

Every daemon and web spec copies this block, word for word, after F-SAMPLES-D in section 6, so that the samples can be
read from the spec alone:

```text
A scenario file (`.in`) is a list of steps, one per line, run in order; a transcript file (`.out`) lists, in the same order, what the program must have sent back. Lines starting with `#` are comments; indentation means nothing. Strings are in double quotes with the escapes `\r` `\n` `\t` `\0` `\\` `\"` `\$` and `\xHH`; adjacent strings are joined, and `"ab"*3` repeats one. `${name}` is a value bound earlier; in a transcript, `${:type}` stands for any value of that type (`int`, `uint`, `hex`, `uuid`, `token`, or `ts`, a UTC time such as `2026-01-02T03:04:05Z`). Durations are written `250ms` or `2s`.
The steps: `env` and `file` set up the program's environment, and `start` runs `bash run.sh` and waits until the program is ready; `connect` opens a TCP or HTTP connection to it, and `stdin` and `stdout` are its standard input and output; `send` writes bytes (with `split`, in pieces cut at the given byte offsets, with a `pause` between them); `request` sends one HTTP request made of the `header` and `body` lines after it; `recv` reads one reply — on TCP or `stdout` a number of `bytes`, everything `until` a delimiter, a number of `lines`, or up to `eof`; on HTTP one whole response — optionally `chunk` bytes at a time with a `pause`, and optionally binding a value of it with `capture <name> json <path>` or `capture <name> regex "<pattern>"`; `expect <connection> quiet for <duration>` checks that nothing arrives, and `expect <connection> closed` that the program closed the connection; `close` and `shutdown` close a connection or its sending half; `sleep` waits; `together` … `end` releases the sends and requests inside it at the same instant, one per connection; `repeat <n> <name>` … `end` runs its lines n times with `${name}` = 0, 1, …; `signal TERM` sends SIGTERM and `wait-exit` waits for the exit. Every step that waits has a deadline: its `within <duration>`, or the file's `default within`.
In the transcript, `ready` is the readiness; `recv <connection> "<bytes>"` is one reply read on TCP or `stdout`; `reply <connection> <status>` with its `header` and `body` lines is one HTTP response, of which only the listed headers are compared, `body json` being compared as a JSON value and `body empty` meaning no body; `quiet` and `closed` are the two checks; `exit <status>` is the exit. Entries between `unordered` and `end` may come in any order, and in a JSON body `{"$unordered": [ … ]}` is an array whose elements may come in any order. Everything else is compared byte for byte.
```

## 6. Worked examples

**These three examples are illustrative.** They show the format working end to end; they are not tasks, and the toy
programs of 6.2 and 6.3 fix nothing about any task of this repository. Each shows both files, complete.

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
byte (F-BYTES-O): a missing final line feed or a trailing space fails the case.

### 6.2 Daemon — a toy key-value store with length-prefixed framing

The toy: three commands, `PING` → `+PONG`, `SET <key> <value>` → `+OK`, `GET <key>` → the value as a bulk string, each an
array of length-prefixed strings. The scenario covers a SET split across two partial writes (with a check that nothing comes
back early), two commands pipelined in one write, a write cut at offsets by `split`, and a slow reader that must not stop
the server from serving another client.

`x2.in`:

```text
scenario 1
# Illustrative only: a toy key-value store with PING, SET and GET in a
# length-prefixed framing. It is not a task of this repository and fixes nothing about one.
default within 2s
start ready port within 10s
connect c1 tcp

# 1. One SET split across two partial writes: no reply may come before the
#    frame is complete (the first part is 30 bytes and ends inside "mango").
send c1 "*3\r\n$3\r\nSET\r\n$5\r\nfruit\r\n$5\r\nma"
expect c1 quiet for 200ms
send c1 "ngo\r\n"
recv c1 until "\r\n"

# 2. Two commands pipelined in one write; the replies come back in order.
send c1 "*2\r\n$3\r\nGET\r\n$5\r\nfruit\r\n*1\r\n$4\r\nPING\r\n"
recv c1 until "\r\n" count 2
recv c1 until "\r\n"

# 3. The same GET written in three pieces, cut at byte offsets 1 and 9.
send c1 "*2\r\n$3\r\nGET\r\n$5\r\nfruit\r\n" split 1,9 pause 50ms
recv c1 until "\r\n" count 2

# 4. A slow reader: 'lazy' pipelines 64 GETs of a 16 KiB value in one write
#    and reads nothing; meanwhile c1 is still served. Then 'lazy' reads its
#    64 replies of 16394 bytes each, 4096 bytes at a time.
send c1 "*3\r\n$3\r\nSET\r\n$3\r\nbig\r\n$16384\r\n" "v"*16384 "\r\n"
recv c1 until "\r\n"
connect lazy tcp rcvbuf 4096
send lazy "*2\r\n$3\r\nGET\r\n$3\r\nbig\r\n"*64
sleep 200ms
send c1 "*1\r\n$4\r\nPING\r\n"
recv c1 until "\r\n" within 500ms
repeat 64 i
  recv lazy bytes 16394 chunk 4096 pause 2ms within 5s
end

# 5. The stop.
signal TERM
wait-exit within 5s
```

`x2.out`:

```text
transcript 1
ready
# 1.
quiet c1
recv c1 "+OK\r\n"
# 2.
recv c1 "$5\r\nmango\r\n"
recv c1 "+PONG\r\n"
# 3.
recv c1 "$5\r\nmango\r\n"
# 4.
recv c1 "+OK\r\n"
recv c1 "+PONG\r\n"
repeat 64 i
  recv lazy "$16384\r\n" "v"*16384 "\r\n"
end
# 5.
exit 0
```

A server that answers the first part of the SET early fails at entry 2 with `noise c1 "…"`; one that blocks every
client while `lazy` reads nothing fails at entry 8 with `timeout c1 ""`.

### 6.3 Web — two clients racing on one idempotency key

The toy: `POST /orders` with an `Idempotency-Key` header creates one order and answers `201` with a `Location` header and
the order as JSON; a request that repeats a key gets the reply that key's first request got — the same status, the same
body — even when the two requests race; `GET /orders/<id>` answers `200` with the order; `GET /orders` lists every order
in no defined order. Ids are UUIDs; `at` is the creation time.

`x3.in`:

```text
scenario 1
# Illustrative only: a toy order endpoint with idempotency keys. It is not a
# task of this repository and fixes nothing about one.
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
transcript 1
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

A toy server that let both racing requests create an order fails at entry 3:

```text
FAIL x3 — entry 3 of 7 differs
  step    x3.in:21   recv b
  expect  x3.out:7   reply b 201
  actual             reply b 201
  body $.id: expected "${order}" = "0b6f…", actual "5d2a…"
```

## 7. Conformance (for the maintainers)

A spec conforms to this format when a checker finds all of the following. The checker is the spec conformance check, a
tool of the bench plan's stories for the visible samples and for this document, used by every spec's gate. **It is to be
built later; nothing runs it yet.** Until it exists, the same list is checked by hand and the result recorded with the spec's
gate.

1. **The file.** The name matches 1.1; the title's `NN` equals the file's.
2. **The header.** The five rows of 1.2, in order, each value either its placeholder or its later form; the `Kind` value
   one of the three phrases; the `Freeze rule` sentence exact.
3. **The sections.** Exactly the eight `##` headings of the kind (1.4), in order; section 8 is `None yet.` or a list of
   amendment items of the form 1.4 gives, numbered `A1` … `Ak` without gaps, matching the `Amendments` row.
4. **The R-ids.** Every requirement paragraph opens with `**R<n>.**`; the ids are unique and are exactly `R1` … `Rn`;
   before the freeze they ascend through the document; F-REQ's `{n}` equals n; every id the amendments name exists.
5. **The fixed sentences.** Every sentence that 2.1 requires for the kind is present, in its section and, where the table
   says so, inside a requirement paragraph, verbatim after whitespace is collapsed, with every slot filled by an allowed
   value; F-BYTES-O-N's items come from 2.5 only; exactly one of F-INVALID and F-ERRORS; in daemon and web specs, the
   reading block of 5.6 follows F-SAMPLES-D.
6. **The samples.** `tasks/<NN-slug>/samples/` holds exactly the pairs `s01` … `s<k>` that section 6 shows, nothing else;
   the olympiad table's cells (with `⏎` read as a line feed) or the daemon and web code blocks equal the files' bytes;
   every daemon and web pair parses under sections 4 and 5 — the head, the steps, the closed keyword list, strings and
   escapes, every variable bound before its use and bound once, `together` and `repeat` well formed, the last statement
   `wait-exit`, and the transcript's entries corresponding one to one, after expansion, to the entries the script
   produces.
7. **No forbidden needle.** No hit, case-insensitive, of any needle of a list the spec author supplies with the check (for
   a known task, its id, title and site names; for every task, anything else the test-design notes say a spec must not
   carry). The list is kept outside git with those notes, since it may name what it guards. Always added to it: the
   bench plan's identifiers (story, epic, decision, gate and test ids, by pattern), web links (the http and https schemes), and the
   names of the implementation languages, runtimes and web frameworks of the series.
8. **The `⏎` glyph** occurs only in section 6.

The same parser checks this document's own examples: the files of 6.2 and 6.3 parse, and the runner, against one stub
program per kind, gives the verdicts the examples state.

## 8. Amendments (for the maintainers)

**This document.** Before the freeze, a change is a revision of version 1, recorded in the changelog (section 9). After
the freeze — the tag `spec-frozen-<yyyymmdd>` that also freezes this document, whose sha256 is registered with the specs'
— a change is a **dated amendment**: it raises `Format version` by one, adds a changelog line
`- v<k> · <YYYY-MM-DD> · amendment · touches <NN, NN | all daemon and web tasks | all> — <what changed and why>`, and
names every task it touches: every task whose spec, samples or held-out cases read differently under the change. A
change to the scenario grammar or to the comparison rules touches every daemon and web task; a change to a fixed sentence
touches every spec that carries it. Each touched spec takes an amendment of its own on the same date (1.4), bringing it to
the new version.

**What an amendment costs.** An amendment of this document, like an amendment of a spec, **voids the registered limits of
every task it touches** (the bench plan's calibration-order check): those tasks are validated jointly, calibrated
and registered again, for both stacks, before the pre-registration. After the pre-registration, neither this document
nor a spec changes.

**A spec.** After its freeze tag a spec changes only by a dated amendment applied to both stacks before the
pre-registration: the item in section 8, the `Amendments` row updated, R-ids appended or withdrawn as 1.3 says, never
renumbered.

## 9. Changelog

- v1 · 2026-10-31 · draft — the first version, written from the pilot spec of task 01: the spec file and header, R-ids,
  the fixed sentences, the sample pair of each kind, the scenario grammar and its comparison rules, three illustrative
  examples, conformance and amendment rules. Not yet gated.
