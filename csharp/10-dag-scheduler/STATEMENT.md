# Task 10 — DAG job scheduler

| | |
|---|---|
| Kind | a long-running program that serves until it is stopped by SIGTERM |
| Gate verdict | `proceed` · 2026-11-17 |
| Freeze tag | `spec-frozen-B-20261118` |
| Freeze rule | this text changes only by a dated amendment from its authors |
| Amendments | none yet |

Every numbered sentence (**R1** … **R38**) is a requirement. Hidden tests refer to these numbers; nothing outside them
is tested.

## 1. What the program does

**R1.** The program is a job scheduler served over TCP. Clients submit directed acyclic graphs (DAGs) of jobs. The
program runs each job once every job it depends on has succeeded, runs at most k jobs at a time across all the DAGs it
holds, retries a job that fails after a bounded backoff, ends an attempt that exceeds the job's timeout, skips the
dependents of a job that has finally failed, cancels a DAG on request, and reports the state of every job and every
attempt of a DAG. Jobs are virtual: a job does nothing but take time and end as its submission says (section 3.3), so
the program's behaviour is a function of the requests it receives and of the clock of section 3.4.

**R2.** The program starts with no DAG and with the clock at 0, keeps its state only in memory, reads and writes no
file, and keeps nothing across a stop.

## 2. Start, readiness and stop

**R3.** The environment variable `PORT` holds a TCP port number; the program listens on it, and the tests connect to it
at 127.0.0.1. The tests may set the variable `SLOTS` (R22) and the variable `CLOCK` (R20); they set no other variable
beyond the base environment of R36. The program is started with no arguments, and its standard input receives nothing.

**R4.** The program is ready when it accepts a TCP connection on that port. A test waits at most 10 seconds for
readiness; a program that is not ready by then fails that test.

**R5.** The program keeps serving whatever any one client does. A client may connect and disconnect without sending a
byte (the readiness probe does exactly that), may disconnect in the middle of a request, or may disconnect while replies
to it are still unsent; in each case the program drops that connection and the incomplete request at its end, if any;
the disconnect itself changes neither the DAGs nor the clock, every other connection continues, and the program does
not exit. A client may instead shut down only its sending side (a half-close) after its last complete request and go on
reading: that is not a disconnect. The program processes every complete request that arrived before the half-close,
writes each reply as R8 says, and may close the connection after the last one.

**R6.** A test ends by sending the signal SIGTERM to the program. The program must then exit with status 0 within 5
seconds, leaving no process it started still running. A program still running after 5 seconds is killed and fails that
test, and so does one that exits with another status or leaves a process running.

**R7.** On SIGTERM the program, at the current time t, cancels every job of every DAG that is not in a final state, all at
once: each such job changes as a `cancel` request at t changes it (R14) — a running job's attempt ends at t with the
result `cancelled`, and a job that is pending, ready or in backoff is cancelled with no new attempt — and no scheduling
step runs, so no attempt starts at the stop. Then it writes to standard output one line per DAG it holds, in the
order of submission: the report object of the DAG (R26) exactly as a `report` request would then answer it, without the
`{"ok":true,"report":` wrapper, followed by one line feed. Then it exits. The tests read these lines. The program does
not wait for a client to disconnect and may close every connection at once. The tests send SIGTERM only after they have
read every reply they expect; in real mode (R20) they also keep the margin of R21 around the signal.

## 3. Interface

### 3.1 Lines, requests and replies

**R8.** On one connection a client sends requests, each one line: the text of R9, then one line feed (0x0A); the line
holds no other line feed and no carriage return. The lines may arrive in any number of pieces, cut at any byte, over any
span of time, and several requests may arrive in one piece. The program processes a request only once its line feed has
arrived, processes the requests of one connection in the order they arrive, and answers each request with exactly one
reply line on the same connection, in the order of the requests, each reply complete before the next begins; it writes
nothing else to a connection. A client may keep a connection open for any number of requests or open a new one per
request; a reply is never withheld until more input arrives.

**R9.** A request is a JSON object whose `cmd` member names the request (section 3.2) and whose other members are the
ones its row names, every one present and no other, in any order, with JSON whitespace other than a line feed allowed
between tokens and none required. An **id** (a DAG's or a job's) is a JSON string of 1 to 64 characters, each one of
`A`–`Z`, `a`–`z`, `0`–`9`, `_`, `-` and `.`, written as those characters with no escape sequence. A **count** is an
integer from 0 to 1 000 000 000 000 (10^12), written in decimal with no sign, no fraction, no exponent and no leading
zero. A request line is at most 1 048 576 bytes long, its line feed counted.

