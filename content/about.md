---
title: "About"
layout: "single"
url: "/about/"
---

## About This Blog

**O11y Alchemy** documents the emergence of autonomous observability—AI agents that can perceive anti-patterns in distributed systems, reason about their causes, implement fixes, deploy them through GitOps, and verify improvements. All without human intervention.

The name references the alchemical tradition of transmutation. I'm transmuting passive observability telemetry into active, intelligent systems that operate infrastructure.

This isn't speculative. The case studies on this site document production systems where autonomous agents detected performance anti-patterns, generated code fixes, deployed them, and validated latency improvements ranging from 59% to 4,700x—all autonomously.

## What's Been Demonstrated

### VALIS: Autonomous Anti-Pattern Detection
A fully implemented system that detects 11+ anti-patterns in production telemetry through geometric analysis of trace topology. Detection is language-agnostic—the same framework identifies N+1 queries in Go gRPC calls, Python SQL queries, or Node.js HTTP requests. It classifies shapes, not implementations.

Core capabilities proven in production:
- **Trace topology analysis**: Fan-out, homogeneity, temporality, duration distribution
- **Statistical reasoning**: Bayesian confidence scoring, temporal analysis, anomaly detection
- **Predictive modeling**: Monte Carlo time-to-failure simulation, forecasting
- **Causal inference**: Granger causality testing, correlation analysis across telemetry signals

### Closed-Loop Remediation Architecture
Complete perceive-reason-act-verify cycles demonstrated end-to-end:

**Case 1: N+1 Query (Go + gRPC)**
- Detected 2N+2 sequential service calls scaling linearly with cart size
- Generated concurrent fan-out fix using errgroup to parallelize RPCs
- Deployed via ArgoCD GitOps
- Verified 59% latency reduction in production telemetry

The fix was implemented, deployed, and validated without human intervention. The commits are public. The telemetry is verifiable.

### MCP-Based Open Architecture
The autonomy is built on composable infrastructure, not proprietary platforms:

- **100+ MCP tools** providing complete API coverage across observability (Dash0), GitOps (ArgoCD), source control (GitHub), and orchestration (Kubernetes)
- **Vendor-agnostic by design**: Swap Dash0 for Datadog, Honeycomb, or Dynatrace by changing configuration—the orchestration layer is platform-independent
- **Extensible**: Add new capabilities by adding MCP servers, not filing vendor feature requests
- **Open models**: Claude-based orchestration, but the architecture works with any LLM supporting tool use

This architecture demonstrates why closed platforms will lose to composable AI infrastructure. Integration is no longer a moat—it's a commodity when AI agents orchestrate through open protocols.

### Geometric Detection Theory
The most significant finding: **anti-patterns have geometric signatures in trace topology space that are invariant across programming languages, protocols, and observability platforms.**

An N+1 query produces a characteristic "comb" shape—high fan-out, homogeneous children, sequential execution. This geometry is identical whether the implementation is Go with gRPC, Python with SQL, or JavaScript with REST. The detector classifies shapes, not code.

This means:
- Detection scales across polyglot architectures without per-language tuning
- New anti-patterns can be discovered empirically by clustering trace geometries
- The same detection framework works with any observability backend that preserves trace topology

## Why This Matters

The observability industry has spent a decade building better dashboards and more sophisticated alerting. But fundamentally, the workflow remained reactive: humans investigate, diagnose, fix, deploy, verify.

Autonomous observability inverts this. The system investigates. The system diagnoses. The system fixes. The system deploys. The system verifies. Humans architect the systems that enable this autonomy.

The leverage changes by orders of magnitude.

## What's Coming

Current work focuses on:
- **Pattern taxonomy expansion**: Discovering new anti-pattern signatures through unsupervised clustering of production trace geometries
- **Multi-signal correlation**: Combining spans, logs, and metrics for comprehensive incident analysis
- **Deployment safety**: Pre-deployment error budget checks and post-deployment rollback decision circuits
- **Code correlation**: Mapping span names to source code locations for targeted fixes

The goal isn't to replace engineers. It's to give them Maxwell's Demon—a system that continuously reduces entropy in production systems by detecting and correcting pathological patterns as they emerge.

## About the Author

**Aaron Jacobs** — Principal Technical Consultant specializing in observability, OpenTelemetry, and autonomous infrastructure systems.

Background:
- Pre-IPO AppDynamics through Cisco's $3.7B acquisition (2017)
- Enterprise observability architecture across Fortune 500 deployments
- OpenTelemetry instrumentation and custom receiver development
- Production-grade Kubernetes observability on Raspberry Pi clusters (for science)

Current focus: Building autonomous systems that operate at the boundary between AI reasoning and production infrastructure. The work combines observability domain expertise, statistical analysis, causal inference, and agentic AI orchestration.

Everything documented on this site has been implemented and demonstrated. The code is public. The commits include verification data. The telemetry improvements are reproducible.

This is not observability theory. This is observability alchemy—transmuting passive telemetry into active intelligence.

**Connect:**
- GitHub: [@npcomplete777](https://github.com/npcomplete777)
- LinkedIn: [aaronjacobs777](https://linkedin.com/in/aaronjacobs777)
