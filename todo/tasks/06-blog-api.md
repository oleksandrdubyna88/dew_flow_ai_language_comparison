# Task 06 — Blog API

| | |
|---|---|
| Kind | a long-running HTTP server that serves until it is stopped by SIGTERM |
| Gate verdict | `proceed` · 2026-11-17 |
| Freeze tag | `spec-frozen-B-20261118` |
| Freeze rule | this text changes only by a dated amendment from its authors |
| Amendments | none yet |

Every numbered sentence (**R1** … **R52**) is a requirement. Hidden tests refer to these numbers; nothing outside them is
tested.

## 1. What the program does

**R1.** The program is the HTTP service of a small blogging site. Users register and log in and receive a signed
*token* that identifies them in later requests; every user has a public *profile* that other users may *follow*;
users write *articles*, each reachable under a *slug* derived from its title and carrying a list of *tags*; users
write *comments* on articles. All state — users, follows, articles, comments — lives in the process's memory and
nowhere else: a fresh process starts with no user, no article and no comment, and nothing survives the process.

### 1.1 Values

**R2.** A *text* is a JSON string. The length of a text is its number of Unicode code points (so an emoji or a
Chinese character counts one, whatever its encoding), and a character set given for a text (`A`–`Z`, `a`–`z`, `0`–`9`,
`_`, `-`) is exact: letter case matters, and no character outside the set is accepted, trimmed, folded or replaced.
Two texts are equal when their code points are equal, character for character: `Ann` and `ann` are different
usernames, `Ann@example.org` and `ann@example.org` different emails.

**R3.** A *time* is a text of exactly this form: `YYYY-MM-DDThh:mm:ss.mmmZ` — the year in four digits, the month, day,
hour (00–23), minute and second in two digits each, exactly three digits of fraction (the whole milliseconds, the
rest of the fraction dropped), and the letter `Z`; it is the time in UTC (`2026-10-31T22:15:04.123Z`), never in
another zone and never with an offset. The service takes the time from the system clock at the moment it handles the
request that creates or changes the object; the value of the environment variable `TZ` (R17) does not affect it.

### 1.2 Users and tokens

