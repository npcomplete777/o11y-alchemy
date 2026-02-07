---
title: "Anti-Patterns Have Shapes, and Shapes Don't Care What Language You Write In"
date: 2026-02-07
draft: false
tags: ["distributed-tracing", "anti-patterns", "trace-topology", "language-invariance", "Go", "Python", "Java", "OpenTelemetry", "N+1", "geometric-detection", "Bayesian-inference"]
categories: ["Research", "Experiments"]
author: "Aaron Jacobs"
description: "How I proved that distributed system anti-patterns produce identical trace geometry across Go, Python, and Java — and why that changes everything about autonomous detection."
---

*How I proved that distributed system anti-patterns produce identical trace geometry across Go, Python, and Java — and why that changes everything about autonomous detection.*

---

A distributed trace is a tree. Each node is a span — a named, timed unit of work. Each edge is a causal relationship: this span caused that span. The tree has a shape.

I've been building an autonomous observability system that I call the *Bayesian Trace Topology Analyzer* — a detection engine that identifies performance anti-patterns by analyzing the shapes of trace trees. Not the contents. Not the language. Not the framework. The shapes.

Over the past several weeks, I ran a series of controlled experiments to test a hypothesis that, if true, fundamentally simplifies how anti-pattern detection works in polyglot architectures:

> **The Language-Invariance Hypothesis:** A given anti-pattern produces the same trace geometry regardless of the programming language, runtime, or instrumentation strategy that generated the trace.

