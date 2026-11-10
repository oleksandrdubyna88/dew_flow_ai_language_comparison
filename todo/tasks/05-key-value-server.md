# Task 05 — Key-value server with expiry

| | |
|---|---|
| Kind | a long-running program that serves until it is stopped by SIGTERM |
| Gate verdict | not yet gated |
| Freeze tag | none yet |
| Freeze rule | this text changes only by a dated amendment from its authors |
| Amendments | none yet |

Every numbered sentence (**R1** … **R40**) is a requirement. Hidden tests refer to these numbers; nothing outside them
is tested.

## 1. What the program does

**R1.** The program is a key-value store served over TCP. It holds a set of entries; each entry maps one key to one
value, and may carry an expiry instant after which the entry no longer exists. Keys and values are sequences of bytes.
A client connects, sends requests in the frame format of section 3, and receives exactly one reply per request, on the
same connection, in the order the requests were received. Many clients are connected at once, and the program serves
each of them at the same time as the others.

**R2.** The store is empty when the program starts and lives only in the program's memory: nothing is read from or
written to disk, and nothing survives a stop.

## 2. Start, readiness and stop

**R3.** The environment variable `PORT` holds a TCP port number; the program listens on it, and the tests connect to it
at 127.0.0.1. `PORT` is the only variable the tests set beyond the runner's base environment; the program is started
with no arguments, and its standard input receives nothing.

**R4.** The program is ready when it accepts a TCP connection on that port. A test waits at most 10 seconds for
readiness; a program that is not ready by then fails that test.

**R5.** The program keeps serving whatever any one client does. A client may connect and disconnect without sending a
byte (the readiness probe does exactly that), may disconnect in the middle of a request, or may disconnect while
replies to it are still unsent; in each case the program drops that connection, an incomplete frame of that connection
changes nothing, every other connection continues, and the program does not exit.

**R6.** A test ends by sending the signal SIGTERM to the program. The program must then exit with status 0 within 5
seconds, leaving no process it started still running. A program still running after 5 seconds is killed and fails that
test, and so does one that exits with another status or leaves a process running.

**R7.** On SIGTERM the program exits even while client connections are open: it does not wait for a client to
disconnect, and it may close every connection at once. The tests send SIGTERM only after they have read every reply they
expect, so nothing is required of replies after the signal.

## 3. Interface

### 3.1 Frames

**R8.** Every request is one **array frame**: the byte `*`, the number of elements n written in decimal, the two bytes
CR LF (0x0D 0x0A), then n **bulk strings** one after another. A bulk string is the byte `$`, its length L in bytes
written in decimal, CR LF, then exactly L bytes of payload, then CR LF. Here n ≥ 1 and L ≥ 0; a number written in
decimal is ASCII digits only, with no sign, no spaces and no leading zero unless the number is 0. The L payload bytes may
be any bytes at all — CR, LF, `$`, `*` and 0x00 included — and nothing in them ends the string: only the stated length
does. Every element of a request is a bulk string; no other element type occurs in a request.

**R9.** On one connection the bytes form a sequence of frames with nothing between them. A frame may arrive in any
number of pieces, cut at any byte, over any span of time, and several frames may arrive in one piece. The program
replies to a frame only once the whole frame has arrived, replies to each frame exactly once, and sends the replies in
the order of the frames, each reply complete before the next begins. It never waits for more input before replying to a
frame it has received whole.

**R10.** A reply is one of five forms, each the stated bytes and nothing else:

| Form | Bytes |
|---|---|
| status | `+`, a text, CR LF — for example `+OK` CR LF |
| error | `-`, a text, CR LF — for example `-ERR value is not an integer or out of range` CR LF |
| integer | `:`, an integer written in decimal with a leading `-` when negative, CR LF — for example `:-2` CR LF |
| bulk string | `$`, the length L in decimal, CR LF, the L bytes, CR LF — exactly as in a request |
| null | the five bytes `$-1` CR LF |

A text holds no CR and no LF. Which form each command answers with is fixed by section 3.2; the program writes nothing
to a connection but these replies.

**R11.** The first element of a request is the command name and the remaining elements are its arguments, in order. A
command name is matched without regard to the case of its ASCII letters, so `get`, `Get` and `GET` are the same
command. An argument is a sequence of bytes and is interpreted only as the command's description says.

### 3.2 Commands

**R12.** The commands, each with the number of elements its frame has:

| Command | Elements | Reply |
|---|---|---|
| `PING` | 1 | status `+PONG` |
| `SET key value` | 3 | status `+OK` |
| `SET key value EX seconds` | 5 | status `+OK` |
| `SET key value PX milliseconds` | 5 | status `+OK` |
| `GET key` | 2 | bulk string, or null |
| `DEL key [key …]` | 2 or more | integer |
| `EXPIRE key seconds` | 3 | integer `:1` or `:0` |
| `PEXPIRE key milliseconds` | 3 | integer `:1` or `:0` |
| `INCR key` | 2 | integer, or error |

`PING` answers `+PONG` CR LF.

**R13.** `SET key value` stores the value under the key. If the key exists, its value is replaced and its expiry, if
any, is removed, so the entry then has no expiry. The reply is `+OK` CR LF.

**R14.** `SET key value EX seconds` and `SET key value PX milliseconds` do what `SET key value` does and give the entry an
expiry instant: the moment the command is processed plus the stated lifetime. The option name `EX` or `PX` is matched
without regard to the case of its letters; the lifetime is a positive integer written in decimal as R8 defines, in
seconds after `EX` and in milliseconds after `PX`. The reply is `+OK` CR LF.

**R15.** `GET key` answers the bulk string holding the key's current value, exactly the bytes last stored under it,
when the entry exists; otherwise it answers null. An entry that was never created, that was deleted, or whose expiry
instant has passed does not exist.

**R16.** `DEL key [key …]` removes the entries of the named keys and answers the integer count of the entries that
existed and were removed. The keys are processed in order; a key that is named twice is removed at its first mention
and counts once; a key with no entry counts 0.

**R17.** `EXPIRE key seconds` gives an existing entry an expiry instant — the moment the command is processed plus the
lifetime, a positive integer written in decimal, in seconds — replacing any expiry the entry had, and answers `:1`
CR LF. When the entry does not exist it changes nothing and answers `:0` CR LF. `PEXPIRE key milliseconds` is the same
with the lifetime in milliseconds.

**R18.** A value is **integer text** when it is the byte `0` alone, or an optional `-` followed by a digit `1`–`9` and
then zero or more digits `0`–`9`, and the number it writes lies in −9 223 372 036 854 775 808 … 9 223 372 036 854 775 807
inclusive. No other value is integer text: not an empty value, not one with a `+`, a space, a leading zero or `-0`, and
not one outside that range.

**R19.** `INCR key` adds 1 to the number the key's value writes and stores the result as its decimal text, the same
notation as integer text; the reply is the integer form of the new number. An entry that does not exist is treated as
holding `0`, so `INCR` on it creates the entry with value `1`, with no expiry, and answers `:1` CR LF. When the entry
exists, its expiry instant, if any, is kept unchanged.

**R20.** When the key's value is not integer text, `INCR` changes nothing and answers the error reply whose text is
`ERR value is not an integer or out of range`, that is, the bytes `-ERR value is not an integer or out of range` CR LF.
When the value is integer text and writes 9 223 372 036 854 775 807, so that adding 1 would leave the range of R18,
`INCR` changes nothing and answers the error whose text is `ERR increment or decrement would overflow`, that is, the
bytes `-ERR increment or decrement would overflow` CR LF. In both cases the entry keeps its value and its expiry.

**R21.** Keys and values are compared and stored as byte sequences: two keys are the same key when they are the same
bytes, and `GET` returns the value's bytes unchanged, whatever they are — including values that are empty (`$0` CR LF
CR LF in both request and reply), values holding CR LF or 0x00, and byte sequences that are not valid text in any
encoding.

### 3.3 Expiry

**R22.** An entry's expiry instant is fixed when `SET … EX`, `SET … PX`, `EXPIRE` or `PEXPIRE` is processed, from the
program's clock at that moment plus the lifetime. From that instant on the entry does not exist for every command —
`GET` answers null, `DEL` does not count it, `EXPIRE` and `PEXPIRE` answer `:0`, `INCR` starts from `0`, `SET` creates
a fresh entry — whether or not any command named the key since the instant passed. Before the instant the entry exists
and every command sees it.

**R23.** The tests keep a margin around every expiry instant: no test sends a command on a key in the 500 milliseconds
before or the 500 milliseconds after that key's expiry instant, counted from the moment the test sent the command that
set the instant. So a clock that is accurate to a tenth of a second is enough.

**R24.** The memory an expired entry occupied is released without any later command naming it: an entry whose expiry
instant passed more than 10 seconds ago is removed from the store, and the program keeps no reference to it. The memory
the program needs depends only on the entries that exist or expired less than 10 seconds ago (R33), not on everything
stored over the test's whole run.