**R4.** A *user* has a `username`, an `email`, a `password`, a `bio` and an `image`. `username` is a text of 1 to 32
characters from `A`–`Z`, `a`–`z`, `0`–`9` and `_`. `email` is a text of 3 to 254 characters holding exactly one `@`,
which is neither its first nor its last character; no other rule applies to an email. `password` is a text of 8 to 72
characters, any characters. `bio` and `image` are each a text or null, and both are null for every user: no request
of this task sets them. No two users share a `username`, and no two users share an `email` (R2's equality).

**R5.** A *user object* is the JSON object `{"user": {"username": <text>, "email": <text>, "bio": null, "image":
null, "token": <token>}}`, with the user's values and a token (R6) that names the user. The password is never written
in any reply.

**R6.** A *token* is a text of three parts joined by `.`, each part the base64url encoding (the URL-safe alphabet
`A`–`Z`, `a`–`z`, `0`–`9`, `-`, `_`, with no `=` padding) of bytes: the first part encodes the bytes of the 27
characters `{"alg":"HS256","typ":"JWT"}` exactly as written here, so every token the service issues begins with
`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.`; the second part encodes the UTF-8 bytes of a JSON object with exactly three
members, in any order and with no other member — `sub`, the username as a JSON string; `iat`, the moment of issue as a
JSON integer of whole seconds since 1970-01-01T00:00:00Z; and `exp`, the moment of expiry, `iat` plus the value of
`TOKEN_TTL_SECONDS` (R17), as a JSON integer of the same kind; the third part encodes the 32 bytes of HMAC-SHA256,
keyed with the UTF-8 bytes of the environment variable `JWT_SECRET` (R17), over the ASCII bytes of the first part, a
`.` and the second part. The service issues a token at registration (R25) and at login (R26); the two tokens of one
user may be equal or different.

**R7.** A token presented in a request is *valid* when all of the following hold, and *invalid* otherwise: it has
exactly three non-empty parts joined by `.`, each the exact base64url encoding of R6, without `=`, of the bytes it
decodes to (a part whose length is one more than a multiple of four, or whose last character sets bits the encoding
leaves unused, is not); the first part decodes to a JSON object whose `alg` member is the string `HS256` (its other
members are not examined); the third part decodes to exactly the HMAC-SHA256 of R6 over the first part, the `.` and
the second part exactly as received, keyed with `JWT_SECRET`; the second part decodes to a JSON object whose `sub`
member is the username of an existing user and whose `exp` member is a JSON number written as decimal digits, with an
optional `-` and no fraction part or exponent, greater than the current time in whole seconds since
1970-01-01T00:00:00Z. The user a valid token names is the one its `sub` names. A token the tests build themselves to
these rules — with the test's `JWT_SECRET`, an `exp` in the future and an existing user — is valid; a token whose
signature does not match, whose `exp` is not in the future or whose `sub` names no user is invalid. A token's
expiry is the moment `exp` names; the tests keep at least one second clear of it: a token presented more than one
second before its expiry must be accepted and one presented more than one second after it must be rejected, and no
test presents a token within one second of its expiry.

**R8.** The `Authorization` request header carries a token as the six characters `Token ` (the word `Token`, letter
case as written, then one space) followed by the token and nothing else. An endpoint whose row in R20 says *required*
needs this header with a valid token; without the header, with a header of another form, or with an invalid token
the reply is 401 (R23). An endpoint whose row says *optional* serves a request without the header as *anonymous*,
and a request with the header as the user the token names — a header of another form or an invalid token is 401
there too. An endpoint whose row says *none* ignores the header. The *requester* of a request is the user its valid
token names, or nobody when the request is anonymous.

### 1.3 Profiles and follows

**R9.** A *profile object* is the JSON object `{"profile": {"username": <text>, "bio": null, "image": null,
"following": <boolean>}}` for one user. `following` is `true` when the requester is a user who follows that user, and
`false` when the requester does not follow them or is anonymous. A user may follow any user, themselves included; a
follow has no other effect than this value.

### 1.4 Articles

**R10.** An *article* has a `title`, a text of 1 to 120 characters holding at least one character from `A`–`Z`,
`a`–`z`, `0`–`9`; a `description`, a text of 1 to 500 characters; a `body`, a text of 1 to 10 000 characters; a
`tagList` of 0 to 10 *tags*, each a text of 1 to 32 characters from `A`–`Z`, `a`–`z`, `0`–`9`, `_` and `-`; an
*author*, the user who created it; a `slug` (R11); and `createdAt` and `updatedAt`, times (R3): `createdAt` is the time
of creation and never changes, `updatedAt` is `createdAt` until the article is changed (R33) and then the time of the
latest change.

**R11.** The *slug* is chosen by the service at creation from the title and never changes afterwards. The *base* of
the slug is the title transformed as follows: every character from `A`–`Z` becomes the corresponding character from
`a`–`z`; every character from `a`–`z` and `0`–`9` stays; every maximal run of other characters, whatever they are,
becomes one `-`; then a `-` at the beginning and a `-` at the end are removed (`Hello, World!` gives `hello-world`,
`  Über 2 Äpfel ` gives `ber-2-pfel`, `A--B` gives `a-b`). When no existing article has the base as its slug, the slug
is the base; otherwise it is the base, `-` and the smallest whole number k ≥ 2 such that no existing article has that
slug (`hello-world-2`, then `hello-world-3`). An *existing* article is one that was created and not deleted (R34): the
slug of a deleted article may be issued again.

**R12.** An *article object* is the JSON object `{"article": {"slug": <text>, "title": <text>, "description": <text>,
"body": <text>, "tagList": [<text>, …], "createdAt": <time>, "updatedAt": <time>, "author": <profile>}}`, where
`author` is the inner object of the author's profile object (R9) — `username`, `bio`, `image`, `following` — with
`following` computed for the requester. `tagList` holds the tags given at creation in the order they were given, each
tag once: a tag given more than once is kept at its first position only. An absent `tagList` at creation is an empty
list.

### 1.5 Comments

**R13.** A *comment* belongs to one article and has an `id`, a `body`, a text of 1 to 2 000 characters, an *author*
and `createdAt` and `updatedAt`, times (R3) that are equal: the time of creation. The `id` is a whole number: the first
comment the process creates has `id` 1, the next 2, and so on across all articles, in the order the service creates
them; an id is never reused, also after the comment is deleted. A *comment object* is the JSON object `{"comment":
{"id": <integer>, "body": <text>, "createdAt": <time>, "updatedAt": <time>, "author": <profile>}}`, `author` as in
R12.

## 2. Start, readiness and stop

**R14.** The environment variable `PORT` holds a TCP port number; the program listens on it, and the tests connect to
it at 127.0.0.1.

**R15.** The program is ready when it accepts a TCP connection on that port. A test waits at most 10 seconds for
readiness; a program that is not ready by then fails that test.

**R16.** A test ends by sending the signal SIGTERM to the program. The program must then exit with status 0 within 5
seconds, leaving no process it started still running. A program still running after 5 seconds is killed and fails
that test, and so does one that exits with another status or leaves a process running. SIGTERM may arrive while
client connections are open; the program may close them.

**R17.** Every test sets two further environment variables before it starts the program. `JWT_SECRET` holds the
signing key of R6, a text of 1 to 256 characters from `A`–`Z`, `a`–`z`, `0`–`9`, `-`, `_` and `.`, used as its UTF-8
bytes. `TOKEN_TTL_SECONDS` holds the lifetime of a token in whole seconds, decimal digits denoting a number from 1 to
86 400. Both are fixed for the life of the process. A test may also set the environment variable `TZ` to the name of
any time zone, or leave it as the base environment sets it (R50); no reply depends on it (R3).

## 3. Interface

### 3.1 HTTP, JSON and how replies are compared

**R18.** The service speaks HTTP/1.1. A client may send several requests one after another on one connection, and the
service keeps a connection open after a reply for the client's next request. A request with a body carries the header
`content-type: application/json` and a body that is one JSON object in UTF-8 with the members the endpoint's row names
and no other member; a request without a body has no body. Path segments written `<username>`, `<slug>` and `<id>` in
R20 are texts in the character set and form of a username (R4), a slug (R11) and a comment id (decimal digits); the
query of `GET /api/articles` is as R35 says, and no other request has a query.

**R19.** Every reply with a status other than 204 has a JSON body and the header `content-type` with the value
`application/json; charset=utf-8`; a reply with status 204 has no body. Every reply is compared on its status code, on
each response header that a requirement of this spec names, and on its body; other headers are not compared. The body
is compared byte for byte, except where a requirement of this spec names another comparison. Nothing else is
normalized.

**R20.** The tests send these requests and no other:

| Request | Authorization | Body | Success |
|---|---|---|---|
| `POST /api/users` | none | `{"user": {"username", "email", "password"}}` | 201, a user object (R25) |
| `POST /api/users/login` | none | `{"user": {"email", "password"}}` | 200, a user object (R26) |
| `GET /api/user` | required | — | 200, a user object (R27) |
| `GET /api/profiles/<username>` | optional | — | 200, a profile object (R29) |
| `POST /api/profiles/<username>/follow` | required | — | 200, a profile object (R30) |
| `DELETE /api/profiles/<username>/follow` | required | — | 200, a profile object (R30) |
| `POST /api/articles` | required | `{"article": {"title", "description", "body", "tagList"?}}` | 201, an article object (R31) |
| `GET /api/articles/<slug>` | optional | — | 200, an article object (R32) |
| `PUT /api/articles/<slug>` | required | `{"article": {"title"?, "description"?, "body"?}}` | 200, an article object (R33) |
| `DELETE /api/articles/<slug>` | required | — | 204, no body (R34) |
| `GET /api/articles` | optional | — | 200, a list (R35) |
| `GET /api/tags` | none | — | 200, the tags (R36) |
| `POST /api/articles/<slug>/comments` | required | `{"comment": {"body"}}` | 201, a comment object (R37) |
| `GET /api/articles/<slug>/comments` | optional | — | 200, the comments (R38) |
| `DELETE /api/articles/<slug>/comments/<id>` | required | — | 204, no body (R39) |

A member marked `?` may be absent, with the meaning R12 or R33 gives; a member not so marked may also be absent in a
test's request, or hold a JSON value of any type — R23 says how such a request is answered.

**R21.** The body of every reply that has a body is compared as a JSON value: the members of an object may come in any
order, numbers are compared by their exact decimal value with no rounding (so `2` equals `2.0` and `-0` equals `0`, and
two numbers that differ in any digit differ), and whitespace outside strings is ignored; arrays keep their order
unless a requirement says otherwise.

**R22.** The service writes values in JSON as follows. A text and a time are JSON strings. `following` is `true` or
`false`. A comment `id` and `articlesCount` are JSON numbers written as decimal digits with no sign, no fraction part
and no exponent. Null is written as JSON `null` and is always written: `bio` and `image` are present with the value
`null`, never left out. An empty `tagList`, an empty `articles`, `tags` or `comments` list is written `[]`.

### 3.2 Errors

**R23.** A reply that is not a success of R20 is an *error reply*: its body is `{"errors": {"body": ["<message>"]}}`
with exactly one message, chosen with its status from this table:

| Status | Message | When |
|---|---|---|
| 401 | `unauthorized` | the `Authorization` header is missing on a *required* endpoint, has another form than R8, or carries an invalid token (R7), on a *required* or an *optional* endpoint |
| 404 | `not found` | no existing article has the `<slug>`, no user has the `<username>`, or no existing comment of that article has the `<id>` |
| 403 | `forbidden` | the requester is not the author of the article (`PUT`, `DELETE /api/articles/<slug>`) or of the comment (`DELETE … /comments/<id>`) |
| 422 | `<field> is invalid` | the named body member is absent, is `null`, is not a JSON string (for `tagList`: is not a JSON array of JSON strings), or does not meet its rule in R4, R10 or R13; `<field>` is the member's name: `username`, `email`, `password`, `title`, `description`, `body` or `tagList` |
| 422 | `<field> is taken` | at `POST /api/users`, a user already has the given `username` or `email`; `<field>` is `username` or `email` |
| 422 | `email or password is invalid` | at `POST /api/users/login`, no user has the given `email`, or the user's password differs from the given one |

**R24.** The conditions are checked in this order, and the first that holds decides the reply; the later ones are not
examined: first 401; then 404 for the path's `<username>`, `<slug>` or `<id>`; then 403; then 422 `is invalid` for the
body's members, in the order the endpoint's row of R20 lists them (an absent member marked `?` is not invalid); then
422 `is taken`, `username` before `email`; then 422 `email or password is invalid`. At `POST /api/users/login` the
members `email` and `password` are *invalid* only when absent, `null` or not a JSON string; their lengths are not
examined there. An error reply changes nothing: no user, follow, article or comment is created, changed or deleted,
and no token is issued.

### 3.3 Users

**R25.** `POST /api/users` creates a user from `username`, `email` and `password` (R4) with `bio` and `image` null,
and replies 201 with the user object (R5) carrying a token issued now.

**R26.** `POST /api/users/login` finds the user whose `email` equals the given one; when that user's `password` equals
the given one, the reply is 200 with the user object carrying a token issued now; otherwise 422 (R23).

**R27.** `GET /api/user` replies 200 with the requester's user object, whose `token` member is the token the request
carried, unchanged.

**R28.** The three requests above, and the user objects they return, are the only ones that handle passwords; how the
service stores a password is not specified, and the password's characters are compared exactly (R2).

### 3.4 Profiles

**R29.** `GET /api/profiles/<username>` replies 200 with the profile object of that user, `following` computed for
the requester (R9).

**R30.** `POST /api/profiles/<username>/follow` makes the requester follow that user, and `DELETE
/api/profiles/<username>/follow` makes the requester not follow them; following a user already followed, or
unfollowing one not followed, changes nothing. Both reply 200 with the profile object of that user as it is after the
request (`following` `true` after `POST`, `false` after `DELETE`).

### 3.5 Articles and tags

**R31.** `POST /api/articles` creates an article from `title`, `description`, `body` and `tagList` (R10, R12), with the
requester as author, the slug of R11, and `createdAt` and `updatedAt` both the current time, and replies 201 with the
article object.

**R32.** `GET /api/articles/<slug>` replies 200 with the article object of the existing article that has that slug,
`author.following` computed for the requester.

**R33.** `PUT /api/articles/<slug>` changes the existing article that has that slug, when the requester is its
author: each of `title`, `description` and `body` that is present replaces the article's value (R10's rules apply to
the new value; a member that is absent leaves its value), `tagList` and `slug` stay as they are, `updatedAt` becomes
the current time, and the reply is 200 with the article object as it is after the change. Every test's `PUT` body has
at least one of the three members.

