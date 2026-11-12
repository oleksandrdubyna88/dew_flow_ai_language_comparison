# Task 07 — Stream processor with sliding windows

| | |
|---|---|
| Kind | a long-running program that serves until it is stopped by SIGTERM |
| Gate verdict | not yet gated |
| Freeze tag | none yet |
| Freeze rule | this text changes only by a dated amendment from its authors |
| Amendments | none yet |

Every numbered sentence (**R1** … **R30**) is a requirement. Hidden tests refer to these numbers; nothing outside them
is tested.

## 1. What the program does

**R1.** The program is a stream processor. It reads a stream of events from standard input, one event per line in the
form of section 3.1; each event names a key, an event time and an integer value. For every key it keeps sliding windows
over event time (section 3.2) and it writes to standard output, as lines in the form of section 3.3: one line for every
window when the window closes, holding the count, the sum, the minimum and the maximum of the values in it; one line
for every event it drops as late; one line for every key it expires; and one final line when it stops. Time in this
task is event time alone — the `ts` member of the events: nothing the program does depends on the moment at which a
line arrives, only on the order of the lines and on what they hold.

**R2.** The program's output is a function of its input: the same sequence of input lines, however it is cut into pieces
and at whatever pace it arrives, produces the same sequence of output lines. The program starts with no state, keeps
its state only in memory, reads and writes no file, and keeps nothing across a stop.

## 2. Start, readiness and stop

**R3.** The program is started with no arguments; it reads standard input and writes standard output. It listens on no
port, and it ignores the environment variable `PORT` when that is set. The tests set no environment variable for it
beyond the base environment of R28. The program does not exit before the end of its input or the signal SIGTERM,
whatever the pace of the input: while standard input is open it waits for more.

**R4.** The program is ready when it has written the line `{"type":"ready"}` to standard output. A test waits at most 10
seconds for readiness; a program that is not ready by then fails that test. That line is the first line the program
writes, and it writes it before it reads any input.

**R5.** A test may instead end at the end of its input: it closes the program's standard input, and the program then
does what R15 says and must exit with status 0 within 5 seconds of the end of its input, leaving no process it started
still running. A program still running after 5 seconds is killed and fails that test, and so does one that exits with
another status or leaves a process running. A test that ends this way sends no signal; it reads the lines of the flush
and waits for the exit.

**R6.** A test that does not end at the end of its input ends by sending the signal SIGTERM to the program. The program
must then exit with status 0 within 5 seconds, leaving no process it started still running. A program still running
after 5 seconds is killed and fails that test, and so does one that exits with another status or leaves a process
running. On SIGTERM the program performs the same flush as at the end of input (R15), writes its lines to standard
output, and then exits; the tests read those lines. The tests send SIGTERM only after they have read every output line
that the input they wrote had to produce, and no test writes to standard input during the second before it sends the
signal, so when the signal arrives every line the program has received has been processed (R9) and no further line is on
its way. A test may send SIGTERM before writing any input: the flush then holds the end line alone.

## 3. Interface

### 3.1 Input lines

**R7.** Standard input is a sequence of lines, each ended by one line feed (0x0A); the input holds nothing but complete
lines, so the last byte of a non-empty input is a line feed. Every line is one event: a JSON object with exactly three
members, `key`, `ts` and `value`, in any order, with JSON whitespace other than a line feed allowed between its tokens
and no whitespace required. `key` is a JSON string of 1 to 64 characters, each one of `A`–`Z`, `a`–`z`, `0`–`9`, `_`,
`-` and `.`, written as those characters with no escape sequence. `ts`, the event time, is an integer from 0 to
1 000 000 000 000 000 (10^15), written in decimal with no sign, no fraction, no exponent and no leading zero. `value` is
an integer from −9 223 372 036 854 775 808 to 9 223 372 036 854 775 807, written in decimal with a leading `-` when it
is negative and otherwise as `ts` is. A line is at most 256 bytes long, its line feed counted.

**R8.** The lines may arrive in any number of pieces, cut at any byte, over any span of time, and several lines may
arrive in one piece. The program treats a line as received only once its line feed has arrived, and processes the lines
in the order in which they arrive.

