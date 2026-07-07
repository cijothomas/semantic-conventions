# `otel.sdk.log.created` — counting semantics analysis & path to stable

Working notes for clarifying and (eventually) stabilizing the
`otel.sdk.log.created` SDK self-observability metric. This is a design/analysis
document, **not** a proposed spec/semconv text yet. It captures the full context
so the work can be picked up later.

Status: **analysis complete, nothing filed yet.** Owner: @cijothomas.

---

## 1. The overarching goal

Drive **one** OpenTelemetry SDK self-observability metric to **stable** in the
semantic conventions, to prove out the stabilization path for the whole
`otel.sdk.*` self-observability metric family.

`otel.sdk.log.created` was chosen as the first candidate because:

- It has **zero attributes** — no transitive attribute stabilization required
  (every other candidate drags along development-stability attributes).
- It already has the **broadest cross-SDK implementation coverage**.

### Prerequisite (DONE)

Rust did not implement this metric. It has now been added so Rust joins
Java/JS/Python/Go as an implementing SDK:

- PR: **open-telemetry/opentelemetry-rust#3530** ("feat(sdk): add
  otel.sdk.log.created self-observability metric").
- `SdkLogger` holds a `BoundCounter<u64>` built via `global::meter("otel.sdk")`,
  incremented in `emit()` after the telemetry-suppression check.
- Gated behind the `experimental_metrics_bound_instruments` Cargo feature
  (because the semconv is still `stability: development`).

---

## 2. Current metric definition

`model/otel/metrics.yaml`:

```yaml
- id: metric.otel.sdk.log.created
  type: metric
  metric_name: otel.sdk.log.created
  annotations:
    code_generation:
      metric_value_type: int
  stability: development
  brief: "The number of logs submitted to enabled SDK Loggers."
  instrument: counter
  unit: "{log_record}"
```

- Instrument: `counter`, monotonic, `int` (u64)
- Unit: `{log_record}`
- **No attributes**
- **No `note:`** — the entire semantics rest on the one-line brief.

Originating PR: **open-telemetry/semantic-conventions#1921** ("Add SDK Log health
metrics"), a follow-up to **#1631** (spans). Stated motivation:

> "SDK self-monitoring metrics to give insights into how the SDK is performing,
> e.g. whether data is being dropped due to overload / misconfiguration or
> everything is healthy."

The #1921 PR text also says *"only enabled loggers count"*, but neither the PR
nor the metric text defines what "enabled" means.

---

## 3. The core problem: "enabled" is undefined

The brief says "logs submitted to **enabled** SDK Loggers." "Enabled" is
ambiguous, and the 5 SDKs interpret it differently. This produces materially
different counts for the same application behavior.

### Cross-SDK implementation audit

Where each SDK places the `otel.sdk.log.created` increment inside its emit path:

| SDK | Increment position | Gated on... |
|---|---|---|
| **Java** | After shutdown check, after `isEnabled()` | `LoggerConfig.enabled` + `minimum_severity` + `trace_based` |
| **JS** | After `enabled()` | `LoggerConfig.disabled` + `minimum_severity` + `trace_based` + **every processor's `enabled()`** |
| **Python** | After `_is_enabled()` | `LoggerConfig.is_enabled` only |
| **Go** | First action in `Emit()` | nothing (no `LoggerConfig` concept) |
| **Rust** | First action in `emit()` (after telemetry-suppression early-return) | nothing (no `LoggerConfig` concept) |

Feature-flag / opt-in state (orthogonal to counting position):

- Java: always on (no flag)
- JS: always on (no flag) — 3/17 self-obs metrics unflagged incl. `log.created`
- Python: behind `OTEL_PYTHON_SDK_INTERNAL_METRICS_ENABLED` (off by default)
- Go: behind `OTEL_GO_X_OBSERVABILITY` env var (experimental)
- Rust: behind `experimental_metrics_bound_instruments` Cargo feature

### Concrete divergence

Workload: user code submits **100 `INFO` records** via a Logger with
`minimum_severity = WARN` (for SDKs that support it; Go/Rust have no
`LoggerConfig`, so they simply submit unfiltered).

| SDK | `otel.sdk.log.created` reports |
|---|---|
| Java | **0** (severity filter runs before the counter) |
| JS | **0** (same) |
| Python | **100** (no severity gate) |
| Go | **100** (no filtering at all) |
| Rust | **100** (no filtering) |