**R34.** `DELETE /api/articles/<slug>` deletes the existing article that has that slug, when the requester is its
author, together with its comments, and replies 204 with no body. Afterwards the slug names no existing article: `GET
/api/articles/<slug>` and the comment requests on it reply 404, and its tags count no longer for `GET /api/tags`.

**R35.** `GET /api/articles` lists existing articles. Its query holds any of the parameters `tag`, `author`, `limit`
and `offset`, each at most once, in any order, as `name=value` pairs joined by `&` after `?`, or no query at all.
`tag=<tag>` keeps only the articles whose `tagList` holds that tag; `author=<username>` keeps only the articles whose
author has that username; both given, both apply; a tag or username that matches nothing gives an empty list, not an
error. The matching articles stand in the order the service created them, the most recently created first; the reply
skips the first `offset` of them (a whole number from 0 to 10 000, default 0) and holds at most `limit` of the rest (a
whole number from 1 to 100, default 20), as `{"articles": [<article>, …], "articlesCount": <integer>}`, where each
`<article>` is the inner object of the article object (R12) — `slug` through `author` — with `author.following`
computed for the requester, and `articlesCount` is the number of matching articles before `offset` and `limit` are
applied.

**R36.** `GET /api/tags` replies 200 with `{"tags": [<text>, …]}`: every tag that is in the `tagList` of at least one
existing article, each once, in ascending order of their bytes compared byte by byte (so `-` sorts before digits,
digits before `A`–`Z`, `A`–`Z` before `_`, and `_` before `a`–`z`; a tag that is a prefix of another sorts first:
`2026`, `Go`, `go`, `go-lang`). With no existing article, or none with a tag, the list is empty.

