# Task 04 — Rule API

| | |
|---|---|
| Kind | a long-running HTTP server that serves until it is stopped by SIGTERM |
| Gate verdict | not yet gated |
| Freeze tag | none yet |
| Freeze rule | this text changes only by a dated amendment from its authors |
| Amendments | none yet |

Every numbered sentence (**R1** … **R41**) is a requirement. Hidden tests refer to these numbers; nothing outside them is
tested.

## 1. What the program does

**R1.** The program is an HTTP service over a registry of named objects that this section defines completely. A client
submits a *rule*: a list of *steps*, each naming one object of the registry and one *member* of that object — a property
to read, a property to set to a given value, or an operation to call with given arguments. The service applies the steps
in order, changing the objects where a step says so, and answers with one result per step. The registry lives in the
service's memory and nowhere else; a fresh process starts from the initial state of 1.3.

### 1.1 Types and values

**R2.** Every property, parameter and result has one of five types. `integer`: a whole number from
−9 223 372 036 854 775 808 to 9 223 372 036 854 775 807 (−2^63 to 2^63 − 1), every value exact. `decimal`: a number
with a whole part and at most four fraction digits, every value exact — arithmetic on decimals is decimal arithmetic,
so 0.1 + 0.2 is 0.3 and 21.5 + (−0.25) is 21.25; the magnitude of every decimal in this task stays below
1 000 000 000 (R36). `string`: a sequence of characters. `boolean`: true or false. `list of strings`: an ordered list
of strings. A type written "or null" (for example `string or null`) also admits the value null; no other type admits it.

**R3.** Every object has a property `id`, an `integer` from 1 to 2^63 − 1, fixed in 1.3. Two objects never share an id.

### 1.2 The three types of objects

**R4.** An object's members are exactly the properties and operations that the table of its type lists below; it has no
other member, under any name. A property has a type and is read-write or read-only. An operation has a list of
parameters, each with a name and a type, and a result type; `→ null` means its result is always null. "Fails" in a
row means the failure `operation` of R21.

Type `registry` — the registry itself, one object:

| Member | Kind | Type or signature | Meaning |
|---|---|---|---|
| `id` | property, read-only | `integer` | the object's identifier |
| `size` | property, read-only | `integer` | the number of objects in the registry, this one included: 5 |
| `find` | operation | `(id: integer) → string` | the name of the object whose `id` equals the argument; fails when no object has that id |
| `names` | operation | `() → list of strings` | the names of all objects, in the order of the table of 1.3 |

Type `gauge`:

| Member | Kind | Type or signature | Meaning |
|---|---|---|---|
| `id` | property, read-only | `integer` | the object's identifier |
| `label` | property, read-write | `string or null` | a free text, or null |
| `unit` | property, read-only | `string` | the unit of `reading` |
| `reading` | property, read-write | `decimal` | the measured value |
| `offset` | property, read-write | `decimal` | a correction added to `reading` |
| `enabled` | property, read-write | `boolean` | whether `shift` may change `reading` |
| `corrected` | operation | `() → decimal` | `reading` + `offset`; changes nothing |
| `rounded` | operation | `(digits: integer) → decimal` | the value of `corrected` rounded to `digits` fraction digits as R8 says; fails when `digits` is below 0 or above 4; changes nothing |
| `shift` | operation | `(delta: decimal) → decimal` | when `enabled` is true: adds `delta` to `reading` and returns the new `reading`; when `enabled` is false: fails and changes nothing |
| `reset` | operation | `() → null` | sets `reading` and `offset` to 0; `label`, `unit` and `enabled` stay |

Type `counter`:

| Member | Kind | Type or signature | Meaning |
|---|---|---|---|
| `id` | property, read-only | `integer` | the object's identifier |
| `label` | property, read-write | `string or null` | a free text, or null |
| `count` | property, read-only | `integer` | the current count |
| `stride` | property, read-write | `integer` | what one `increment` adds |
| `limit` | property, read-write | `integer or null` | the largest `count` an `increment` may reach; null means no limit |
| `increment` | operation | `() → integer` | when `limit` is null, or `count` + `stride` is not above `limit`: adds `stride` to `count` and returns the new `count`; otherwise fails and changes nothing |
| `cap` | operation | `(limit: integer or null) → integer or null` | sets `limit` to the argument and returns the new `limit` |
| `reset` | operation | `() → null` | sets `count` to 0; `label`, `stride` and `limit` stay |

