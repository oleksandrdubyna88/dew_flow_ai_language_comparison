# Task 08 — Ledger API

| | |
|---|---|
| Kind | a long-running HTTP server that serves until it is stopped by SIGTERM |
| Gate verdict | `proceed` · 2026-11-17 |
| Freeze tag | `spec-frozen-B-20261118` |
| Freeze rule | this text changes only by a dated amendment from its authors |
| Amendments | none yet |

Every numbered sentence (**R1** … **R38**) is a requirement. Hidden tests refer to these numbers; nothing outside them is
tested.

## 1. What the program does

**R1.** The program is the HTTP service of a ledger in one currency. A client opens *accounts*, each with a
*balance*; moves money between two accounts with a *transfer*, which charges a *fee* to the paying account and books
the fee as *collected*; and protects every transfer with an *idempotency key*, so that a retried request has one
effect at most. All state — accounts, transfers, keys and the collected fees — lives in the process's memory and
nowhere else: a fresh process starts with no account, no transfer, no key and `0.00` collected, and nothing survives
the process.

### 1.1 Money

**R2.** A *money value* is a whole number of hundredths of the currency unit, from 0 to 99 999 999 999 999 999
hundredths: 0.00 to 999 999 999 999 999.99. It is written, in a request and in a reply, as a JSON string in exactly
this grammar: the whole part, either `0` or a digit `1`–`9` followed by 0 to 14 digits `0`–`9`; a full stop `.`; and
exactly two digits `0`–`9`. So `0.00`, `0.01`, `100.00`, `999999999999999.99` and `20.50` are money values, and `100`,
`100.5`, `100.500`, `0100.00`, `+1.00`, `-1.00`, `1e2`, `100,00`, ` 100.00`, `.50` and `100.00 ` are not. Every value a
money value can take is written in exactly one way, and the service writes every money value in this grammar.

**R3.** Arithmetic on money values is exact: a sum or difference of money values is the money value of the sum or
difference of their hundredths, with no rounding and no loss of digits at any magnitude of R2. The one operation that
rounds is the fee (R6).

### 1.2 Accounts

**R4.** An *account* has an `id` and a `balance`. The `id` is a UUID chosen by the service — 8, 4, 4, 4 and 12
hexadecimal digits `0`–`9`, `a`–`f` in lower case, joined by `-` — different from the id of every other account the
process has created; it never changes. The `balance` is a money value (R2): the account's *opening balance* when the
account is created, then changed only by transfers (R5). An *account object* is the JSON object `{"account": {"id":
<id>, "balance": <money>}}`.

### 1.3 Transfers and fees

**R5.** A *transfer* moves an `amount` from the account `from` to the account `to` and charges `from` a `fee`: when it
is applied, the balance of `from` decreases by `amount` plus `fee`, the balance of `to` increases by `amount`, and the
*collected* total increases by `fee`, all three at once. It has an `id`, a UUID of the form of R4 different from the id
of every other transfer the process has created. A *transfer object* is the JSON object `{"transfer": {"id": <id>,
"from": <id>, "to": <id>, "amount": <money>, "fee": <money>}}`, `from` and `to` being the ids of the two accounts.

**R6.** The *fee* of a transfer is one half of one percent of its amount — the amount's hundredths multiplied by 5
and divided by 1 000 — rounded to a whole number of hundredths: to the nearest one, and when the exact value lies
halfway between two, to the larger one. So the fee of `100.00` is `0.50`; of `40.00` is `0.20`; of `1.00` is `0.01`
(from 0.005); of `3.00` is `0.02` (from 0.015); of `7.00` is `0.04` (from 0.035); of `0.99` is `0.00` (from 0.00495);
of `1.99` is `0.01` (from 0.00995); of `333.33` is `1.67` (from 1.66665); and of `0.01` is `0.00`.

**R7.** A balance is never negative: a transfer is applied only when the balance of `from` is at least `amount` plus
`fee` (R19). At every moment the sum of all balances plus the collected total equals the sum of the opening balances
of all accounts.

### 1.4 Idempotency keys

**R8.** An *idempotency key* is the value of the request header `Idempotency-Key` of a `POST /transfers`: a text of
1 to 64 characters from `A`–`Z`, `a`–`z`, `0`–`9`, `.`, `_`, `~` and `-`, compared exactly, character for character.
Keys are one namespace for the whole process, not per account.