### 3.6 Comments

**R37.** `POST /api/articles/<slug>/comments` creates a comment on the existing article that has that slug, from
`body` (R13), with the requester as author and the current time, and replies 201 with the comment object.

**R38.** `GET /api/articles/<slug>/comments` replies 200 with `{"comments": [<comment>, …]}`: every existing comment
of the existing article that has that slug, in ascending order of `id`, each `<comment>` the inner object of the
comment object (R13) with `author.following` computed for the requester; `[]` when the article has no comment.

**R39.** `DELETE /api/articles/<slug>/comments/<id>` deletes the comment with that `id` on the existing article that
has that slug, when the requester is its author, and replies 204 with no body. A comment is *existing* from its
creation until it is deleted, or until its article is deleted (R34).

### 3.7 Concurrency

**R40.** Requests may arrive at the same time on different connections. The service applies them as if one at a
time, in some order: every request is applied whole, and every reply shows a state that some sequence of whole
requests produced. So when k clients send, at the same instant, `POST /api/users` with the same `username` and k
different emails, exactly one reply is 201 and k − 1 are 422 `username is taken`; when k clients send, at the same
instant, `POST /api/articles` with the same title `T` whose base is `t`, the k replies carry k different slugs, `t`,
`t-2`, …, `t-k`, and `GET /api/articles` afterwards lists k articles; when k clients send, at the same instant, a
comment each on one article, the k replies carry k different ids and the comments list holds all k.

