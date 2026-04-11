# Ecotone Documentation Enterprise Repositioning

**Date:** 2026-04-10
**Status:** Draft
**Scope:** Full content and positioning overhaul of docs.ecotone.tech (GitBook)

## Context

Ecotone's documentation site currently explains *what Ecotone does* but not *what problem it solves for you*. The positioning is generic ("stop wrestling with infrastructure") and doesn't frame Ecotone as what it actually is: the enterprise architecture layer that makes Laravel and Symfony production-ready for complex business domains.

The docs need to shift from pattern-first documentation to problem-first documentation, so that:
- Laravel/Symfony developers who hit a complexity wall recognize their pain and find the solution
- Enterprise architects evaluating PHP see that it has the same caliber of tooling as Java (Spring/Axon) and .NET (NServiceBus/MassTransit)
- Every page reinforces: Ecotone is additive to your framework, not competitive

### Target audiences (priority order)
1. **Laravel/Symfony developers** who've outgrown basic framework patterns — controllers doing too much, async processing failing silently, business processes scattered across files
2. **Technical leads / architects** evaluating whether PHP can handle enterprise-grade architecture

### Constraints
- Platform is GitBook (markdown-based, limited custom components)
- Minor URL restructuring is OK, but major page URLs must stay stable
- Free tier should feel production-ready; Enterprise tier should feel like the natural next step as systems grow

---

## Section 1: Home Page Overhaul (README.md)

### Current state
Generic "stop wrestling with infrastructure" hero, feature bullet list, single "get started" CTA. Doesn't mention Laravel/Symfony prominently, doesn't speak to specific pain points.

### Proposed structure

**Hero tagline:**
> "CQRS, Event Sourcing, Workflows, and Production Resilience for Laravel and Symfony"

**Sub-tagline:**
> "Add production-grade architecture to your existing framework with a single Composer package. No framework change. No base classes. Just PHP attributes."

**"What problem does Ecotone solve?" section** — 3 short pain-point blocks:
- **"Your app grew, your architecture didn't"** — Controllers doing too much, business logic scattered, hard to test
- **"Async processing is fragile"** — Failed jobs disappear, no retry strategy, no visibility
- **"Enterprise patterns feel out of reach in PHP"** — You've seen CQRS/Event Sourcing but every PHP implementation requires rewriting your app

**Framework-specific entry paths** — Side-by-side cards:
- **Laravel developers** — "Keep Eloquent, add enterprise messaging" -> Laravel quick start
- **Symfony developers** — "Go beyond Symfony Messenger" -> Symfony quick start
- **Architects** — "PHP can do what Spring and Axon do" -> Why Ecotone page

**"How it works" — 3-step visual:**
1. Install via Composer
2. Add attributes to your code
3. Ecotone wires the messaging

**Social proof / credibility line:**
> "Built on Enterprise Integration Patterns. Used in production by teams running multi-tenant, event-sourced systems at scale."

**Existing CTAs preserved:** Quick start, Tutorial, Workshops

---

## Section 2: New "Why Ecotone?" Page

New page, positioned right after "About" in navigation. Core positioning page for architects and evaluators.

### Structure

**1. "What Ecotone Is (and Isn't)"**
- Ecotone is NOT a framework replacement. It's the enterprise architecture layer on top of Laravel and Symfony.
- Analogy: "The way API Platform provides the API layer on top of Symfony, Ecotone provides the enterprise messaging layer on top of your framework."
- You keep your ORM, your routing, your templates. Ecotone handles the messaging architecture.

**2. "Every ecosystem has this layer — except PHP"**
- Java: Spring + Axon Framework
- .NET: NServiceBus, MassTransit, Wolverine
- PHP had nothing production-ready. Until Ecotone.
- Not about PHP being inferior — about PHP maturing into enterprise domains.

**3. "What you get"** — organized by problem, not pattern name:
- Clear command/query separation -> CQRS
- Full audit trail and time travel -> Event Sourcing
- Long-running business processes -> Sagas & Orchestrators
- Reliable async with self-healing -> Resilient Messaging
- Cross-service communication -> Distributed Bus
- Multi-tenant isolation -> Dynamic Channels

**4. "How it integrates"**
- Laravel: works with Eloquent, Laravel Queues, Octane
- Symfony: works with Doctrine, Symfony Messenger Transport
- Standalone: works with any PHP app via Ecotone Lite

**5. "Start free, scale with Enterprise"**
- Free tier: everything you need for production CQRS, Event Sourcing, Workflows
- Enterprise: when you outgrow single-tenant, single-service, or need advanced resilience and scalability

---

## Section 3: New "Solutions" Section

New top-level section in navigation, positioned before Installation. 6 problem-oriented pages that bridge developer pain points to Ecotone patterns.

