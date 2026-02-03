---
title: "Closed-Loop Observability: From Trace Detection to Automated Remediation"
date: 2026-02-01
draft: false
tags: ["MCP", "distributed-tracing", "anti-patterns", "GitOps", "OpenTelemetry", "closed-loop"]
categories: ["Case Study"]
author: "Aaron Jacobs"
description: "A case study in closed-loop observability: using distributed traces, MCP-integrated tooling, and GitOps to identify, fix, and verify a performance anti-pattern in a microservices application—demonstrating the shift from passive monitoring to active remediation."
---

Modern observability platforms generate vast amounts of telemetry data—traces, metrics, and logs—that often go underutilized beyond basic dashboarding and alerting. This article demonstrates a complete workflow for extracting actionable intelligence from distributed traces: detecting a performance anti-pattern, implementing a fix, deploying it through GitOps, and validating the improvement—all orchestrated through an AI assistant integrated with observability and infrastructure tooling via the Model Context Protocol (MCP).

The target application is the OpenTelemetry Astronomy Shop, a reference microservices application running on k3s (via OrbStack) and exporting OTLP telemetry to Dash0. The entire workflow was orchestrated using Claude as an AI assistant with MCP servers providing access to Dash0 (observability), kubectl (Kubernetes), GitHub (source control), and ArgoCD (GitOps deployment).

## The Technology Stack

Before diving into the detection and remediation process, it's worth understanding the infrastructure that made this workflow possible:

- **OrbStack with k3s**: A lightweight Kubernetes distribution running locally for development, providing a realistic multi-node cluster environment.
- **OpenTelemetry Demo (Astronomy Shop)**: A polyglot microservices application with 15+ services demonstrating real-world observability patterns.
- **Dash0**: Cloud-native observability platform receiving OTLP telemetry, providing span querying and visualization.
- **ArgoCD**: GitOps continuous delivery, syncing Kubernetes manifests from GitHub to the cluster.
- **MCP (Model Context Protocol)**: Anthropic's protocol enabling AI assistants to interact with external tools and services through standardized interfaces.

### Why MCP Matters for Observability

Traditional observability tools provide UIs and dashboards—human interfaces for human investigation. MCP inverts this: it exposes observability platforms as programmatic APIs that AI assistants can query, correlate, and act upon.

This enables a shift from:
- **Observability 1.0**: "Set up dashboards, hope you graphed the right thing"
- **Observability 2.0**: "Store rich events, query anything ad-hoc"
- **Observability 3.0**: "AI agents query telemetry, detect patterns, take action"

The Model Context Protocol provides standardized tool interfaces (like LSP did for code editors) so AI assistants can interact with any observability backend, infrastructure system, or development tool through a common protocol.

## 1. Anti-Pattern Detection Through Trace Analysis

### Querying Span Data

The investigation began with a simple query to the Dash0 MCP server, requesting recent spans from the checkout service:

```
dash0_spans_query(service_name="checkout", time_range_minutes=30, limit=100)
```

The returned spans revealed a concerning pattern in the `prepareOrderItemsAndShippingQuoteFromCart` function. For each item in a shopping cart, the checkout service was making sequential RPC calls:

| Span Name | Duration | Pattern |
|-----------|----------|---------|
| PlaceOrder | 10.78ms | Root operation |
| prepareOrderItems... | 5.58ms | N+1 initiator |
| GetProduct #1 | 0.57ms | **N+1: Item 1** |
| Convert #1 | 1.06ms | **N+1: Item 1 currency** |
| GetProduct #2 | 0.20ms | **N+1: Item 2** |
| Convert #2 | 0.43ms | **N+1: Item 2 currency** |

The pattern was unmistakable: for each item in the cart, the service made one call to `ProductCatalogService.GetProduct` and one call to `CurrencyService.Convert`. This results in 2N sequential RPC calls for N cart items—classic N+1 query behavior, but manifesting as RPC calls rather than database queries.

### Correlating Traces to Source Code

Using the GitHub MCP server, I retrieved the source code for the checkout service. The problematic function was immediately apparent in `src/checkout/main.go`:

```go
func (cs *checkout) prepOrderItems(ctx context.Context,
    items []*pb.CartItem, userCurrency string) ([]*pb.OrderItem, error) {
    out := make([]*pb.OrderItem, len(items))
    for i, item := range items {
        // N+1: GetProduct called for EACH item
        product, err := cs.productCatalogSvcClient.GetProduct(...)
        // N+1: Convert called for EACH item
        price, err := cs.convertCurrency(ctx, product.GetPriceUsd()...)
    }
```

