---
title: "The Geometry of Failure: Language-Agnostic Anti-Pattern Signatures in Distributed Trace Topology"
date: 2026-02-04
draft: false
tags: ["distributed-tracing", "anti-patterns", "trace-topology", "language-invariance", "Go", "Python", "Java", "OpenTelemetry", "geometric-detection", "Bayesian-inference", "VALIS", "autonomous-remediation"]
categories: ["Research", "Experiments"]
author: "Aaron Jacobs"
description: "Anti-patterns in distributed systems produce characteristic geometric signatures in trace topology. I tested this hypothesis across Go, Python, and Java — and the geometry was identical every time."
---

> "The real cycle you're working on is a cycle called yourself." — Robert M. Pirsig, *Zen and the Art of Motorcycle Maintenance*

## The Claim

I'm going to make a claim that I haven't seen anyone else make explicitly, and then back it up with empirical evidence from three production experiments.

**Anti-patterns in distributed systems produce characteristic geometric signatures in trace topology space. These signatures are invariant across programming languages, runtimes, and instrumentation strategies. A system that classifies trace geometry can detect anti-patterns without knowing anything about the underlying implementation.**

I've proven this for the N+1 query pattern across Go, Python, and Java — three fundamentally different runtimes, three different instrumentation strategies, identical geometry every time. I've also observed it for a second anti-pattern (sync-over-async blocking) in a Go + Kafka system. Whether it holds across all anti-pattern types and all languages is the subject of ongoing investigation — but the evidence so far is compelling enough to formalize the framework.

## What I Mean by "Trace Geometry"

A distributed trace is a tree. Each node (span) represents a unit of work. Edges represent causal relationships — "this span caused that span."

Most observability practice treats traces as debugging artifacts: you look at a trace when something is broken, find the slow span, fix it. But traces have *topology*. The tree has a shape. That shape has measurable properties:

**Fan-out.** How many children does a parent span produce, and how does that number relate to input size? In an N+1 pattern, fan-out scales linearly with the input collection.

**Homogeneity.** Are child spans the same type of operation? In an N+1 pattern, all the repeating children have the same span name and target service. The tree looks like a comb — one spine with many identical teeth.

**Temporality.** Are children concurrent or sequential? In an N+1 pattern, each child starts only after its predecessor completes. There are small inter-span gaps (typically 10–20 microseconds) between the end of one child and the start of the next — the signature of a `for` loop, not a `Promise.all`.

**Scaling.** Does fan-out change with input? In an N+1 pattern, doubling the input collection doubles the number of child spans. The relationship is strictly linear.

These four dimensions define a point in what I call *trace topology space*. My hypothesis: the same anti-pattern, regardless of language, occupies the same point in this space.

## The Experiments

The OpenTelemetry Astronomy Shop is a polyglot microservices application with services in Go, Python, Java, JavaScript, C++, Rust, and others. It ships with feature flags managed by flagd that can inject controlled failures and anti-patterns. I ran the same N+1 query anti-pattern across three services:

**Experiment 0 (baseline): Go checkout service.** This service was already exhibiting a naturally occurring N+1 pattern — `prepOrderItems` iterates over cart items and makes individual `GetProduct` and `Convert` calls per item. No injection needed; the anti-pattern was in production code.