### Page template
Each page follows the same structure:
1. **Problem you recognize** — described in terms a Laravel/Symfony dev knows
2. **What the industry calls it** — name the pattern
3. **How Ecotone solves it** — concrete explanation with brief illustrative code snippet (5-10 lines showing an attribute and handler, not full runnable examples — those live in the technical docs)
4. **Next steps** — links to detailed technical docs
5. **As You Scale** — natural transition to Enterprise features (where applicable)

### Pages

**Sidebar titles use short labels:**

#### 1. "Scattered Application Logic"
- Pain: Controllers doing reads and writes, business rules in listeners, services calling services. Hard to test, hard to onboard.
- Pattern: CQRS, Command/Query separation
- Solution: Command Handlers, Query Handlers, Event Handlers with single responsibility, wired automatically
- Framework context: "In Laravel, you might have a 300-line Controller. In Symfony, a service with 10 injected dependencies."
- Links to: CQRS Introduction, Aggregates

#### 2. "Unreliable Async Processing"
- Pain: Failed jobs silently disappear, no retry strategy, can't replay failed messages, no visibility into what's queued
- Solution: Automatic retries, error channels, dead letter queues, outbox pattern, idempotency — all declarative via attributes
- Framework context: "Laravel Queues and Symfony Messenger give you basic async. Ecotone gives you production resilience on top."
- Links to: Async Handling, Resiliency, Error Channel
- As You Scale: instant retries, command bus error channel, gateway-level deduplication

#### 3. "Complex Business Processes"
- Pain: Order fulfillment, subscription lifecycle, onboarding flows — state scattered across tables, flags, and cron jobs
- Solution: Sagas (stateful workflows), Orchestrators (declarative step sequences), Handler Chaining (simple pipelines)
- Links to: Business Workflows, Sagas, Orchestrators
- As You Scale: Orchestrators for declarative workflow automation

#### 4. "Audit Trail & State Rebuild"
- Pain: "What happened to this order?" requires reading logs. Schema migrations risk data loss. No way to replay history.
- Solution: Event Sourcing with projections, event versioning, snapshots, rebuild capabilities
- Links to: Event Sourcing, Projections
- As You Scale: Partitioned projections, async backfill, blue-green deployments

#### 5. "Microservice Communication"
- Pain: Services calling each other via HTTP, no guaranteed delivery, no event sharing, custom glue code per service pair
- Solution: Distributed Bus with Service Map, multiple broker support, shared event streams
- Links to: Distributed Bus, Microservices
- As You Scale: Service Map with multi-broker topology

#### 6. "PHP for Enterprise Architecture"
- For architects: positions PHP + Ecotone against Java/Spring, .NET/NServiceBus
- Enterprise Integration Patterns foundation, production resilience, multi-tenant support, observability via OpenTelemetry
- Links to: Why Ecotone, Enterprise page

---

## Section 4: Reframe Existing Feature Pages

Add a problem-context header to each major page in the "Modelling" section. Short block before existing technical content.

### Pattern
```
## The Problem
[2-3 sentences describing the pain in terms a Laravel/Symfony dev recognizes]

## How Ecotone Solves It
[1-2 sentences connecting the pattern to the solution]

---
[Existing technical content unchanged]
```

### Pages and their problem framing

1. **Message Bus & CQRS Introduction** — "Your service classes mix reading and writing. A single change to how orders are placed breaks the order listing page. Business rules are scattered across controllers, listeners, and services."

2. **Aggregates** — "Business rules are enforced in multiple places — a validation here, a check there. When rules change, you update three files and miss a fourth. There's no single source of truth for what an Order or User can do."

3. **Event Sourcing Introduction** — "You store the current state but not how you got there. When a customer disputes a charge, you can't answer 'what exactly happened?' Rebuilding read models after a schema change means writing migration scripts by hand."

4. **Resiliency** — "A failed HTTP call crashes your handler. A duplicate webhook triggers double-processing. You've wrapped handlers in try/catch blocks and retry loops — each one slightly different."

5. **Async Handling** — "You added async processing, but now you can't tell which messages are stuck, which failed silently, and which will retry forever. Going async required touching every handler."

6. **Business Workflows** — "Your order fulfillment process spans 6 steps across 4 services. The logic is spread across event listeners, cron jobs, and status flags. Nobody can explain the full flow without reading all the code."

7. **Distributed Bus / Microservices** — "You're calling other services via HTTP. When Service B is down, Service A fails too. You've built custom retry logic, custom serialization, and custom routing for each service pair."

8. **Multi-Tenancy** — "Adding a new tenant means configuring new queues, new database connections, and custom routing. A slow tenant's queue backlog delays everyone else's messages."