**R9.** The first `POST /transfers` with a given well-formed key that the service handles and does not answer 413 —
its reply is a 201 or a 422 — is *the request that fixed the key*: the service records under the key the bytes of that
request's body and the reply it gave (status and body). Every later `POST /transfers` with the same key has no
effect on any account, transfer or collected total and is answered by the record: when its body equals the recorded
body byte for byte, the reply is the recorded reply, identical in status and body (so a replayed 201 carries the same
transfer `id`, and a replayed 422 stays a 422 even when the condition that caused it no longer holds); when its body
differs in any byte, the reply is 409 (R18) and the record stays as it was. A request answered 413 (R24) does not fix
a key and is not recorded.

**R10.** When several `POST /transfers` with the same key arrive at the same time, exactly one of them fixes the key
and is applied as R9 says, and every other one is answered as a later request with that key: a reply identical to the
fixed one when the bodies are equal, 409 when they differ. Which request fixes the key is not defined.

## 2. Start, readiness and stop

**R11.** The environment variable `PORT` holds a TCP port number; the program listens on it, and the tests connect to
it at 127.0.0.1.

**R12.** The program is ready when it accepts a TCP connection on that port. A test waits at most 10 seconds for
readiness; a program that is not ready by then fails that test.

**R13.** A test ends by sending the signal SIGTERM to the program. The program must then exit with status 0 within 5
seconds, leaving no process it started still running. A program still running after 5 seconds is killed and fails
that test, and so does one that exits with another status or leaves a process running. SIGTERM may arrive while
client connections are open; the program may close them.

## 3. Interface

### 3.1 HTTP, JSON and how replies are compared

**R14.** The service speaks HTTP/1.1. A client may send several requests one after another on one connection, and
the service keeps a connection open after a reply for the client's next request, except as R24 allows. The tests send
these requests and no other:

| Request | Headers | Body | Success |
|---|---|---|---|
| `POST /accounts` | `content-type: application/json` | `{"openingBalance": <money>}` | 201, an account object (R20) |
| `GET /accounts/<id>` | — | — | 200, an account object (R21) |
| `POST /transfers` | `content-type: application/json`, `Idempotency-Key` (R8) | `{"from": <id>, "to": <id>, "amount": <money>}` | 201, a transfer object (R22) |
| `GET /transfers/<id>` | — | — | 200, a transfer object (R23) |
| `GET /fees` | — | — | 200, the collected total (R25) |

A request with a body carries the `content-type` header shown and a body that is one JSON object in UTF-8 with
exactly the members shown, each present and holding a JSON value of any type (R18 says how a value that is not what
the table shows is answered); the `Idempotency-Key` header may be absent or malformed on a `POST /transfers` (R18) and
is sent on no other request. A request without a body has no body. A path segment `<id>` is a text of 1 to 64
characters from `A`–`Z`, `a`–`z`, `0`–`9` and `-`; no request has a query.

**R15.** Every reply has a JSON body and the header `content-type` with the value `application/json;
charset=utf-8`. Every reply is compared on its status code, on each response header that a requirement of this spec
names, and on its body; other headers are not compared. The body is compared byte for byte, except where a
requirement of this spec names another comparison. Nothing else is normalized.

**R16.** The body of every reply is compared as a JSON value: the members of an object may come in any order, numbers
are compared by their exact decimal value with no rounding (so `2` equals `2.0` and `-0` equals `0`, and two numbers
that differ in any digit differ), and whitespace outside strings is ignored; arrays keep their order unless a
requirement says otherwise.

**R17.** The service writes a money value as a JSON string in the grammar of R2 and an id as a JSON string in the
form of R4; it writes no JSON number in this task. A value is never left out of an object of R4, R5 or R25.

### 3.2 Errors

**R18.** A reply that is not a success of R14 is an *error reply*: its body is `{"error": "<code>"}` with exactly one
code, chosen with its status from this table:

| Status | Code | When |
|---|---|---|
| 413 | `payload-too-large` | the request's body is longer than 4 096 bytes (R24) |
| 400 | `idempotency-key-missing` | `POST /transfers` without an `Idempotency-Key` header, or with one whose value is not a key of R8 |
| 409 | `idempotency-key-mismatch` | `POST /transfers` with a key already fixed, and a body that differs from the recorded one (R9) |
| 404 | `not-found` | `GET /accounts/<id>` or `GET /transfers/<id>` where no account, or no transfer, has that id |
| 422 | `invalid-amount` | `openingBalance` or `amount` is not a money value of R2; or `amount` is `0.00` |
| 422 | `same-account` | `from` and `to` are equal: both are JSON strings with the same characters (a value that is not a JSON string is equal to no other value and is the id of no account) |
| 422 | `unknown-account` | `from`, or else `to`, is not the id of an account |
| 422 | `insufficient-funds` | the balance of `from` is less than `amount` plus the fee of `amount` (R6, R7) |

**R19.** For a `POST /transfers` the conditions are checked in this order, and the first that holds decides the
reply; the later ones are not examined: 413; then 400; then, when the key is fixed, the replay or the 409 of R9; then
422 `invalid-amount`, `same-account`, `unknown-account` (`from` before `to`) and `insufficient-funds` in this order.
For a `POST /accounts`: 413, then 422. An error reply changes no account, transfer or collected total; of the error
replies only those of R9 are recorded under a key.

### 3.3 Accounts

**R20.** `POST /accounts` creates an account whose `balance` is `openingBalance` (R2; `0.00` is allowed) and replies
201 with the account object. No idempotency key applies: every such request creates one account.

**R21.** `GET /accounts/<id>` replies 200 with the account object of the account that has that id, showing its
current balance.

### 3.4 Transfers

**R22.** `POST /transfers` whose key is not yet fixed and whose body passes the checks of R19 applies the transfer of
`amount` from `from` to `to` with the fee of R6 (R5), records the key (R9), and replies 201 with the transfer object.

**R23.** `GET /transfers/<id>` replies 200 with the transfer object of the transfer that has that id; a transfer that
was applied is kept for the life of the process.

**R24.** A request whose body is longer than 4 096 bytes is answered 413 (R18) whatever else it holds: the body is
not examined, no key is read or recorded, nothing changes. After the 413 reply the service may close the connection
or keep it open.

### 3.5 Fees

**R25.** `GET /fees` replies 200 with `{"fees": {"collected": <money>}}`, the sum of the fees of every transfer
applied so far (`0.00` for none).

### 3.6 Concurrency

**R26.** Requests may arrive at the same time on different connections. The service applies them as if one at a
time, in some order: every request is applied whole — the checks of R19, the three balance changes of R5 and the
recording of R9 together — and every reply shows a state that some sequence of whole requests produced. So when k ≥ 10
clients send, at the same instant, `POST /transfers` with k different keys, each moving `1.00` from one account with
balance `10.00` to another account, the number of 201 replies is the largest n such that n × `1.01` is at most
`10.00` — that is 9 — and the rest are 422 `insufficient-funds`; the paying account's balance is then `0.91`, the
receiving account's `9.00` more than before, and the collected total `0.09` more. And when k clients send, at the same
instant, `POST /transfers` with one key and equal bodies, every reply is identical and the transfer is applied once
(R10).

### 3.7 Not defined, not compared

**R27.** A request or input that violates any of R14, R31 and R32 does not occur in any visible or hidden test. What the
program does with it is unspecified and is never tested.

**R28.** Standard output is compared only where a requirement of this spec says what it holds; otherwise it is
recorded, not judged.

**R29.** Standard error is not compared and may hold anything; it is recorded, not judged. Only its size is bounded: a
run that writes more than 256 MiB to standard error is stopped, and that test fails.

## 4. Limits

**R30.** Each test starts the program once, in a fresh process, and drives one scenario against it under a bound on the
program's memory. The bound is the one the task's statement states: one registered value, the same for every daemon and
web task, which the test harness adds to the statement from its registration, so a spec never writes the number. It is
registered before the freeze, only once the reference solutions of every daemon and web task pass every test under it in
both stacks, and enforced as F-MEMORY says. The load scenarios are hidden scenarios of up to the bounds of R31 and R32,
each lasting at most 600 seconds; the bound on memory holds in every test, whether it is a load scenario or not. Memory
is measured as the peak memory of the program's own control group (cgroup `memory.peak`), which counts the program and
every process it starts, including the page cache and kernel memory charged to them, with no swap. The program's memory
reaches the limit when the system kills a process of the program's control group at the limit, or when the program's own
runtime stops it for lack of memory; either way the run is stopped and fails that test. The test machine caps each
runtime's own heap at 90 % of the limit, by that runtime's own setting. The tests run only on a machine that can read
this peak, and no other measurement of memory passes or fails a test. The program and every process it starts may hold
at most 512 processes and threads together. Beyond that the system refuses to create another, and a run that needs more
fails that test.

