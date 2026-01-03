---
title: "Building AI Agents for Observability Platforms"
date: 2025-01-03
draft: false
tags: ["MCP", "agentic-ai", "observability", "VALIS"]
categories: ["Architecture"]
author: "Aaron Jacobs"
description: "Why full API coverage matters for autonomous observability operations"
---

## The Gap in Vendor AI Integrations

Every major observability vendor shipped AI capabilities in 2024-2025. Dynatrace, Datadog, New Relic, Honeycomb—they all have chatbots now that can answer questions about your environment.

But there's a pattern: they expose **read access** generously while keeping **write access** tightly controlled.

You can ask "what problems are open?" but you can't say "create a dashboard for this incident." You can query metrics but you can't configure alerts. The AI can observe but it can't operate.

This makes sense from a vendor perspective—write operations are risky. But it creates a ceiling on what agentic AI can actually accomplish.

## Full Surface Coverage

What happens when you wire up the *entire* API surface of an observability platform?

We built comprehensive MCP (Model Context Protocol) coverage for a major observability vendor—not the 15-20 curated endpoints vendors typically expose, but complete coverage across environment APIs, configuration APIs, and platform APIs.

The result: nearly **100 tools** spanning everything from entity queries to dashboard creation to alert configuration to problem management.

The difference isn't just quantitative. It's qualitative. With full coverage, an AI agent can complete entire workflows autonomously:

- Detect an anomaly
- Investigate root cause across metrics, traces, and logs
- Correlate with recent deployments
- Create a war room dashboard
- Document findings
- Configure follow-up alerts
- Close the incident when resolved

With partial coverage, the agent gets stuck midway through and hands off to a human. With full coverage, it completes the loop.

## Skills: Encoding Operational Expertise

Raw API access isn't enough. An AI with dozens of tools still needs to know *when* and *how* to use them effectively.

We developed a "Skills" layer—structured documentation that encodes domain expertise. Think of it as operational runbooks that the AI reads before executing tasks.

A dashboard creation skill, for example, captures:
- Which query patterns work for different visualization types
- Common pitfalls in the platform's schema
- Best practices for layout and organization

The AI consults relevant skills before acting, giving it the equivalent of an experienced operator's intuition.

## What This Enables

With full API coverage plus skills, we've seen significant acceleration in operational workflows. Tasks that previously required switching between UIs, consulting documentation, and iterating through trial-and-error now complete in single autonomous runs.

More importantly, the AI can handle scenarios it's never explicitly seen—because it has the tools and knowledge to reason through novel situations rather than following rigid scripts.

## The Takeaway

The vendor AI integrations are a starting point, not the destination. If you want AI that can actually *operate* your observability stack—not just answer questions about it—you need comprehensive API coverage and encoded expertise.

Full coverage transforms an AI assistant into an AI operator. The question isn't whether AI will run operations—it's whether you'll build the infrastructure to let it.

---

*Next: [The Receiver Factory: Accelerating OpenTelemetry Development](/posts/otel-receiver-factory/)*
