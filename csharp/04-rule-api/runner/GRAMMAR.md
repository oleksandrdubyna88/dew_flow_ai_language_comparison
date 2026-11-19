# Scenario and sample grammar (from FORMAT.md v2, sha256 6c3546bb03cdc70df32a6795b68c52d5c03d0c4efbb968824b1c177cc8cb5d47)

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