**R9.** The program processes every received line without waiting for later input or for the end of input: the output
lines that a line causes (section 3.3) are written to standard output, complete, once that line has been processed,
and a test reads them within its step deadline (R25) while standard input stays open and silent. Nothing is written
for a line that causes no output. A test may read the lines that one input line causes before it writes the next line,
or write many lines and read all their output after; the output is the same either way (R2).

### 3.2 Windows, the watermark, late events and expiry

**R10.** Three constants fix the windows, in the unit of `ts`: the window size **W = 10 000**, the slide **S = 5 000** and
the allowed lateness **L = 3 000**. A window is the half-open interval of event time [a, a + W) whose start a is an
integer multiple of S; a may be negative. Writing ⌊t⌋ for the largest multiple of S that is not above t, an event with
event time t lies in exactly two windows, the ones with starts ⌊t⌋ and ⌊t⌋ − S. Windows are kept per key: the window
(k, a) holds the accepted events of key k whose event time lies in [a, a + W). A window exists from the moment its
first event is accepted into it until it closes (R12 or R15).

**R11.** Let M be the largest `ts` of any event received so far. The **watermark** is M − L; before the first event there
is none. An event is **late** when a watermark exists and the event's `ts` is below it; a late event is dropped: it
enters no window, changes nothing, and the program writes a late line (R16) for it. Every other event — an event whose
`ts` is at or above the watermark, and the first event — is **accepted**: it enters its two windows (R10), and its key
is live from then on (R13). An event whose `ts` equals the watermark is accepted; an event's `ts` equal to M does not
raise M.

**R12.** When an accepted event raises M, the watermark advances to the new M − L, and every existing window of every key
whose end a + W is at or below the new watermark **closes**: the program writes its window line (R16) and the window
exists no more. The window lines of one advance are written in the order of the window start, smaller first, and
among windows with the same start in the order of their keys (R14). No window closes except by an advance of the
watermark or by the flush (R15). Hence an accepted event never belongs to a closed window, and an event that would
belong to a closed window is always late.

**R13.** A key is **live** from the acceptance of its first event until the moment it has no existing window left. At
that moment the key **expires**: the program writes its expired line (R16) and keeps nothing about the key. The expired
lines of one advance are written after all the window lines of that advance, in the order of the keys (R14). A later
event of an expired key is treated as an event of a key never seen: if it is accepted, the key is live again with fresh
windows. A late event never makes its key live and never keeps it live.

**R14.** Where this spec orders keys, they are ordered as byte sequences: of two keys, the one that is a proper prefix of
the other comes first; otherwise the first byte at which they differ decides, the smaller byte value first. So `B` comes
before `_`, which comes before `a`; `k-1`, `k.1`, `k1`, `k10`, `k2` stand in that order.

**R15.** The **flush**, performed at the end of input (R5) and on SIGTERM (R6), closes every existing window as if the
watermark had advanced beyond every window's end, and then expires every live key: it writes the window lines of all
existing windows in the order of R12 (by start, then by key), then the expired line of every key that is live after
those windows closed, in the order of the keys, then one **end line** `{"type":"end","accepted":<A>,"late":<B>}`, where
A is the number of accepted events and B the number of late events over the whole run. After the end line the program
writes nothing more and exits (R5, R6).

### 3.3 Output lines

**R16.** Every output line is exactly the bytes its row below gives, then one line feed: no space anywhere, no other
member, the members in the given order, every integer written in decimal with a leading `-` when it is negative and
otherwise with no sign, no leading zero, no fraction and no exponent, and the key written as its characters, as it
appeared in the input.

| Line | Written when | Bytes |
|---|---|---|
| ready | at start, before any input is read (R4) | `{"type":"ready"}` |
| window | a window closes (R12, R15) | `{"type":"window","key":"<key>","start":<a>,"end":<a+W>,"count":<c>,"sum":<s>,"min":<m>,"max":<x>}` |
| late | a late event is dropped (R11) | `{"type":"late","key":"<key>","ts":<ts>}` |
| expired | a key expires (R13, R15) | `{"type":"expired","key":"<key>"}` |
| end | the flush ends (R15) | `{"type":"end","accepted":<A>,"late":<B>}` |

