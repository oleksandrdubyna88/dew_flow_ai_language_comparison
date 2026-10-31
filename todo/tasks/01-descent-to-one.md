# Task 01 — Descent to one

| | |
|---|---|
| Task | 01 of 10 — a program that reads standard input once and writes standard output once |
| Gate verdict | not yet gated (the document gate, a consultant and the bias review run before the freeze) |
| Freeze tag | none yet |
| Freeze rule | frozen: changes only by a dated amendment before the v2 pre-registration |
| Amendments | none (section 8) |

Every numbered sentence (**R1** … **R17**) is a requirement. Hidden tests refer to these numbers; nothing outside them is
tested.

## 1. What the program does

**R1.** The program reads one positive integer n and prints the sequence of values that starts at n and ends at 1,
built by this step rule: from the current value v, the next value is v / 2 when v is even and 3·v + 1 when v is odd.
The rule is applied to the latest value, again and again, until the value 1 is produced. The value 1 is the last term;
the rule is not applied to it.

**R2.** The first term is n itself, also when n is already 1: for n = 1 the whole sequence is the single term 1.

**R3.** For every n allowed by section 2 the sequence reaches 1. The program may rely on this and need not detect or
report a sequence that would not.

**R4.** Every term is printed as the exact value the step rule defines. A term is written as a positive integer in
decimal notation: digits only, no sign, no leading zeros, no digit grouping, no fractional part and no exponent form.

## 2. Input

**R5.** Standard input holds exactly one line: the decimal digits of n, with 1 ≤ n ≤ 1 000 000, written with no sign,
no leading zeros and no spaces, followed by one line feed (byte 0x0A). Nothing else is present.

**R6.** Input that violates R5 does not occur in any visible or hidden test. What the program does with such input is
unspecified and is never tested.

## 3. Output

**R7.** Standard output holds exactly one line: the terms of the sequence in order, from n to 1, each separated from
the next by exactly one space (byte 0x20), and the last term followed by exactly one line feed (byte 0x0A). There is no
leading space, no trailing space, no second line and no other byte.

**R8.** The output is compared with the expected output byte for byte. Nothing is normalized: a missing or extra line
feed, a trailing space or a carriage return makes the test fail.

**R9.** Standard error is not compared and may hold anything; it is recorded, not judged.

**R10.** The process exits with status 0 after writing its output. Any other exit status fails the test, whatever was
written.

## 4. Limits

**R11.** Each test runs the program once, in a fresh process, under a limit on wall-clock time and a limit on memory.
The exact values are registered when the hidden suite is calibrated against reference solutions in both stacks, before
any session, and are given to the test runner, not to the program's author. The registered values are never below these
floors: **1 second** of wall-clock time per run and **256 MB** of memory. (The registration rule: at least three times
the wall time of the slower reference, and at least twice the peak memory of the larger reference.)

**R12.** A run that exceeds the time limit, exceeds the memory limit, exits with a non-zero status or prints anything
other than the expected bytes fails that test.

## 5. Build and run — the workspace contract

**R13.** The workspace holds `STATEMENT.md`, `samples/`, `build.sh`, `run.sh`, `PACKAGES.md` and the toolchain pin
files of the stack. These are frozen inputs: do not change, rename or remove them. The program's sources go where
`build.sh` and `run.sh` expect them; the two scripts say where.

**R14.** `bash build.sh`, run in the workspace directory, builds the program with no network access and with no
package outside the set listed in `PACKAGES.md`. It must exit 0 within 600 seconds. It runs once before the tests; a
build that does not exit 0 fails every test.

**R15.** `bash run.sh`, run in the workspace directory with no arguments, starts the built program once: the test's
input is its standard input, its standard output is collected whole, its exit status is read. One fresh process per
test; nothing carries over from one run to the next.

**R16.** The tests run offline, on a pristine copy of the workspace, after the build. The program must not depend on
the network, on files outside the workspace, or on anything written during an earlier run.

## 6. Visible samples

The same three cases are in `samples/`, one input file and one expected-output file per case (`samples/<id>.in`,
`samples/<id>.out`). The bytes below are exact; `⏎` marks the single line feed that ends each line.

| Id | Standard input | Expected standard output |
|---|---|---|
| `s01` | `1⏎` | `1⏎` |
| `s02` | `16⏎` | `16 8 4 2 1⏎` |
| `s03` | `7⏎` | `7 22 11 34 17 52 26 13 40 20 10 5 16 8 4 2 1⏎` |

How `s03` is built: 7 is odd, so 3·7 + 1 = 22; 22 is even, so 11; 11 is odd, so 34; and so on down to 16, 8, 4, 2, 1.

## 7. Hidden tests

**R17.** Hidden tests exist. They are inputs that satisfy section 2, chosen across the whole range 1 ≤ n ≤ 1 000 000,
with expected outputs computed from section 1 and compared as section 3 says. The visible samples are a small part of
the same suite; a program that passes them has not thereby passed the hidden tests.

## 8. Amendments

None.