**R10.** A reply is exactly the bytes its row below gives, then one line feed: no space anywhere, no other member, the
members in the given order, integers written in decimal with no sign and no leading zero, `true`, `false` and `null` as
JSON writes them, ids as their characters.

| Reply | Bytes |
|---|---|
| done | `{"ok":true}` |
| time | `{"ok":true,"now":<t>}` |
| report | `{"ok":true,"report":<report object, R26>}` |
| error | `{"ok":false,"error":"<code>"}` |

The error codes form a closed list: `duplicate-dag`, `duplicate-job`, `unknown-dependency`, `cycle` (R12) and
`unknown-dag` (R14, R15). A request that is answered with an error changes nothing.

### 3.2 Requests

**R11.** The four requests, their members besides `cmd`, and their replies:

| `cmd` | Other members | Reply |
|---|---|---|
| `submit` | `dag` (an id), `backoff` (an object with the counts `base` and `cap`), `jobs` (an array of 1 to 1 000 job objects, R16) | done, or error |
| `tick` | `ms` (a count, at least 1) | time |
| `cancel` | `dag` (an id) | done, or error |
| `report` | `dag` (an id) | report, or error |

**R12.** `submit` offers a DAG under the id `dag`. The program checks, in this order, and answers the first error that
applies: `duplicate-dag` when a DAG with that id was accepted earlier; `duplicate-job` when two job objects carry the
same `id`; `unknown-dependency` when a job's `deps` names an id that is no job of this submission; `cycle` when some
job depends on itself, directly or through a chain of dependencies. A rejected submission leaves no trace. Otherwise
the DAG is accepted with the next sequence number (the first accepted DAG has 1), every job of it is in the state R19
gives it at the current time t, the scheduling step of R23 runs at t, and the reply is done. A dependency that a job's
`deps` names more than once counts as one dependency.

**R13.** `tick` advances the clock (R20, virtual mode) from the current time t to t + `ms`: every instant from t + 1 to
t + `ms` at which something is due — an attempt that ends, a backoff that ends — is processed in increasing order as R24
says, each instant completely before the next, and the clock then reads t + `ms`. The reply is the time reply with
`now` = t + `ms`, written after all of that.

**R14.** `cancel` names a DAG. When no accepted DAG has that id, the reply is `unknown-dag`. Otherwise, at the current time
t, every job of the DAG that is not in a final state becomes `cancelled`: a running job's attempt ends at t with the
result `cancelled` and frees its slot; a job that is pending, ready or in backoff is cancelled with no new attempt. Then
the scheduling step of R23 runs at t (jobs of other DAGs may start), and the reply is done. A `cancel` of a DAG whose
jobs are all in a final state changes nothing and is answered with done.

**R15.** `report` names a DAG and is answered with the report reply holding the DAG's report object (R26) as of the
current time, or with `unknown-dag`.

### 3.3 Jobs

**R16.** A **job object** has exactly these six members: `id`, an id unique within the submission; `deps`, an array of
zero or more ids of jobs of the same submission; `duration` D, a count of at least 1; `timeout` T, a count — 0 means
that the job has no timeout; `retries` R, a count; and `results`, an array of 1 or more strings, each `"ok"` or
`"fail"`. The **outcome** of the job's n-th attempt is the n-th element of `results` when n is at most the array's length
and the last element otherwise.

**R17.** An **attempt** of a job begins when the scheduling step starts the job (R23) at a time s and occupies one of the
k slots while it runs. When T is 0 or T is at least D, the attempt ends at s + D with its outcome: `ok`, after which the
job has `succeeded`, or `fail`. When T is positive and below D, the attempt ends at s + T with the result `timeout`. The
timeout covers one attempt's run alone — the time in backoff or waiting for a slot does not count — and every attempt of
a job has the same D and T. An attempt may also end by a cancellation (R14) or at the stop (R7), at the current time,
with the result `cancelled`; a cancellation is never retried.

**R18.** When an attempt — the job's n-th — ends with `fail` or `timeout` at a time e: if n is at most R, the job enters
**backoff** until e + min(`base` · 2^(n−1), `cap`), with `base` and `cap` of the job's DAG, and becomes ready at that
instant (R24; a backoff of 0 ends at e itself); otherwise the job has `failed`. So the first retry waits `base`, the
second 2·`base`, the third 4·`base`, each wait capped at `cap`, and a job makes at most R + 1 attempts. A job in backoff
holds no slot.