In a window line, `<a>` is the window's start and `<a+W>` its end, `<c>` the number of accepted events in the window,
`<s>` the sum of their values, `<m>` the smallest and `<x>` the largest of their values. In a late line, `<key>` and
`<ts>` are the dropped event's.

**R17.** Counts, sums, minima and maxima are exact: a sum is the exact integer sum of the values, never rounded, and
every value is carried with all its digits, so that a window whose values are 9 007 199 254 740 993 and −1 has the sum
9 007 199 254 740 992. In every test the sum of every window — closed or not, at every moment — lies within the range of
`value` in R7, so arithmetic in that range suffices; a line whose integer differs in any digit from the exact one is
wrong.

### 3.4 What is not defined

**R18.** A request or input that violates any of R7 and R22 does not occur in any visible or hidden test. What the
program does with it is unspecified and is never tested. Such input is: a line that is not a JSON object of exactly the
three members with the forms R7 gives; a key, a `ts` or a `value` outside its range or form; a line longer than 256
bytes; an input that ends without a line feed; more events, keys or live keys than R22 allows.

**R19.** Standard output is compared only where a requirement of this spec says what it holds; otherwise it is recorded,
not judged. Here R4, R9, R15 and R16 say what every line of standard output holds: the program writes nothing else to
it, and the tests read and compare the lines those requirements define.

**R20.** Standard error is not compared and may hold anything; it is recorded, not judged. Only its size is bounded: a
run that writes more than 256 MiB to standard error is stopped, and that test fails.

**R21.** Every reply is compared with the expected reply byte for byte, except a field that a requirement of this spec
says is compared otherwise. Nothing else is normalized. In this task a reply is a line of standard output, and no
requirement names another comparison, so every line is compared byte for byte, its line feed included.

## 4. Limits

**R22.** In every test: at most 10 000 000 events are written to standard input; at most 4 000 000 distinct keys occur
over the test; at every moment at most 100 000 keys are live (R13); a line is at most 256 bytes; input may arrive at any
rate and in pieces of any size; a test lasts at most 600 seconds. A live key has at most three existing windows (this
follows from R10 to R12), so the state the program must hold at any moment is bounded by the live keys, not by the
events or the keys it has read.

**R23.** A window that has closed and a key that has expired occupy no memory of the program: the memory the program
needs depends only on the live keys and their existing windows (R22), not on everything the program has ever read, and
a test may feed the program far more keys over its run than the bound of R24 on the program's memory would hold at once.

**R24.** Each test starts the program once, in a fresh process, and drives one scenario against it under a bound on the
program's memory. The bound is the one the task's statement states: one registered value, the same for every daemon and
web task, which the test harness adds to the statement from its registration, so a spec never writes the number. It is
registered before the freeze, only once the reference solutions of every daemon and web task pass every test under it in
both stacks, and enforced as F-MEMORY says. The load scenarios are hidden scenarios of up to the bounds of R22, each
lasting at most 600 seconds; the bound on memory holds in every test, whether it is a load scenario or not. Memory is
measured as the peak memory of the program's own control group (cgroup `memory.peak`), which counts the program and
every process it starts, including the page cache and kernel memory charged to them, with no swap. The program's memory
reaches the limit when the system kills a process of the program's control group at the limit, or when the program's own
runtime stops it for lack of memory; either way the run is stopped and fails that test. The test machine caps each
runtime's own heap at 90 % of the limit, by that runtime's own setting. The tests run only on a machine that can read
this peak, and no other measurement of memory passes or fails a test. The program and every process it starts may hold
at most 512 processes and threads together. Beyond that the system refuses to create another, and a run that needs more
fails that test.

**R25.** Every step of a scenario that waits — for readiness, for a reply, for the exit — has a deadline, and a step that
does not complete by its deadline fails the test. The visible samples state their deadlines in their files; the
deadlines of the hidden scenarios are fixed when the suite is calibrated, so that the reference solutions in both stacks
meet each of them with margin.

## 5. Build and run — the workspace contract

**R26.** The workspace holds `STATEMENT.md`, `samples/`, `runner/`, `build.sh`, `run.sh`, `PACKAGES.md` and the
toolchain pin files named in `build.sh`. These are frozen inputs: do not change, rename or remove them. The program's sources
go where `build.sh` and `run.sh` expect them; the two scripts say where.

