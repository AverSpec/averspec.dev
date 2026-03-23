---
title: "Telemetry"
---

> **Experimental.** Telemetry verification is functional and tested but has not been validated in production environments. The correlation model, `causes` API, and contract verification workflow may evolve based on real-world usage. If you're using this in practice, we'd love to hear about your experience — [open an issue](https://github.com/averspec/aver/issues) or [start a discussion](https://github.com/averspec/aver/discussions).

This guide covers decisions and advanced usage beyond the [Telemetry Tutorial](/tutorial-telemetry/). If you haven't done the tutorial, start there — it walks through declarations, collector setup, failure modes, and basic correlation step by step.

## When to add telemetry

Not every domain needs telemetry declarations. Add them when:

- **Business-critical flows** — payment, order, auth — where missing observability is a production risk
- **Cross-boundary operations** — HTTP calls, queues, external services — where trace propagation matters
- **Compliance requirements** — audit trails that must be proven observable

Skip telemetry for internal plumbing, admin CRUD, and dev tooling.

## Causal correlation with `causes`

The tutorial covers basic correlation — operations sharing an attribute key with the same value. But some operations trigger other operations asynchronously: a task assignment fires a notification via a queue, a checkout triggers fulfillment via an event bus.

The `causes` declaration tells the verifier about these causal relationships:

```typescript
assignTask: action<{ title: string; assignee: string }>({
  telemetry: (p) => ({
    span: 'task.assign',
    attributes: { 'task.title': p.title },
    causes: ['notification.process'],
  }),
}),
```

This says: "when `task.assign` runs, it should produce a `notification.process` span that is causally connected — either in the same trace or linked via a span link." If the spans are in different traces with no link, verification fails:

```
'task.assign' declares causes: ['notification.process'] but spans are in
different traces with no link. Propagate trace context or add a span link
at the async boundary.
```

Use `causes` when your code explicitly triggers the downstream operation (queues, event buses, async workers). Don't use it for operations that happen to share an entity but are triggered independently.

## Practical tips

### Spans must arrive before verification runs

Aver verifies telemetry immediately after each adapter handler returns. If your application emits spans asynchronously (batched exporter, background worker), flush them before the handler returns:

```typescript
actions: {
  async createTask(ctx, payload) {
    const res = await ctx.post('/api/tasks', payload)
    await flushTracing() // Ensure spans reach the OTLP receiver
    return res.json()
  },
}
```

For async operations like queued workers, drain the queue first:

```typescript
async assignTask(ctx, payload) {
  await ctx.patch(`/api/tasks/${payload.title}`, { assignee: payload.assignee })
  await drainQueue()    // Wait for background worker to finish
  await flushTracing()  // Then flush all spans
}
```

### What gets verified and what doesn't

- Operations without `telemetry` declarations are not verified — no warning, no error. You won't get feedback about missing declarations unless you add them.
- Adapters without a `TelemetryCollector` on their protocol skip telemetry verification entirely. This is by design for unit adapters that don't use real OTel spans.

## Dev-to-production verification

Test-time telemetry verification proves your system emits the right spans in a controlled environment. But does production actually emit the same spans with the same attributes? Code paths differ, middleware interferes, instrumentation gets refactored away. The `@averspec/telemetry` package closes this gap with `extractContract()` and `verifyContract()`.

### The flow

1. Run your tests. Each passing test produces a trace of domain operations with telemetry expectations.
2. `extractContract()` distills those traces into a **behavioral contract** — a portable description of what spans production must emit.
3. Collect OTLP traces from production (or staging).
4. `verifyContract()` checks the contract against those real traces and reports violations.

### Extracting a contract

```typescript
import { extractContract } from '@averspec/telemetry'
import { signupFlow } from './domains/signup-flow'

const contract = extractContract({
  domain: signupFlow,
  results: testResults, // from your test runner
})
```

The contract captures two kinds of attribute bindings:

- **Literal** — fixed values from static telemetry declarations. `telemetry: { span: 'order.cancel', attributes: { 'order.status': 'cancelled' } }` becomes `{ kind: 'literal', value: 'cancelled' }`. Production must emit that exact value.
- **Correlated** — parameterized values discovered via proxy-based field tracking. `telemetry: (p) => ({ attributes: { 'user.email': p.email } })` becomes `{ kind: 'correlated', symbol: '$email' }`. Production doesn't need to match the test's specific email — but every span referencing `$email` within a single trace must carry the *same* value.

### Verifying against production traces

```typescript
import { verifyContract } from '@averspec/telemetry'
import type { ProductionTrace } from '@averspec/telemetry'

const productionTraces: ProductionTrace[] = [
  {
    traceId: 'abc123',
    spans: [
      { name: 'user.signup', attributes: { 'user.email': 'jane@example.com' } },
      { name: 'account.created', attributes: { 'account.email': 'jane@example.com' } },
    ],
  },
]

const report = verifyContract(contract, productionTraces)
```

### Violation types

**`missing-span`** — a span the contract expects was not found in a matching trace.

**`literal-mismatch`** — a span attribute has a different value than the contract requires. The domain says cancellation sets status to `'cancelled'`; production says `'pending'`.

**`correlation-violation`** — two spans that should reference the same entity carry different values. Within a single trace, the signup and account-creation spans reference different email addresses.

Traces that don't contain the anchor span are silently skipped — unrelated traffic won't generate false positives.

### Exporting traces from Jaeger

Query the Jaeger HTTP API and save the response:

```bash
curl -s 'http://localhost:16686/api/traces?service=my-app&limit=100' -o traces.json
```

Pass the file to `FileTraceSource` and use it with `verifyContract()`. This is the fastest way to get production traces into the verification pipeline.

## Span naming conventions

Follow OTel semantic conventions: `{noun}.{verb}` or `{service}.{operation}`.

- `order.checkout`, `order.fulfill`, `notification.send`
- NOT `doCheckout`, `handleFulfillment`

## Design considerations

Telemetry declarations can live on domain markers or on adapters. Both are valid:

- **Domain markers** — when observability is a business requirement. "Checkout must be traceable" or "every payment must emit an audit span." Declaring it on the marker makes it visible to anyone reading the domain and ensures every adapter satisfies the requirement.
- **Adapters** — when telemetry is an implementation detail. If only the HTTP adapter needs spans for debugging, putting telemetry on the adapter keeps the domain clean.

Some teams start with adapter-level telemetry and promote to domain when they realize observability is load-bearing. Others start at domain level and move down when the constraints are too rigid. Either direction works — be intentional about whether a span is a business promise or an engineering convenience.