**R31.** The load of a test stays within these bounds: at most 16 client connections are open at once; at most 20 000
requests are sent in one test; a request body is at most 65 536 bytes (so a body of 4 097 to 65 536 bytes occurs and
is answered 413); one test creates at most 2 000 accounts and uses at most 20 000 different keys.

**R32.** Every value stays within its range: no transfer makes a balance exceed the largest money value of R2, and
the sum of the opening balances of a test's accounts is at most that value; a JSON string in a request body is at most
4 096 bytes long; an `Idempotency-Key` header value, well formed or not, is at most 256 bytes long and holds only
printable ASCII characters.

**R33.** Every step of a scenario that waits — for readiness, for a reply, for the exit — has a deadline, and a step
that does not complete by its deadline fails the test. The visible samples state their deadlines in their files; the
deadlines of the hidden scenarios are fixed when the suite is calibrated, so that the reference solutions in both
stacks meet each of them with margin.

## 5. Build and run — the workspace contract

**R34.** The workspace holds `STATEMENT.md`, `samples/`, `runner/`, `build.sh`, `run.sh`, `PACKAGES.md` and the
toolchain pin files named in `build.sh`. These are frozen inputs: do not change, rename or remove them. The program's
sources go where `build.sh` and `run.sh` expect them; the two scripts say where.

**R35.** `bash build.sh`, run in the workspace directory, builds the program with no network access and with no
package outside the set listed in `PACKAGES.md`. It must exit 0 within 600 seconds. It runs once before the tests; a
build that does not exit 0 fails every test.

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
# Two accounts, one transfer with its fee; the balances and the collected
# total afterwards.
default within 2s
start ready port within 10s
connect c http

request c POST "/accounts"
  header "content-type" "application/json"
  body json {"openingBalance":"100.00"}
recv c capture a json $.account.id as uuid

request c POST "/accounts"
  header "content-type" "application/json"
  body json {"openingBalance":"0.00"}
recv c capture b json $.account.id as uuid

request c POST "/transfers"
  header "content-type" "application/json"
  header "idempotency-key" "k-1"
  body json {"from":"${a}","to":"${b}","amount":"40.00"}
recv c capture t json $.transfer.id as uuid

request c GET "/accounts/${a}"
recv c

request c GET "/accounts/${b}"
recv c

request c GET "/fees"
recv c

signal TERM
wait-exit within 5s
```

`samples/s01.out`

```text
transcript 2
ready
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${a}","balance":"100.00"}}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${b}","balance":"0.00"}}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"transfer":{"id":"${t}","from":"${a}","to":"${b}","amount":"40.00","fee":"0.20"}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${a}","balance":"59.80"}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${b}","balance":"40.00"}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"fees":{"collected":"0.20"}}
exit 0
```

How `s01` is built: the fee of `40.00` is `0.20` (R6); the paying account loses `40.20`, the receiving one gains
`40.00`, and `59.80` + `40.00` + `0.20` = `100.00` (R7).

### s02

`samples/s02.in`

```text
scenario 2
# One key: the same body again gives the same reply and no second effect; a
# different body gives 409; a new key gives a new transfer.
default within 2s
start ready port within 10s
connect c http

request c POST "/accounts"
  header "content-type" "application/json"
  body json {"openingBalance":"10.00"}
recv c capture a json $.account.id as uuid

request c POST "/accounts"
  header "content-type" "application/json"
  body json {"openingBalance":"0.00"}
recv c capture b json $.account.id as uuid

request c POST "/transfers"
  header "content-type" "application/json"
  header "idempotency-key" "pay-7"
  body json {"from":"${a}","to":"${b}","amount":"2.00"}
recv c capture t json $.transfer.id as uuid

request c POST "/transfers"
  header "content-type" "application/json"
  header "idempotency-key" "pay-7"
  body json {"from":"${a}","to":"${b}","amount":"2.00"}
recv c