**R19.** A job is at every moment in one of eight **states**: `pending` — some job it depends on has not succeeded and
none has failed or been skipped; `ready` — every job it depends on has succeeded and it waits for a slot; `running`;
`backoff`; and the four **final states** `succeeded`, `failed`, `skipped` and `cancelled`. At its submission a job with
no dependency is `ready` and every other job is `pending`. A pending job becomes `ready` at the instant the last of its
dependencies succeeds and becomes `skipped` at the instant any of its dependencies becomes `failed` or `skipped`; both
propagate at that same instant through the whole DAG. A job in a final state never changes again.

### 3.4 The clock

**R20.** Time is a count of milliseconds, an integer that is 0 at the moment the program becomes ready (R4) and never
decreases. The environment variable `CLOCK` selects the mode: unset, or `virtual`, is **virtual mode**, in which time
advances only by `tick` (R13) and nothing happens between two requests; `real` is **real mode**, in which the time is the
whole number of milliseconds that the program's own monotonic clock has counted since readiness, and every due instant
(R24) is processed when that clock reaches it. In real mode `tick` does not occur.

**R21.** In real mode the tests keep a margin: no test sends a request, or the signal SIGTERM, within 500 milliseconds
before or after an instant at which an attempt or a backoff is due to end, so a program that processes each due instant
within 100 milliseconds of it gives the same states as one that is exact. In real mode every time in a reply or in a
standard-output line — `now`, `start`, `end` — is compared by its form only: an integer written as R10 says. In virtual
mode every time is compared byte for byte, like everything else.

### 3.5 Scheduling

**R22.** The environment variable `SLOTS` holds k, the concurrency limit: an integer from 1 to 1 000 written in decimal.
When it is not set, k is 1. At every moment at most k jobs are running, counted over all the DAGs the program holds.

**R23.** The **ready queue** holds every job in the state `ready`, ordered by the instant at which the job became ready
(earlier first), then by the sequence number of its DAG (smaller first), then by the position of the job in its
submission's `jobs` array (earlier first). The **scheduling step**, run at a time t, starts the first job of the ready
queue whenever fewer than k jobs are running and the queue is not empty, and repeats until one of the two fails. A job
started at t begins its attempt at t (R17).

**R24.** An instant t at which something is due is processed in four steps, in this order: (1) every attempt that ends
at t ends with its result (R17), and each job whose attempt failed enters backoff or has failed (R18); (2) every
backoff that ends at t ends, and its job becomes ready at t — a backoff of 0 that began in step (1) of this same
instant ends here; (3) every pending job whose dependencies have all succeeded becomes ready at t, and every pending
job that depends on a failed or skipped job becomes skipped, through the whole DAG (R19); (4) the scheduling step
(R23) runs at t. A submission (R12) and a cancellation (R14) at a time t apply their own changes and then run the
scheduling step at t; the stop (R7) applies its own changes and runs no scheduling step. Nothing else changes a job's
state.

**R25.** There is one clock and one set of DAGs for all connections. Every request is processed whole, one at a time: no
two requests interleave, and the requests from all connections are processed in some single order that respects each
connection's own order. Hence a request sent on any connection after the reply to another request was received sees
that request's effects, and the effects of a `tick` are complete when its reply is written.

### 3.6 The report

**R26.** The **report object** of a DAG is the bytes
`{"dag":"<id>","now":<t>,"done":<done>,"jobs":[<job>,…]}` with no space anywhere: `<t>` the current time, `<done>`
`true` when every job of the DAG is in a final state and `false` otherwise, and one `<job>` per job in the order of the
submission's `jobs` array, each `{"id":"<id>","state":"<state>","attempts":[<attempt>,…]}` with the state's name as
R19 writes it and one `<attempt>` per attempt in the order they were made, each
`{"n":<n>,"start":<s>,"end":<e>,"result":"<r>"}`: `<n>` the attempt's number from 1, `<s>` the time it began, `<e>` the
time it ended and `<r>` its result — `ok`, `fail`, `timeout` or `cancelled` — or, while the attempt runs, `null` for
both so that the attempt reads `{"n":<n>,"start":<s>,"end":null,"result":null}`. A job that never ran has
`"attempts":[]`.

### 3.7 What is not defined

