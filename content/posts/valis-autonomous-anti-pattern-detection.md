---
title: "VALIS: Autonomous Anti-Pattern Detection for Production Observability"
date: 2026-02-01
draft: false
tags: ["VALIS", "MCP", "Dash0", "anti-patterns", "autonomous-observability", "statistical-analysis"]
categories: ["Architecture", "Innovation"]
author: "Aaron Jacobs"
description: "A deep technical exploration of VALIS—the Vast Active Living Intelligence System—and how its native integration with Dash0 telemetry enables autonomous detection of performance anti-patterns in distributed systems."
---

## The Problem with Passive Observability

Most observability workflows are fundamentally reactive. We instrument our services, ship telemetry to a platform, build dashboards, configure alerts, and wait for something to break. When it does, we manually sift through traces, correlate logs, and pattern-match against our tribal knowledge of "things that have gone wrong before."

This works—until it doesn't. The complexity of distributed systems has outpaced human cognitive bandwidth. N+1 queries hide in trace waterfalls. Memory leaks creep up over days. Retry storms cascade across service boundaries in milliseconds. By the time a human notices, the damage is done.

What if observability could be *active*? What if the system could perceive anti-patterns as they form, reason about their probability and impact, and act before they escalate?

This is VALIS.

## Introducing VALIS: Vast Active Living Intelligence System

Named after Philip K. Dick's novel about a vast machine intelligence that perceives patterns in reality, VALIS is an MCP (Model Context Protocol) server that brings autonomous anti-pattern detection to observability platforms. It doesn't replace human operators—it augments them with continuous statistical analysis and probabilistic reasoning.

VALIS provides 14 specialized tools spanning:

- **Pattern Detection**: Signature-based detection for 11 known anti-patterns including N+1 queries, retry storms, chatty APIs, memory leaks, connection pool exhaustion, and more
- **Statistical Analysis**: Anomaly detection (Z-score, MAD, IQR), changepoint detection (CUSUM, PELT, BOCPD), and SPC process control
- **Predictive Capabilities**: Time-series forecasting (Holt, Holt-Winters), Monte Carlo time-to-failure simulation
- **Probabilistic Reasoning**: Bayesian probability calculation with evidence chain tracking
- **Correlation Analysis**: Pearson, Spearman, and Granger causality testing

The key insight: these tools are *composable*. An AI agent can chain them together to perform comprehensive analysis that would take a human engineer hours.

## Native Dash0 Integration

Here's where it gets interesting for the Dash0 engineering team.

VALIS analyzes observability data *natively*. The `valis_analyze_spans`, `valis_analyze_logs`, and `valis_analyze_metrics` tools accept raw OTLP-compatible data—exactly what Dash0 collects and stores. There's no transformation layer, no proprietary format, no vendor lock-in.

A typical workflow looks like this:

```
1. Query spans from Dash0: dash0_spans_query(service_name="checkout", limit=200)
2. Feed directly to VALIS: valis_analyze_spans(spans=<dash0_response>)
3. Get findings with calibrated confidence scores
```

The integration is seamless because both systems speak OpenTelemetry natively. Dash0's span attributes map directly to VALIS's semantic convention expectations. Trace IDs correlate. Service names align. It just works.

### What I Observed Today

Running VALIS against live Dash0 telemetry from an OpenTelemetry demo deployment, I captured:

- **200 spans** across 15+ microservices (frontend, checkout, cart, payment, product-catalog, recommendation, etc.)
- **200 logs** with correlated trace context
- Full trace propagation through Kafka messaging, gRPC calls, and HTTP requests

VALIS assessed the instrumentation quality at **0.78 (Good tier)** with 100% semantic compliance on spans. This matters because VALIS calibrates its confidence scores based on data quality—poor instrumentation leads to appropriately skeptical findings.

The analysis revealed:

1. **4 latency anomalies** at the 49ms range—all attributable to the AI assistant endpoint making LLM calls (expected behavior)
2. **A statistically significant changepoint** indicating a regime shift in latency patterns
3. **8 SPC violations** including Western Electric Rule violations suggesting process instability
4. **Lag-2 cross-correlation of 0.796** between request sequences, indicating propagation effects through the service mesh

No critical anti-patterns (N+1 queries, retry storms, connection pool exhaustion) were detected—the system is healthy. But VALIS didn't just say "everything's fine." It provided *evidence-based reasoning* for that conclusion.

## The Technical Architecture

### Signature-Based Detection

VALIS maintains a library of 11 anti-pattern signatures, each with:

- **Evidence indicators** (what to look for in spans/logs/metrics)
- **Base rates** (prior probability in typical production systems)
- **True/False positive rates** (for Bayesian updating)

