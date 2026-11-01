# dew_flow_ai_language_comparison

Task workspaces for a controlled comparison of AI coding agents on two stacks: **C# / .NET 10** and **TypeScript /
Node 26**. Each task has one specification that both stacks implement to the same requirements, and one set of black-box
checks that judges both.

## Layout

| Path | What it holds |
|---|---|
| `todo/tasks/` | the task specifications, one file per task, and `FORMAT.md`, the format of the visible samples |
| `tasks/<NN-slug>/samples/` | the visible samples of each task (`<id>.in` / `<id>.out`) |
| `csharp/`, `node/` | one workspace template per task and stack |
| `runner/` | the visible-sample runner (Python, standard library only) |

## The tasks

| # | Task | Kind |
|---|---|---|
| 01 | Descent to one | olympiad (stdin/stdout) |
| 02 | Open areas | olympiad (stdin/stdout) |
| 03 | Wrapping arithmetic | olympiad (stdin/stdout) |
| 04 | Rule API | web API |
| 05 | Key-value server | daemon (TCP) |
| 06 | Blog API | web API |
| 07 | Stream processor | daemon |
| 08 | Ledger API | web API |
| 09 | Switched routes | olympiad (stdin/stdout) |
| 10 | DAG scheduler | daemon |

## Stacks

Both stacks use the current LTS line at its latest patch, pinned: .NET SDK 10.0.403 (runtime 10.0.14, C# 14) and
Node 26.13.0 with TypeScript 6.0.3 in `strict` mode. Both get first-party or de-facto-standard pieces at the same level of
abstraction:

| Need | C# / .NET 10 | TypeScript / Node 26 |
|---|---|---|
| HTTP routing, binding | ASP.NET Core Minimal API | Fastify 5.12.7 |
| Validation and serialization | Minimal API built-in validation + System.Text.Json | TypeBox 0.34.x through `@fastify/type-provider-typebox` |
| JWT | `Microsoft.AspNetCore.Authentication.JwtBearer` 10.x | `@fastify/jwt` 10.x |
| Lifecycle, graceful stop | Generic Host, `IHostApplicationLifetime` | Fastify hooks and `close()`; `process.on('SIGTERM')` |
| Logging | `ILogger` | pino |
| TCP | `System.Net.Sockets` | `node:net` |
| Tests | xUnit v3 offered | `node:test` |

Why Fastify for Node: it is the counterpart of Minimal API — built-in JSON-schema validation and fast serialization,
lifecycle hooks and graceful close, a structured logger, first-party JWT and TypeBox plugins. Express was rejected (no
built-in validation or serialization, a weaker tool than Minimal API), NestJS (a decorator and DI framework, the analogue of
MVC controllers, not of Minimal API), Hono (its Node ecosystem for JWT and validation is younger) and raw `node:http` (it
would make TypeScript write its own framework while C# gets one).

## Licence

The repository is under the MIT licence (`LICENSE`), except the files named in the attribution section below.

## Attribution

This section is the only place in the repository that names a source; it is removed from every run base an agent sees.

**Known tasks.**

Task 01 is adapted from the CSES Problem Set problem "Weird Algorithm" (https://cses.fi/problemset/task/1068) by Antti Laaksonen, used under CC BY-NC-SA 4.0.

Task 02 is adapted from the CSES Problem Set problem "Counting Rooms" (https://cses.fi/problemset/task/1192) by Antti Laaksonen, used under CC BY-NC-SA 4.0.

The files todo/tasks/01-descent-to-one.md and todo/tasks/02-open-areas.md adapt CSES Problem Set material and are therefore licensed under CC BY-NC-SA 4.0 (https://creativecommons.org/licenses/by-nc-sa/4.0/), not under the repository's MIT licence.

Task 05 describes a subset of the RESP wire protocol in its own words; the protocol is specified at https://redis.io/docs/latest/develop/reference/protocol-spec/.

Task 06 is adapted from the RealWorld "Conduit" API specification (https://github.com/gothinkster/realworld), used under the MIT licence; its notice is reproduced below.

Task 09's problem idea is based on IOI 2018 Werewolf (https://stats.ioinformatics.org/tasks/2018/werewolf); its statement is written wholly in our own words and copies no text.

The RealWorld notice, as read on 2026-11-01:

```
MIT License

Copyright (c) 2021 Thinkster
Copyright (c) 2026 c4ffein

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

**Where this comes from.** The experiment that uses these tasks — its plan, its harness and its results — is in the public
repository https://github.com/oleksandrdubyna88/dew_flow_benchmark (`todo/PLAN_lang_comparison_v2.md`).