request c POST "/transfers"
  header "content-type" "application/json"
  header "idempotency-key" "pay-7"
  body json {"from":"${a}","to":"${b}","amount":"3.00"}
recv c

request c GET "/accounts/${a}"
recv c

request c GET "/transfers/${t}"
recv c

request c POST "/transfers"
  header "content-type" "application/json"
  header "idempotency-key" "pay-8"
  body json {"from":"${a}","to":"${b}","amount":"2.00"}
recv c capture u json $.transfer.id as uuid

request c GET "/accounts/${b}"
recv c

signal TERM
wait-exit within 5s
```

`samples/s02.out`

```text
transcript 2
ready
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${a}","balance":"10.00"}}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${b}","balance":"0.00"}}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"transfer":{"id":"${t}","from":"${a}","to":"${b}","amount":"2.00","fee":"0.01"}}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"transfer":{"id":"${t}","from":"${a}","to":"${b}","amount":"2.00","fee":"0.01"}}
reply c 409
  header "content-type" "application/json; charset=utf-8"
  body json {"error":"idempotency-key-mismatch"}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${a}","balance":"7.99"}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"transfer":{"id":"${t}","from":"${a}","to":"${b}","amount":"2.00","fee":"0.01"}}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"transfer":{"id":"${u}","from":"${a}","to":"${b}","amount":"2.00","fee":"0.01"}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${b}","balance":"4.00"}}
exit 0
```

How `s02` is built: the second request with `pay-7` has the same body as the first and gets the recorded reply, with
the same transfer id, and nothing moves; the third has another amount under the same key and gets 409 (R9). The
balance `7.99` shows one transfer of `2.00` with its fee `0.01`. The key `pay-8` is new, so its transfer is applied,
and the receiving account holds `4.00`.

### s03

`samples/s03.in`

```text
scenario 2
# Each error of a transfer; a failed key replayed after the funds arrived
# stays failed; an unknown transfer id.
default within 2s
start ready port within 10s
connect c http

request c POST "/accounts"
  header "content-type" "application/json"
  body json {"openingBalance":"2.00"}
recv c capture a json $.account.id as uuid

request c POST "/accounts"
  header "content-type" "application/json"
  body json {"openingBalance":"5.00"}
recv c capture b json $.account.id as uuid

request c POST "/transfers"
  header "content-type" "application/json"
  body json {"from":"${a}","to":"${b}","amount":"2.00"}
recv c

request c POST "/transfers"
  header "content-type" "application/json"
  header "idempotency-key" "e1"
  body json {"from":"${a}","to":"${b}","amount":"1.5"}
recv c

request c POST "/transfers"
  header "content-type" "application/json"
  header "idempotency-key" "e2"
  body json {"from":"${a}","to":"${a}","amount":"2.00"}
recv c

request c POST "/transfers"
  header "content-type" "application/json"
  header "idempotency-key" "e3"
  body json {"from":"${a}","to":"no-such-account","amount":"2.00"}
recv c

request c POST "/transfers"
  header "content-type" "application/json"
  header "idempotency-key" "e4"
  body json {"from":"${a}","to":"${b}","amount":"2.00"}
recv c

request c POST "/transfers"
  header "content-type" "application/json"
  header "idempotency-key" "e5"
  body json {"from":"${b}","to":"${a}","amount":"2.00"}
recv c capture t json $.transfer.id as uuid

request c POST "/transfers"
  header "content-type" "application/json"
  header "idempotency-key" "e4"
  body json {"from":"${a}","to":"${b}","amount":"2.00"}
recv c

request c GET "/accounts/${a}"
recv c

request c GET "/accounts/${b}"
recv c

request c GET "/transfers/00000000-0000-0000-0000-000000000000"
recv c

signal TERM
wait-exit within 5s
```

`samples/s03.out`

```text
transcript 2
ready
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${a}","balance":"2.00"}}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${b}","balance":"5.00"}}
reply c 400
  header "content-type" "application/json; charset=utf-8"
  body json {"error":"idempotency-key-missing"}
reply c 422
  header "content-type" "application/json; charset=utf-8"
  body json {"error":"invalid-amount"}
reply c 422
  header "content-type" "application/json; charset=utf-8"
  body json {"error":"same-account"}
reply c 422
  header "content-type" "application/json; charset=utf-8"
  body json {"error":"unknown-account"}