**R5.** A member name denotes the member that the table of the object's own type lists under that name, and nothing
else: `reset` on a gauge is the gauge's `reset` and `reset` on a counter the counter's; `find` and `names` exist on the
registry object only; `corrected` exists on gauges only.

### 1.3 The registry and its initial state

**R6.** The registry holds exactly these five objects, named and initialised as follows. No object is ever added,
renamed or removed; the table is the whole registry.

| Name | Type | `id` | The other properties at start |
|---|---|---|---|
| `registry` | `registry` | 1 | `size` 5 |
| `boiler` | `gauge` | 9007199254740993 | `label` `"boiler room"`, `unit` `"C"`, `reading` 21.5, `offset` −0.25, `enabled` true |
| `attic` | `gauge` | 72057594037927937 | `label` null, `unit` `"C"`, `reading` −3.75, `offset` 0.5, `enabled` false |
| `visits` | `counter` | 1000000007 | `label` `"front door"`, `count` 0, `stride` 1, `limit` null |
| `tickets` | `counter` | 9223372036854775807 | `label` `"day pass"`, `count` 10, `stride` 5, `limit` 25 |

**R7.** Every test starts a fresh process (R39), so every test begins from this state. The state changes only through
the steps of rules (section 3): a `set` step and the operations whose rows say they change a property.

### 1.4 Rounding

**R8.** `rounded` with `digits` = d gives, among the decimals with at most d fraction digits, the one nearest to the
value of `corrected`; when that value lies exactly halfway between two of them, the one with the larger magnitude (the
one farther from zero) is chosen. With d = 0: 2.5 gives 3, −2.5 gives −3, 2.4 gives 2 and −2.6 gives −3. With d = 2:
0.125 gives 0.13 and −0.125 gives −0.13. With d = 1: 21.25 gives 21.3 and 21.24 gives 21.2. A value that already has at
most d fraction digits is returned unchanged.

## 2. Start, readiness and stop

**R9.** The environment variable `PORT` holds a TCP port number; the program listens on it, and the tests connect to
it at 127.0.0.1.

**R10.** The program is ready when it accepts a TCP connection on that port. A test waits at most 10 seconds for
readiness; a program that is not ready by then fails that test.

**R11.** A test ends by sending the signal SIGTERM to the program. The program must then exit with status 0 within 5
seconds, leaving no process it started still running. A program still running after 5 seconds is killed and fails that
test, and so does one that exits with another status or leaves a process running. SIGTERM may arrive while client
connections are open; the program may close them.

## 3. Interface

### 3.1 HTTP, JSON and how replies are compared

**R12.** The service speaks HTTP/1.1. The tests send three kinds of request and no other: `POST /rules` (3.3),
`GET /objects/<name>` (3.4) and `GET /rules/<token>` (3.5). Every `POST /rules` carries the header
`content-type: application/json` and a body that is one JSON value in UTF-8; the `GET` requests carry no body. A client
may send several requests one after another on one connection, and the service keeps a connection open after a reply
for the client's next request.

**R13.** Every reply has a JSON body and the header `content-type` with the value `application/json; charset=utf-8`.
Every reply is compared on its status code, on each response header that a requirement of this spec names, and on its
body; other headers are not compared. The body is compared byte for byte, except where a requirement of this spec names
another comparison. Nothing else is normalized.

**R14.** The body of every reply is compared as a JSON value: the members of an object may come in any order, numbers
are compared by their exact decimal value with no rounding (so `2` equals `2.0` and `-0` equals `0`, and two numbers that
differ in any digit differ), and whitespace outside strings is ignored; arrays keep their order unless a requirement
says otherwise.

**R15.** The service writes values in JSON as follows. An `integer` is a JSON number written as decimal digits with an
optional leading minus sign, no fraction part and no exponent, denoting exactly the value: the id 9007199254740993 is
written `9007199254740993`. A `decimal` is a JSON number written as decimal digits with an optional leading minus sign
and an optional fraction part, no exponent, denoting exactly the value; since numbers are compared by value, `22` and
`22.0` are the same reply, and `22.0` and `22.01` are not. A `string` is a JSON string. A `boolean` is `true` or
`false`. A `list of strings` is a JSON array of JSON strings in the list's order. Null is written as JSON `null`, and
it is always written: a result or property whose value is null is present with the value `null`, never left out.

