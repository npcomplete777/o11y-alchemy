---
title: "The Receiver Factory: Accelerating OpenTelemetry Development"
date: 2025-01-03
draft: false
tags: ["opentelemetry", "code-generation", "automation"]
categories: ["Implementation"]
author: "Aaron Jacobs"
description: "Generating production-ready OTel receivers from API specifications"
---

## The OpenTelemetry Coverage Problem

OpenTelemetry has become the standard for observability instrumentation. But there's a gap between the standard and reality: most tools and platforms don't have native OTel receivers.

The OpenTelemetry Collector contrib repository has receivers for major databases, cloud providers, and infrastructure components. But what about your Airflow deployment? Your Snowflake warehouse? Your custom internal APIs?

Building a production-quality OTel receiver typically takes days to weeks. You need to understand the source API, map data to semantic conventions, handle authentication, implement proper error handling, write tests, and integrate with the collector build system.

What if you could generate receivers automatically?

## The Factory Approach

I built a system that takes an API specification (OpenAPI/Swagger) and produces a working OpenTelemetry receiver. Not a skeleton—a production-ready receiver with proper semantic conventions, error handling, and configuration.

The key insight: receiver structure is highly consistent. The variation is in *what* you're scraping and *how* you map it to OTel conventions. If you can formalize those mappings, you can automate the generation.

My factory uses:

- **Schema enforcement** via OTel semantic conventions
- **Template-based generation** for consistent structure
- **Automated validation** against the collector build system
- **Production verification** to confirm data actually flows

## Closed-Loop Verification

Generation isn't enough. You need to know the receiver actually works.

My system implements a closed-loop: generate code, build it, deploy to a test collector, verify metrics appear in the backend. If something breaks, the error feeds back into the next generation attempt.

This isn't just CI/CD—it's the AI observing the consequences of its own code and iterating. The verification backend becomes ground truth that the system learns from.

## Results

I've used this factory to generate receivers for several platforms that lacked native OTel support:

- Apache Airflow (workflow orchestration metrics)
- Snowflake (data warehouse performance)
- GitHub (repository and workflow metrics)
- Various internal APIs

Development time dropped from days to minutes. More importantly, the generated receivers follow consistent patterns and semantic conventions—something that's hard to maintain across hand-written code.

## Why This Matters

The observability ecosystem has a long-tail problem. Major platforms get receivers; niche tools don't. This creates blind spots in environments that use specialized tooling.

Automated generation changes the economics. When building a receiver takes minutes instead of days, you can justify coverage for tools that wouldn't otherwise make the cut.

It also enables self-service: teams can generate receivers for their own tools without waiting for central platform teams to prioritize the work.

## What's Next

The factory currently handles REST APIs with OpenAPI specs. I'm expanding to cover:

- GraphQL APIs
- Database metric scraping
- Log-based metric extraction

The goal is comprehensive coverage—if a system exposes data, I should be able to generate an OTel receiver for it.

---

*Next: [Ontologies for Vendor-Agnostic Observability Migration](/posts/observability-ontologies/)*