9. **Testing Support** — "Testing your message handlers requires bootstrapping the entire framework, setting up queues, and hoping the async parts work. Unit testing a saga means mocking half the application."

### Additional per-page changes
- Add "Works with Laravel & Symfony" callout where relevant
- Add framework-specific code snippets where current examples are framework-agnostic

---

## Section 5: Enterprise Growth Path

Make the free-to-Enterprise transition feel natural throughout the docs, not just on the Enterprise page.

### Changes

**In Solutions pages (Section 3):** Each problem page ends with an "As You Scale" callout introducing relevant Enterprise features. Not a sales pitch — a progression:
- "Unreliable Async" -> free: retries, error channels, dead letter -> as you scale: instant retries, command bus error channel, gateway-level deduplication
- "Complex Business Processes" -> free: sagas, handler chaining -> as you scale: orchestrators
- "Microservice Communication" -> free: distributed bus -> as you scale: service map, multi-broker topology

**In reframed feature pages (Section 4):** Where a feature has both free and Enterprise capabilities, show the free approach first as the complete solution, then a brief "Going further with Enterprise" section at the bottom — not interrupting the technical flow.

**Enterprise page adjustments:**
- Add positioning context at the top: "Ecotone Free gives you production-ready CQRS, Event Sourcing, and Workflows. Enterprise is for when your system outgrows single-tenant, single-service, or needs advanced resilience and scalability."
- Keep the existing "Signs You're Ready" structure (already well-written)
- Add a brief comparison table: Free vs Enterprise at a glance

---

## Section 6: Navigation Changes

### Current SUMMARY.md top-level
```
About -> Installation -> How to use -> Tutorial -> Enterprise -> Modelling -> Messaging -> Modules -> Other
```

### Proposed SUMMARY.md top-level
```
About -> Why Ecotone? (NEW) -> Solutions (NEW) -> Installation -> How to use -> Tutorial -> Enterprise -> Modelling -> Messaging -> Modules -> Other
```

### Solutions sub-navigation
```
* Solutions
  * Scattered Application Logic
  * Unreliable Async Processing
  * Complex Business Processes
  * Audit Trail & State Rebuild
  * Microservice Communication
  * PHP for Enterprise Architecture
```

### Reasoning
- "Why Ecotone?" right after About — visitors who are intrigued find positioning immediately
- "Solutions" before Installation — developers who know their problem find their entry point before being asked to install
- Everything else stays in current position

---

## Section 7: Cross-Cutting Improvements

### Introduction page rewrite (modelling/message-driven-php-introduction.md)
- Currently opens with OOP history and theory
- Rewrite to lead with practical problem: what Ecotone does for you
- Ground it in Enterprise Integration Patterns for credibility
- Move theoretical foundation lower

### Quick Start section (quick-start-php-ddd-cqrs-event-sourcing/)
- Add one-sentence problem framing to each quick start page header
- Ensure each quick start links to both Laravel and Symfony demo repos

### Module pages (Symfony, Laravel, etc.)
- Add "What this gives you" opening connecting to enterprise layer positioning
- Laravel: "Ecotone integrates with your existing Laravel application — Eloquent, Queues, Octane. You keep Laravel's developer experience and add enterprise messaging architecture."
- Symfony: "Ecotone builds on top of your Symfony application — Doctrine, Messenger Transport, Bundle configuration. Everything you know stays, enterprise patterns get added."

### Event Sourcing section intro (modelling/event-sourcing/README.md)
- Currently just lists demo links with no context
- Add proper introduction: what Event Sourcing is, why you'd want it, what Ecotone gives you vs building it yourself

### Resiliency section (modelling/recovering-tracing-and-monitoring/resiliency/README.md)
- Currently empty (just "# Resiliency")
- Add proper introduction covering the problem landscape, linking to sub-pages

### Consistent "Works with" callouts
- Each major section page gets: "Works with: Laravel, Symfony, Standalone"
- Reinforces additive positioning on every entry point

---

## Summary of all changes

| # | Section | Type | Scope |
|---|---------|------|-------|
| 1 | Home page overhaul | Rewrite | README.md |
| 2 | Why Ecotone? | New page | 1 new file |
| 3 | Solutions section | New pages | 6 new files + SUMMARY.md entry |
| 4 | Feature page reframing | Edit existing | ~9 major pages get problem headers |
| 5 | Enterprise growth path | Edit existing | Solutions pages, feature pages, enterprise.md |
| 6 | Navigation changes | Edit existing | SUMMARY.md restructure |
| 7 | Cross-cutting improvements | Edit existing | Introduction, quick starts, modules, empty pages |

**Total new files:** ~8 (Why Ecotone + 6 Solutions pages + Solutions README)
**Total edited files:** ~20-25 existing pages
**No files deleted. No major URLs broken.**