When analyzing telemetry, VALIS doesn't just pattern-match. It calculates the *probability* that an anti-pattern is present given the observed evidence. A finding with 95% confidence means something different than one with 35% confidence.

### Statistical Process Control

The SPC analysis applies manufacturing quality control principles to observability:

- **Shewhart control charts** with UCL/LCL bounds
- **Western Electric rules** for detecting non-random patterns
- **Process capability indices** (Cp, Cpk) for measuring stability

Today's analysis showed Cpk of 0.51 against latency targets—below the 1.33 threshold that indicates a capable process. This isn't an alert, but it's actionable intelligence: the checkout service has latency variability that warrants investigation.

### Time-to-Failure Prediction

VALIS can run Monte Carlo simulations to predict when a degrading metric will cross a failure threshold. Feed it:

- Current value (e.g., 16.7ms latency)
- Growth rate and variance
- Failure threshold (e.g., 100ms SLA)

It returns a probability distribution of time-to-failure. Today's simulation indicated 165 minutes (2.75 hours) until SLA breach if current trends continued—giving operators a window for proactive intervention.

## Enterprise Production Implications

This is a demo deployment. What happens when we point VALIS at enterprise production traffic?

### Scale Considerations

Dash0's architecture handles high-cardinality telemetry efficiently. VALIS's analysis tools are O(n) or O(n log n) for most operations. The bottleneck isn't computation—it's sample selection. At enterprise scale, you don't analyze every span; you analyze statistically significant samples.

A production integration might:

1. **Sample intelligently**: Use Dash0's sampling rules to capture representative traffic plus 100% of errors
2. **Run continuous background analysis**: VALIS detects anti-patterns before they alert
3. **Correlate with deployment events**: Did that canary deploy introduce an N+1 query?
4. **Feed findings into remediation pipelines**: Auto-create Jira tickets, trigger runbooks, or even auto-rollback

### The Closed-Loop Vision

Imagine this workflow, fully automated:

1. Dash0 ingests spans from a new deployment
2. VALIS detects N+1 query pattern with 87% confidence
3. VALIS correlates with GitHub to find the offending code
4. AI agent generates a batch query fix
5. CI runs, tests pass
6. ArgoCD deploys the fix
7. VALIS confirms the anti-pattern is resolved

This isn't science fiction. I've demonstrated each component working. The integration is the engineering challenge—and Dash0 is uniquely positioned to enable it.

### Why Dash0?

Dash0's OpenTelemetry-native architecture makes VALIS integration trivial. But there's a deeper opportunity:

1. **Built-in instrumentation assessment**: Dash0 could surface VALIS's data quality scores, helping users understand where their instrumentation gaps are
2. **Native anti-pattern detection**: Embed VALIS analysis in the Dash0 UI, surfacing findings alongside traces and logs
3. **Proactive alerting**: Move beyond threshold alerts to probability-based warnings ("73% chance of connection pool exhaustion in next 2 hours")
4. **Remediation suggestions**: VALIS findings include actionable recommendations—Dash0 could integrate with ticketing/runbook systems

## The Philosophical Shift

The name VALIS is intentional. In Dick's novel, VALIS perceives information in reality that humans cannot see. Production observability is similar—the telemetry contains patterns that human cognition struggles to extract at scale.

We've spent decades building systems that *record* what happens. VALIS represents the shift to systems that *understand* what's happening. Not through black-box ML that nobody trusts, but through transparent statistical methods with calibrated confidence.

The human remains in the loop—but as a strategic decision-maker, not a pattern-matching machine.

## Try It Yourself

VALIS is available as an MCP server. Point it at your Dash0 data and see what it finds. The integration is straightforward:

1. Query telemetry from Dash0 (`dash0_spans_query`, `dash0_logs_query`)
2. Pipe to VALIS (`valis_analyze_spans`, `valis_analyze_logs`, `valis_analyze_metrics`)
3. Explore findings with probability scores and evidence chains

For the Dash0 engineering team: I'd love to explore deeper integration. VALIS's statistical engine combined with Dash0's data platform could redefine what "observability" means.

The telemetry is already there. The algorithms exist. The question is: are we ready for observability that thinks?

---

*VALIS capabilities demonstrated in this article: `valis_list_signatures`, `valis_assess_instrumentation_data`, `valis_analyze_spans`, `valis_analyze_logs`, `valis_analyze_metrics`, `valis_detect_anomalies`, `valis_detect_changepoints`, `valis_correlate`, `valis_forecast`, `valis_probability`, `valis_spc_analysis`, `valis_predict_ttf`, `valis_recommend_action`*

*Previous: [Building AI Agents for Observability Platforms](/posts/building-ai-agents-for-observability/)*