A 0-vs-100 divergence for identical app behavior defeats the metric's purpose as
a *cross-SDK* health signal.

**Key insight:** the 0-vs-100 split is driven almost entirely by
`minimum_severity`, which is a **`LoggerConfig`** field. `LoggerConfig` is itself
`development`-status and thinly implemented. See §6.

---

## 4. What the metric is *for* (original intent)

From #1921 / #1631: these are **SDK health metrics**. The intended usage is to
compare **intake** (`log.created`) against **downstream** counters
(`otel.sdk.processor.log.processed`, exporter metrics). The **delta** shows where
records are dropped (severity filter, disabled logger, queue overflow, exporter
failure, shutdown...).

**Implication:** if `log.created` itself already drops records (severity,
trace-based, processor-enabled), those filter effects become invisible from this
metric — you can no longer observe "the severity filter is dropping 60% of my
intake." So the intent points toward `log.created` being a **pre-filter intake
count**.

### Precedent: `otel.sdk.span.started`

The analogous span metric already resolved this exact question:

```yaml
brief: "The number of created spans."
note: "Implementations MUST record this metric for all spans, even for non-recording ones."
attributes:
  - ref: otel.span.sampling_result
```

`span.started` counts **all** spans regardless of the sampling decision and
exposes `otel.span.sampling_result` as an attribute so users can break down
intake by outcome. Same shape wanted for `log.created`: count intake, attribute
or downstream-metric the outcome.

Caveat: not a perfect analogy — spans always return a span object, while logs
have `Logger.Enabled()` explicitly so bridges can avoid *building* a record.

---

## 5. The spec-layering complication (`LoggerConfig`)

The Logs SDK spec models `LoggerConfig` with three sibling fields that operate at
**different scopes** — this is the root cause of the confusion:

Spec: <https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/logs/sdk.md#loggerconfig>

| Field | Spec says | Scope |
|---|---|---|
| `enabled = false` | "Logger MUST behave equivalently to **No-op Logger**" | **Logger-level** (whole logger off) |
| `minimum_severity` | "log record MUST be **dropped by the Logger**" | **per-record** filter |
| `trace_based = true` | unsampled "log record MUST be **dropped**" | **per-record** filter |

So `enabled` is a lifecycle switch (the whole Logger becomes no-op), while
`minimum_severity` / `trace_based` are per-record filters applied *within* an
enabled Logger.

Two further sources of confusion:

1. **"Emit a LogRecord" enumerates only 2 filtering rules** (min severity,
   trace-based) and does **not** mention `enabled`. Consistent with the
   "enabled=false ⇒ whole logger no-op, handled at creation" reading, but never
   stated explicitly. Java nonetheless checks `enabled` dynamically inside
   `emit()`.
2. **`Logger.Enabled()` collapses 5 conditions** into one boolean: no processors,
   `LoggerConfig.enabled=false`, below `minimum_severity`, `trace_based`+unsampled,
   all processors' `Enabled()` false. So when prose says "enabled Logger," it's
   ambiguous whether it means the `LoggerConfig.enabled` *field* or the
   `Enabled()` *API result*. This is exactly what makes the metric brief
   ("enabled SDK Loggers") ambiguous.

### Candidate spec issues (identified, not filed)

- **A. `LoggerConfig` mixes two operational scopes in one struct.** Ask: add a
  sentence distinguishing `enabled` (logger lifecycle) from `minimum_severity` /
  `trace_based` (per-record filters).
- **B. "Emit a LogRecord" omits `enabled`.** Ask: add a leading sentence saying a
  Logger with `enabled=false` MUST behave as no-op including for `Emit`, and that
  implementations MAY enforce at creation, at each `Emit`, or both.
- **C. `Logger.Enabled()` collapses 5 conditions.** Not a bug (it's a useful
  optimization hint) but a note recommending prose specify *which* sense of
  "enabled" is meant would help.

Recommendation from the session: if pursued, file **one** spec issue covering
**A + B** (same root cause); leave C to surface via the semconv discussion.

---

## 6. Decision point: pursue spec-first, or semconv-first with LoggerConfig set aside?

