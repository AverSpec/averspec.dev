---
title: "Six Languages, No Loop"
draft: true
---

The Ruby port had 331 passing tests and I rewrote the entire API before it shipped. The code worked. I didn't like how it felt. Block DSL to class inheritance. Every public surface changed. The behavioral spec validated the result in seconds. 327 tests passed. I hadn't looked at the implementation.

Then I did the same kind of pass across every port. Identified a DX issue, told the agent to fix it, checked the acceptance tests. Go, Rust, Kotlin, Python — each had language-specific friction that the first draft got wrong. Each fix was one or two agent dispatches. None required me to read the generated code. I checked two things: does the acceptance test count hold, and would I want to write code like this.

[Aver](https://github.com/averspec/aver) is a testing framework where you define behavior once and run it against unit, HTTP, and browser adapters. I built the TypeScript version in about three weeks. Then I ported it to Python, Ruby, Go, Rust, and Kotlin in a single session on a $100/month Claude Max plan. ~2,000 tests across all six, all CI green, all self-testing through their own API.

## The wall

There's a moment in every AI-assisted project where the complexity catches up. The agent produces code fast. Then you need to change something foundational. The refactor touches ten files. Three of them break in ways the agent doesn't notice. You spend an hour verifying by hand. Next time you're afraid to change anything. You start working around problems instead of fixing them. The codebase calcifies.

I never hit that wall. Not on the original three-week TypeScript build. Not on any of the five ports. Not during the DX polish passes that touched every public API surface across every language.

The Ruby API rewrite changed every adapter, every test, every example. Go's domain system was redesigned to eliminate a generics limitation that forced an awkward `.D` indirection on every marker access — thirty files changed. Python's async handling was redesigned twice. Kotlin's test API was reworked to avoid a language keyword collision. The acceptance tests across every port were migrated from plain assertions to domain-mediated operations — forty files per language.

Any of those could have been a "burn it down" moment. They weren't, because the behavioral spec caught every regression within seconds. Green or red. The agent could be aggressive because the feedback was immediate.

The same spec that made the ports cheap is what kept the refactors safe. And the refactors are what made the APIs good. Without confidence to change things, you ship the first draft.

## Why it didn't need a loop

[Cloudflare rebuilt Next.js](https://blog.cloudflare.com/vinext/) with 800 AI sessions. [Geoff Huntley clones products](https://ghuntley.com/loop/) with a bash loop that retries until tests pass. Both work. Both iterate hundreds of times because each attempt starts with limited context.

The Aver ports ran in a single conversation with a million-token context window. By the third language, the agent had seen how Python solved the async/sync boundary, how Ruby's RSpec load-time evaluation broke adapter registration, how Go's parallel test execution raced on environment variables, which simulated panel review caught a design flaw before any code was written. It didn't rediscover any of that through error messages. It applied it.

The behavioral spec gave the agent an unambiguous target. "Register two adapters, the test runs against both" is true in any language. The context window gave it accumulated judgment. Both together meant each port was two passes: the agent builds, I check the acceptance count and the API feel, the agent closes the gaps.

The real leverage came after the initial ports. The DX improvement cycle — auditing every port for language-idiomatic feel, fixing the friction points, re-validating against the spec — happened across five languages simultaneously with minimal effort because every fix was verified instantly. The spec doesn't just enable porting. It enables polishing. It enables building docs with multi-language code examples that stay synced because each language's API is tested against the same behavioral contract.

Honest caveat: the spec wasn't free. Three weeks of TypeScript work produced the 534 tests that made everything after it possible. Huntley starts from zero and ships in days. Different trade-off. But once a behavioral spec exists, it amortizes. Every port. Every refactor. Every polish pass.

## How it played out

Python was first and most hands-on. Design decisions about decorators, async strategy, pytest plugin architecture. A simulated panel of Python experts caught three design flaws before a line of implementation was written. That review cost ten minutes and saved what would have been a full rewrite.

Ruby was mostly hands-off. The agent built the core. I checked acceptance count and API feel. Both triggered more work. The agent handled it without further direction.

Go, Rust, and Kotlin launched as parallel background agents. Each came back with a working framework, tests, CI, and an example app. The Rust agent solved a borrow checker problem I hadn't anticipated — separating the immutable domain reference from the mutable test context in a way that produced a cleaner API than I'd have designed manually.

Then the polish cycle. A panel reviewed all six APIs for idiom fit and principle of least surprise. Every port had issues:

- Go forced an awkward `.D` field on every marker access — a generics limitation. The agent redesigned the domain registry to eliminate it. `TaskBoard.D.AddTask` became `TaskBoard.AddTask`.
- Rust used opaque tuples for payloads and named every marker twice. A declarative macro and named structs fixed both.
- Kotlin needed backtick escaping on `when` because it's a reserved keyword. Capitalizing all three — `ctx.Given()`, `ctx.When()`, `ctx.Then()` — reads like BDD specs and sidesteps the keyword entirely.
- Python forced dataclass imports for every payload. Kwargs support on the proxy made them optional. `ctx.when.create_task(title="Fix bug")` instead of importing and instantiating a class.
- Ruby's block DSL was clever but not how Ruby developers write code. Classes with methods replaced blocks with handles.

Each fix was verified against the spec. No regression. No wall. The APIs went from working to good.

The TypeScript original has one DX issue that none of the ports share: every test must be `async` with `await` on every proxy call, even for purely synchronous unit adapters. JavaScript can't synchronously block on a Promise, so if any adapter might be async, every test must be async. Python solved this by hiding the event loop inside the proxy. The ports taught me something about the original.

## What carries

The behavioral spec is the load-bearing piece. If your tests describe what the system does without mentioning how, an agent can build against them in any language and verify its own changes. If your tests are coupled to your runtime, every change is a gamble.

The context window replaces the retry loop. A conversation that carries the full implementation, the design decisions, and the mistakes from every previous attempt produces code that accounts for lessons already learned.

The combination prevents the wall. But it also enables something more interesting than just "working code fast." It enables *good* code iteratively. The spec gives you the confidence to throw away the first draft. The context gives the agent enough judgment to make the second draft better. The polish cycle — audit, fix, verify — is where the quality actually lives. Speed gets you to the first draft. The spec is what lets you keep improving it.

Every AI-assisted refactor is either verified against a spec or verified by hope. The first compounds. The second calcifies.

---

~2,000 tests. Six languages. One session. [$100](https://github.com/averspec).

[Aver](https://github.com/averspec/aver) · [Python](https://github.com/averspec/aver-py) · [Ruby](https://github.com/averspec/aver-rb) · [Go](https://github.com/averspec/aver-go) · [Rust](https://github.com/averspec/aver-rs) · [Kotlin](https://github.com/averspec/aver-jvm)
