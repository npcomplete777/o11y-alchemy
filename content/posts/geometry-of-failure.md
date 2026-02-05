---
title: "The Geometry of Failure: Language-Agnostic Anti-Pattern Signatures in Distributed Trace Topology"
date: 2026-03-01
draft: false
tags: ["distributed-tracing", "anti-patterns", "trace-topology", "OpenTelemetry", "VALIS", "geometric-detection", "observability", "autonomous-remediation"]
categories: ["Research", "Case Study"]
author: "Aaron Jacobs"
description: "Anti-patterns in distributed systems have geometric shapes that are invariant across languages, frameworks, and protocols. Empirical evidence from two production systems demonstrates how trace topology analysis enables language-agnostic anti-pattern detection."
---

> "The real cycle you're working on is a cycle called yourself." — Robert M. Pirsig, *Zen and the Art of Motorcycle Maintenance*

## The Claim

I'm going to make a claim that I haven't seen anyone else make explicitly, and then I'm going to prove it with two empirical case studies from production systems.

**Anti-patterns in distributed systems produce characteristic geometric signatures in trace topology space. These signatures are invariant across programming languages, communication protocols, and observability platforms. A system that can classify trace geometry can detect anti-patterns without knowing anything about the underlying implementation.**

This isn't a theory. I've demonstrated it twice, on two completely different technology stacks, using the same detection framework. The detector didn't change. The languages changed. The protocols changed. The platforms changed. The geometry was identical.

## What Do I Mean by "Trace Geometry"?

A distributed trace is a tree. Each node (span) represents a unit of work. Edges represent causal relationships — "this span caused that span." Each node carries metadata: a name, a duration, a service attribution, and a status.

Most observability practice treats traces as debugging artifacts. You look at a trace when something goes wrong. You find the slow span. You fix it. This is like using a microscope to look at one cell at a time.

But traces have *topology*. The tree has a shape. That shape has measurable properties:

- **Fan-out**: How many children does a parent span have?
- **Homogeneity**: Are the children the same type of operation?
- **Temporality**: Are children sequential or concurrent?
- **Scaling behavior**: Does fan-out vary with input size?
- **Depth**: How deep is the call chain?

These properties define a point in what I'm calling *trace topology space* — a multi-dimensional space where each axis is a structural property of the trace tree. Different executions of the same code path produce points that cluster together in this space. Different anti-patterns produce clusters in different regions.

The insight: **these regions are the same regardless of what language generated the trace.**

An N+1 query in Go producing gRPC spans occupies the same region of trace topology space as an N+1 query in Python producing SQL spans. The implementation details are different. The geometry is identical.

## Case Study 1: The Checkout Service (Go + gRPC)

The first detection occurred in the checkout service of the OpenTelemetry Astronomy Shop, a polyglot microservices application running on k3s and exporting OTLP telemetry to Dash0.

VALIS — my autonomous observability system built on Claude with MCP-integrated tooling — queried Dash0 for recent spans from the checkout service and found this structure:

```
PlaceOrder (297ms)
└─ prepareOrderItemsAndShippingQuoteFromCart (132ms)
   ├─ CartService/GetCart (32ms)            ← 1 call
   ├─ ProductCatalogService/GetProduct (14ms)  ← Item 1
   ├─ CurrencyService/Convert (6ms)            ← Item 1
   ├─ ProductCatalogService/GetProduct (8ms)   ← Item 2
   ├─ CurrencyService/Convert (13ms)           ← Item 2
   ├─ ProductCatalogService/GetProduct (5ms)   ← Item 3
   └─ CurrencyService/Convert (9ms)            ← Item 3
```

The structural properties:

| Property | Value |
|----------|-------|
| Fan-out from parent | 2N + 1 (scales with cart size) |
| Child homogeneity | Repeating pair: GetProduct, Convert |
| Temporal pattern | Sequential (non-overlapping) |
| Scaling behavior | Linear with input cardinality |
| Protocol | gRPC |
| Language | Go |
| Data layer | Remote service calls |