reply c 422
  header "content-type" "application/json; charset=utf-8"
  body json {"error":"insufficient-funds"}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"transfer":{"id":"${t}","from":"${b}","to":"${a}","amount":"2.00","fee":"0.01"}}
reply c 422
  header "content-type" "application/json; charset=utf-8"
  body json {"error":"insufficient-funds"}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${a}","balance":"4.00"}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${b}","balance":"2.99"}}
reply c 404
  header "content-type" "application/json; charset=utf-8"
  body json {"error":"not-found"}
exit 0
```

How `s03` is built: `1.5` is not in the grammar of R2. With `2.00` in the paying account, a transfer of `2.00` needs
`2.01` and fails under `e4`; the transfer under `e5` then brings the account to `4.00`, but the replay of `e4` with the
same body returns the recorded 422 (R9). The second account paid `2.00` and the fee `0.01`.

### s04

`samples/s04.in`

```text
scenario 2
# Two clients send one key with equal bodies at the same instant: one transfer,
# two identical replies.
default within 2s
start ready port within 10s
connect x http
connect y http

request x POST "/accounts"
  header "content-type" "application/json"
  body json {"openingBalance":"50.00"}
recv x capture a json $.account.id as uuid

request x POST "/accounts"
  header "content-type" "application/json"
  body json {"openingBalance":"0.00"}
recv x capture b json $.account.id as uuid

together
  request x POST "/transfers"
    header "content-type" "application/json"
    header "idempotency-key" "race-1"
    body json {"from":"${a}","to":"${b}","amount":"20.00"}
  request y POST "/transfers"
    header "content-type" "application/json"
    header "idempotency-key" "race-1"
    body json {"from":"${a}","to":"${b}","amount":"20.00"}
end
recv x capture t json $.transfer.id as uuid
recv y

request x GET "/accounts/${a}"
recv x

request y GET "/accounts/${b}"
recv y

request x GET "/fees"
recv x

signal TERM
wait-exit within 5s
```

`samples/s04.out`

```text
transcript 2
ready
reply x 201
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${a}","balance":"50.00"}}
reply x 201
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${b}","balance":"0.00"}}
reply x 201
  header "content-type" "application/json; charset=utf-8"
  body json {"transfer":{"id":"${t}","from":"${a}","to":"${b}","amount":"20.00","fee":"0.10"}}
reply y 201
  header "content-type" "application/json; charset=utf-8"
  body json {"transfer":{"id":"${t}","from":"${a}","to":"${b}","amount":"20.00","fee":"0.10"}}
reply x 200
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${a}","balance":"29.90"}}
reply y 200
  header "content-type" "application/json; charset=utf-8"
  body json {"account":{"id":"${b}","balance":"20.00"}}
reply x 200
  header "content-type" "application/json; charset=utf-8"
  body json {"fees":{"collected":"0.10"}}
exit 0
```

How `s04` is built: whichever request fixed the key, both replies carry the one transfer's id (R10), and the balances
show one transfer of `20.00` with its fee `0.10`.

## 7. Hidden tests

**R38.** Hidden tests exist. They are scenarios of the kind shown in section 6: accounts with opening balances across
the range of R2, transfers whose amounts exercise the fee of R6 and the grammar of R2, keys replayed with equal and
with different bodies after a 201 and after an error, every error of R18 with the state it leaves, bodies above the
size of R24, reads of accounts, transfers and the collected total, requests sent by several clients at the same
instant with one key and with different keys, the readiness of section 2 and the stop by SIGTERM — all within the
bounds of section 4, with expected replies computed from sections 1 and 3 and compared as 3.1 says. The visible
samples are a small part of the same suite; a program that passes them has not thereby passed the hidden tests.

## 8. Amendments

None yet.

<!-- stack-sentence -->
Use C# 14 on .NET 10 (SDK 10.0.403 is installed).

<!-- memory-sentence -->
The program and its descendants run under a 512 MiB (536,870,912-byte) cgroup memory limit, including runtime overhead. Build resources are separate.

<!-- heap-sentence -->
The .NET garbage-collected heap is capped by DOTNET_GCHeapHardLimit=0x1ccccccc (483,183,820 bytes, 90 % of the limit).

<!-- framework-sentence -->
Use ASP.NET Core Minimal API (the shared framework of SDK 10.0.403) for the HTTP server.