### 3.2 Names

**R16.** A request names an object by a string — the `object` member of a step, or the `<name>` segment of
`GET /objects/<name>` — and a member by the string value of a step's `get`, `set` or `call`. A name is matched against
the names of 1.2 and 1.3 exactly, character for character: letter case matters (`boiler`, `Boiler` and `BOILER` are
three different names, and only the first names an object), no character is trimmed, folded or replaced, and no name
matches anything but itself. A string that is not one of the five object names of 1.3 is an unknown object, whatever
else the string may be; a string that is not a member name in the table of the object's type is an unknown member of
that object, whatever else the string may be.

### 3.3 `POST /rules` — evaluate a rule

**R17.** The body is a JSON object `{"steps": [ <step>, … ]}` whose `steps` is an array of one or more steps. A step is
a JSON object of one of three shapes, where `object` and exactly one of `get`, `set` and `call` are present and are JSON
strings:

| Shape | Step | Meaning |
|---|---|---|
| read | `{"object": "<name>", "get": "<property>"}` | read the property |
| set | `{"object": "<name>", "set": "<property>", "value": <JSON value>}` | set the property to the value |
| call | `{"object": "<name>", "call": "<operation>", "args": [ <JSON value>, … ]}` | call the operation with the arguments, in order; `args` may be left out, and an absent `args` is an empty argument list |

**R18.** The steps are applied in the order of the array, each one against the state the steps before it left. A read
step's result is the property's current value. A set step converts `value` to the property's type (3.6), stores it, and
its result is the property's new value. A call step converts each argument to its parameter's type (3.6), runs the
operation, and its result is the operation's result — null for an operation whose result type is `→ null`.

**R19.** When every step succeeds, the reply has status 200 and the body `{"rule": "<token>", "results": [ {"value":
<value0>}, {"value": <value1>}, … ]}` with exactly one entry per step, in the order of the steps, `<valuei>` being the result of step
i written as R15 says. `rule` is a *token* the service chooses: a string of 1 to 64 characters from `A`–`Z`, `a`–`z`,
`0`–`9`, `.`, `_`, `~` and `-`, different from every token the process has issued before. The service keeps the body it
answered under that token for `GET /rules/<token>` (R24).

**R20.** When a step fails, the evaluation stops at that step: the steps before it have been applied and stay applied,
the failing step changes nothing, the steps after it are not applied, no token is issued and nothing is kept. The
reply's status is the failure's status (R21) and its body is `{"error": "<code>", "step": <index>}`, `<index>` being the
position of the failing step in `steps`, counted from 0. A failed request has no other effect: the service goes on
serving every later request exactly as if the failed request had been a rule of only the steps before the failing one.

**R21.** The failures of a step, their codes and their statuses:

| Code | Status | When |
|---|---|---|
| `unknown-object` | 404 | the step's `object` is not one of the five names of 1.3 (R16) |
| `unknown-member` | 404 | the step's `get`, `set` or `call` is not a member name in the table of the object's type (R16) |
| `not-a-property` | 400 | `get` or `set` names an operation |
| `not-callable` | 400 | `call` names a property |
| `read-only` | 400 | `set` names a read-only property |
| `arity` | 400 | the number of arguments differs from the number of the operation's parameters |
| `argument` | 400 | a set step has no `value` member; or `args` is present and is not a JSON array; or `value`, or an argument, cannot be converted to the type of the property or parameter as 3.6 says — for arguments, the first one that cannot |
| `operation` | 422 | the operation fails as its row in 1.2 says, the arguments having been converted |

**R22.** The conditions of R21 are checked in the order of its table, and the first that holds decides the failure; the
later ones are not examined. A call of an unknown member with the wrong number of arguments is `unknown-member`; a set
of a read-only property with an unconvertible value is `read-only`; a call with too many arguments, one of them
unconvertible, is `arity`.

### 3.4 `GET /objects/<name>` — read an object

**R23.** For an object name of 1.3, the reply has status 200 and the body `{"name": "<name>", "type": "<type>",
"properties": { … }}`, where `type` is the object's type name (`registry`, `gauge` or `counter`) and `properties` is a
JSON object with one member per property in the table of that type — `id` included — holding the property's current
value written as R15 says. For any other `<name>` the reply has status 404 and the body
`{"error": "unknown-object", "step": null}`.

### 3.5 `GET /rules/<token>` — read back an answered rule