**R27.** `bash build.sh`, run in the workspace directory, builds the program with no network access and with no package
outside the set listed in `PACKAGES.md`. It must exit 0 within 600 seconds. It runs once before the tests; a build that
does not exit 0 fails every test.

**R28.** `bash run.sh`, run in the workspace directory with no arguments, starts the built program once per test, with
the environment section 2 describes. One fresh process per test; nothing carries over from one test to the next.
`run.sh` ends by replacing itself with the program (`exec`), so the process the tests start, signal and measure is the
program itself. Every test gives the program the same base environment, identical for both stacks apart from a few
documented runtime settings of each stack: in it `LANG` and `LC_ALL` are `de_DE.UTF-8`, nothing else selects a locale,
and `TZ` is `UTC`.

**R29.** The tests run offline. They use a clean copy of the workspace as you left it: build output and installed
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
# One key. Two events in one write open the windows [5000,15000) and
# [10000,20000); nothing is written until the watermark passes a window's
# end. The third event raises the watermark to 23000 and closes both. End
# of input closes the two windows that remain, expires the key, writes the
# end line, and the program exits by itself.
default within 2s
start ready stdout "{\"type\":\"ready\"}" within 10s
# 1. Two events, no output yet.
send stdin "{\"key\":\"a\",\"ts\":11000,\"value\":5}\n" "{\"key\":\"a\",\"ts\":13000,\"value\":7}\n"
expect stdout quiet for 300ms
# 2. The watermark advances to 23000: two windows close.
send stdin "{\"key\":\"a\",\"ts\":26000,\"value\":1}\n"
recv stdout lines 2
# 3. End of input: the flush.
close stdin
recv stdout lines 4
wait-exit within 5s
```

`samples/s01.out`

```text
transcript 2
ready
# 1. Two events, no output yet.
quiet stdout
# 2. The watermark advances to 23000: two windows close.
recv stdout "{\"type\":\"window\",\"key\":\"a\",\"start\":5000,\"end\":15000,\"count\":2,\"sum\":12,\"min\":5,\"max\":7}\n"
recv stdout "{\"type\":\"window\",\"key\":\"a\",\"start\":10000,\"end\":20000,\"count\":2,\"sum\":12,\"min\":5,\"max\":7}\n"
# 3. End of input: the flush.
recv stdout "{\"type\":\"window\",\"key\":\"a\",\"start\":20000,\"end\":30000,\"count\":1,\"sum\":1,\"min\":1,\"max\":1}\n"
recv stdout "{\"type\":\"window\",\"key\":\"a\",\"start\":25000,\"end\":35000,\"count\":1,\"sum\":1,\"min\":1,\"max\":1}\n"
recv stdout "{\"type\":\"expired\",\"key\":\"a\"}\n"
recv stdout "{\"type\":\"end\",\"accepted\":3,\"late\":0}\n"
exit 0
```

### s02

`samples/s02.in`

```text
scenario 2
# Four keys, events out of order. A key's event below the watermark is
# late and dropped even while the key has open windows. One advance closes
# five windows: the lines come by window start, then by key; the keys
# whose last window closed are expired after them. SIGTERM flushes the
# rest and the program writes its end line before exiting.
default within 2s
start ready stdout "{\"type\":\"ready\"}" within 10s
# 1. Out of order but not late: b at 7000, then a at 6000 (watermark 4000).
send stdin "{\"key\":\"b\",\"ts\":7000,\"value\":3}\n" "{\"key\":\"a\",\"ts\":6000,\"value\":10}\n"
# 2. c at 12000 moves the watermark to 9000; a at 5000 is now late.
send stdin "{\"key\":\"c\",\"ts\":12000,\"value\":2}\n" "{\"key\":\"a\",\"ts\":5000,\"value\":4}\n"
recv stdout lines 1
# 3. b at 9500 is accepted: 9500 is not below the watermark 9000.
send stdin "{\"key\":\"b\",\"ts\":9500,\"value\":1}\n"
expect stdout quiet for 300ms
# 4. d at 20000 moves the watermark to 17000: windows ending at 10000 and 15000 close.
send stdin "{\"key\":\"d\",\"ts\":20000,\"value\":0}\n"
recv stdout lines 7
# 5. The stop: open windows, then every live key, then the end line.
sleep 1s
signal TERM
recv stdout lines 6
wait-exit within 5s
```

`samples/s02.out`

```text
transcript 2
ready
# 1. Out of order but not late: b at 7000, then a at 6000 (watermark 4000).
# 2. c at 12000 moves the watermark to 9000; a at 5000 is now late.
recv stdout "{\"type\":\"late\",\"key\":\"a\",\"ts\":5000}\n"
# 3. b at 9500 is accepted: 9500 is not below the watermark 9000.
quiet stdout
# 4. d at 20000 moves the watermark to 17000: windows ending at 10000 and 15000 close.
recv stdout "{\"type\":\"window\",\"key\":\"a\",\"start\":0,\"end\":10000,\"count\":1,\"sum\":10,\"min\":10,\"max\":10}\n"
recv stdout "{\"type\":\"window\",\"key\":\"b\",\"start\":0,\"end\":10000,\"count\":2,\"sum\":4,\"min\":1,\"max\":3}\n"
recv stdout "{\"type\":\"window\",\"key\":\"a\",\"start\":5000,\"end\":15000,\"count\":1,\"sum\":10,\"min\":10,\"max\":10}\n"
recv stdout "{\"type\":\"window\",\"key\":\"b\",\"start\":5000,\"end\":15000,\"count\":2,\"sum\":4,\"min\":1,\"max\":3}\n"
recv stdout "{\"type\":\"window\",\"key\":\"c\",\"start\":5000,\"end\":15000,\"count\":1,\"sum\":2,\"min\":2,\"max\":2}\n"
recv stdout "{\"type\":\"expired\",\"key\":\"a\"}\n"
recv stdout "{\"type\":\"expired\",\"key\":\"b\"}\n"
# 5. The stop: open windows, then every live key, then the end line.
recv stdout "{\"type\":\"window\",\"key\":\"c\",\"start\":10000,\"end\":20000,\"count\":1,\"sum\":2,\"min\":2,\"max\":2}\n"
recv stdout "{\"type\":\"window\",\"key\":\"d\",\"start\":15000,\"end\":25000,\"count\":1,\"sum\":0,\"min\":0,\"max\":0}\n"
recv stdout "{\"type\":\"window\",\"key\":\"d\",\"start\":20000,\"end\":30000,\"count\":1,\"sum\":0,\"min\":0,\"max\":0}\n"
recv stdout "{\"type\":\"expired\",\"key\":\"c\"}\n"
recv stdout "{\"type\":\"expired\",\"key\":\"d\"}\n"
recv stdout "{\"type\":\"end\",\"accepted\":5,\"late\":1}\n"
exit 0
```

### s03

`samples/s03.in`

```text
scenario 2
# Values at the ends of the 64-bit range and just above 2^53, summed
# exactly. The first line arrives in two pieces, cut inside the number;
# the next two arrive in one write. Nothing closes before end of input.
default within 2s
start ready stdout "{\"type\":\"ready\"}" within 10s
# 1. One line in two pieces, cut after the digits 900719.
send stdin "{\"key\":\"n\",\"ts\":10000,\"value\":9007199254740993}\n" split 36 pause 100ms
send stdin "{\"key\":\"n\",\"ts\":10001,\"value\":-1}\n" "{\"key\":\"m\",\"ts\":10002,\"value\":9223372036854775807}\n"
send stdin "{\"key\":\"m\",\"ts\":10003,\"value\":-9223372036854775807}\n"
expect stdout quiet for 300ms
# 2. End of input.
close stdin
recv stdout lines 7
wait-exit within 5s
```

`samples/s03.out`

```text
transcript 2
ready
# 1. One line in two pieces, cut after the digits 900719.
quiet stdout
# 2. End of input.
recv stdout "{\"type\":\"window\",\"key\":\"m\",\"start\":5000,\"end\":15000,\"count\":2,\"sum\":0,\"min\":-9223372036854775807,\"max\":9223372036854775807}\n"
recv stdout "{\"type\":\"window\",\"key\":\"n\",\"start\":5000,\"end\":15000,\"count\":2,\"sum\":9007199254740992,\"min\":-1,\"max\":9007199254740993}\n"
recv stdout "{\"type\":\"window\",\"key\":\"m\",\"start\":10000,\"end\":20000,\"count\":2,\"sum\":0,\"min\":-9223372036854775807,\"max\":9223372036854775807}\n"
recv stdout "{\"type\":\"window\",\"key\":\"n\",\"start\":10000,\"end\":20000,\"count\":2,\"sum\":9007199254740992,\"min\":-1,\"max\":9007199254740993}\n"
recv stdout "{\"type\":\"expired\",\"key\":\"m\"}\n"
recv stdout "{\"type\":\"expired\",\"key\":\"n\"}\n"
recv stdout "{\"type\":\"end\",\"accepted\":4,\"late\":0}\n"
exit 0
```

### s04

`samples/s04.in`

```text
scenario 2
# A key that never returns is expired by the watermark that other keys
# carry. A window closes when its end equals the watermark. After the stop
# the end line counts four accepted events and no late one.
default within 2s
start ready stdout "{\"type\":\"ready\"}" within 10s
# 1. a has events at 6000 and 7000: windows [0,10000) and [5000,15000).
send stdin "{\"key\":\"a\",\"ts\":6000,\"value\":6}\n" "{\"key\":\"a\",\"ts\":7000,\"value\":7}\n"
expect stdout quiet for 300ms
# 2. t at 13000: the watermark is 10000, and [0,10000) ends at 10000, so it closes.
send stdin "{\"key\":\"t\",\"ts\":13000,\"value\":0}\n"
recv stdout lines 1
# 3. t at 18000: the watermark is 15000; a's last window closes and a expires.
send stdin "{\"key\":\"t\",\"ts\":18000,\"value\":0}\n"
recv stdout lines 3
# 4. The stop.
sleep 1s
signal TERM
recv stdout lines 4
wait-exit within 5s
```

`samples/s04.out`

```text
transcript 2
ready
# 1. a has events at 6000 and 7000: windows [0,10000) and [5000,15000).
quiet stdout
# 2. t at 13000: the watermark is 10000, and [0,10000) ends at 10000, so it closes.
recv stdout "{\"type\":\"window\",\"key\":\"a\",\"start\":0,\"end\":10000,\"count\":2,\"sum\":13,\"min\":6,\"max\":7}\n"
# 3. t at 18000: the watermark is 15000; a's last window closes and a expires.
recv stdout "{\"type\":\"window\",\"key\":\"a\",\"start\":5000,\"end\":15000,\"count\":2,\"sum\":13,\"min\":6,\"max\":7}\n"
recv stdout "{\"type\":\"window\",\"key\":\"t\",\"start\":5000,\"end\":15000,\"count\":1,\"sum\":0,\"min\":0,\"max\":0}\n"
recv stdout "{\"type\":\"expired\",\"key\":\"a\"}\n"
# 4. The stop.
recv stdout "{\"type\":\"window\",\"key\":\"t\",\"start\":10000,\"end\":20000,\"count\":2,\"sum\":0,\"min\":0,\"max\":0}\n"
recv stdout "{\"type\":\"window\",\"key\":\"t\",\"start\":15000,\"end\":25000,\"count\":1,\"sum\":0,\"min\":0,\"max\":0}\n"
recv stdout "{\"type\":\"expired\",\"key\":\"t\"}\n"
recv stdout "{\"type\":\"end\",\"accepted\":4,\"late\":0}\n"
exit 0
```

## 7. Hidden tests

**R30.** Hidden tests exist. They are scenarios in the same form as the visible ones: event streams within the bounds of
section 4, written to standard input in pieces of every size and at every pace — lines cut at any byte, many lines in
one write, long runs of millions of events over millions of keys of which few are live at once — with events in order
and out of order, at and around the watermark, at the ends of the range of `value`, with keys that return and keys that
never return, with the end of input and with SIGTERM at every stage of a run; the expected transcripts are derived from
section 3 and compared as R21 says, under the limits of section 4. The visible samples are a small part of the same
suite; a program that passes them has not thereby passed the hidden tests.

## 8. Amendments

None yet.