### 3.4 Many clients

**R25.** There is one store for all connections. Every command is processed whole, one at a time: no two commands
interleave, and the commands from all connections are processed in some single order that respects each connection's
own order. Hence a command sent on any connection after the reply to a `SET`, `DEL`, `INCR`, `EXPIRE` or `PEXPIRE` was
received sees that command's effect, and k `INCR` commands on one key, from any number of connections, raise its number
by exactly k. Each reply is one of the k consecutive values that follow the number the key held before the first of
these commands, and the replies on one connection increase.

**R26.** A client that sends requests and does not read its replies for a while, or reads them slowly, does not slow
or stop the service of the other connections: their replies keep coming within the deadlines. The slow client's replies
are neither lost nor reordered; they are delivered, complete and in order, as it reads. While replies to a client are
unread, the program may stop reading that client's requests until the client reads its replies; no test writes more
than 64 KiB of requests to a connection while earlier replies on it are unread.

**R27.** When a client closes the sending half of its connection after writing one or more complete frames, the program
still answers every one of those frames, in order, and only then may close the connection. When a client closes the
connection entirely, R5 applies.

### 3.5 What is not defined

**R28.** A request or input that violates any of R8, R11, R12, R14, R17, R32 and R33 does not occur in any visible or
hidden test. What the program does with it is unspecified and is never tested. Such input is: a byte sequence that is
not an array frame of bulk strings; a command name not in section 3.2; a command with another number of elements than
its row states; a `SET` of five elements whose fourth element is not `EX` or `PX`; a lifetime that is not a positive
integer written in decimal; a key, value, key count, connection count, entry count or stored size outside the limits
of section 4.

**R29.** Standard output is compared only where a requirement of this spec says what it holds; otherwise it is recorded,
not judged. No requirement of this spec says what standard output holds.

**R30.** Standard error is not compared and may hold anything; it is recorded, not judged. Only its size is bounded: a
run that writes more than 256 MiB to standard error is stopped, and that test fails.

**R31.** Every reply is compared with the expected reply byte for byte, except a field that a requirement of this spec
says is compared otherwise. Nothing else is normalized. No requirement of this spec names another comparison, so every
reply is compared byte for byte.

## 4. Limits

**R32.** In every test: a key is 1 to 512 bytes; a value is 0 to 1 048 576 bytes; a `DEL` names at most 100 keys; a
lifetime is at most 1 000 000 000 seconds or 1 000 000 000 000 milliseconds; at most 256 connections are open at
once, and at most 10 000 connections are opened over one test; a test lasts at most 600 seconds.

**R33.** In every test, at every instant, the entries that exist or whose expiry instant passed less than 10 seconds
ago number at most 200 000 and hold at most 64 MiB (67 108 864 bytes) of keys and values together. Over a whole test,
at most 1 GiB (1 073 741 824 bytes) of keys and values is stored in all, counting every `SET`; all of it beyond the
entries above belongs to entries that expired, were deleted or were replaced.

**R34.** Each test starts the program once, in a fresh process, and drives one scenario against it under a bound on the
program's memory. The bound is the one the task's statement states: one registered value, the same for every daemon and
web task, which the test harness adds to the statement from its registration, so a spec never writes the number. It is
registered before the freeze, only once the reference solutions of every daemon and web task pass every test under it in
both stacks, and enforced as F-MEMORY says. The load scenarios are hidden scenarios of up to the bounds of R32 and R33,
each lasting at most 600 seconds; the bound on memory holds in every test, whether it is a load scenario or not. Memory
is measured as the peak memory of the program's own control group (cgroup `memory.peak`), which counts the program and
every process it starts, including the page cache and kernel memory charged to them, with no swap. The program's memory
reaches the limit when the system kills a process of the program's control group at the limit, or when the program's own
runtime stops it for lack of memory; either way the run is stopped and fails that test. The test machine caps each
runtime's own heap at 90 % of the limit, by that runtime's own setting. The tests run only on a machine that can read
this peak, and no other measurement of memory passes or fails a test. The program and every process it starts may hold
at most 512 processes and threads together. Beyond that the system refuses to create another, and a run that needs more
fails that test.

**R35.** Every step of a scenario that waits — for readiness, for a reply, for the exit — has a deadline, and a step
that does not complete by its deadline fails the test. The visible samples state their deadlines in their files; the
deadlines of the hidden scenarios are fixed when the suite is calibrated, so that the reference solutions in both stacks
meet each of them with margin.