### 3.8 Not defined, not compared

**R41.** A request or input that violates any of R17, R18, R20, R33, R35, R46 and R47 does not occur in any visible or
hidden test. What the program does with it is unspecified and is never tested.

**R42.** Standard output is compared only where a requirement of this spec says what it holds; otherwise it is
recorded, not judged.

**R43.** Standard error is not compared and may hold anything; it is recorded, not judged. Only its size is bounded: a run
that writes more than 256 MiB to standard error is stopped, and that test fails.

## 4. Limits

**R44.** Each test starts the program once, in a fresh process, and drives one scenario against it under a bound on the
program's memory. The bound is the one the task's statement states: one registered value, the same for every daemon and
web task, which the test harness adds to the statement from its registration, so a spec never writes the number. It is
registered before the freeze, only once the reference solutions of every daemon and web task pass every test under it in
both stacks, and enforced as F-MEMORY says. The load scenarios are hidden scenarios of up to the bounds of R46 and R47,
each lasting at most 600 seconds; the bound on memory holds in every test, whether it is a load scenario or not. Memory
is measured as the peak memory of the program's own control group (cgroup `memory.peak`), which counts the program and
every process it starts, including the page cache and kernel memory charged to them, with no swap. The program's memory
reaches the limit when the system kills a process of the program's control group at the limit, or when the program's own
runtime stops it for lack of memory; either way the run is stopped and fails that test. The test machine caps each
runtime's own heap at 90 % of the limit, by that runtime's own setting. The tests run only on a machine that can read
this peak, and no other measurement of memory passes or fails a test. The program and every process it starts may hold
at most 512 processes and threads together. Beyond that the system refuses to create another, and a run that needs more
fails that test.

**R45.** Every step of a scenario that waits — for readiness, for a reply, for the exit — has a deadline, and a step
that does not complete by its deadline fails the test. The visible samples state their deadlines in their files; the
deadlines of the hidden scenarios are fixed when the suite is calibrated, so that the reference solutions in both
stacks meet each of them with margin.

**R46.** The load of a test stays within these bounds: at most 8 client connections are open at once; at most 10 000
requests are sent in one test; a request body is at most 65 536 bytes; one test creates at most 2 000 users, 2 000
articles and 5 000 comments, and makes at most 10 000 follows.

**R47.** Every value in a request stays within these bounds: a text in a request body is at most 10 000 characters
long, and a JSON string of a request body holds no character that is not a Unicode scalar value; a `<username>`,
`<slug>` or `<id>` path segment and a query value are at most 256 characters long; a `tag` or `author` query value
holds only characters a tag or a username may hold; a `limit` or `offset` value is written as decimal digits.

