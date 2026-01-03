---
title: "Ontologies for Vendor-Agnostic Observability Migration"
date: 2025-01-03
draft: false
tags: ["ontology", "migration", "observability", "semantic-mapping", "VALIS"]
categories: ["Architecture"]
author: "Aaron Jacobs"
description: "Building semantic bridges between observability platforms for intelligent configuration migration"
---

## The Migration Problem

Observability platform migrations are notoriously painful. Whether you're moving from Datadog to Dynatrace, New Relic to Grafana, or any other combination, the process typically involves:

1. Exporting dashboards and alerts as JSON
2. Manually mapping fields between schemas
3. Rewriting queries in the target platform's language
4. Rebuilding what doesn't translate
5. Hoping nothing breaks

This is **syntactic translation**—moving symbols between systems without understanding what they mean.

The result: migrations take months, cost more than expected, and leave gaps that only surface in production.

## Beyond Syntax: Semantic Translation

What if migration tools understood *intent* rather than just format?

Consider a Datadog dashboard monitoring service latency. A syntactic tool tries to convert the JSON structure. A semantic tool understands: "this dashboard shows service health via latency percentiles" and can express that concept idiomatically in any target platform.

The difference matters because observability platforms model reality differently:

- **Service identity**: Auto-detected entities vs tag-based identification vs explicit configuration
- **Dependencies**: Graph databases vs trace inference vs manual declaration  
- **Anomaly detection**: Automatic AI vs threshold-based vs statistical baselines

A 1:1 field mapping misses these conceptual differences. Semantic translation accounts for them.

## The Ontology Approach

We're building an ontology layer that sits between observability platforms—a canonical model of observability concepts that maps to each vendor's specific implementation.

The architecture has three components:

**Canonical Model**: Platform-agnostic definitions of core concepts (service, metric, alert, dashboard, SLO). This is the Rosetta Stone.

**Platform Mappings**: How each vendor's data model maps to canonical concepts. Where Dynatrace has "entities" and Datadog has "tags," both map to the canonical "service" concept—but with documented differences in semantics.

**Gap Analysis**: The most valuable output. What *doesn't* translate cleanly between platforms, and what decisions need to be made.

## Gap Detection is the Killer Feature

The ontology's real value isn't what translates—it's explicitly documenting what doesn't.

When migrating from Platform A to Platform B:

> "Platform A's automatic dependency mapping has no direct equivalent in Platform B. You'll need to either:
> (a) Rely on trace-based inference, which misses non-traced calls
> (b) Implement a manual service catalog integration
> (c) Accept reduced topology visibility
> 
> Here's what you'll lose and options to compensate..."

This is consultancy-grade insight generated systematically. Instead of discovering gaps in production six months post-migration, you surface them before work begins.

## Beyond Migration: Multi-Platform Correlation

Once you have semantic mappings for multiple platforms, migration is just one use case.

**Cross-platform correlation**: Query both platforms and unify results using the canonical model. Useful during migration windows or in environments that legitimately run multiple tools.

**Vendor-agnostic dashboards**: Define dashboards against the canonical model; render on any platform.

**Best-of-breed composition**: Use each platform for what it does best, unified through semantic translation.

## Current Status

We have working mappings for two major observability platforms with a third in development. The gap analysis component is operational and has already surfaced non-obvious translation issues in real migration planning.

The goal is coverage across the major players—Dynatrace, Datadog, New Relic, Grafana/Prometheus, Splunk—with a shared canonical model that enables intelligent migration and multi-platform operation.

## The Business Case

Observability migrations are a significant market. Enterprises spend millions on platform transitions that often underdeliver. The consultancies doing this work rely on tribal knowledge and manual effort.

Semantic migration tooling changes the equation: faster assessments, explicit gap documentation, and automated translation where possible. The human expertise focuses on decisions that actually require judgment, not mechanical conversion work.

---

*This is the third article in the O11y Alchemy series. See also: [Building AI Agents for Observability](/posts/building-ai-agents-for-observability/) and [The Receiver Factory](/posts/otel-receiver-factory/).*
