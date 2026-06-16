# Adventure Idea: 🧭 Dead Reckoning

## Overview

**Theme:** The Grand Fleet's commission office is the last thing standing between a newly built vessel and open water, a single portal where manifests are filed, provisions ordered, and routes set. Without it, nothing sails. With a broken one, nothing arrives where it should. You're the fleet's engineer: restore the commission office, connect the harbors, and when vessels start arriving with the wrong provisions even though every system reported a successful voyage, find out why.

**Skills:**

- Commission new services through an internal developer platform using software templates
- Debug and restore a broken end-to-end delivery pipeline across multiple platform tools
- Diagnose silent delivery failures in a distributed system using distributed tracing

**Technologies:** Backstage, Gitea, Argo Workflows, Argo CD, OpenTelemetry, Jaeger, Kubernetes

---

## Levels

### 🟢 Beginner: Laying the Keel

#### Description

Fix a broken Backstage software template so the commission office can register new vessels for service.

#### Story

The harbour master has filed a formal complaint. No vessels have cleared port in three weeks. The commission office is staffed but producing nothing: manifests are submitted, but repositories never appear, and no components are making it into the registry.

Your mission: investigate the Backstage software template, find what's broken, and restore the commission office so captains can register new vessels again.

#### The Problem

The vessel commissioning template is misconfigured. It is intended to scaffold a new service, create a repository in Gitea, and register the component in the Backstage catalog, but multiple issues prevent it from completing successfully.

#### Objective

- Successfully scaffold a new service using the vessel commissioning template with no errors
- See the new service appear as a registered component in the Backstage catalog
- Confirm a repository was created in Gitea containing the correct `catalog-info.yaml`

#### What You'll Learn

- How Backstage software templates are structured: parameters, steps, and output
- How scaffolder actions work (`fetch:template`, `publish:gitea`, `catalog:register`)
- How the catalog registration step connects a scaffolded repository to the Backstage catalog

#### Tools & Infrastructure

- **Tools:** Backstage UI, `kubectl`, `k9s`
- **Infrastructure:** Kubernetes Cluster, Backstage, Gitea

---

### 🟡 Intermediate: Sea Trial

#### Description

Fix the broken integration points in the delivery pipeline so that commissioning a vessel in Backstage results in a running deployment.

#### Story

The commission office is open again. Manifests are filed, repositories are created, components are registered. But the captains are back at the door with complaints: their vessels are stuck in the harbor. The manifest reaches the registry, but nothing moves after that. Provisions are never loaded, routes are never set, and no vessel has actually sailed.

Somewhere between the commission office and the open water, the golden path is broken. Your mission: trace the delivery pipeline from end to end, find where it breaks, and connect the harbors so that a filed manifest results in a vessel underway.

#### The Problem

Several integration points between platform components are misconfigured. The connection between Gitea and the workflow engine, the workflow step ordering, and the Argo CD configuration each have issues that together prevent a commissioned service from running in the cluster.

#### Objective

- Commission a new service end-to-end: template run, Gitea repository created, Argo Workflow completes, Argo CD Application created and synced, service running in the cluster
- Confirm the service is healthy by accessing it directly
- Identify which component broke at each integration point by reading its logs or UI

#### What You'll Learn

- How webhooks connect git events to workflow engines
- How Argo Workflows orchestrate multi-step delivery pipelines
- How Argo CD's app-of-apps pattern manages dynamically created applications
- How to trace a silent failure across multiple platform components

#### Tools & Infrastructure

- **Tools:** Backstage UI, `kubectl`, `k9s`, Argo CD UI, Argo Workflows UI
- **Infrastructure:** Kubernetes Cluster, Backstage, Gitea, Argo Workflows, Argo CD

---

### 🔴 Expert: The Chronometer

#### Description

Repair the fleet's broken navigation log, then use the complete traces it produces to diagnose why vessels are arriving with the wrong provisions.

#### Story

The fleet is operational. Vessels are being commissioned, routes are set, and the harbor is busy. But reports have been arriving from distant ports: some vessels are reaching their destination with the wrong provisions. The captains filed correct manifests. The commission office recorded everything. Every system along the route reported a successful voyage.

And yet the cargo doesn't match the orders.

The fleet's telemetry system is supposed to keep a precise record of every voyage: departure time, each leg of the journey, what was loaded at every stop, and the exact parameters carried from one hand to the next. But the log is dark. Someone left the commission office's instruments unconfigured, and the one section of the route that was logging isn't connecting its records to the rest.

You can't read a log that was never written. Fix the instruments first. Then find where the voyage went wrong.

#### The Problem

Two things are broken before the traces can even be read. Backstage is not emitting spans: its OpenTelemetry plugin is misconfigured in `app-config.yaml`, so nothing from the commission office reaches Jaeger. Once that is fixed, traces appear but are fragmented: the workflow is not propagating trace context in its calls, so the Argo Workflows and Argo CD spans are disconnected from the Backstage spans.

With a complete trace finally visible, the root cause becomes clear: a workflow step executes more than once due to a retry logic bug, and the repeated execution uses different parameters than the original. Every tool reported success. Only the full trace reveals what actually happened.

#### Objective

- Fix Backstage's OpenTelemetry configuration so the commission office emits spans to Jaeger
- Fix trace context propagation so the full delivery pipeline appears as a single connected trace
- Identify how and where the delivery pipeline produced incorrect service configuration by reading the complete traces
- Fix the workflow so each commissioning produces exactly one execution of every step with the correct parameters

#### What You'll Learn

- How to configure Backstage's OpenTelemetry plugin to emit traces
- How trace context propagation connects spans across system boundaries
- How distributed traces reveal causal chains that per-tool logs cannot: not just that something went wrong, but what data flowed through each step
- Why observability is qualitatively different from logging, and what it means to navigate by dead reckoning versus taking a fix

#### Tools & Infrastructure

- **Tools:** Backstage UI, `kubectl`, `k9s`, Jaeger UI
- **Infrastructure:** Kubernetes Cluster, Backstage, Gitea, Argo Workflows, Argo CD, OpenTelemetry Collector, Jaeger