**R24.** For a token the process has issued (R19), the reply has status 200 and, as its body, the same JSON value the
service answered to the `POST /rules` that issued the token — the same `rule`, the same `results` with the values of
that moment; a later change of the registry does not alter it. For any other `<token>` the reply has status 404 and the
body `{"error": "unknown-rule", "step": null}`.

### 3.6 Arguments and set values: conversion from JSON

**R25.** JSON `null` is accepted for a parameter or property whose type is written "or null", and denotes null; for a
parameter or property of any other type it is an `argument` failure. Null is a value and absence is not: `"args":
[null]` passes one argument, which is null; an absent `args` passes no argument, so `{"object": "visits", "call":
"cap"}` is an `arity` failure while `{"object": "visits", "call": "cap", "args": [null]}` sets `limit` to null; a set
step without a `value` member is an `argument` failure, not a set to null.

**R26.** A value for an `integer` parameter or property is either a JSON number written without a fraction part, or a
JSON string in the integer grammar: an optional minus sign `-`, then one or more decimal digits `0`–`9`, and nothing
else. The string denotes the integer its digits spell: `"9007199254740993"` and the number `9007199254740993` denote the
same value, `"-7"` denotes −7, `"007"` denotes 7. A JSON number with a fraction part (`2.5`), a string outside the
grammar (`"+7"`, `" 7"`, `"7 "`, `"7.0"`, `"7,0"`, `"1e3"`, `"1_000"`, `"seven"`, `""`), and a JSON value of any other
type (`true`, an array, an object) are `argument` failures.

**R27.** A value for a `decimal` parameter or property is either a JSON number, or a JSON string in the decimal
grammar: an optional minus sign `-`, then one or more decimal digits, then optionally a full stop `.` followed by one or
more decimal digits, and nothing else — no plus sign, no space, no exponent, no digit grouping, and no character but `.`
between the whole part and the fraction. The string denotes exactly the decimal its digits spell: `"22.75"` denotes
22.75 (twenty-two and three quarters), `"-0.5"` denotes −0.5, `"7"` denotes 7, `"1234.5"` denotes one thousand two
hundred thirty-four and a half. A string outside the grammar (`"22,75"`, `"22.75 "`, `" 22.75"`, `"+1"`, `"1e2"`,
`"1.234,5"`, `"."`, `"1."`, `".5"`, `""`) and a JSON value of any other type are `argument` failures.

**R28.** A value for a `string` parameter or property is a JSON string and nothing else; a value for a `boolean`
property is JSON `true` or `false` and nothing else. A value of another JSON type is an `argument` failure: the number
`5` is not the string `"5"`, the string `"true"` is not the boolean `true`, and `1` is not `true`.

### 3.7 Concurrency

**R29.** Requests may arrive at the same time on different connections. The service applies them as if one at a time,
in some order: every `POST /rules` is applied whole, so the steps of two rules never interleave, and every reply — to a
`POST` or to a `GET` — shows a state that some sequence of whole rules produced. So when k clients each send, at the
same instant, a rule of one `increment` on the same counter whose `limit` is null, the k results are k different values,
the start count plus `stride`, plus 2·`stride`, … plus k·`stride`, each client getting one of them; `count` afterwards is
the start count plus k·`stride`; and the k replies carry k different tokens.

### 3.8 Not defined, not compared

**R30.** A request or input that violates any of R12, R17, R35 and R36 does not occur in any visible or hidden test.
What the program does with it is unspecified and is never tested.

**R31.** Standard output is compared only where a requirement of this spec says what it holds; otherwise it is recorded,
not judged.

**R32.** Standard error is not compared and may hold anything; it is recorded, not judged. Only its size is bounded: a
run that writes more than 256 MiB to standard error is stopped, and that test fails.

## 4. Limits