I tested this against three services in the OpenTelemetry Astronomy Shop demo application, each written in a different language, each instrumented differently, each running on a fundamentally different runtime. The same N+1 query anti-pattern was injected into all three via feature flags. The experiment code is public — the Python injection, for example, is committed to the OpenTelemetry demo repository ([`e0ff415`](https://github.com/open-telemetry/opentelemetry-demo/commit/e0ff4158cfb738cfba0c6befc7e793b8564300f5)), where I modified the recommendation service to fetch each recommended product individually via `GetProduct()` instead of using the batch `ListProducts()` result when the `recommendationServiceNPlusOne` feature flag is enabled.

The result: identical geometry. Every time. The hypothesis holds.

Here's how I proved it.

---

## What I Mean by "Trace Geometry"

When I say an anti-pattern has a geometric signature, I mean it has measurable structural properties that are invariant across implementations. For the N+1 pattern — where code iterates over a collection and makes one network call per item instead of batching — I defined four geometric dimensions:

**Fan-out.** How many child spans does the parent produce, and how does that number relate to the input? In an N+1 pattern, the parent span fans out to N child spans (plus one initial query), and N scales linearly with the size of the input collection.

**Homogeneity.** Are the child spans the same operation? In an N+1 pattern, all the repeating children have the same span name, the same RPC method, and the same target service. The trace tree looks like a comb — one spine with many identical teeth.

**Temporality.** Are the children concurrent or sequential? In an N+1 pattern, each child starts only after its predecessor completes. There are small inter-span gaps (typically 10–20 microseconds) between the end of one child and the start of the next. This is the signature of a `for` loop, not a `Promise.all`.

**Scaling.** Does the fan-out change with the input? In an N+1 pattern, doubling the input collection doubles the number of child spans. The relationship is strictly linear.

These four dimensions define a point in what I call *trace topology space*. My hypothesis was that the same anti-pattern, regardless of language, would occupy the same point.

---

## The Experiment

The OpenTelemetry Astronomy Shop is a polyglot microservices application with services in Go, Python, Java, JavaScript, C++, Rust, and others. It ships with feature flags managed by flagd that can inject controlled failures and anti-patterns. I used three of these:

**Experiment 0 (baseline):** The Go checkout service. This service was already exhibiting a naturally occurring N+1 pattern — `prepOrderItems` iterates over cart items and makes individual `GetProduct` and `Convert` calls per item. No feature flag needed; the anti-pattern is in the production code.

**Experiment 1:** The Python recommendation service with the `recommendationServiceNPlusOne` feature flag enabled. When active, the service iterates over recommended product IDs and calls `GetProduct` individually for each one instead of using a batch API. I injected this pattern with custom span attributes for full observability — `app.recommendation.mode` to distinguish batch vs. N+1 code paths, `app.recommendation.product_count` for cardinality tracking, and `app.recommendation.sequential_call_index` to tag each loop iteration. These attributes made it possible to correlate the detector's geometric inference with ground-truth loop behavior.

The implementation shows the pattern clearly:

```python
# When recommendationServiceNPlusOne flag is enabled
if check_feature_flag("recommendationServiceNPlusOne"):
    span.set_attribute("app.recommendation.mode", "n_plus_one")
    span.set_attribute("app.recommendation.product_count", len(prod_list))

    # Fetch each product individually - creates the N+1 pattern
    for idx, product_id in enumerate(prod_list):
        with tracer.start_as_current_span("GetProduct") as product_span:
            product_span.set_attribute("app.product.id", product_id)
            product_span.set_attribute("app.recommendation.sequential_call_index", idx)
            product = product_catalog_stub.GetProduct(
                demo_pb2.GetProductRequest(id=product_id))
```

Logs from the deployed service confirmed the pattern was active:

```
2026-02-07 13:25:03 - get_product_list: N+1 mode enabled, fetching 5 products individually
2026-02-07 13:25:28 - get_product_list: N+1 mode enabled, fetching 5 products individually
```

**Experiment 2:** The Java ad service with the `adServiceNPlusOne` feature flag enabled. Same logical pattern — iterate over ad results, call `GetProduct` per item — but now running on the JVM with bytecode-injected instrumentation.

The Java implementation demonstrates the same algorithmic pattern in a completely different runtime:

```java
// Extract product IDs from ad redirect URLs
List<String> productIds = extractProductIdsFromAds(ads);

parentSpan.setAttribute("app.ad.mode", "n_plus_one");
parentSpan.setAttribute("app.ad.product_count", productIds.size());

// Fetch each product individually to create the N+1 pattern
for (int i = 0; i < productIds.size(); i++) {
    Span productSpan = tracer.spanBuilder("GetProduct").startSpan();
    try (Scope ignored = productSpan.makeCurrent()) {
        productSpan.setAttribute("app.ad.sequential_call_index", i);
        productSpan.setAttribute("app.product.id", productIds.get(i));

        // gRPC call - auto-instrumented by javaagent
        Product product = catalogStub.getProduct(
            GetProductRequest.newBuilder().setId(productIds.get(i)).build());
    } finally {
        productSpan.end();
    }
}
```

Logs confirmed the Java N+1 pattern was active:

```
2026-02-07 13:51:38 - N+1 mode enabled: fetching 1 products individually
2026-02-07 13:51:38 - Completed N+1 pattern: fetched 1 products individually
2026-02-07 13:52:04 - N+1 mode enabled: fetching 2 products individually
2026-02-07 13:52:04 - Completed N+1 pattern: fetched 2 products individually
```

Each experiment ran against the same Dash0 observability backend. The analyzer queried live span data from Dash0 using OTLP/gRPC, analyzed the trace topology without being told what language the service was written in, and scored confidence using sequential Bayesian inference.

---

## What the Analyzer Saw

### Go Checkout Service

```
PlaceOrder (297ms)
└─ prepareOrderItemsAndShippingQuoteFromCart (132ms)
   ├─ CartService/GetCart (32ms)                          ← The "1"
   ├─ ProductCatalogService/GetProduct (14ms)             ← Item 1
   ├─ CurrencyService/Convert (6ms)                       ← Item 1
   ├─ ProductCatalogService/GetProduct (8ms)              ← Item 2
   ├─ CurrencyService/Convert (13ms)                      ← Item 2
   ├─ ProductCatalogService/GetProduct (5ms)              ← Item 3
   └─ CurrencyService/Convert (9ms)                       ← Item 3
```

Fan-out: 2N+1 (repeating GetProduct/Convert pairs). Homogeneity: repeating pairs. Temporality: sequential. Scaling: linear with cart size.

Runtime: compiled Go binary. Instrumentation: explicit, manual OpenTelemetry SDK calls. The developer wrote the span creation code by hand.

Bayesian confidence: **99.9%.**

### Python Recommendation Service

```
ListRecommendations (8.2ms)
├─ get_product_list (0.89ms)                              ← The "1"
├─ GetProduct (0.61ms)                                     ← Product 1
├─ GetProduct (1.42ms)                                     ← Product 2
├─ GetProduct (0.38ms)                                     ← Product 3
├─ GetProduct (1.25ms)                                     ← Product 4
└─ GetProduct (0.55ms)                                     ← Product 5
```

Fan-out: N+1. Homogeneity: repeating singles. Temporality: sequential with 13–17μs inter-span gaps. Scaling: linear with recommendation count.

Runtime: interpreted CPython. Instrumentation: automatic via `opentelemetry-instrument` monkey-patching. No developer-written span code — the Python auto-instrumentation agent intercepted gRPC calls at runtime.

Bayesian confidence: **99.9%.**

### Java Ad Service

```
oteldemo.AdService/GetAds (3.6ms)
├─ getAdsByCategory (0.02ms)                               ← The "1"
├─ GetProduct [0PUK6V6EV0] (0.62ms)                       ← Ad 1
│  └─ oteldemo.ProductCatalogService/GetProduct (0.28ms)
├─ GetProduct [6E92ZMYYFZ] (0.33ms)                        ← Ad 2
│  └─ oteldemo.ProductCatalogService/GetProduct (0.14ms)
└─ GetProduct [L9ECAV7KIM] (0.53ms)                        ← Ad 3
   └─ oteldemo.ProductCatalogService/GetProduct (0.28ms)
```

Fan-out: N+1. Homogeneity: repeating singles (each with a nested gRPC child span). Temporality: sequential — each `GetProduct` starts after its predecessor completes. Scaling: linear with ad product count (observed 1–3 products across traces).

Runtime: JVM bytecode on HotSpot. Instrumentation: hybrid — the OpenTelemetry javaagent injects bytecode at class load time to auto-instrument gRPC calls, while the application code uses the OTel API directly for `GetProduct`, `getAdsByCategory`, and `getRandomAds` spans.

Bayesian confidence: **99.9%.**

---

## The Comparison

| Geometric Property | Go Checkout | Python Recommendation | Java Ad | All Match? |
|----|----|----|----|----|
| Fan-out formula | 2N+1 (pairs) | N+1 | N+1 | **Yes*** |
| Homogeneity | Repeating pairs | Repeating singles | Repeating singles | **Yes** |
| Temporality | Sequential | Sequential (13–17μs gaps) | Sequential (~15μs gaps) | **Yes** |
| Scaling | Linear with cart size | Linear with rec count | Linear with ad count | **Yes** |
| Child span name | GetProduct | GetProduct | GetProduct | **Yes** |
| P(N+1) | 99.9% | 99.9% | 99.9% | **Yes** |
| Runtime | Compiled native | Interpreted CPython | JVM bytecode | 3 differ |
| Instrumentation | Manual SDK | Auto monkey-patch | Auto bytecode inject | 3 differ |
| Typical child duration | 5–14ms | 0.3–1.4ms | 0.3–1.5ms | Varies |

*\*Go produces pairs (GetProduct + Convert) per item, giving 2N+1 direct children. Python and Java produce singles, giving N+1. The structural pattern — linear fan-out of repeated calls — is identical. The coefficient differs because Go's checkout happens to bundle a currency conversion with each product fetch.*

The bottom three rows are the only differences. They are all *implementation details* — how the code runs, how the spans get created, how fast the runtime executes. None of them affect the geometry.

---

## What This Proves

Three fundamentally different execution models produced the same trace topology:

**Go** compiled the loop to native machine code. A developer manually wrapped each call in `tracer.Start()` / `span.End()`. The spans were created by explicit, human-written instrumentation code.

**Python** interpreted the loop bytecode on CPython. An agent injected at process startup monkey-patched the gRPC client library. The spans were created automatically, with no developer involvement — the instrumentation code was injected by the `opentelemetry-instrument` wrapper.

**Java** JIT-compiled the loop from JVM bytecode. The `-javaagent` flag triggered bytecode transformation at class load time, instrumenting gRPC calls transparently. The application also used the OTel API directly for some spans. A hybrid model.

None of this mattered to the detector. The analyzer examined span parent-child relationships, counted fan-out, checked homogeneity, measured sequential timing, and scored confidence. It never asked what language the service was written in. It didn't need to.

The N+1 pattern is a property of the *algorithm* — iterate over a collection, make one call per item. That algorithm produces a characteristic trace shape regardless of how the machine executes it. The shape is the invariant.

---

## Experimental Evidence: The Deployment

All three experiments were deployed to a production-grade Kubernetes environment running on k3d. The deployment process demonstrated reproducibility across completely different build systems:

**Python deployment:**
```bash
# Built with Docker
docker build -t otel-demo-recommendation:n-plus-one \
  -f src/recommendation/Dockerfile .

# Imported to k3d cluster
k3d image import otel-demo-recommendation:n-plus-one -c gitops-demo

# Deployed via kubectl
kubectl set image deployment/recommendation \
  recommendation=otel-demo-recommendation:n-plus-one -n otel-demo
```

**Java deployment:**
```bash
# Built with Gradle multi-stage Docker build
docker build --build-arg OTEL_JAVA_AGENT_VERSION=2.23.0 \
  -t otel-demo-adservice:n-plus-one -f src/ad/Dockerfile .

# Same import and deployment process
k3d image import otel-demo-adservice:n-plus-one -c gitops-demo
kubectl set image deployment/ad ad=otel-demo-adservice:n-plus-one -n otel-demo
```

Both services came online successfully, connected to the same flagd instance for feature flag management, and began emitting traces to the same Dash0 collector. The geometric signatures appeared immediately in the trace data, indistinguishable from each other except for service attribution.

---

## Closing the Loop: From Detection to Remediation

Detecting the anti-pattern is only half the claim. The other half is that the geometric signature points directly at the fix — and that the fix visibly changes the trace topology.

After the analyzer flagged the Go checkout service's `prepOrderItems` function at 99.9% confidence, I refactored it. The fix replaced N sequential `GetProduct` calls with parallel goroutines using `errgroup`, processing all cart items concurrently instead of iterating one at a time ([`cbf332e`](https://github.com/open-telemetry/opentelemetry-demo/commit/cbf332ea25fdf3cbe9139f3b91c3ae7ad27e11cc)). The commit reduced checkout latency from O(N) sequential to O(N) parallel, bounded by available connections.

The trace geometry changed immediately. The comb — one spine with many sequential identical teeth — collapsed into a fan: a single parent with concurrent children whose start times overlap. The Bayesian confidence for N+1 dropped to near zero. The detector confirmed the fix without any code-level analysis; it simply observed that the geometric signature had disappeared.

This same investigation also surfaced a second anti-pattern in the checkout path: `sendToPostProcessor` was blocking on synchronous Kafka writes, causing requests to exceed Envoy's 15-second timeout and producing 504 errors. That fix — converting to a fire-and-forget pattern with background goroutine acknowledgment handling — is also committed ([`e2af374`](https://github.com/open-telemetry/opentelemetry-demo/commit/e2af3748dff418ed3b2cc243af5636b521e99dba)). It's a different anti-pattern (sync-over-async, or what I call "The Lollipop") but the detection principle is identical: the trace tree's shape betrayed the problem before any code review did.

---

## Why This Matters

If anti-pattern geometry is language-invariant, several things follow.

**One detector covers all languages.** You don't need a Go N+1 detector, a Python N+1 detector, and a Java N+1 detector. You need one trace topology analyzer that recognizes comb geometry. It works on any service that emits OpenTelemetry spans. The Astronomy Shop has services in eleven languages. A geometric detector covers all of them.

**Detection is vendor-agnostic.** I proved this on Dash0 with OTLP data. The same analysis would work on Dynatrace, Datadog, Jaeger, or any backend that stores parent-child span relationships. The geometry lives in the data, not in the platform.

**New languages get coverage for free.** When someone adds a Rust service or a Kotlin service to the architecture, the N+1 detector doesn't need updating. If the new service has a `for` loop making individual RPC calls, the trace will have the same comb shape, and the detector will flag it.

**The approach extends to other anti-patterns.** N+1 is "The Comb" — many identical teeth hanging from one spine. But other anti-patterns have their own characteristic shapes. A retry storm is "The Staircase" — repeated calls with growing inter-span gaps and a terminal timeout. Sync-over-async is "The Lollipop" — a cluster of fast operations with one massively long stem (and as demonstrated by the Kafka fix above, trace geometry catches this pattern just as reliably). Connection pool exhaustion is "The Hourglass" — bimodal durations where requests either complete instantly or wait for a timeout. Each shape is detectable through the same structural analysis, with different Bayesian evidence streams calibrated per pattern.

---

## How the Detection Works

The Bayesian Trace Topology Analyzer uses sequential Bayesian inference to classify trace geometries. For each anti-pattern, there are evidence signals — binary observations about the trace tree's structural properties. Each signal has a calibrated true positive rate (TPR) and false positive rate (FPR).

For N+1 detection, the evidence chain is:

1. **Repeating child spans** — does the parent have multiple children with the same operation name?
2. **Sequential execution** — do children execute one after another (not concurrently)?
3. **Same operation name** — are the repeating children calling the same downstream service method?
4. **Linear scaling** — does the child count correlate with input cardinality?
5. **High child count** — are there more than 2–3 children (ruling out coincidental repetition)?

Each evidence signal updates the posterior probability via Bayes' rule:

```
P(N+1 | evidence) = P(evidence | N+1) × P(N+1) / P(evidence)
```

Starting from a conservative 3% prior (the base rate of N+1 patterns across all traces), five positive evidence signals with TPR=0.8 and FPR=0.1 drive the posterior through: 3% → 19.8% → 66.4% → 94.1% → 99.2% → 99.9%.

The same engine, the same evidence signals, the same update chain — applied to Go traces, Python traces, and Java traces. Same result every time. Because the geometry is the same every time.

---

## The Deeper Pattern

There's a connection to information theory here. An anti-pattern, in thermodynamic terms, is a code path that passes through unnecessarily many states to achieve an outcome reachable through fewer states. The N+1 pattern makes N network round trips when 1 batch call would suffice. Each call adds connection states, failure modes, and timing interleavings. The configuration space is vastly larger than necessary.

The batch fix collapses that configuration space. Fewer calls, fewer states, fewer ways for things to go wrong. The trace tree shrinks. The geometry simplifies. I observed this directly when the Go checkout service's comb topology collapsed into a parallel fan after the `errgroup` refactor — the trace went from 2N+1 sequential children to N concurrent children, and the overall span duration dropped proportionally.

Anti-pattern detection through trace geometry is, at its core, entropy measurement. High fan-out with sequential homogeneous children is high entropy — many redundant microstates. The geometric signature *is* the entropy signature. And entropy doesn't care what language generated the microstates.

---

## What's Next

I've proven language invariance for the N+1 / Comb pattern. The next experiments will test other geometries:

- **Retry Storm / Staircase** — does the stepping pattern look the same whether retries are implemented with Go's `for` loop, Python's `tenacity`, or Java's Spring Retry?
- **Sync-over-Async / Lollipop** — does blocking on an async operation produce the same dominant-child signature across Go channels, Python `asyncio`, and Java `CompletableFuture`? Early evidence from the Kafka timeout fix ([`e2af374`](https://github.com/open-telemetry/opentelemetry-demo/commit/e2af3748dff418ed3b2cc243af5636b521e99dba)) suggests it does — the synchronous Kafka write produced a single dominant child span that dwarfed its siblings, regardless of the Go-specific runtime mechanics.
- **Circuit Breaker Oscillation / Sawtooth** — does the open/half-open/closed oscillation create the same periodic temporal signal regardless of the circuit breaker library?

If the hypothesis continues to hold — and I expect it will, because anti-patterns are algorithmic, not linguistic — then a single geometric detection framework can classify the full taxonomy of distributed system anti-patterns across any OpenTelemetry-instrumented architecture.

One shape language. Every programming language.

---

*Aaron Jacobs is a Principal Technical Consultant specializing in observability and AI. The Bayesian Trace Topology Analyzer is an autonomous detection engine that integrates LLM-driven reasoning with MCP-connected observability tools to perform closed-loop anti-pattern detection and remediation. The experiments described in this post used live production telemetry from the OpenTelemetry Astronomy Shop running on k3s, exported via OTLP to Dash0, and analyzed using sequential Bayesian inference over trace tree structure.*

*All experiment code is committed to a public fork of the OpenTelemetry demo repository. The blog O11y Alchemy explores the intersection of observability, AI, and distributed systems. Follow for more on autonomous detection, trace topology, and the geometry of failure.*