## 5. Build and run — the workspace contract

**R48.** The workspace holds `STATEMENT.md`, `samples/`, `runner/`, `build.sh`, `run.sh`, `PACKAGES.md` and the
toolchain pin files named in `build.sh`. These are frozen inputs: do not change, rename or remove them. The program's
sources go where `build.sh` and `run.sh` expect them; the two scripts say where.

**R49.** `bash build.sh`, run in the workspace directory, builds the program with no network access and with no package
outside the set listed in `PACKAGES.md`. It must exit 0 within 600 seconds. It runs once before the tests; a build that
does not exit 0 fails every test.

**R50.** `bash run.sh`, run in the workspace directory with no arguments, starts the built program once per test, with
the environment section 2 describes. One fresh process per test; nothing carries over from one test to the next.
`run.sh` ends by replacing itself with the program (`exec`), so the process the tests start, signal and measure is the
program itself. Every test gives the program the same base environment, identical for both stacks apart from a few
documented runtime settings of each stack: in it `LANG` and `LC_ALL` are `de_DE.UTF-8`, nothing else selects a locale,
and `TZ` is `UTC`.

**R51.** The tests run offline. They use a clean copy of the workspace as you left it: build output and installed
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
# One user registers; the token names the current user; the public profile.
default within 2s
env JWT_SECRET "sample-secret"
env TOKEN_TTL_SECONDS "3600"
start ready port within 10s
connect c http

request c POST "/api/users"
  header "content-type" "application/json"
  body json {"user":{"username":"ann","email":"ann@example.org","password":"correct horse"}}
recv c capture t json $.user.token as token

request c GET "/api/user"
  header "authorization" "Token ${t}"
recv c

request c GET "/api/profiles/ann"
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
  body json {"user":{"username":"ann","email":"ann@example.org","bio":null,"image":null,"token":"${t}"}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"user":{"username":"ann","email":"ann@example.org","bio":null,"image":null,"token":"${t}"}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"profile":{"username":"ann","bio":null,"image":null,"following":false}}
exit 0
```

### s02

`samples/s02.in`

```text
scenario 2
# Login after registration; an article with tags, read back by its slug
# anonymously; the tag list in byte order.
default within 2s
env JWT_SECRET "sample-secret"
env TOKEN_TTL_SECONDS "3600"
start ready port within 10s
connect c http

request c POST "/api/users"
  header "content-type" "application/json"
  body json {"user":{"username":"bob","email":"bob@example.org","password":"hunter2hunter2"}}
recv c

request c POST "/api/users/login"
  header "content-type" "application/json"
  body json {"user":{"email":"bob@example.org","password":"hunter2hunter2"}}
recv c capture t json $.user.token as token

request c POST "/api/articles"
  header "content-type" "application/json"
  header "authorization" "Token ${t}"
  body json {"article":{"title":"Hello, World!","description":"A first post","body":"Some text.","tagList":["intro","Go","go","intro"]}}
recv c capture created json $.article.createdAt as ts

request c GET "/api/articles/hello-world"
recv c

request c GET "/api/tags"
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
  body json {"user":{"username":"bob","email":"bob@example.org","bio":null,"image":null,"token":"${:token}"}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"user":{"username":"bob","email":"bob@example.org","bio":null,"image":null,"token":"${t}"}}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"article":{"slug":"hello-world","title":"Hello, World!","description":"A first post","body":"Some text.","tagList":["intro","Go","go"],"createdAt":"${:ts~10s}","updatedAt":"${created}","author":{"username":"bob","bio":null,"image":null,"following":false}}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"article":{"slug":"hello-world","title":"Hello, World!","description":"A first post","body":"Some text.","tagList":["intro","Go","go"],"createdAt":"${created}","updatedAt":"${created}","author":{"username":"bob","bio":null,"image":null,"following":false}}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"tags":["Go","go","intro"]}
exit 0
```

How `s02` is built: the title `Hello, World!` gives the base `hello-world` (R11), which no article has yet. The tag
`intro` is given twice and kept once, at its first position; `Go` and `go` are two tags (R2, R12). `createdAt` is
within 10 seconds of the runner's clock and `updatedAt` equals it (R10). The tag list of R36 sorts `Go` before `go`
before `intro` by bytes.

### s03

`samples/s03.in`

```text
scenario 2
# Two users; a follow; two articles; the list with a limit, then filtered by a
# tag with an offset; a comment created, listed and deleted.
default within 2s
env JWT_SECRET "sample-secret"
env TOKEN_TTL_SECONDS "3600"
start ready port within 10s
connect c http