**R27.** A request or input that violates any of R8, R9, R16, R20 and R31 does not occur in any visible or hidden test.
What the program does with it is unspecified and is never tested. Such input is: a line that is not a JSON object of
the members its request names, with the forms R9 and R16 give; a `cmd` not in R11; an id, a count or a `results`
element outside its form; a `tick` in real mode; a `CLOCK` value other than `virtual` or `real`; more jobs,
dependencies, requests or connections, or a longer line, than section 4 allows.

**R28.** Standard output is compared only where a requirement of this spec says what it holds; otherwise it is recorded,
not judged. R7 is the one requirement that says what standard output holds: the report lines written on SIGTERM. The
program writes nothing to standard output before the signal, so those lines are the first and the only lines there, and
the tests read and compare them.

**R29.** Standard error is not compared and may hold anything; it is recorded, not judged. Only its size is bounded: a
run that writes more than 256 MiB to standard error is stopped, and that test fails.

**R30.** Every reply is compared with the expected reply byte for byte, except a field that a requirement of this spec
says is compared otherwise. Nothing else is normalized. R21 names the one such field: in real mode, the times.

## 4. Limits

**R31.** In every test: at most 200 DAGs are accepted and at most 200 000 jobs over all of them, each DAG holding 1 to
1 000 jobs and each job naming at most 100 dependencies; `duration`, `timeout`, `base` and `cap` are at most
1 000 000 000, `retries` at most 10, `ms` at most 1 000 000 000, and the time never exceeds 10^12; k is at most 1 000; a
request line is at most 1 048 576 bytes; at most 10 000 requests are sent; at most 64 connections are open at once and
at most 1 000 are opened over one test; a test lasts at most 600 seconds.

**R32.** Each test starts the program once, in a fresh process, and drives one scenario against it under a bound on the
program's memory. The bound is the one the task's statement states: one registered value, the same for every daemon and
web task, which the test harness adds to the statement from its registration, so a spec never writes the number. It is
registered before the freeze, only once the reference solutions of every daemon and web task pass every test under it in
both stacks, and enforced as F-MEMORY says. The load scenarios are hidden scenarios of up to the bounds of R31, each
lasting at most 600 seconds; the bound on memory holds in every test, whether it is a load scenario or not. Memory is
measured as the peak memory of the program's own control group (cgroup `memory.peak`), which counts the program and every
process it starts, including the page cache and kernel memory charged to them, with no swap. The program's memory
reaches the limit when the system kills a process of the program's control group at the limit, or when the program's own
runtime stops it for lack of memory; either way the run is stopped and fails that test. The test machine caps each
runtime's own heap at 90 % of the limit, by that runtime's own setting. The tests run only on a machine that can read
this peak, and no other measurement of memory passes or fails a test. The program and every process it starts may hold
at most 512 processes and threads together. Beyond that the system refuses to create another, and a run that needs more
fails that test.

**R33.** Every step of a scenario that waits — for readiness, for a reply, for the exit — has a deadline, and a step that
does not complete by its deadline fails the test. The visible samples state their deadlines in their files; the
deadlines of the hidden scenarios are fixed when the suite is calibrated, so that the reference solutions in both stacks
meet each of them with margin.

## 5. Build and run — the workspace contract

**R34.** The workspace holds `STATEMENT.md`, `samples/`, `runner/`, `build.sh`, `run.sh`, `PACKAGES.md` and the
toolchain pin files named in `build.sh`. These are frozen inputs: do not change, rename or remove them. The program's sources
go where `build.sh` and `run.sh` expect them; the two scripts say where.

**R35.** `bash build.sh`, run in the workspace directory, builds the program with no network access and with no package
outside the set listed in `PACKAGES.md`. It must exit 0 within 600 seconds. It runs once before the tests; a build that
does not exit 0 fails every test.

**R36.** `bash run.sh`, run in the workspace directory with no arguments, starts the built program once per test, with
the environment section 2 describes. One fresh process per test; nothing carries over from one test to the next.
`run.sh` ends by replacing itself with the program (`exec`), so the process the tests start, signal and measure is the
program itself. Every test gives the program the same base environment, identical for both stacks apart from a few
documented runtime settings of each stack: in it `LANG` and `LC_ALL` are `de_DE.UTF-8`, nothing else selects a locale,
and `TZ` is `UTC`.