The source code confirmed the geometry. In `src/checkout/main.go`, the `prepOrderItems` function iterated over cart items in a `for` loop, making one `GetProduct` and one `Convert` call per iteration. Sequential. Unbatched. O(2N+2).

The fix: batch RPC methods (`GetProducts`, `ConvertCurrencies`) that collapsed the fan-out to a constant 3 calls regardless of cart size. Post-deployment telemetry confirmed a 59% latency reduction.

The full detection-to-verification cycle — perceive, reason, act, verify — was executed autonomously. The commit is at [npcomplete777/opentelemetry-demo](https://github.com/npcomplete777/opentelemetry-demo), branch `fix/n-plus-one-checkout-batch`.

## Case Study 2: The Async Kafka Failure (Go + Kafka)

The second detection was structurally different but revealed through the same geometric analysis.

VALIS queried Dash0 for checkout spans and found an anomalous trace:

```
PlaceOrder (165,084ms)          ← 2.75 MINUTES
├─ prepareOrderItems (49ms)     ← Fast (batch fix working)
├─ ChargePayment (3ms)          ← Fast
├─ SendConfirmation (7ms)       ← Fast
├─ EmptyCart (2ms)               ← Fast
└─ sendToPostProcessor (165,000ms)  ← BLOCKING KAFKA WRITE
   └─ orders publish (142,732ms)
      └─ Kafka ack wait...
```

The structural properties:

| Property | Value |
|----------|-------|
| Fan-out from parent | N children, one dominant |
| Child homogeneity | Heterogeneous — different operations |
| Temporal pattern | Sequential, with one extreme outlier |
| Duration distribution | Bimodal: fast cluster + extreme tail |
| Protocol | Kafka (async messaging used synchronously) |
| Language | Go |
| Data layer | Message broker |

This is a different anti-pattern — *synchronous blocking on an asynchronous operation* — but the detection method was identical: analyze the trace tree's structural properties, classify the geometry, identify the pathology.

The source code confirmed it. In `sendToPostProcessor`, the checkout service was blocking on a Kafka acknowledgment `select` statement, turning a fire-and-forget operation into a synchronous call that could block for minutes. Meanwhile, the payment was charged, the confirmation email was sent, and the user saw a 504 timeout believing their order failed.

The fix: async fire-and-forget with background acknowledgment handling. Post-deployment: 4,700x latency improvement, from 165 seconds to 35 milliseconds. Zero human intervention. PR #3 on the same repository.

## The Geometry Is the Invariant

Now look at what's the same across both cases and what's different:

| Dimension | Case 1: N+1 gRPC | Case 2: Kafka Block |
|-----------|-------------------|---------------------|
| Language | Go | Go |
| Protocol | gRPC | Kafka |
| Anti-pattern | N+1 Query | Sync-over-Async |
| Fan-out | 2N+1 (linear scaling) | N (one dominant child) |
| Homogeneity | Repeating pairs | Heterogeneous |
| Temporality | Sequential, uniform | Sequential, one outlier |
| Duration dist. | Uniform, scaling | Bimodal, extreme tail |
| Detection method | **Trace topology analysis** | **Trace topology analysis** |
| Detection framework | **VALIS Bayesian engine** | **VALIS Bayesian engine** |

The anti-patterns are different. The geometries are different. But the *detection framework* is the same. It operates on structural properties of the trace tree — fan-out, homogeneity, temporality, duration distribution — and classifies the geometry against known signatures.

The framework didn't need to know it was looking at gRPC calls in Case 1 and Kafka writes in Case 2. It didn't need to read Go source code. It didn't need to understand protocol buffers or Kafka producer semantics. It classified *shapes*, not implementations.

This is the key insight: **anti-patterns have geometric signatures that are independent of implementation details.**

## Beyond N+1: A Taxonomy of Trace Geometries

If anti-patterns have characteristic geometries, then we can define a taxonomy — a classification system based on structural properties of trace trees.

Here are the signatures I've identified, expressed as regions in trace topology space:

**N+1 Query / Chatty API**
- Fan-out: Linear with input cardinality
- Homogeneity: High (repeating operation type)
- Temporality: Sequential
- Signature: "Comb" — many identical teeth hanging from one spine

**Sync-over-Async**
- Fan-out: Moderate, one dominant child
- Homogeneity: Low (mixed operation types)
- Duration distribution: Extreme bimodal
- Signature: "Lollipop" — small cluster of fast operations, one massively long stem

**Retry Storm**
- Fan-out: Repeated calls to same target service
- Temporality: Sequential with increasing inter-span gaps
- Terminal state: Timeout or error
- Signature: "Staircase" — descending steps of increasing latency

**Circuit Breaker Oscillation**
- Duration distribution: Periodic alternation between fast-fail and slow-fail
- Temporal pattern: Oscillating signal
- Signature: "Sawtooth" — repeating rise-and-crash pattern

**Connection Pool Exhaustion**
- Duration distribution: Bimodal (immediate vs. wait-timeout)
- Degradation: Progressive — healthy cluster shrinks, waiting cluster grows
- Signature: "Hourglass" — flow constricts at a single resource bottleneck

Each of these geometries is detectable through the same structural analysis framework. The Bayesian evidence streams differ — different properties matter for different patterns — but the *methodology* is invariant:

1. Extract structural properties from trace topology
2. Classify against known geometric signatures
3. Calculate confidence via Bayesian inference
4. Act on high-confidence detections

## Why This Hasn't Been Said Before

The pieces of this observation exist independently in the literature:

**Brendan Gregg's flame graphs** proved that performance problems have visual signatures — you can *see* the pathology in the shape. But flame graphs are single-process. They don't capture distributed topology.

**Charity Majors and the Honeycomb team** proved that high-cardinality, high-dimensionality trace data enables novel debugging. But Honeycomb's interaction model is human-driven query exploration, not automated geometric classification.

**The OpenTelemetry project** standardized span semantics across languages and frameworks, making cross-platform structural analysis possible. But OTel focuses on data collection and export, not topological analysis.

**The SEI at Carnegie Mellon** defined architectural anti-patterns in graph-theoretic terms. But they work at the static dependency level, not runtime trace topology.

Nobody connected these pieces because, until recently, there was no *agent* that could operate on trace geometry programmatically. Humans could see shapes in flame graphs. Monitoring tools could apply threshold rules. But nothing could ingest a trace tree, measure its topological properties, run probabilistic classification, and *act* on the result.

The theoretical framework wasn't useful without an execution engine. So nobody formalized it.

I built the execution engine first. The theory followed.

## The Execution Engine: VALIS

VALIS (Vast Active Living Intelligence System) is an autonomous observability platform built on Claude with MCP servers providing access to:

- **Dash0**: Span querying, log analysis, alerting, sampling rules
- **kubectl**: Kubernetes cluster state
- **GitHub**: Source code access, branch creation, PR workflow
- **ArgoCD**: GitOps deployment and verification
- **VALIS statistical engine**: Bayesian inference, temporal analysis, anomaly detection, Granger causality, Monte Carlo simulation

The MCP (Model Context Protocol) architecture is what makes geometric detection possible. Each MCP server exposes a specific capability as a tool the AI agent can invoke. The agent orchestrates across tools — querying telemetry from Dash0, analyzing topology with the statistical engine, correlating to source code via GitHub, deploying fixes through ArgoCD, and verifying improvements by querying Dash0 again.

No single tool detects anti-patterns. The detection is an **emergent property** of connected tools orchestrated by a reasoning agent. Dash0 provides perception. The statistical engine provides classification. GitHub provides code correlation. ArgoCD provides remediation. Claude provides the reasoning that connects them.

This emergence is why the framework generalizes across anti-pattern types. The agent doesn't follow a script that says "count GetProduct calls." It reasons: "this trace tree has high fan-out, homogeneous children, sequential execution, and linear scaling. That geometric signature matches the N+1 pattern with 99.2% Bayesian confidence."

Change the children from GetProduct to SQL queries to HTTP calls to Kafka writes — the geometry doesn't change. The confidence doesn't change. The detection doesn't change.

## The Entropy Connection

There's a deeper principle underneath the geometry.

An anti-pattern, in thermodynamic terms, is a code path that passes through *unnecessarily many states* to achieve an outcome reachable through fewer states. The N+1 pattern makes 2N+2 RPC calls when 3 would suffice. Each call adds network round trips, connection states, failure modes, timing interleavings. The *configuration space* of possible system states during execution is vastly larger than necessary.

The batch fix collapses that configuration space. Fewer calls, fewer states, fewer ways for things to go wrong. The trace tree shrinks. The geometry simplifies.

Anti-pattern detection through trace geometry is, at its core, **entropy measurement**. High fan-out with sequential homogeneous children is high entropy — many redundant microstates. The geometric signature *is* the entropy signature.

Fixing the anti-pattern reduces local entropy. The system's behavior becomes more predictable. The trace tree occupies a smaller, tighter region of topology space.

VALIS is, in effect, a Maxwell's Demon for distributed systems — observing microstates (spans), acquiring information (Bayesian inference), and acting to reduce local entropy (code fixes), all while exporting the entropic cost into computation and token consumption.

## Implications

If anti-patterns have language-agnostic geometric signatures, several things follow:

**Detection scales across polyglot architectures.** The OpenTelemetry Astronomy Shop has services in Go, Python, JavaScript, Java, Ruby, C++, Rust, PHP, .NET, Erlang, and Kotlin. A geometric detector doesn't care. Spans are spans. Trees are trees. Shapes are shapes.

**Detection is vendor-agnostic.** I proved the N+1 detection on Dash0 with OTLP data. The same framework works on any backend that exposes trace data through a queryable interface — Honeycomb, Jaeger, Tempo, Datadog, Dynatrace. The geometry exists in the trace, not in the platform.

**New anti-patterns can be discovered empirically.** If you can measure trace topology, you can cluster traces by geometric similarity and discover anti-pattern signatures you didn't know to look for. The taxonomy doesn't need to be predefined. It can be *learned from production traffic*.

**The human role shifts from pattern-matching to architecture.** If an agent can detect geometric signatures and implement fixes autonomously, engineers stop being debuggers and become architects of the systems that enable autonomous detection. The leverage changes by orders of magnitude.

## What This Means for Observability

The observability industry has spent a decade arguing about three pillars versus events versus traces versus metrics. The geometric perspective dissolves most of that debate.

What matters is the *topology*. If your telemetry preserves parent-child relationships, timing, service attribution, and operation names — if it preserves the *shape* of execution — then geometric analysis can extract anti-pattern signatures regardless of how you label the data.

Traces preserve topology. Metrics don't. Logs don't (unless correlated). This is why Charity Majors has been right about traces for years, but for a reason she may not have fully articulated: traces are the only telemetry type that preserves the geometric structure of distributed execution.

And geometric structure is where the anti-patterns live.

## Conclusion

I've demonstrated, with two empirical case studies on production systems, that anti-patterns in distributed systems produce characteristic geometric signatures in trace topology space. These signatures are invariant across programming languages, communication protocols, and observability platforms.

The N+1 pattern in Go gRPC calls looks the same as an N+1 pattern in any other language and protocol. The sync-over-async pattern in Kafka has a distinct but equally recognizable geometry. Both were detected by the same framework operating on structural properties of trace trees — fan-out, homogeneity, temporality, duration distribution — without knowledge of the underlying implementation.

This isn't monitoring. This isn't dashboarding. This isn't even observability in the traditional sense.

This is **computational geometry applied to distributed system behavior**.

The traces are the substrate. The shapes are the signal. The agent is the classifier.

And the shapes are universal.

---

*Aaron Jacobs is a Principal Technical Consultant specializing in observability and AI. He is the creator of VALIS (Vast Active Living Intelligence System), an autonomous observability platform. His previous posts on closed-loop observability are available at [o11y-alchemy.com](https://o11y-alchemy.com). The detection framework, fixes, and deployment history are publicly available at [github.com/npcomplete777/opentelemetry-demo](https://github.com/npcomplete777/opentelemetry-demo).*

*The code commits are the proof. The geometry is the insight. The question isn't whether this is possible — it's who will formalize it next.*