request c POST "/api/users"
  header "content-type" "application/json"
  body json {"user":{"username":"ann","email":"ann@example.org","password":"correct horse"}}
recv c capture a json $.user.token as token

request c POST "/api/users"
  header "content-type" "application/json"
  body json {"user":{"username":"bob","email":"bob@example.org","password":"hunter2hunter2"}}
recv c capture b json $.user.token as token

request c POST "/api/profiles/ann/follow"
  header "authorization" "Token ${b}"
recv c

request c POST "/api/articles"
  header "content-type" "application/json"
  header "authorization" "Token ${a}"
  body json {"article":{"title":"One","description":"First","body":"Body one.","tagList":["x"]}}
recv c capture t1 json $.article.createdAt as ts

request c POST "/api/articles"
  header "content-type" "application/json"
  header "authorization" "Token ${a}"
  body json {"article":{"title":"Two","description":"Second","body":"Body two.","tagList":["x","y"]}}
recv c capture t2 json $.article.createdAt as ts

request c GET "/api/articles?limit=1"
  header "authorization" "Token ${b}"
recv c

request c GET "/api/articles?tag=x&offset=1"
recv c

request c POST "/api/articles/one/comments"
  header "content-type" "application/json"
  header "authorization" "Token ${b}"
  body json {"comment":{"body":"Nice."}}
recv c capture c1 json $.comment.createdAt as ts

request c GET "/api/articles/one/comments"
  header "authorization" "Token ${a}"
recv c

request c DELETE "/api/articles/one/comments/1"
  header "authorization" "Token ${b}"
recv c

request c GET "/api/articles/one/comments"
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
  body json {"user":{"username":"ann","email":"ann@example.org","bio":null,"image":null,"token":"${a}"}}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"user":{"username":"bob","email":"bob@example.org","bio":null,"image":null,"token":"${b}"}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"profile":{"username":"ann","bio":null,"image":null,"following":true}}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"article":{"slug":"one","title":"One","description":"First","body":"Body one.","tagList":["x"],"createdAt":"${:ts~10s}","updatedAt":"${t1}","author":{"username":"ann","bio":null,"image":null,"following":false}}}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"article":{"slug":"two","title":"Two","description":"Second","body":"Body two.","tagList":["x","y"],"createdAt":"${:ts~10s}","updatedAt":"${t2}","author":{"username":"ann","bio":null,"image":null,"following":false}}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"articles":[{"slug":"two","title":"Two","description":"Second","body":"Body two.","tagList":["x","y"],"createdAt":"${t2}","updatedAt":"${t2}","author":{"username":"ann","bio":null,"image":null,"following":true}}],"articlesCount":2}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"articles":[{"slug":"one","title":"One","description":"First","body":"Body one.","tagList":["x"],"createdAt":"${t1}","updatedAt":"${t1}","author":{"username":"ann","bio":null,"image":null,"following":false}}],"articlesCount":2}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"comment":{"id":1,"body":"Nice.","createdAt":"${:ts~10s}","updatedAt":"${c1}","author":{"username":"bob","bio":null,"image":null,"following":false}}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"comments":[{"id":1,"body":"Nice.","createdAt":"${c1}","updatedAt":"${c1}","author":{"username":"bob","bio":null,"image":null,"following":false}}]}
reply c 204
  body empty
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"comments":[]}
exit 0
```

How `s03` is built: Bob follows Ann, so in the list Bob requests the author's `following` is `true`, while Ann's own
creation replies show `false` (Ann does not follow herself) and the anonymous list shows `false`. The list with
`limit=1` holds the most recent article, `two`, and counts both; the list filtered by `x` matches both, skips one and
holds `one`. The comment gets `id` 1; Ann does not follow Bob, so its author shows `following` `false` to her; Bob,
its author, deletes it.

### s04

`samples/s04.in`

```text
scenario 2
# Error replies: a taken username, a wrong password, a missing header, a
# stranger's delete, an unknown slug; a token built by the test from the
# secret is accepted, an expired one is not.
default within 2s
env JWT_SECRET "sample-secret"
env TOKEN_TTL_SECONDS "3600"
start ready port within 10s
connect c http

request c POST "/api/users"
  header "content-type" "application/json"
  body json {"user":{"username":"ann","email":"ann@example.org","password":"correct horse"}}
recv c capture a json $.user.token as token

request c POST "/api/users"
  header "content-type" "application/json"
  body json {"user":{"username":"ann","email":"other@example.org","password":"correct horse"}}