**R37.** The tests run offline. They use a clean copy of the workspace as you left it: build output and installed
packages are removed, and the frozen inputs are put back as they were given. Every other file is yours, including the
project files that declare or lock your packages, and is built as you left it, by `bash build.sh`. The program must not
depend on the network, on files outside the workspace, or on anything written during an earlier run.

## 6. Visible samples

The same four scenarios are in `samples/`, one scenario file and one expected-transcript file per scenario
(`samples/<id>.in`, `samples/<id>.out`), shown below exactly as stored. `runner/GRAMMAR.md` describes their format, and
`python3 -I runner/run_samples.py samples/` builds the program and runs them.

### s01

`samples/s01.in`

```text
scenario 2
# One DAG: b depends on a. Time advances only by tick. The report shows a
# running at once, then succeeded with b running, then both succeeded and
# the DAG done. The stop writes the DAG's report to standard output.
default within 2s
env SLOTS "1"
start ready port within 10s
connect c1 tcp
# 1. Submit; a starts at 0, b waits for a.
send c1 "{\"cmd\":\"submit\",\"dag\":\"d1\",\"backoff\":{\"base\":100,\"cap\":800},\"jobs\":[{\"id\":\"a\",\"deps\":[],\"duration\":100,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]},{\"id\":\"b\",\"deps\":[\"a\"],\"duration\":50,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]}]}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"report\",\"dag\":\"d1\"}\n"
recv c1 lines 1
# 2. 100 ms: a ends with ok, b starts.
send c1 "{\"cmd\":\"tick\",\"ms\":100}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"report\",\"dag\":\"d1\"}\n"
recv c1 lines 1
# 3. 50 ms more: b ends; the DAG is done.
send c1 "{\"cmd\":\"tick\",\"ms\":50}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"report\",\"dag\":\"d1\"}\n"
recv c1 lines 1
# 4. The stop.
close c1
signal TERM
recv stdout lines 1
wait-exit within 5s
```

`samples/s01.out`

```text
transcript 2
ready
# 1. Submit; a starts at 0, b waits for a.
recv c1 "{\"ok\":true}\n"
recv c1 "{\"ok\":true,\"report\":{\"dag\":\"d1\",\"now\":0,\"done\":false,\"jobs\":[{\"id\":\"a\",\"state\":\"running\",\"attempts\":[{\"n\":1,\"start\":0,\"end\":null,\"result\":null}]},{\"id\":\"b\",\"state\":\"pending\",\"attempts\":[]}]}}\n"
# 2. 100 ms: a ends with ok, b starts.
recv c1 "{\"ok\":true,\"now\":100}\n"
recv c1 "{\"ok\":true,\"report\":{\"dag\":\"d1\",\"now\":100,\"done\":false,\"jobs\":[{\"id\":\"a\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":0,\"end\":100,\"result\":\"ok\"}]},{\"id\":\"b\",\"state\":\"running\",\"attempts\":[{\"n\":1,\"start\":100,\"end\":null,\"result\":null}]}]}}\n"
# 3. 50 ms more: b ends; the DAG is done.
recv c1 "{\"ok\":true,\"now\":150}\n"
recv c1 "{\"ok\":true,\"report\":{\"dag\":\"d1\",\"now\":150,\"done\":true,\"jobs\":[{\"id\":\"a\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":0,\"end\":100,\"result\":\"ok\"}]},{\"id\":\"b\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":100,\"end\":150,\"result\":\"ok\"}]}]}}\n"
# 4. The stop.
recv stdout "{\"dag\":\"d1\",\"now\":150,\"done\":true,\"jobs\":[{\"id\":\"a\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":0,\"end\":100,\"result\":\"ok\"}]},{\"id\":\"b\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":100,\"end\":150,\"result\":\"ok\"}]}]}\n"
exit 0
```

### s02

`samples/s02.in`

```text
scenario 2
# Two slots. d1 has four independent jobs; d2, submitted at the same
# instant, has one. Ready jobs start in the order they became ready, then
# by DAG, then by position: a and b at 0; c and d at 100; e at 150. One
# tick of 300 ms carries every instant.
default within 2s
env SLOTS "2"
start ready port within 10s
connect c1 tcp
# 1. Two submissions at time 0.
send c1 "{\"cmd\":\"submit\",\"dag\":\"d1\",\"backoff\":{\"base\":100,\"cap\":800},\"jobs\":[{\"id\":\"a\",\"deps\":[],\"duration\":100,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]},{\"id\":\"b\",\"deps\":[],\"duration\":100,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]},{\"id\":\"c\",\"deps\":[],\"duration\":50,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]},{\"id\":\"d\",\"deps\":[],\"duration\":50,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]}]}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"submit\",\"dag\":\"d2\",\"backoff\":{\"base\":100,\"cap\":800},\"jobs\":[{\"id\":\"e\",\"deps\":[],\"duration\":100,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]}]}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"report\",\"dag\":\"d2\"}\n"
recv c1 lines 1
# 2. One tick to 300.
send c1 "{\"cmd\":\"tick\",\"ms\":300}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"report\",\"dag\":\"d1\"}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"report\",\"dag\":\"d2\"}\n"
recv c1 lines 1
# 3. The stop.
close c1
signal TERM
recv stdout lines 2
wait-exit within 5s
```