**R33.** Each test starts the program once, in a fresh process, and drives one scenario against it under a bound on the
program's memory. The bound is the one the task's statement states: one registered value, the same for every daemon and
web task, which the test harness adds to the statement from its registration, so a spec never writes the number. It is
registered before the freeze, only once the reference solutions of every daemon and web task pass every test under it in
both stacks, and enforced as F-MEMORY says. The load scenarios are hidden scenarios of up to the bounds of R35 and R36,
each lasting at most 600 seconds; the bound on memory holds in every test, whether it is a load scenario or not. Memory
is measured as the peak memory of the program's own control group (cgroup `memory.peak`), which counts the program and
every process it starts, including the page cache and kernel memory charged to them, with no swap. The program's memory
reaches the limit when the system kills a process of the program's control group at the limit, or when the program's own
runtime stops it for lack of memory; either way the run is stopped and fails that test. The test machine caps each
runtime's own heap at 90 % of the limit, by that runtime's own setting. The tests run only on a machine that can read
this peak, and no other measurement of memory passes or fails a test. The program and every process it starts may hold
at most 512 processes and threads together. Beyond that the system refuses to create another, and a run that needs more
fails that test.

**R34.** Every step of a scenario that waits — for readiness, for a reply, for the exit — has a deadline, and a step
that does not complete by its deadline fails the test. The visible samples state their deadlines in their files; the
deadlines of the hidden scenarios are fixed when the suite is calibrated, so that the reference solutions in both stacks
meet each of them with margin.

**R35.** The load of a test stays within these bounds: at most 8 client connections are open at once; at most 10 000
requests are sent in one test; a request body is at most 8 192 bytes; a rule has from 1 to 50 steps; every name in a
request is 1 to 64 characters long, and the `<name>` of `GET /objects/<name>` holds only ASCII letters, digits, `_`,
`-` and `.`; the `<token>` of `GET /rules/<token>` has the form of a token (R19).

**R36.** Every value stays within its type: every `integer` in a request, and every `integer` a step produces, lies in
the range of R2; every `decimal` in a request, and every `decimal` a step produces, has at most four fraction digits and
a magnitude below 1 000 000 000; every `string` value is at most 256 characters long. A JSON number in a request is
written as decimal digits with an optional leading minus sign and an optional fraction part of one or more digits, never
with an exponent; a JSON number meant as an `integer` carries no fraction part.

## 5. Build and run — the workspace contract

**R37.** The workspace holds `STATEMENT.md`, `samples/`, `runner/`, `build.sh`, `run.sh`, `PACKAGES.md` and the
toolchain pin files named in `build.sh`. These are frozen inputs: do not change, rename or remove them. The program's
sources go where `build.sh` and `run.sh` expect them; the two scripts say where.

**R38.** `bash build.sh`, run in the workspace directory, builds the program with no network access and with no
package outside the set listed in `PACKAGES.md`. It must exit 0 within 600 seconds. It runs once before the tests; a
build that does not exit 0 fails every test.

**R39.** `bash run.sh`, run in the workspace directory with no arguments, starts the built program once per test, with
the environment section 2 describes. One fresh process per test; nothing carries over from one test to the next.
`run.sh` ends by replacing itself with the program (`exec`), so the process the tests start, signal and measure is the
program itself. Every test gives the program the same base environment, identical for both stacks apart from a few
documented runtime settings of each stack: in it `LANG` and `LC_ALL` are `de_DE.UTF-8`, nothing else selects a locale,
and `TZ` is `UTC`.

**R40.** The tests run offline. They use a clean copy of the workspace as you left it: build output and installed
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
# Read one object; then a rule of two steps on another: a property read and a call.
default within 2s
start ready port within 10s
connect c http

request c GET "/objects/visits"
recv c

request c POST "/rules"
  header "content-type" "application/json"
  body json {"steps":[{"object":"boiler","get":"reading"},{"object":"boiler","call":"corrected"}]}
recv c

signal TERM
wait-exit within 5s
```

`samples/s01.out`

```text
transcript 2
ready
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"name":"visits","type":"counter","properties":{"id":1000000007,"label":"front door","count":0,"stride":1,"limit":null}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"rule":"${:token}","results":[{"value":21.5},{"value":21.25}]}
exit 0
```

### s02

`samples/s02.in`

```text
scenario 2
# A set with a string value, two increments and a read in one rule; the rule's
# token read back; then nulls given and returned, a rounding and a shift.
default within 2s
start ready port within 10s
connect c http

request c POST "/rules"
  header "content-type" "application/json"
  body json {"steps":[{"object":"tickets","set":"stride","value":"3"},{"object":"tickets","call":"increment"},{"object":"tickets","call":"increment"},{"object":"tickets","get":"count"}]}
recv c capture r json $.rule as token

request c GET "/rules/${r}"
recv c