recv c

request c POST "/api/users/login"
  header "content-type" "application/json"
  body json {"user":{"email":"ann@example.org","password":"wrong horse"}}
recv c

request c GET "/api/user"
recv c

request c GET "/api/user"
  header "authorization" "Token eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhbm4iLCJpYXQiOjE3MDAwMDAwMDAsImV4cCI6NDEwMjQ0NDgwMH0.tuM7HFn6-cRH5grJaE6iX4jGj9dLcEzcOMIqoALVUNA"
recv c

request c GET "/api/user"
  header "authorization" "Token eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhbm4iLCJpYXQiOjE3MDAwMDAwMDAsImV4cCI6MTcwMDAwMzYwMH0.-utSF76SLp7TJLpgFFvE71OL3P9Y8kxg6R_7JQdbRB8"
recv c

request c POST "/api/articles"
  header "content-type" "application/json"
  header "authorization" "Token ${a}"
  body json {"article":{"title":"Hi there","description":"Short","body":"Text."}}
recv c

request c POST "/api/users"
  header "content-type" "application/json"
  body json {"user":{"username":"bob","email":"bob@example.org","password":"hunter2hunter2"}}
recv c capture b json $.user.token as token

request c DELETE "/api/articles/hi-there"
  header "authorization" "Token ${b}"
recv c

request c GET "/api/articles/nothing-here"
recv c

signal TERM
wait-exit within 5s
```

`samples/s04.out`

```text
transcript 2
ready
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"user":{"username":"ann","email":"ann@example.org","bio":null,"image":null,"token":"${a}"}}
reply c 422
  header "content-type" "application/json; charset=utf-8"
  body json {"errors":{"body":["username is taken"]}}
reply c 422
  header "content-type" "application/json; charset=utf-8"
  body json {"errors":{"body":["email or password is invalid"]}}
reply c 401
  header "content-type" "application/json; charset=utf-8"
  body json {"errors":{"body":["unauthorized"]}}
reply c 200
  header "content-type" "application/json; charset=utf-8"
  body json {"user":{"username":"ann","email":"ann@example.org","bio":null,"image":null,"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhbm4iLCJpYXQiOjE3MDAwMDAwMDAsImV4cCI6NDEwMjQ0NDgwMH0.tuM7HFn6-cRH5grJaE6iX4jGj9dLcEzcOMIqoALVUNA"}}
reply c 401
  header "content-type" "application/json; charset=utf-8"
  body json {"errors":{"body":["unauthorized"]}}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"article":{"slug":"hi-there","title":"Hi there","description":"Short","body":"Text.","tagList":[],"createdAt":"${:ts~10s}","updatedAt":"${:ts~10s}","author":{"username":"ann","bio":null,"image":null,"following":false}}}
reply c 201
  header "content-type" "application/json; charset=utf-8"
  body json {"user":{"username":"bob","email":"bob@example.org","bio":null,"image":null,"token":"${b}"}}
reply c 403
  header "content-type" "application/json; charset=utf-8"
  body json {"errors":{"body":["forbidden"]}}
reply c 404
  header "content-type" "application/json; charset=utf-8"
  body json {"errors":{"body":["not found"]}}
exit 0
```

How `s04` is built: the two tokens written out in the scenario are built by the rules of R6 with the secret
`sample-secret`: both have the second part `{"sub":"ann","iat":1700000000,"exp":…}`, the first with `exp` 4102444800
(the year 2100), the second with `exp` 1700003600 (the year 2023), so the first is valid once `ann` exists and the
second is expired (R7). `GET /api/user` returns the token the request carried (R27). Bob is not the author of
`hi-there`, so his delete is 403 and the article stays; an article without `tagList` has `[]`.

## 7. Hidden tests

**R52.** Hidden tests exist. They are scenarios of the kind shown in section 6: registrations and logins with valid
and invalid fields and with taken names, tokens issued by the service and tokens built by the test — valid, expired,
altered or naming no user — on every endpoint of R20, profiles and follows seen by different requesters, articles
created, read, changed and deleted with slugs derived from many titles, lists with every combination of filter,
`limit` and `offset`, tag lists, comments created, listed and deleted, every error of R23 with the state it leaves,
requests sent by several clients at the same instant, the readiness of section 2 and the stop by SIGTERM — all within
the bounds of section 4, with expected replies computed from sections 1 and 3 and compared as 3.1 says. The visible
samples are a small part of the same suite; a program that passes them has not thereby passed the hidden tests.

## 8. Amendments

None yet.