`LoggerConfig` is `development`-status and **not widely implemented** (only
Java/JS/Python have it; Go/Rust don't). Two paths:

### Path 1 — spec-first
Fix the `LoggerConfig` layering (issues A+B) before touching semconv, because the
semconv counting rule depends on the Logger-level-vs-per-record reading. Cleaner
foundationally, but OTel spec changes are slow and could block the
path-to-stable.

### Path 2 — semconv-first, LoggerConfig set aside (CURRENT LEANING)
Since `LoggerConfig` is barely implemented, defer everything that depends on it
and clarify only the LoggerConfig-independent parts of `log.created` now.

**What remains to clarify once `LoggerConfig` is set aside:**

1. **Pre-filter vs post-filter intake** — reduces to: does the counter fire at
   `Emit` entry, or only after processor-side `Enabled()` checks? (JS is the only
   SDK gating on processor `enabled()`; everyone else counts at/near entry.)
2. **Bridge `Logger.Enabled()` skip → not counted.** Structural truth, not a
   filter. Uncontroversial. Worth stating.
3. **"No processors registered"** — should such records still count? Proposed:
   **yes** (they reached `Emit`; absence of downstream counts then explains the
   drop).
4. **Post-shutdown emits** — count or not? Java/JS exclude; Python/Go/Rust don't
   special-case. `processor.log.processed` already tracks this downstream via
   `error.type=already_shutdown`. Proposed lean: **count it**, let the downstream
   metric explain.
5. **Brief wording** — drop/redefine "enabled". Proposed:
   **"The number of log records submitted to the SDK."**

**What drops off the table** (LoggerConfig-dependent, defer):
`minimum_severity`, `trace_based`, and `enabled=false` whole-logger-no-op — i.e.
the hardest, most contentious cases (incl. the entire Java/JS 0-vs-100 split).

**Attractive property of Path 2:** it requires **no behavior change from any SDK
that hasn't implemented `LoggerConfig`** — Go, Python, Rust are already compliant
with the proposed "count at Emit entry" rule; only JS's processor-`enabled()`
gating would be up for discussion.

---

## 7. Proposed direction (LoggerConfig set aside)

Draft (for a future semconv issue/PR — **not final**):

```yaml
brief: "The number of log records submitted to the SDK."
note: |
  Count every log record submitted via `Logger.Emit` (or the equivalent).

  Records that a log bridge chose not to build — e.g. after calling
  `Logger.Enabled()` and getting `false` — are not counted, because no record
  was submitted to the SDK. Records submitted when no processors are registered
  ARE counted if they reach `Emit`.

  Interaction with `LoggerConfig` filtering (`minimum_severity`, `trace_based`,
  and the `enabled` lifecycle flag) is deferred until those features are more
  widely implemented and the underlying spec layering is clarified.
```

Two decisions still needed before drafting the actual issue:

- **(a)** Does processor-`Enabled()` / "no processors" affect the count?
  Session lean: **no** — count at `Emit` entry.
- **(b)** Post-shutdown — count or not? Session lean: **count**, let downstream
  (`processor.log.processed` + `error.type=already_shutdown`) explain.

---

## 8. Path to stable (once semantics are settled)

1. Land the counting-semantics clarification (semconv note, and/or spec A+B).
2. Confirm each SDK conforms (Rust: PR #3530; verify Java/JS/Python/Go).
3. Flip `otel.sdk.log.created` from `stability: development` → `stable` in
   `model/otel/metrics.yaml` + chloggen entry.
4. Gather sign-off from Java, Python, Go, JS, Rust SDK maintainers.
5. Spec follow-up: promote the relevant self-observability section to stable.

Regeneration/validation for any model change:
`make check` (full), `make table-generation registry-generation` (docs/registry).

---

## 9. Cross-references

- Rust impl PR: open-telemetry/opentelemetry-rust#3530
- Originating semconv PR (logs health metrics): #1921
- Spans health metrics PR (precedent): #1631
- Metric def: `model/otel/metrics.yaml` → `metric.otel.sdk.log.created`
- Sibling metric (downstream, has `error.type` incl. `already_shutdown`):
  `otel.sdk.processor.log.processed`
- Logs SDK spec (LoggerConfig, Emit, Enabled):
  <https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/logs/sdk.md>
- Verification for future stabilization via Weaver Live Check
  (in-progress: open-telemetry/opentelemetry-rust-contrib#613).

---

## 10. Next action (where we stopped)

Deciding between **Path 1 (spec-first, issues A+B)** and **Path 2 (semconv-first
with LoggerConfig set aside)**. Current leaning: **Path 2** — smaller,
lower-controversy, no SDK behavior change for non-LoggerConfig SDKs, filable
today with no spec dependency. Before filing, settle decisions (a) and (b) in §7.