**Experiment 1: Python recommendation service** with the `recommendationServiceNPlusOne` feature flag enabled. When active, the service iterates over recommended product IDs and calls `GetProduct` individually for each one instead of using the batch `ListProducts()` result. The injection is committed to the public OpenTelemetry demo repository ([`e0ff415`](https://github.com/open-telemetry/opentelemetry-demo/commit/e0ff4158cfb738cfba0c6befc7e793b8564300f5)), with custom span attributes for full observability — `app.recommendation.mode`, `app.recommendation.product_count`, and `app.recommendation.sequential_call_index` to tag each loop iteration.

```python
if check_feature_flag("recommendationServiceNPlusOne"):
    span.set_attribute("app.recommendation.mode", "n_plus_one")
    span.set_attribute("app.recommendation.product_count", len(prod_list))

    for idx, product_id in enumerate(prod_list):
        with tracer.start_as_current_span("GetProduct") as product_span:
            product_span.set_attribute("app.recommendation.sequential_call_index", idx)
            product = product_catalog_stub.GetProduct(
                demo_pb2.GetProductRequest(id=product_id))
```

**Experiment 2: Java ad service** with the `adServiceNPlusOne` feature flag enabled. Same logical pattern — iterate over ad results, call `GetProduct` per item — but running on the JVM with bytecode-injected instrumentation.

```java
for (int i = 0; i < productIds.size(); i++) {
    Span productSpan = tracer.spanBuilder("GetProduct").startSpan();
    try (Scope ignored = productSpan.makeCurrent()) {
        productSpan.setAttribute("app.ad.sequential_call_index", i);
        Product product = catalogStub.getProduct(
            GetProductRequest.newBuilder().setId(productIds.get(i)).build());
    } finally {
        productSpan.end();
    }
}
```

Each experiment ran against the same observability backend. The analyzer queried live span data via OTLP, analyzed the trace topology *without being told what language the service was written in*, and scored confidence using sequential Bayesian inference.

## What the Analyzer Saw

### Go Checkout Service

```
PlaceOrder (297ms)
└─ prepareOrderItemsAndShippingQuoteFromCart (132ms)
   ├─ CartService/GetCart (32ms)                ← The "1"
   ├─ ProductCatalogService/GetProduct (14ms)   ← Item 1
   ├─ CurrencyService/Convert (6ms)             ← Item 1
   ├─ ProductCatalogService/GetProduct (8ms)    ← Item 2
   ├─ CurrencyService/Convert (13ms)            ← Item 2
   ├─ ProductCatalogService/GetProduct (5ms)    ← Item 3
   └─ CurrencyService/Convert (9ms)             ← Item 3
```

Runtime: compiled Go binary. Instrumentation: manual OTel SDK calls written by the developer. Bayesian confidence: **99.9%.**

### Python Recommendation Service

```
ListRecommendations (8.2ms)
├─ get_product_list (0.89ms)          ← The "1"
├─ GetProduct (0.61ms)                 ← Product 1
├─ GetProduct (1.42ms)                 ← Product 2
├─ GetProduct (0.38ms)                 ← Product 3
├─ GetProduct (1.25ms)                 ← Product 4
└─ GetProduct (0.55ms)                 ← Product 5
```

Runtime: interpreted CPython. Instrumentation: automatic via `opentelemetry-instrument` monkey-patching — no developer-written span code. Bayesian confidence: **99.9%.**

### Java Ad Service

```
oteldemo.AdService/GetAds (3.6ms)
├─ getAdsByCategory (0.02ms)                           ← The "1"
├─ GetProduct [0PUK6V6EV0] (0.62ms)                    ← Ad 1
│  └─ oteldemo.ProductCatalogService/GetProduct (0.28ms)
├─ GetProduct [6E92ZMYYFZ] (0.33ms)                    ← Ad 2
│  └─ oteldemo.ProductCatalogService/GetProduct (0.14ms)
└─ GetProduct [L9ECAV7KIM] (0.53ms)                    ← Ad 3
   └─ oteldemo.ProductCatalogService/GetProduct (0.28ms)
```

Runtime: JVM bytecode on HotSpot. Instrumentation: hybrid — the OpenTelemetry javaagent injects bytecode at class load time, while the application uses the OTel API directly for some spans. Bayesian confidence: **99.9%.**

## The Comparison

| Geometric Property | Go Checkout | Python Recommendation | Java Ad | Match? |
|---|---|---|---|---|
| Fan-out formula | 2N+1 (pairs) | N+1 | N+1 | **Yes*** |
| Homogeneity | Repeating pairs | Repeating singles | Repeating singles | **Yes** |
| Temporality | Sequential | Sequential (13–17μs gaps) | Sequential (~15μs gaps) | **Yes** |
| Scaling | Linear | Linear | Linear | **Yes** |
| P(N+1) | 99.9% | 99.9% | 99.9% | **Yes** |
| Runtime | Compiled native | Interpreted CPython | JVM bytecode | Differ |
| Instrumentation | Manual SDK | Auto monkey-patch | Auto bytecode inject | Differ |

*\*Go produces pairs (GetProduct + Convert) per item, giving 2N+1 direct children. Python and Java produce singles, giving N+1. The structural pattern — linear fan-out of repeated calls — is identical. The coefficient differs because Go's checkout bundles a currency conversion with each product fetch.*

The bottom two rows are the only differences, and they're all implementation details. None of them affected the geometry.

## What This Proves

Three fundamentally different execution models produced the same trace topology.

**Go** compiled the loop to native machine code. A developer manually wrapped each call in `tracer.Start()` / `span.End()`.

**Python** interpreted the loop bytecode. An agent injected at process startup monkey-patched the gRPC library — no developer involvement in span creation.

**Java** JIT-compiled the loop from bytecode. The javaagent transformed classes at load time. A hybrid model where some spans came from automatic injection and some from explicit API calls.

None of this mattered to the detector. It examined span parent-child relationships, counted fan-out, checked homogeneity, measured sequential timing. It never asked what language the service was written in.

The N+1 pattern is a property of the *algorithm* — iterate over a collection, make one call per item. That algorithm produces a characteristic trace shape regardless of how the machine executes it. The shape is the invariant.

## Beyond N+1: A Taxonomy of Trace Geometries

If anti-patterns have characteristic geometries, we can define a taxonomy — a classification based on structural properties of trace trees. Five signatures emerge from this work. N+1 / Comb is experimentally validated across three languages; the others are based on observed cases and represent the next candidates for multi-language validation.

**N+1 Query / Chatty API — "The Comb"**
- Fan-out: Linear with input cardinality
- Homogeneity: High (repeating operation type)
- Temporality: Sequential
- *Status: Experimentally validated across Go, Python, Java*

**Sync-over-Async — "The Lollipop"**
- Fan-out: Moderate, one dominant child
- Duration distribution: Extreme bimodal (fast cluster + one extreme outlier)
- *Status: Observed in Go + Kafka; cross-language validation pending*

**Retry Storm — "The Staircase"**
- Fan-out: Repeated calls to same target
- Temporality: Sequential with increasing inter-span gaps
- Terminal state: Timeout or error
- *Status: Hypothesized; not yet validated experimentally*

**Circuit Breaker Oscillation — "The Sawtooth"**
- Duration distribution: Periodic alternation between fast-fail and slow-fail
- *Status: Hypothesized; not yet validated experimentally*

**Connection Pool Exhaustion — "The Hourglass"**
- Duration distribution: Bimodal (immediate vs. wait-timeout)
- Degradation: Progressive — healthy cluster shrinks, waiting cluster grows
- *Status: Hypothesized; not yet validated experimentally*

Each geometry is detectable through the same structural analysis framework. The Bayesian evidence streams differ — different properties matter for different patterns — but the methodology is invariant.

## The Three-Layer Architecture

The N+1 case study reveals a clean separation into three layers:

**Layer 1: Geometry (universal).** The structural properties of the trace tree — fan-out, temporal pattern, duration distribution, homogeneity. This is where detection happens. Language-agnostic. Framework-agnostic. Vendor-agnostic.

**Layer 2: Semantics (language-aware).** The span attributes that explain the geometry — `code.function`, `thread.id`, `runtime.name`, `rpc.system`. OpenTelemetry semantic conventions provide this layer. When the geometric detector flags a pattern, semantic attributes narrow the diagnosis: "this is a Python service, the repeating span is `GetProduct`, the sequential call index attribute is incrementing — N+1 in a recommendation loop."

**Layer 3: Remediation (implementation-specific).** The actual code fix. The N+1 fix in Go uses `errgroup` for parallel execution. The same anti-pattern in Java might use `CompletableFuture.allOf`. The async-blocking fix in Go uses a goroutine. Each language has its own idiom — but the fix is only applied because the geometry was detected, and the geometry was detected without knowing the language.

This layering means the same detection framework generalizes across every language that emits OpenTelemetry spans, while still producing diagnosis and remediation guidance that's specific enough to be actionable.

## How the Detection Works

The Bayesian Trace Topology Analyzer uses sequential Bayesian inference to classify trace geometries. For each anti-pattern, evidence signals are binary observations about the trace tree's structural properties, each with a calibrated true positive rate (TPR) and false positive rate (FPR).

For N+1 detection, the evidence chain is:

1. **Repeating child spans** — does the parent have multiple children with the same operation name?
2. **Sequential execution** — do children execute one after another, not concurrently?
3. **Same operation name** — are the repeating children calling the same downstream method?
4. **Linear scaling** — does child count correlate with input cardinality?
5. **High child count** — are there more than 2–3 children?

Starting from a conservative 3% prior (the base rate of N+1 patterns across all traces), five positive evidence signals with TPR=0.8 and FPR=0.1 drive the posterior through: 3% → 19.8% → 66.4% → 94.1% → 99.2% → 99.9%.

The same engine, the same evidence signals, the same update chain — applied to Go traces, Python traces, and Java traces. Same result every time. Because the geometry is the same every time.

## Closing the Loop

Detecting the anti-pattern is only half the claim. The other half: the geometric signature points directly at the fix, and the fix visibly changes the trace topology.

After the analyzer flagged the Go checkout service's `prepOrderItems` function at 99.9% confidence, I refactored it to use parallel goroutines via `errgroup`, processing all cart items concurrently ([`cbf332e`](https://github.com/open-telemetry/opentelemetry-demo/commit/cbf332ea25fdf3cbe9139f3b91c3ae7ad27e11cc)). The trace geometry changed immediately. The comb — one spine with many sequential identical teeth — collapsed into a fan: concurrent children with overlapping start times. Bayesian confidence for N+1 dropped to near zero. The detector confirmed the fix without any code-level analysis; it simply observed that the geometric signature had disappeared.

The 59% latency reduction followed from the geometry change. The full investigation is documented in detail [here](/posts/trace-to-fix-n-plus-one-case-study/).

## Why This Hasn't Been Said Before

The pieces of this observation exist independently in the literature:

**Brendan Gregg's flame graphs** proved that performance problems have visual signatures — you can *see* the pathology in the shape. But flame graphs are single-process and don't capture distributed topology.

**Charity Majors and the Honeycomb team** proved that high-cardinality trace data enables novel debugging. But the interaction model is human-driven query exploration, not automated geometric classification.

**OpenTelemetry** standardized span semantics across languages and frameworks, making cross-platform structural analysis possible. But OTel focuses on data collection and export, not topological analysis.

**The SEI at Carnegie Mellon** defined architectural anti-patterns in graph-theoretic terms. But they work at the static dependency level, not runtime trace topology.

Nobody connected these pieces because, until recently, there was no *agent* that could operate on trace geometry programmatically. Humans could see shapes in flame graphs. Monitoring tools could apply threshold rules. But nothing could ingest a trace tree, measure its topological properties, run probabilistic classification, and *act* on the result.

The theoretical framework wasn't useful without an execution engine. So nobody formalized it. I built the execution engine first. The theory followed.

## What This Means for Observability

**One detector covers all languages.** You don't need a Go N+1 detector, a Python N+1 detector, and a Java N+1 detector. You need one trace topology analyzer that recognizes comb geometry. It works on any service that emits OpenTelemetry spans. A polyglot architecture with eleven languages gets coverage from one classifier.

**Detection is vendor-agnostic.** The experiments ran against an OTel-native backend with OTLP data. The same analysis works on Dynatrace, Datadog, Jaeger, Tempo, or any backend that stores parent-child span relationships. The geometry lives in the data, not the platform.

**New languages get coverage for free.** When someone adds a Rust service or a Kotlin service, the N+1 detector doesn't need updating. If the new service has a `for` loop making individual RPC calls, the trace will have the same comb shape.

**Traces are the only telemetry that preserves execution topology.** Metrics don't. Logs don't (unless correlated). This is why traces are the right foundation for anti-pattern detection — and why the observability industry's long argument about "which pillar matters most" misses the point. The answer is whichever telemetry preserves the geometric structure of distributed execution. That's traces.

## What's Next

I've proven language invariance for the N+1 / Comb pattern. The next experiments will test the other geometries in the taxonomy:

- **Retry Storm / Staircase** — does the stepping pattern look the same whether retries are implemented with Go's `for` loop, Python's `tenacity`, or Java's Spring Retry?
- **Sync-over-Async / Lollipop** — does blocking on an async operation produce the same dominant-child signature across Go channels, Python `asyncio`, and Java `CompletableFuture`? Early evidence from the Kafka fix suggests it does — but that's one language and one runtime.
- **Circuit Breaker Oscillation / Sawtooth** — does the open/half-open/closed oscillation create the same periodic temporal signal across different circuit breaker libraries?

If the hypothesis continues to hold — and the algorithmic nature of anti-patterns suggests it should — then a single geometric detection framework can classify the full taxonomy across any OpenTelemetry-instrumented architecture.

One shape language. Every programming language.

---

*Aaron Jacobs is a Principal Technical Consultant specializing in observability and AI. He is the creator of VALIS (Vast Active Living Intelligence System), an autonomous observability platform that performs Bayesian inference over trace topology to detect and remediate anti-patterns autonomously. The detection framework, fixes, and deployment history are publicly available at [github.com/npcomplete777/opentelemetry-demo](https://github.com/npcomplete777/opentelemetry-demo).*

*The code commits are the proof. The geometry is the insight.*
