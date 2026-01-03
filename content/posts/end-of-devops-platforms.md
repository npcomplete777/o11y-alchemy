---
title: "The End of DevOps Platforms"
date: 2025-01-03
draft: false
tags: ["devops", "MCP", "agentic-ai", "CI/CD"]
categories: ["Industry"]
author: "Aaron Jacobs"
description: "Why MCP-based AI agents will replace monolithic DevOps platforms"
---

## The Platform Era

For the past decade, the DevOps market has consolidated around platforms. Harness, GitLab Ultimate, GitHub Enterprise, CircleCI—they all sell the same promise: unified DevOps under one roof.

The pitch makes sense. Before platforms, teams stitched together Jenkins, custom scripts, and a dozen point solutions. It was fragile and expensive to maintain. Platforms offered integration, governance, and a single pane of glass.

But platforms come with tradeoffs: vendor lock-in, lowest-common-denominator features, and pricing that scales with headcount rather than value.

Now there's an alternative.

## MCP Changes Everything

The Model Context Protocol (MCP) standardizes how AI agents interact with tools. Any system with an API can become an MCP server. Any MCP server can be orchestrated by an LLM.

This matters because the value of DevOps platforms was *integration*—making tools work together. But if an AI agent can orchestrate any tool through a standard protocol, the integration layer becomes commoditized.

I've built full-surface MCP servers that expose complete API coverage for:

- Git platforms (GitLab, GitHub)
- GitOps controllers (ArgoCD, Flux)
- Observability platforms (complete API coverage, not curated subsets)
- Container orchestration (Kubernetes)
- Data stores (ClickHouse, Prometheus)

Each server exposes dozens to hundreds of tools. An LLM orchestrates them through natural language.

## What Platforms Sell vs. What MCP Provides

| Platform Feature | MCP Equivalent |
|------------------|----------------|
| Pipeline generation | LLM + Git MCP creates configs from natural language |
| AI DevOps Assistant | LLM + any MCP—not locked to one vendor |
| Continuous Verification | Query observability backend directly, closed-loop |
| Auto-rollback | GitOps MCP + problem detection |
| AI Autofix | Code generation + build verification loop |
| Security scanning | Add a security scanner MCP |
| Database DevOps | Add a database MCP |

The pattern: anything a platform does can be decomposed into API calls. Wrap those APIs in MCP servers. Let an LLM orchestrate.

## The Closed-Loop Advantage

Platform AI features are typically reactive: monitor for problems, alert, maybe rollback. They observe and respond.

I've built something different: closed-loop systems where the AI takes action, verifies the result through observability, and iterates until success.

Example: my receiver factory generates OpenTelemetry collectors from API specs. It doesn't just generate code—it builds, deploys, queries the backend to confirm data flows, and iterates if something fails. The observability backend is ground truth, not just a monitoring signal.

This pattern generalizes. Any deployment can be verified against actual production behavior, not just health checks. The AI learns from real outcomes, not proxy metrics.

## Skills Over Features

Platforms ship features. New capability = new release = wait for the vendor.

I use a Skills layer instead: structured documentation that encodes expertise. The AI reads relevant skills before executing tasks.

New capability = new skill file. No vendor dependency. No release cycle. I extend the system's capabilities by writing documentation, not code.

This inverts the platform model. Instead of waiting for Harness to add a feature, I teach the AI how to do it.

## The Economics

DevOps platforms charge per seat. $50-500/developer/month depending on tier.

MCP-based orchestration costs:
- LLM API calls (pennies per operation)
- Infrastructure you already run
- Time to build MCP servers (once, then reusable)

For a 100-developer org paying $200/seat/month, that's $240K/year for a platform. The MCP alternative is a fraction of that, with no lock-in.

## What This Means

The platform era solved a real problem: tool fragmentation. But it created new problems: lock-in, inflexibility, cost.

MCP dissolves the integration moat. When any tool is AI-accessible through a standard protocol, the value shifts from "unified platform" to "best orchestration."

The winners will be:
- **Best-of-breed tools** that expose clean APIs
- **MCP server builders** who wrap those APIs well
- **AI orchestration layers** that coordinate across tools

The losers will be platforms whose primary value was integration.

I'm not saying Harness or GitLab will disappear tomorrow. But their moat is eroding. The future is composable, not monolithic.

---

*See also: [Building AI Agents for Observability](/posts/building-ai-agents-for-observability/) and [The Receiver Factory](/posts/otel-receiver-factory/).*