## 5. Build and run — the workspace contract

**R36.** The workspace holds `STATEMENT.md`, `samples/`, `runner/`, `build.sh`, `run.sh`, `PACKAGES.md` and the
toolchain pin files named in `build.sh`. These are frozen inputs: do not change, rename or remove them. The program's
sources go where `build.sh` and `run.sh` expect them; the two scripts say where.

**R37.** `bash build.sh`, run in the workspace directory, builds the program with no network access and with no package
outside the set listed in `PACKAGES.md`. It must exit 0 within 600 seconds. It runs once before the tests; a build that
does not exit 0 fails every test.

**R38.** `bash run.sh`, run in the workspace directory with no arguments, starts the built program once per test, with
the environment section 2 describes. One fresh process per test; nothing carries over from one test to the next.
`run.sh` ends by replacing itself with the program (`exec`), so the process the tests start, signal and measure is the
program itself. Every test gives the program the same base environment, identical for both stacks apart from a few
documented runtime settings of each stack: in it `LANG` and `LC_ALL` are `de_DE.UTF-8`, nothing else selects a locale,
and `TZ` is `UTC`.

**R39.** The tests run offline. They use a clean copy of the workspace as you left it: build output and installed
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
# One client: PING, SET, GET, a key that was never set, DEL of two names of
# which one is present, GET after the deletion.
default within 2s
start ready port within 10s
connect c1 tcp
send c1 "*1\r\n$4\r\nPING\r\n"
recv c1 until "\r\n"
send c1 "*3\r\n$3\r\nSET\r\n$5\r\ncolor\r\n$4\r\nblue\r\n"
recv c1 until "\r\n"
send c1 "*2\r\n$3\r\nGET\r\n$5\r\ncolor\r\n"
recv c1 until "\r\n" count 2
send c1 "*2\r\n$3\r\nGET\r\n$5\r\nshape\r\n"
recv c1 until "\r\n"
send c1 "*3\r\n$3\r\nDEL\r\n$5\r\ncolor\r\n$5\r\nshape\r\n"
recv c1 until "\r\n"
send c1 "*2\r\n$3\r\nGET\r\n$5\r\ncolor\r\n"
recv c1 until "\r\n"
close c1
signal TERM
wait-exit within 5s
```

`samples/s01.out`

```text
transcript 2
ready
recv c1 "+PONG\r\n"
recv c1 "+OK\r\n"
recv c1 "$4\r\nblue\r\n"
recv c1 "$-1\r\n"
recv c1 ":1\r\n"
recv c1 "$-1\r\n"
exit 0
```

### s02

`samples/s02.in`

```text
scenario 2
# Counters: INCR on a key that was never set, then again; the stored text;
# INCR on a negative value; INCR on a value that is not an integer, which
# leaves the value unchanged.
default within 2s
start ready port within 10s
connect c1 tcp
send c1 "*2\r\n$4\r\nINCR\r\n$4\r\nhits\r\n"
recv c1 until "\r\n"
send c1 "*2\r\n$4\r\nINCR\r\n$4\r\nhits\r\n"
recv c1 until "\r\n"
send c1 "*2\r\n$3\r\nGET\r\n$4\r\nhits\r\n"
recv c1 until "\r\n" count 2
send c1 "*3\r\n$3\r\nSET\r\n$4\r\ntemp\r\n$2\r\n-3\r\n"
recv c1 until "\r\n"
send c1 "*2\r\n$4\r\nINCR\r\n$4\r\ntemp\r\n"
recv c1 until "\r\n"
send c1 "*3\r\n$3\r\nSET\r\n$4\r\nword\r\n$3\r\nten\r\n"
recv c1 until "\r\n"
send c1 "*2\r\n$4\r\nINCR\r\n$4\r\nword\r\n"
recv c1 until "\r\n"
send c1 "*2\r\n$3\r\nGET\r\n$4\r\nword\r\n"
recv c1 until "\r\n" count 2
close c1
signal TERM
wait-exit within 5s
```

`samples/s02.out`

```text
transcript 2
ready
recv c1 ":1\r\n"
recv c1 ":2\r\n"
recv c1 "$1\r\n2\r\n"
recv c1 "+OK\r\n"
recv c1 ":-2\r\n"
recv c1 "+OK\r\n"
recv c1 "-ERR value is not an integer or out of range\r\n"
recv c1 "$3\r\nten\r\n"
exit 0
```

### s03

`samples/s03.in`

```text
scenario 2
# Expiry: SET with a lifetime in milliseconds, read before and after it has
# passed; EXPIRE on an absent key and on a present one; the key gone after
# the lifetime set by EXPIRE. The pauses leave a wide margin on both sides
# of each expiry instant.
default within 2s
start ready port within 10s
connect c1 tcp
send c1 "*5\r\n$3\r\nSET\r\n$5\r\nfruit\r\n$5\r\nmango\r\n$2\r\nPX\r\n$4\r\n1500\r\n"
recv c1 until "\r\n"
send c1 "*2\r\n$3\r\nGET\r\n$5\r\nfruit\r\n"
recv c1 until "\r\n" count 2
sleep 2500ms
send c1 "*2\r\n$3\r\nGET\r\n$5\r\nfruit\r\n"
recv c1 until "\r\n"
send c1 "*3\r\n$6\r\nEXPIRE\r\n$5\r\nfruit\r\n$2\r\n10\r\n"
recv c1 until "\r\n"
send c1 "*3\r\n$3\r\nSET\r\n$5\r\nfruit\r\n$5\r\nmango\r\n"
recv c1 until "\r\n"
send c1 "*3\r\n$6\r\nEXPIRE\r\n$5\r\nfruit\r\n$1\r\n1\r\n"
recv c1 until "\r\n"
send c1 "*2\r\n$3\r\nGET\r\n$5\r\nfruit\r\n"
recv c1 until "\r\n" count 2
sleep 2s
send c1 "*2\r\n$3\r\nGET\r\n$5\r\nfruit\r\n"
recv c1 until "\r\n"
close c1
signal TERM
wait-exit within 5s
```

`samples/s03.out`

```text
transcript 2
ready
recv c1 "+OK\r\n"
recv c1 "$5\r\nmango\r\n"
recv c1 "$-1\r\n"
recv c1 ":0\r\n"
recv c1 "+OK\r\n"
recv c1 ":1\r\n"
recv c1 "$5\r\nmango\r\n"
recv c1 "$-1\r\n"
exit 0
```

### s04

`samples/s04.in`

```text
scenario 2
# Two clients share one store. c1 sends two requests in one write and gets
# two replies in order; c2 reads what c1 stored and overwrites it; c1 reads
# the new value. DEL of one present key answers 1.
default within 2s
start ready port within 10s
connect c1 tcp
connect c2 tcp
send c1 "*3\r\n$3\r\nSET\r\n$4\r\nteam\r\n$5\r\nalpha\r\n*2\r\n$3\r\nGET\r\n$4\r\nteam\r\n"
recv c1 until "\r\n"
recv c1 until "\r\n" count 2
send c2 "*2\r\n$3\r\nGET\r\n$4\r\nteam\r\n"
recv c2 until "\r\n" count 2
send c2 "*3\r\n$3\r\nSET\r\n$4\r\nteam\r\n$4\r\nbeta\r\n"
recv c2 until "\r\n"
send c1 "*2\r\n$3\r\nGET\r\n$4\r\nteam\r\n"
recv c1 until "\r\n" count 2
send c2 "*2\r\n$3\r\nDEL\r\n$4\r\nteam\r\n"
recv c2 until "\r\n"
send c1 "*2\r\n$3\r\nGET\r\n$4\r\nteam\r\n"
recv c1 until "\r\n"
close c1
close c2
signal TERM
wait-exit within 5s
```

`samples/s04.out`

```text
transcript 2
ready
recv c1 "+OK\r\n"
recv c1 "$5\r\nalpha\r\n"
recv c2 "$5\r\nalpha\r\n"
recv c2 "+OK\r\n"
recv c1 "$4\r\nbeta\r\n"
recv c2 ":1\r\n"
recv c1 "$-1\r\n"
exit 0
```

## 7. Hidden tests

**R40.** Hidden tests exist. They are scenarios in the same form as the visible ones: one or many connections sending
the commands of section 3.2 within the bounds of section 4 — frames cut into pieces and frames sent several in one
write, keys and values of arbitrary bytes and of the largest allowed sizes, lifetimes in seconds and in milliseconds
observed before and after they pass, counters near the ends of the range of R18, clients that read slowly or not at all
while others are served, many clients at once incrementing one key, long runs that store far more than they keep, and
the stop by SIGTERM with connections open — with the expected transcripts derived from section 3 and compared as R31
says, under the limits of section 4. The visible samples are a small part of the same suite; a program that passes them
has not thereby passed the hidden tests.

## 8. Amendments

None yet.
