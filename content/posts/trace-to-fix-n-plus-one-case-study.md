---
title: "From Trace Data to Production Fix: Detecting and Remediating an N+1 Query Anti-Pattern"
date: 2026-02-01
draft: false
tags: ["MCP", "distributed-tracing", "anti-patterns", "GitOps", "OpenTelemetry"]
categories: ["Case Study"]
author: "Aaron Jacobs"
description: "A case study in closed-loop observability: using distributed traces, MCP-integrated tooling, and GitOps to identify, fix, and verify a performance anti-pattern in a microservices application."
---

Modern observability platforms generate vast amounts of telemetry data—traces, metrics, and logs—that often go underutilized beyond basic dashboarding and alerting. This article demonstrates a complete workflow for extracting actionable intelligence from distributed traces: detecting a performance anti-pattern, implementing a fix, deploying it through GitOps, and validating the improvement—all orchestrated through an AI assistant integrated with observability and infrastructure tooling via the Model Context Protocol (MCP).

The target application is the OpenTelemetry Astronomy Shop, a reference microservices application running on k3s (via OrbStack) and exporting OTLP telemetry to Dash0. The entire workflow was executed using Claude as an AI assistant with MCP servers providing access to Dash0 (observability), kubectl (Kubernetes), GitHub (source control), and ArgoCD (GitOps deployment).

## The Technology Stack

Before diving into the detection and remediation process, it's worth understanding the infrastructure that made this workflow possible:

- **OrbStack with k3s**: A lightweight Kubernetes distribution running locally for development, providing a realistic multi-node cluster environment.
- **OpenTelemetry Demo (Astronomy Shop)**: A polyglot microservices application with 15+ services demonstrating real-world observability patterns.
- **Dash0**: Cloud-native observability platform receiving OTLP telemetry, providing span querying and visualization.
- **ArgoCD**: GitOps continuous delivery, syncing Kubernetes manifests from GitHub to the cluster.
- **MCP (Model Context Protocol)**: Anthropic's protocol enabling AI assistants to interact with external tools and services through standardized interfaces.

## Phase 1: Anti-Pattern Detection Through Trace Analysis

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

## Phase 2: Implementing the Batch RPC Fix

### The Solution: Batch Operations

The fix required two components: adding batch methods to the downstream services and refactoring the checkout service to use them.

**ProductCatalogService**: Added a `GetProducts` method that accepts an array of product IDs and returns all products in a single response.

**CurrencyService**: Added a `ConvertCurrencies` method that accepts an array of amounts and converts them all in one call.

**CheckoutService**: Refactored `prepOrderItems` to collect all product IDs upfront, make a single batch call, then process results.

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

## Phase 3: Validation and an Unexpected Discovery

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

## Key Insights

### 1. Distributed Traces Are Underutilized

Most organizations collect traces but use them only for request debugging. The structural information in traces—parent-child relationships, span counts, timing distributions—can reveal architectural anti-patterns that would be invisible in metrics alone.

### 2. Symptoms Can Mask Root Causes

The "Kafka latency bottleneck" was actually OOMKilled restarts. The N+1 pattern detection led to the correct fix (batch RPCs), but the full picture required correlating telemetry with infrastructure state. Without checking pod status, we might have spent days debugging a Kafka issue that didn't exist.

### 3. MCP Enables Closed-Loop Observability

The Model Context Protocol allowed seamless integration of observability (Dash0), infrastructure (kubectl), source control (GitHub), and deployment (ArgoCD) into a single conversational workflow. The AI assistant could perceive the system state, reason about anti-patterns, and take corrective action—closing the observability loop.

### 4. Resource Limits Matter

A 20Mi memory limit for a Go service with Kafka and OTel instrumentation is a configuration error waiting to cause problems. The Go runtime's `GOMEMLIMIT` environment variable should be set to ~90% of the container memory limit to allow the garbage collector to operate efficiently within bounds.

## Conclusion

This case study demonstrates the power of combining AI assistants with observability tooling through standardized protocols like MCP. What began as an investigation into checkout latency led to discovering and fixing two distinct issues: an N+1 query anti-pattern in the application code and a resource starvation problem in the Kubernetes configuration.

The entire workflow—from detection through validation—was executed through natural language interaction with an AI assistant that had access to the necessary tools. This represents a shift from passive observability (dashboards and alerts) to active observability (AI-assisted detection, diagnosis, and remediation).

As observability platforms expose more capabilities through programmatic interfaces like MCP, we can expect these closed-loop workflows to become increasingly common—and increasingly autonomous.

---

## References

- [OpenTelemetry Demo](https://github.com/open-telemetry/opentelemetry-demo)
- [Model Context Protocol](https://modelcontextprotocol.io)
- [Dash0](https://dash0.com)
- [ArgoCD](https://argo-cd.readthedocs.io)
- [Go GOMEMLIMIT](https://pkg.go.dev/runtime#hdr-Environment_Variables)