`samples/s02.out`

```text
transcript 2
ready
# 1. Two submissions at time 0.
recv c1 "{\"ok\":true}\n"
recv c1 "{\"ok\":true}\n"
recv c1 "{\"ok\":true,\"report\":{\"dag\":\"d2\",\"now\":0,\"done\":false,\"jobs\":[{\"id\":\"e\",\"state\":\"ready\",\"attempts\":[]}]}}\n"
# 2. One tick to 300.
recv c1 "{\"ok\":true,\"now\":300}\n"
recv c1 "{\"ok\":true,\"report\":{\"dag\":\"d1\",\"now\":300,\"done\":true,\"jobs\":[{\"id\":\"a\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":0,\"end\":100,\"result\":\"ok\"}]},{\"id\":\"b\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":0,\"end\":100,\"result\":\"ok\"}]},{\"id\":\"c\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":100,\"end\":150,\"result\":\"ok\"}]},{\"id\":\"d\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":100,\"end\":150,\"result\":\"ok\"}]}]}}\n"
recv c1 "{\"ok\":true,\"report\":{\"dag\":\"d2\",\"now\":300,\"done\":true,\"jobs\":[{\"id\":\"e\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":150,\"end\":250,\"result\":\"ok\"}]}]}}\n"
# 3. The stop.
recv stdout "{\"dag\":\"d1\",\"now\":300,\"done\":true,\"jobs\":[{\"id\":\"a\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":0,\"end\":100,\"result\":\"ok\"}]},{\"id\":\"b\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":0,\"end\":100,\"result\":\"ok\"}]},{\"id\":\"c\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":100,\"end\":150,\"result\":\"ok\"}]},{\"id\":\"d\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":100,\"end\":150,\"result\":\"ok\"}]}]}\n"
recv stdout "{\"dag\":\"d2\",\"now\":300,\"done\":true,\"jobs\":[{\"id\":\"e\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":150,\"end\":250,\"result\":\"ok\"}]}]}\n"
exit 0
```

### s03

`samples/s03.in`

```text
scenario 2
# One slot. a times out every attempt (duration 300, timeout 100, two
# retries; backoff 100 then 150 with base 100, cap 150) and fails at 550;
# b depends on a and is skipped. c and d run while a waits in backoff:
# backoff holds no slot. d fails once (results fail, ok), retries after
# 100 ms and succeeds. Two reports: during the run and at the end.
default within 2s
env SLOTS "1"
start ready port within 10s
connect c1 tcp
# 1. Submit; a takes the slot at 0.
send c1 "{\"cmd\":\"submit\",\"dag\":\"d1\",\"backoff\":{\"base\":100,\"cap\":150},\"jobs\":[{\"id\":\"a\",\"deps\":[],\"duration\":300,\"timeout\":100,\"retries\":2,\"results\":[\"ok\"]},{\"id\":\"b\",\"deps\":[\"a\"],\"duration\":10,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]},{\"id\":\"c\",\"deps\":[],\"duration\":50,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]},{\"id\":\"d\",\"deps\":[],\"duration\":10,\"timeout\":0,\"retries\":1,\"results\":[\"fail\",\"ok\"]}]}\n"
recv c1 lines 1
# 2. To 200: a timed out at 100 and waits; c ran 100-150; d ran 150-160 and failed; a restarts at 200.
send c1 "{\"cmd\":\"tick\",\"ms\":200}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"report\",\"dag\":\"d1\"}\n"
recv c1 lines 1
# 3. To 600: d retried at 300; a failed at 550; b skipped.
send c1 "{\"cmd\":\"tick\",\"ms\":400}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"report\",\"dag\":\"d1\"}\n"
recv c1 lines 1
# 4. The stop.
close c1
signal TERM
recv stdout lines 1
wait-exit within 5s
```

