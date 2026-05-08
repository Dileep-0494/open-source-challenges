# 🔴 Expert: Phase 3 — read the chart

Three sub-tasks:

1. **Wire the OpenTelemetry meter provider** and register the OpenFeature `MetricsHook` so flag evaluations show up as Prometheus counters.
2. **Author a `ContextSpanHook`** of your own — a small `Hook` that copies the merged evaluation context (`species`, `country`, `dose`) onto the active OTel span as `feature_flag.context.<key>` so traces correlate variants with the context that drove them.
3. **Diagnose and roll back a misbehaving fractional rollout.** The `vision_amplifier_v2` flag is at 100% on; it's adding 200ms latency and a 10% HTTP 5xx rate. Identify it on the Grafana dashboard and roll it back via `flags.json` — no redeploy.

Spans are already flowing into Tempo from the OpenFeature `TracesHook`, but the metrics half is dead — the `MeterProvider` has no exporter and the `MetricsHook` was never registered. The dashboard the operator wants to triage from is empty. The k6 loadgen is idle, waiting for a flag flip to turn it on.

The level passes when (a) `feature_flag_evaluation_requests_total` is non-zero in Prometheus, (b) Tempo spans for `fun-with-flags-java-spring` carry `feature_flag.context.*` attributes, (c) `vision_amplifier_v2` is rolled back to 100% off, and (d) the HTTP 5xx rate over the last minute is below 1%.

## 🪐 The Backstory

The trial just went wide. Phase 3 of the new vision amplifier — `vision_amplifier_v2` — was approved for the full cohort yesterday morning. The promise was straightforward: subjects emerge with sharper eyesight than they walked in with. By mid-afternoon the audit log was screaming. Subjects were stabilising 200ms slower, and roughly one in ten of them was emerging **blind** — containment failure recorded as an HTTP 500. The lab director pulled up the **Feature Flag Metrics** dashboard expecting to triage visually. The dashboard was dark. Someone had wired up traces but never finished the metrics half. There is no chart to read. The lab is studying eyesight and the lab itself cannot see.

Your job, in order: **turn on the lights**, find the bad arm of the trial, and **halt enrolment** on the amplifier — all without redeploying the lab. That last constraint is the whole point of feature flags: when a rollout starts misbehaving in production, you need an operational lever that does not take twenty minutes to pull. Save the file, watch the dose drop, watch the 5xx rate fall back to baseline, watch the next batch of subjects walk out seeing.

## ⏰ Deadline

> 🚧 **Coming Soon** — this level is in the planned bucket. Final deadline will be announced when the adventure goes live.

## 💬 Join the discussion

> 🚧 **Coming Soon** — community thread will be linked here at launch.

## 🏗️ Architecture

Four containers and one Spring Boot process, all on a shared Docker network.

```
┌──────────────────────┐      OTLP/gRPC :4317      ┌────────────────────────┐
│  Spring Boot         │ ────────────────────────▶ │  grafana/otel-lgtm     │
│  fun-with-flags-     │      flag eval + HTTP     │   - Grafana   :3000    │
│  java-spring         │                           │   - Prometheus :9090   │
│  :8080               │                           │   - Tempo     :3200    │
└─────┬────────────────┘                           └─────────▲──────────────┘
      │ OpenFeature SDK :8013                                │ scrape / pull
      │ (RPC mode)                                           │
┌─────▼────────────────┐                           ┌─────────┴──────────────┐
│  flagd               │ ◀──── poll loadgen flag ──│  k6 loadgen            │
│  :8013 (gRPC + HTTP  │                           │  HTTP GET /?userId=…   │
│         eval gateway)│                           │  (the lab interceptor  │
│  :8014 management /  │                           │   sets userId as the   │
│        metrics       │                           │   targetingKey, which  │
│  :8015 sync stream   │                           │   is what fractional   │
│  :8016 OFREP         │                           │   rollouts bucket on)  │
│  flags.json mounted  │                           │                        │
└──────────────────────┘                           └────────────────────────┘
```

## 🎯 Objective

By the end of this level, the lab hits each of these observable outcomes:

- **Spans for `fun-with-flags-java-spring` are visible in Tempo** with `feature_flag.context.<key>` attributes — searching `feature_flag.context.dose=underdose` lights up the requests where a tech mis-dosed, with `feature_flag.variant=clouded` on the same span.
- **`feature_flag_evaluation_requests_total` is non-zero in Prometheus** — flag evaluations show up as counters, not just spans.
- **The Feature Flag Metrics dashboard renders.** Variant-distribution, error rate, latency p99 — all populated from the metric counters.
- **The `vision_amplifier_v2` rollout is rolled back to 100% off** — without redeploying the lab.
- **HTTP 5xx rate over the last minute drops below 1%.** The bad arm is contained.

## 🧠 What You'll Learn

- How the OpenFeature OpenTelemetry hooks (`TracesHook` and `MetricsHook`) join
  flag evaluations to the rest of an application's telemetry without a
  separate ingestion path
- How to **author your own `Hook`** — a tiny class that copies merged-eval-context
  attributes onto the active OTel span — to close the loop between *why* a
  flag resolved the way it did and *what* the operator sees in Tempo
- How [`fractional`](https://flagd.dev/reference/custom-operations/fractional-operation/)
  rollout in flagd buckets users by `targetingKey` — same key, same bucket, every
  request — and how to read that bucketing off a dashboard
- How a **flag flip** is a faster operational lever than a redeploy when a
  rollout is misbehaving — the difference between a one-line config change and
  a twenty-minute deployment

## 🧰 Toolbox

Your Codespace comes pre-configured with the following tools:

- [`curl`](https://curl.se/): HTTP client for hitting the lab, flagd, and Prometheus
- [`./mvnw`](https://maven.apache.org/wrapper/): The Maven wrapper to build and run the Spring Boot lab
- A browser pointed at [`http://localhost:3000`](http://localhost:3000) for Grafana (admin / admin)
- [`jq`](https://jqlang.github.io/jq/): Pretty-print and filter JSON from `curl`

flagd, the Grafana LGTM stack, and the k6 loadgen are **sibling devcontainer services** — they come up automatically when the Codespace boots. There is no `docker compose up` step. Inside the workspace they are reachable as `flagd`, `lgtm`, and `loadgen`. The Grafana / Prometheus / Tempo / OTLP ports on `lgtm` are also forwarded onto the Codespace host so you can click them in the Ports tab; flagd stays on the docker-internal network only.

## ✅ How to Play

### 1. Start Your Challenge

> 📖 **First time?** Check out the [Getting Started Guide](../../start-a-challenge)
> for detailed instructions on forking, starting a Codespace, and waiting for
> infrastructure setup.

Quick start:

- Fork the repo
- Create a Codespace
- Select **"Adventure 00 | 🔴 Expert (Phase 3 — read the chart)"**
- Wait ~2-3 minutes for the sibling containers (flagd, Grafana LGTM, k6
  loadgen) to come up. They are part of the devcontainer compose, so they
  start automatically — no `docker compose up` step.

### 2. Start the Lab

The sibling containers (flagd, the LGTM stack, the k6 loadgen) are already up — the Spring Boot lab itself isn't. Boot it before you click into the Ports tab so the forwarded `:8080` is actually serving. Either click **Run** on `Laboratory` in the Spring Boot Dashboard panel (or press **F5** with `Laboratory.java` open), or, from the terminal:

```bash
cd adventures/planned/00-blind-by-design/expert
./mvnw spring-boot:run
```

Spans start flowing into Tempo on the first request — the OpenTelemetry trace pipeline is already wired. The metrics half is dead (task 4a) so the Grafana dashboard panels stay empty until you fix it.

### 3. Access the UIs

Open the **Ports** tab in the bottom panel and click through to:

#### Spring Boot lab (Port `8080`)

The application under test. Open `http://localhost:8080/` to get a vision_state reading
back. Add a `userId` query parameter (e.g. `?userId=subject-42`) to give the
fractional rollout a stable bucketing key.

#### Grafana (Port `3000`)

The single window into the LGTM stack. Login is `admin` / `admin` (skip the
"change your password" prompt).

- **Dashboards → Fun With Flags — Feature Flag Metrics** — the dashboard the
  director keeps reloading. Empty for now.
- **Explore → Tempo** — search by service `fun-with-flags-java-spring`
  to see flag evaluations as span events nested inside HTTP request spans.
  Traces work even before you wire up metrics.

#### Prometheus (Port `9090`)

Exposed by the LGTM container. Useful for `curl`-driven debugging:
`curl 'http://localhost:9090/api/v1/query?query=feature_flag_evaluation_requests_total'`.

#### Tempo (Port `3200`)

Tempo's own HTTP API. The `verify.sh` script uses
`http://localhost:3200/api/search?tags=service.name=fun-with-flags-java-spring`
to assert traces are flowing.

#### flagd

flagd runs on the docker-internal network only. The lab and the loadgen reach it as `flagd:8013`; you don't need to forward its ports onto the Codespace host to play this level. (`verify.sh` runs inside the workspace container so it can reach `flagd:8013` directly.)

#### OTLP receivers (Ports `4317` / `4318`)

The Spring Boot app exports traces (and, after you finish the wiring, metrics)
to the LGTM stack on `4317` (gRPC) and `4318` (HTTP).

### 4. Implement the Objective

Four sub-tasks, in order: wire the meter provider, register the matching `MetricsHook`, write your own `ContextSpanHook` to enrich spans with the flag-decision context, then turn on the loadgen so you can find and roll back the misbehaving fractional rollout.

#### 4a. Wire the OpenTelemetry meter provider

OTel ships two parallel pipelines: **traces** (per-request spans, already flowing into Tempo) and **metrics** (aggregate counters, dead). Each has its own provider, its own SDK, its own exporter. The fix here is on the metrics side — a `MeterProvider` is being created but its exporter is `none`, so any metrics it records go nowhere. Both providers register globally via `GlobalOpenTelemetry`, so once the meter is wired the `MetricsHook` (next step) finds it without any further plumbing.

Open `adventures/planned/00-blind-by-design/expert/src/main/java/dev/openfeature/demo/java/demo/OpenTelemetryConfig.java`. The `@Bean` method already calls `AutoConfiguredOpenTelemetrySdk.builder()`, which produces an `OpenTelemetry` instance with **both** a `SdkTracerProvider` and a `SdkMeterProvider` — but only the tracer provider has an exporter. The meter provider is told `otel.metrics.exporter=none`.

Flip `otel.metrics.exporter` to `otlp` so the SDK attaches an `OtlpGrpcMetricExporter`. The cleanest way is to update both the default in `OpenTelemetryConfig.java` and the value in `src/main/resources/application.properties`. While you're there, set `otel.metric.export.interval=10000` so the dashboard updates within ten seconds of new traffic instead of waiting a minute.

#### 4b. Register `MetricsHook` on the OpenFeature API

The OpenFeature OTel contrib library ships two hooks that turn flag evaluations into telemetry: **`TracesHook`** emits a span event (`feature_flag.evaluation`) on the active span — that's why flag evaluations show up nested inside HTTP request spans in Tempo. **`MetricsHook`** emits four counters per evaluation: `feature_flag_evaluation_requests_total`, `_success_total`, `_error_total`, plus an active-count up/down counter. These power the dashboard panels.

Open `OpenFeatureConfig.java`. `TracesHook` is already registered; `MetricsHook` is not. `MetricsHook` needs the `OpenTelemetry` instance to grab the meter provider, so inject the bean via constructor injection and call `api.addHooks(new MetricsHook(openTelemetry));` next to the `TracesHook` line.

If you compile and run after this step, the **Fun With Flags — Feature Flag Metrics** dashboard in Grafana stays empty — there is no traffic to drive the counters. Move on.

#### 4c. Author and register your own `ContextSpanHook`

The two contrib hooks tell you *what* happened — which flag, which variant, which reason. The `AuditHook` shipped with this level (carried over from Intermediate) writes the durable archive view to disk. What's missing is the **on-call's view in Tempo**: when a span shows `feature_flag.variant=clouded`, the operator can't see *why* without a separate hop into the audit log. Write a third hook that copies the merged eval context attributes onto the active OTel span — same data the audit log records, but visible right next to the variant in the trace UI.

The shape is roughly:

```text
before(hookCtx) {
    span = active OTel span
    for each allowlisted key in merged eval context:
        span.setAttribute("feature_flag.context." + key, value)
}
```

The `before` callback receives a `HookContext`, and `getCtx()` returns the **merged** evaluation context (global + transaction + invocation) — exactly what drove the flag's resolution, so the attributes you copy off it line up with what the variant decision actually saw. Span attributes go on `Span.current()` because that's the active HTTP request span; the OpenFeature hook fires inside its scope.

Register it next to `TracesHook` / `MetricsHook` in `OpenFeatureConfig`. Now in Tempo: **Search → Service: fun-with-flags-java-spring → +Tag → `feature_flag.context.dose=underdose`** lights up exactly the requests where a tech mis-dosed, with the resolved variant on the same span event.

> ⚠️ **Allowlist, don't iterate.** Use a fixed allowlist (`List.of("species", "country", "dose")`) — never iterate the whole eval context. The merged context routinely carries the OpenFeature `targetingKey`, typically a stable user id that joins to email and account data in real apps. Span attributes are retained for days in Tempo and indexed at scale; once they ship, redacting after the fact is hard. Same discipline `AuditHook` already follows for the audit log, same reason. See [OpenTelemetry's security guidance](https://opentelemetry.io/docs/security/).

#### 4d. Turn on the loadgen, find the bad rollout, roll it back

`fractional` is flagd's bucketing operation: given a list of `[variant, percent]` pairs, it deterministically assigns each evaluation to a variant based on a hash of the **`targetingKey`** on the eval context. Same key → same bucket → same variant, every request. Different keys spread across the percentages. **If no targeting key is set, every evaluation hashes the same way, every request lands in the same bucket, and the percentages do nothing.** The `SpeciesInterceptor` shipped with this level reads `?userId=` from each request and threads it through as the targetingKey — the lab is already serving fractional rollouts correctly without you touching it. The k6 loadgen exploits this: it generates a fresh random `userId` per request, which means a different targetingKey per request, which means the fractional rollout spreads across the percentages exactly as configured.

Edit `flags.json` in the expert directory and flip `loadgen_active`'s `defaultVariant` from `"off"` to `"on"`. flagd watches the file and picks up changes within a second. The k6 loadgen container has been polling `loadgen_active` every two seconds — it will notice and start hammering `http://workspace:8080/` with five virtual users (the workspace service name resolves inside the compose network).

Now open the dashboard. When the loadgen turns on you should see latency creep up around 200ms and 5xx rate around 10%; if those don't move, the loadgen flag isn't actually live yet.

That's the diagnosis: the fractional rollout for `vision_amplifier_v2` is inverted. The flag definition currently reads:

```json
"fractional": [
  ["off", 0],
  ["on", 100]
]
```

Edit `flags.json` again — flip the percentages so `off` gets `100` and `on` gets `0`. Save. Within one or two seconds flagd reloads. Because the targetingKey is sticky per `userId` and the loadgen generates a fresh `userId` per request, every subject re-buckets against the new percentages and the population moves to the safe variant. Watch the latency p99 panel collapse back to baseline and the 5xx rate fall to zero.

**No deploy. No rebuild. No restart of the lab.**

### 5. Verify Your Solution

Once you think you've solved the challenge, run the verification script:

```bash
./verify.sh
```

**If the verification fails:**

The script will tell you which checks failed. Fix the issues and run it again.

**If the verification passes:**

1. The script will check if your changes are committed and pushed.
2. Follow the on-screen instructions to commit your changes if needed.
3. Once everything is ready, the script will generate a **Certificate of Completion**.
4. **Copy this certificate** and paste it into the [challenge thread](https://community.open-ecosystem.com/c/open-ecosystem-challenges/) to claim your victory! 🏆