request c POST "/rules"
  header "content-type" "application/json"
  body json {"steps":[{"object":"tickets","call":"cap","args":[null]},{"object":"attic","get":"label"},{"object":"boiler","call":"rounded","args":[0]},{"object":"boiler","call":"shift","args":[0.5]}]}
recv c

signal TERM
wait-exit within 5s
```

`samples/s02.out`

```text
transcript 2
ready
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"rule":"${r}","results":[{"value":3},{"value":13},{"value":16},{"value":16}]}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"rule":"${r}","results":[{"value":3},{"value":13},{"value":16},{"value":16}]}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"rule":"${:token}","results":[{"value":null},{"value":null},{"value":21},{"value":22}]}
exit 0
```

How `s02` is built: `stride` becomes 3, so the two increments take `count` from 10 to 13 and 16; the read-back under the
token returns the same body. In the second rule `cap` with null removes the limit and returns null; `attic`'s `label`
is null; `rounded` with 0 digits turns `corrected` = 21.25 into 21; `shift` adds 0.5 to the reading 21.5 and returns
22.

### s03

`samples/s03.in`

```text
scenario 2
# Three failures - an unknown member after an applied step, a wrong number of
# arguments, an operation that fails - and the state they leave.
default within 2s
start ready port within 10s
connect c http

request c POST "/rules"
  header "content-type" "application/json"
  body json {"steps":[{"object":"visits","call":"increment"},{"object":"visits","get":"pressure"}]}
recv c

request c POST "/rules"
  header "content-type" "application/json"
  body json {"steps":[{"object":"boiler","call":"corrected","args":[1]}]}
recv c

request c POST "/rules"
  header "content-type" "application/json"
  body json {"steps":[{"object":"attic","call":"shift","args":[1]}]}
recv c

request c GET "/objects/visits"
recv c

signal TERM
wait-exit within 5s
```

`samples/s03.out`

```text
transcript 2
ready
reply c 404
  header "content-type" "application/json; charset=utf-8"
  body json {"error":"unknown-member","step":1}
reply c 400
  header "content-type" "application/json; charset=utf-8"
  body json {"error":"arity","step":0}
reply c 422
  header "content-type" "application/json; charset=utf-8"
  body json {"error":"operation","step":0}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"name":"visits","type":"counter","properties":{"id":1000000007,"label":"front door","count":1,"stride":1,"limit":null}}
exit 0
```

How `s03` is built: the first rule's step 0 increments `visits` and stays applied; its step 1 names a member that a
counter does not have. `corrected` takes no argument. `attic` is not enabled, so `shift` fails. The final read shows
`count` 1.

### s04

`samples/s04.in`

```text
scenario 2
# Two clients increment the same counter at the same instant.
default within 2s
start ready port within 10s
connect a http
connect b http

together
  request a POST "/rules"
    header "content-type" "application/json"
    body json {"steps":[{"object":"visits","call":"increment"}]}
  request b POST "/rules"
    header "content-type" "application/json"
    body json {"steps":[{"object":"visits","call":"increment"}]}
end
recv a
recv b

request a GET "/objects/visits"
recv a

signal TERM
wait-exit within 5s
```

`samples/s04.out`

```text
transcript 2
ready
unordered
  reply * 200
    header "content-type" "application/json; charset=utf-8"
    body json {"rule":"${:token}","results":[{"value":1}]}
  reply * 200
    header "content-type" "application/json; charset=utf-8"
    body json {"rule":"${:token}","results":[{"value":2}]}
end
reply a 200
  header "content-type" "application/json; charset=utf-8"
  body json {"name":"visits","type":"counter","properties":{"id":1000000007,"label":"front door","count":2,"stride":1,"limit":null}}
exit 0
```

How `s04` is built: whichever rule the service applies first gets 1 and the other 2 (R29); the read afterwards shows
`count` 2.

## 7. Hidden tests

**R41.** Hidden tests exist. They are scenarios of the kind shown in section 6: reads of objects, rules of one or more
steps with reads, sets and calls on every object and member of 1.2 and 1.3 with values and arguments of every form 3.6
names, failures of every code of R21 and the state they leave, rules sent by several clients at the same instant, the
read-back of tokens, the readiness of section 2 and the stop by SIGTERM — all within the bounds of section 4, with
expected replies computed from sections 1 and 3 and compared as 3.1 says. The visible samples are a small part of the
same suite; a program that passes them has not thereby passed the hidden tests.

## 8. Amendments

None yet.