`samples/s03.out`

```text
transcript 2
ready
# 1. Submit; a takes the slot at 0.
recv c1 "{\"ok\":true}\n"
# 2. To 200: a timed out at 100 and waits; c ran 100-150; d ran 150-160 and failed; a restarts at 200.
recv c1 "{\"ok\":true,\"now\":200}\n"
recv c1 "{\"ok\":true,\"report\":{\"dag\":\"d1\",\"now\":200,\"done\":false,\"jobs\":[{\"id\":\"a\",\"state\":\"running\",\"attempts\":[{\"n\":1,\"start\":0,\"end\":100,\"result\":\"timeout\"},{\"n\":2,\"start\":200,\"end\":null,\"result\":null}]},{\"id\":\"b\",\"state\":\"pending\",\"attempts\":[]},{\"id\":\"c\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":100,\"end\":150,\"result\":\"ok\"}]},{\"id\":\"d\",\"state\":\"backoff\",\"attempts\":[{\"n\":1,\"start\":150,\"end\":160,\"result\":\"fail\"}]}]}}\n"
# 3. To 600: d retried at 300; a failed at 550; b skipped.
recv c1 "{\"ok\":true,\"now\":600}\n"
recv c1 "{\"ok\":true,\"report\":{\"dag\":\"d1\",\"now\":600,\"done\":true,\"jobs\":[{\"id\":\"a\",\"state\":\"failed\",\"attempts\":[{\"n\":1,\"start\":0,\"end\":100,\"result\":\"timeout\"},{\"n\":2,\"start\":200,\"end\":300,\"result\":\"timeout\"},{\"n\":3,\"start\":450,\"end\":550,\"result\":\"timeout\"}]},{\"id\":\"b\",\"state\":\"skipped\",\"attempts\":[]},{\"id\":\"c\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":100,\"end\":150,\"result\":\"ok\"}]},{\"id\":\"d\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":150,\"end\":160,\"result\":\"fail\"},{\"n\":2,\"start\":300,\"end\":310,\"result\":\"ok\"}]}]}}\n"
# 4. The stop.
recv stdout "{\"dag\":\"d1\",\"now\":600,\"done\":true,\"jobs\":[{\"id\":\"a\",\"state\":\"failed\",\"attempts\":[{\"n\":1,\"start\":0,\"end\":100,\"result\":\"timeout\"},{\"n\":2,\"start\":200,\"end\":300,\"result\":\"timeout\"},{\"n\":3,\"start\":450,\"end\":550,\"result\":\"timeout\"}]},{\"id\":\"b\",\"state\":\"skipped\",\"attempts\":[]},{\"id\":\"c\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":100,\"end\":150,\"result\":\"ok\"}]},{\"id\":\"d\",\"state\":\"succeeded\",\"attempts\":[{\"n\":1,\"start\":150,\"end\":160,\"result\":\"fail\"},{\"n\":2,\"start\":300,\"end\":310,\"result\":\"ok\"}]}]}\n"
exit 0
```

### s04

`samples/s04.in`

```text
scenario 2
# Rejected submissions keep nothing: a cycle, an unknown dependency, a
# duplicate job id, a reused DAG id. A dependency named twice counts once.
# Cancelling a DAG ends its running attempt at the current time and
# cancels the rest; a report of an unknown DAG is an error.
default within 2s
env SLOTS "1"
start ready port within 10s
connect c1 tcp
# 1. Rejected: a cycle through three jobs; a dependency on no job; two jobs named x.
send c1 "{\"cmd\":\"submit\",\"dag\":\"d1\",\"backoff\":{\"base\":100,\"cap\":800},\"jobs\":[{\"id\":\"a\",\"deps\":[\"c\"],\"duration\":10,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]},{\"id\":\"b\",\"deps\":[\"a\"],\"duration\":10,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]},{\"id\":\"c\",\"deps\":[\"b\"],\"duration\":10,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]}]}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"submit\",\"dag\":\"d1\",\"backoff\":{\"base\":100,\"cap\":800},\"jobs\":[{\"id\":\"a\",\"deps\":[\"z\"],\"duration\":10,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]}]}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"submit\",\"dag\":\"d1\",\"backoff\":{\"base\":100,\"cap\":800},\"jobs\":[{\"id\":\"x\",\"deps\":[],\"duration\":10,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]},{\"id\":\"x\",\"deps\":[],\"duration\":10,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]}]}\n"
recv c1 lines 1
# 2. Accepted: p, then q which names p twice; the same DAG id is then refused.
send c1 "{\"cmd\":\"submit\",\"dag\":\"d1\",\"backoff\":{\"base\":100,\"cap\":800},\"jobs\":[{\"id\":\"p\",\"deps\":[],\"duration\":100,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]},{\"id\":\"q\",\"deps\":[\"p\",\"p\"],\"duration\":100,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]}]}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"submit\",\"dag\":\"d1\",\"backoff\":{\"base\":100,\"cap\":800},\"jobs\":[{\"id\":\"a\",\"deps\":[],\"duration\":10,\"timeout\":0,\"retries\":0,\"results\":[\"ok\"]}]}\n"
recv c1 lines 1
# 3. 50 ms in, cancel d1: p's attempt ends at 50 as cancelled, q is cancelled, the DAG is done.
send c1 "{\"cmd\":\"tick\",\"ms\":50}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"cancel\",\"dag\":\"d1\"}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"report\",\"dag\":\"d1\"}\n"
recv c1 lines 1
send c1 "{\"cmd\":\"report\",\"dag\":\"d9\"}\n"
recv c1 lines 1
# 4. The stop: the report of the one accepted DAG.
close c1
signal TERM
recv stdout lines 1
wait-exit within 5s
```