The trace data directly mapped to the code structure: each span represented one iteration of the loop. With 10 items in a cart, this pattern would generate 22 RPC calls (2×10 + GetCart + GetShippingQuote) instead of the optimal 4.

## 2. Implementing the Batch RPC Fix

### The Solution: Batch Operations

The fix required two components: adding batch methods to the downstream services and refactoring the checkout service to use them.

**ProductCatalogService**: Added a `GetProducts` method that accepts an array of product IDs and returns all products in a single response.

**CurrencyService**: Added a `ConvertCurrencies` method that accepts an array of amounts and converts them all in one call.

**CheckoutService**: Refactored `prepOrderItems` to collect all product IDs upfront, make a single batch call, then process results.

The implementation can be viewed in the [GitHub commit history](https://github.com/npcomplete777/opentelemetry-demo/commits/fix/n-plus-one-checkout-batch).

### Deploying Through GitOps

With the code changes committed to GitHub, ArgoCD automatically detected the new commit and began synchronization. Using the ArgoCD MCP server, I monitored the deployment:

```
argo_app_sync(name="otel-demo")
argo_app_wait(name="otel-demo", health=true, sync=true)
```

The kubectl MCP server confirmed the new pods were running:

```
k8s_get(resource="pods", namespace="otel-demo",
        selector="app.kubernetes.io/component in (checkout,product-catalog,currency)")
```

## 3. Validation and an Unexpected Discovery

### Initial Validation: Pattern Fix Confirmed

After deployment, I queried Dash0 again to verify the batch pattern was active:

```
dash0_spans_query(service_name="checkout", time_range_minutes=15)
```

The traces now showed the expected batch pattern:

```
prepOrderItems (979.56ms)
├── GetProducts (458.18ms)      ✓ BATCH
└── ConvertCurrencies (272.72ms) ✓ BATCH
```

The N+1 pattern was eliminated. Instead of 2N+2 calls, we now had a constant 3 calls regardless of cart size. However, the absolute latencies were still high—and some traces showed `prepOrderItems` taking 7-10 seconds.

### The Unexpected Discovery: Resource Starvation

While investigating the latency variance, I noticed something alarming in the Kubernetes pod status:

```
checkout-55488cb7cc-7bzlw: CrashLoopBackOff, 44 restarts
Last State: Terminated, Reason: OOMKilled, Exit Code: 137
Memory Limit: 20Mi  ← CRITICALLY LOW
```

The checkout service had a memory limit of only 20Mi—absurdly low for a Go service running gRPC, Kafka producers, and OpenTelemetry instrumentation. The service was being OOMKilled every few minutes, and the high latencies in the traces were actually connection re-establishment delays after restarts.

What initially appeared to be a "Kafka latency bottleneck" (the `orders publish` span was showing 12+ seconds) was actually a symptom of the service restarting mid-request and needing to re-establish its Kafka producer connection.

### Fixing the Resource Allocation

Using kubectl, I patched the deployment to provide adequate resources:

```
k8s_patch(resource="deployment", name="checkout", namespace="otel-demo",
  patch={"spec":{"template":{"spec":{"containers":[{
    "name":"checkout",
    "resources":{"limits":{"memory":"200Mi"},"requests":{"memory":"100Mi"}},
    "env":[{"name":"GOMEMLIMIT","value":"180MiB"}]
  }]}}}})
```

Similar patches were applied to the currency and shipping services, which also had 20Mi limits.

## Results: Before and After

After both fixes (batch RPC pattern + resource allocation), the improvement was dramatic:

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| RPC calls (3 items) | 8 calls | 3 calls | 62.5% |
| RPC calls (10 items) | 22 calls | 3 calls | 86.4% |
| Kafka publish latency | 12,178ms | 0.07-0.17ms | **~100,000x** |
| PlaceOrder latency | 7,000ms+ | 8-41ms | **~99%** |
| Pod restarts (30 min) | 44 | 0 | **100%** |
| Error spans | Intermittent | 0 | **100%** |

## Why This Pattern Matters: From Detection to Remediation

This workflow demonstrates what the observability community calls "closing the loop". Traditional approaches stop at detection:

1. **Monitor**: Set up alerts for known failure modes
2. **React**: Get paged when something breaks
3. **Debug**: Manually investigate logs and metrics
4. **Fix**: Write code, test, deploy
5. **Hope**: Pray it doesn't happen again

The closed-loop approach extends through remediation:

1. **Perceive**: AI queries telemetry for anti-patterns
2. **Reason**: Correlates traces with code structure
3. **Act**: Implements fix, tests, deploys via GitOps
4. **Verify**: Confirms improvement in production telemetry
5. **Learn**: Pattern detection improves for next time

This isn't replacing SREs—it's **amplifying** their effectiveness by handling the mechanical parts of detection and remediation, freeing humans to focus on architectural decisions and complex investigations.

## Key Insights

### 1. Distributed Traces Are Underutilized

Most organizations collect traces but use them only for request debugging. The structural information in traces—parent-child relationships, span counts, timing distributions—can reveal architectural anti-patterns that would be invisible in metrics alone.

### 2. Symptoms Can Mask Root Causes

The "Kafka latency bottleneck" was actually OOMKilled restarts. The N+1 pattern detection led to the correct fix (batch RPCs), but the full picture required correlating telemetry with infrastructure state. Without checking pod status, we might have spent days debugging a Kafka issue that didn't exist.

### 3. MCP Enables Closed-Loop Observability

The Model Context Protocol allowed seamless integration of observability (Dash0), infrastructure (kubectl), source control (GitHub), and deployment (ArgoCD) into a single conversational workflow. The AI assistant could perceive the system state, reason about anti-patterns, and take corrective action—closing the observability loop.

### 4. Resource Limits Matter

A 20Mi memory limit for a Go service with Kafka and OTel instrumentation is a configuration error waiting to cause problems. The Go runtime's `GOMEMLIMIT` environment variable should be set to ~90% of the container memory limit to allow the garbage collector to operate efficiently within bounds.

## What Remained Human Decisions

While the AI assistant orchestrated the technical workflow, several critical decisions remained human:

- **Recognizing** that 2N+2 calls was a problem worth fixing (vs. acceptable)
- **Choosing** batch RPCs over caching or other optimization strategies
- **Validating** that the resource limit fix was correct (not just trusting the AI)
- **Deciding** this was production-ready (not just passing tests)
- **Determining** the architectural approach (batch operations vs. caching vs. async processing)

The AI amplified my engineering productivity by handling the mechanical work—code generation, configuration management, deployment orchestration—but the strategic and architectural decisions were mine. This is observability-driven development, not autonomous coding.

## Conclusion: Observability as Code

This case study demonstrates a fundamental shift in how we think about observability tooling. By exposing observability platforms through programmatic interfaces like MCP, we transform telemetry from a passive data source into an active participant in the development lifecycle.

The key insight: **observability instrumentation is code**, and code can reason about code. When your traces contain rich context (service names, operation types, resource attributes, timing data), AI assistants can:

- Detect anti-patterns structurally (N+1 queries, chatty APIs, retry storms)
- Correlate symptoms with root causes (Kafka latency ← OOMKilled restarts)
- Implement fixes that are architecturally appropriate (batch RPCs, not caching)
- Verify improvements in production telemetry (closing the loop)

This isn't replacing engineers—it's **amplifying their leverage**. What took days of manual investigation, code changes, and deployment became hours of orchestrated automation. The engineer provides the judgment and domain expertise; the AI provides the mechanical execution.

AI is fundamentally an amplifier. It magnifies the strengths of organizations with robust observability while exposing the dysfunctions of those without it. This case study proves the point: rich telemetry data enables AI-assisted remediation, but only when your observability practice is already sound.

The future of observability isn't more dashboards—it's closed-loop systems that perceive, reason, and act on production telemetry. MCP is one protocol making this future possible.

---

## References

- [OpenTelemetry Demo](https://github.com/open-telemetry/opentelemetry-demo)
- [Model Context Protocol](https://modelcontextprotocol.io)
- [Dash0](https://dash0.com)
- [ArgoCD](https://argo-cd.readthedocs.io)
- [Go GOMEMLIMIT](https://pkg.go.dev/runtime#hdr-Environment_Variables)
- [GitHub Commit: N+1 Fix](https://github.com/npcomplete777/opentelemetry-demo/commit/1b14533451b0d2a1ff9070585178925399986fcb)
