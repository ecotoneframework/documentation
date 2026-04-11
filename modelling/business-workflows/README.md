---
description: Building business workflows with Sagas, Orchestrators, and Handler Chaining in PHP
---

# Business Workflows

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

Your order fulfillment process spans 6 steps across 4 services. The logic is spread across event listeners, cron jobs, and status flags. Nobody can explain the full flow without reading all the code. Adding or reordering steps risks breaking the entire process.

## How Ecotone Solves It

Ecotone provides three workflow approaches — from simple handler chaining to stateful sagas to declarative orchestrators. Each step is independently testable. The workflow definition lives in one place, not scattered across the codebase.

---

Ecotone provides three different ways to build Workflows:

### 🔄 **Use** [**Handler Chaining**](connecting-handlers-with-channels.md) **when:**

* You have simple, linear workflows
* Steps are tightly coupled to specific workflows
* You don't need dynamic workflow construction

### 📊 **Use** [**Sagas**](sagas.md) **when:**

* You need to **remember state** between steps
* Workflows span **long periods** (hours, days, weeks)
* You need **compensation logic** for failures
* Workflows involve **human interaction** or external approvals

### ✅ **Use** [**Orchestrator**](orchestrators.md) **when:**

* You have **predefined workflows** with clear steps
* Steps need to be **reusable** across different workflows
* You need **dynamic workflow construction**
* Workflow logic is **separate from step implementation**

## Materials

### Links

* [Building workflows in PHP using Orchestrator](https://blog.ecotone.tech/building-workflows-in-php/) \[Article]
* [Building workflows in PHP with pipe and filter architecture](https://blog.ecotone.tech/building-workflows-in-php-with-ecotone/) \[Article]