`samples/s04.out`

```text
transcript 2
ready
# 1. Rejected: a cycle through three jobs; a dependency on no job; two jobs named x.
recv c1 "{\"ok\":false,\"error\":\"cycle\"}\n"
recv c1 "{\"ok\":false,\"error\":\"unknown-dependency\"}\n"
recv c1 "{\"ok\":false,\"error\":\"duplicate-job\"}\n"
# 2. Accepted: p, then q which names p twice; the same DAG id is then refused.
recv c1 "{\"ok\":true}\n"
recv c1 "{\"ok\":false,\"error\":\"duplicate-dag\"}\n"
# 3. 50 ms in, cancel d1: p's attempt ends at 50 as cancelled, q is cancelled, the DAG is done.
recv c1 "{\"ok\":true,\"now\":50}\n"
recv c1 "{\"ok\":true}\n"
recv c1 "{\"ok\":true,\"report\":{\"dag\":\"d1\",\"now\":50,\"done\":true,\"jobs\":[{\"id\":\"p\",\"state\":\"cancelled\",\"attempts\":[{\"n\":1,\"start\":0,\"end\":50,\"result\":\"cancelled\"}]},{\"id\":\"q\",\"state\":\"cancelled\",\"attempts\":[]}]}}\n"
recv c1 "{\"ok\":false,\"error\":\"unknown-dag\"}\n"
# 4. The stop: the report of the one accepted DAG.
recv stdout "{\"dag\":\"d1\",\"now\":50,\"done\":true,\"jobs\":[{\"id\":\"p\",\"state\":\"cancelled\",\"attempts\":[{\"n\":1,\"start\":0,\"end\":50,\"result\":\"cancelled\"}]},{\"id\":\"q\",\"state\":\"cancelled\",\"attempts\":[]}]}\n"
exit 0
```

## 7. Hidden tests

**R38.** Hidden tests exist. They are scenarios in the same form as the visible ones: one or many connections submitting
DAGs within the bounds of section 4 — requests cut into pieces and sent several in one write, DAGs that overlap in time
and compete for the slots, chains and fans of dependencies, jobs that fail, time out and retry at the ends of their
budgets, backoffs that end at the same instant as other attempts, cancellations at every stage including of a DAG that
is already done, rejected submissions of every kind, reports at every stage, long runs in virtual mode with many DAGs
and a single large `tick`, runs in real mode with the margin of R21, and the stop by SIGTERM with jobs running — with the
expected transcripts derived from section 3 and compared as R30 says, under the limits of section 4. The visible samples
are a small part of the same suite; a program that passes them has not thereby passed the hidden tests.

## 8. Amendments

None yet.

<!-- stack-sentence -->
Use C# 14 on .NET 10 (SDK 10.0.403 is installed).

<!-- memory-sentence -->
The program and its descendants run under a 512 MiB (536,870,912-byte) cgroup memory limit, including runtime overhead. Build resources are separate.

<!-- heap-sentence -->
The .NET garbage-collected heap is capped by DOTNET_GCHeapHardLimit=0x1ccccccc (483,183,820 bytes, 90 % of the limit).
