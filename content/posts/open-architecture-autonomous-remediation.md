---
title: "Why Autonomous Remediation Requires Open Architecture"
date: 2026-02-04
draft: false
tags: ["MCP", "architecture", "autonomous-remediation", "observability", "open-standards"]
categories: ["Architecture", "Industry"]
author: "Aaron Jacobs"
description: "Why composable, extensible MCP infrastructure beats proprietary closed platforms for autonomous observability operations."
---

## The Architecture Question Nobody's Asking

The autonomous remediation market is heating up. Vendors promise AI that fixes production issues automatically—detect the problem, identify the fix, deploy it, verify it works. The demos look impressive.

But there's a foundational question that matters more than any feature comparison: **Is the platform open or closed?**

This isn't about open source licensing. It's about architectural composability. Can you extend the system with new capabilities? Can you swap out components? Can you integrate with arbitrary backends?

The answer determines whether you're building on infrastructure that scales with your needs—or betting on a vendor's roadmap.

## The Closed Platform Approach

Proprietary platforms like Traversal offer an integrated experience. They bundle detection, analysis, remediation, and verification into a single product. Everything is pre-wired.

The value proposition: turnkey deployment. Install their agent, point it at your code, and autonomous fixes start flowing.

The tradeoff: you're locked in.

**Observability backend lock-in**: If the platform integrates with three backends, you pick from those three. Want to use your existing observability infrastructure? You migrate or you don't use the platform.

**Integration lock-in**: Need to connect with an internal tool? A new SaaS product? You file a feature request and wait for the vendor to build it—if they build it at all. Their team controls the integration surface.

**AI lock-in**: The platform uses proprietary models and prompts. You can't swap in a better model, tune the reasoning, or adjust the orchestration logic. The AI is a black box.

This works fine—until it doesn't. Until your observability needs evolve. Until you need an integration they won't build. Until their AI capabilities plateau while the broader ecosystem leaps ahead.

At that point, you're stuck. Migration means rip-and-replace.

## The Open Architecture Alternative

I built autonomous remediation on **Model Context Protocol (MCP)** infrastructure. MCP is Anthropic's standard for AI-tool interoperability. Any system with an API can become an MCP server. Any MCP server can be orchestrated by any LLM.

This architecture has three properties that closed platforms can't match:

### 1. Composability: Any Observability Backend

My system integrates with Dash0, a modern OpenTelemetry-native observability platform. But the integration isn't hardcoded—it's an MCP server that exposes Dash0's API as tools.

Want to use Datadog instead? Swap in a Datadog MCP server. Honeycomb? Dynatrace? New Relic? Same pattern. The orchestration layer doesn't care—it sends MCP requests to whichever backend you configure.

This matters because observability is infrastructure. You don't choose your observability platform based on which autonomous remediation vendor supports it. You choose the remediation system that works with the observability you already have.

Closed platforms force the opposite: pick the platform, then migrate to their supported backends. That's backwards.

### 2. Extensibility: Anyone Can Add Capabilities

My stack includes MCP servers for:
- **VALIS**: Anti-pattern detection with statistical analysis
- **Dash0**: Observability queries (spans, logs, metrics)
- **GitHub**: Code access and PR creation
- **Kubernetes**: Cluster state and operations
- **ArgoCD**: GitOps deployments
- **Dynatrace**: Alternative observability backend

I built most of these. But here's the key: **anyone can add more**.

Need integration with Jira? Build a Jira MCP server (or use an existing one). PagerDuty? Same. Internal deployment tooling? Wrap it in MCP.

The system's capabilities expand through composition, not vendor roadmap. When a new AI model drops, I can swap it in. When a new observability backend emerges, I can integrate it.

Closed platforms can't do this. They control the integration surface. You wait for them to build what you need—and they prioritize the widest market, not your specific requirements.

### 3. AI-Native: MCP Is the Emerging Standard

MCP is rapidly becoming the standard protocol for LLM tool use. Anthropic built it. Model providers are adopting it. Tool builders are exposing MCP interfaces.

This means the ecosystem is working for you. As more tools add MCP support, your autonomous remediation capabilities expand—without vendor dependency.

Closed platforms built proprietary tool integration before MCP existed. Now they're stuck with legacy architectures while the ecosystem moves forward. They'll have to rebuild.

I started with MCP from day one. The ecosystem's growth directly benefits the system.

## What This Looks Like in Practice

Here's an autonomous remediation workflow on open MCP architecture:

1. **VALIS MCP** analyzes spans from **Dash0 MCP** and detects an N+1 query anti-pattern
2. **GitHub MCP** correlates the span to source code
3. LLM generates a batch query fix
4. **GitHub MCP** creates a branch and opens a PR
5. CI runs tests (external to MCP but observable)
6. **ArgoCD MCP** deploys to canary environment
7. **Dash0 MCP** queries post-deployment spans
8. **VALIS MCP** confirms the anti-pattern is resolved
9. **ArgoCD MCP** promotes to production

Each component is swappable. Replace Dash0 with Datadog? Swap one MCP server. Replace GitHub with GitLab? Same. Replace ArgoCD with Flux? Same pattern.

On a closed platform, you're locked into their supported integrations at every step.

## The Long-Term Bet

Proprietary platforms bet on integrated convenience. They make it easy to get started by bundling everything.

Open architecture bets on ecosystem evolution. It's more flexible but requires assembly.

The question is: what happens over 3-5 years?

**Closed platform trajectory**: You're dependent on their integration roadmap, their AI development, their pricing changes. If they plateau, you plateau. If they sunset features, you adapt or migrate.

**Open architecture trajectory**: The MCP ecosystem expands. New tools become available. AI models improve. You compose better capabilities from best-of-breed components. Your infrastructure evolves independently of any single vendor.

This is the same dynamic that played out with monolithic platforms versus composable architectures everywhere else in infrastructure. Kubernetes won over proprietary orchestrators. OpenTelemetry is winning over proprietary instrumentation. Open standards accumulate ecosystem momentum that closed platforms can't match.

## The Right Architecture for Autonomous Operations

Autonomous remediation isn't a feature—it's infrastructure. You're building the system that operates your production environment. The architectural choice matters more than any individual capability comparison.

Ask: **Can I replace components? Can I extend the system? Can I integrate with anything that has an API?**

If the answer is no, you're building on someone else's foundation. If the answer is yes, you're building on yours.

Traversal ships a product. I'm building on a protocol.

Products have features. Protocols have ecosystems.

---

*Related: [VALIS: Autonomous Anti-Pattern Detection](/posts/valis-autonomous-anti-pattern-detection/) and [The End of DevOps Platforms](/posts/end-of-devops-platforms/)*
