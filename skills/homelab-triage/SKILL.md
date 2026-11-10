---
name: homelab-triage
description: Triage a problem on a self-hosted (homelab) stack through the Grafana MCP, read-only. Always starts with a wide pass over every service it can see (blackbox probes, scrape targets, host health, container restarts, log errors per service, trace errors, image updates), narrows to at most five suspects, then tests one hypothesis at a time within a budget of 20 queries, and reports the root cause or the best lead with Grafana links, what was ruled out and the next query. Use when the user says something on their homelab, server or self-hosted services is down, slow, broken, erroring or "feels off", names a service or alert to look into, or asks "what's wrong with my server", "triage this", "is anything broken", "why is X down", and in Spanish "algo anda mal en el servidor", "se cayó X", "X está lento", "revisa el homelab", "qué está fallando", "triage", "investiga esta alerta". Not for building dashboards or alerts, and not for fixing things: it diagnoses and suggests.
argument-hint: "[starting point: service, URL, alert or symptom] [window: 1h | 30m | 2026-11-09 14:00-15:00]"
---

# Homelab triage

Find why something on the stack is wrong, without anchoring on where the user
noticed it. Read `references/debugging.md` once at the start: it holds the
reasoning rules. `references/queries.md` holds the query patterns for each step.

## Hard rules

1. **Read-only.** Never call a Grafana write tool (annotations, incidents,
   alert rules, dashboards, silences) and never run commands that change
   containers or hosts. Suggest fixes in the report; do not apply them.
2. **Budget: 20 data queries.** A data query is any call that reads metrics,
   logs, traces or label values. `list_datasources` and `generate_deeplink` are
   free. Keep a ledger and tag every query in your notes as `[q 7/20]`. At 20,
   stop and report, even mid-drill. If the user grants more ("sigue", "keep
   going", "10 more"), continue with the new total.
3. **Wide pass first, always.** Even when the user names a service, run the
   whole wide pass before querying that service on its own.
4. **"Where you came in" is not the suspect.** Record the starting point as
   context. It becomes a suspect only if the wide pass ranks it.
5. **At most five suspects, one hypothesis at a time.** Each drill query tests
   one stated hypothesis and gets a verdict before the next one starts.
6. **Compare, don't read absolutes.** Some services log thousands of "errors"
   every hour forever. A signal counts when it moved against the baseline.

## Step 0: setup (free)

- Parse the starting point and the window. Default window: the last hour,
  baseline: the same hour one week earlier (`offset 1w`). A window from the
  user ("last 30m", "since 14:00", explicit times) replaces the default; the
  baseline keeps the same length, one week earlier.
- `list_datasources`. Map roles: the Prometheus datasource, every Loki
  datasource (several Lokis usually mean one per host; note which), Tempo if
  present. Skip InfluxDB and others unless the starting point points there.
- Do not hard-code services, hosts or domains: everything comes from query
  results. Discovery happens inside the wide pass, not before it.

## Step 1: wide pass (about 7 queries)

Run these, in parallel when the tools allow, using the patterns in
`references/queries.md`:

| # | Signal | Source |
|---|---|---|
| W1 | Failing blackbox probes and down scrape targets, now and in the baseline | Prometheus |
| W2 | Host health per host: worst filesystem, memory, load per CPU, vs baseline | Prometheus |
| W3 | Containers started (restarted) in the window, per host | Prometheus (cadvisor) |
| W4 | Error-looking log lines per service, window vs baseline, one query each per Loki | Loki (2 queries for 2 Lokis, each with `offset`) |
| W5 | Error spans per service in the window | Tempo search |
| W6 | Image updates and failed updates in the window: the homelab's deploys | Loki, the watchtower (or similar updater) service |

If a source is missing (no Tempo, no cadvisor), say so in the report and move
on; do not spend queries hunting for it.

## Step 2: narrow (no queries)

Rank at most five suspects from the wide pass. A suspect is a service or a host
with a signal that moved: probe failing now but not in the baseline, errors up
several times over baseline, a restart or image update inside the window, a
resource near exhaustion and growing. For each one, one line: the signal, the
baseline, and why it ranks where it does. If nothing moved, say so and go to
the report: "nothing stands out" is a valid result.

Ranking hints: an image update or restart shortly before the symptom started
ranks high; a host-level problem (disk, memory) outranks the services on that
host, because it explains all of them; a steady signal with the same value in
the baseline ranks low or drops out.

## Step 3: drill (the remaining budget)

For the top suspect, write before querying:

```
H1: <hypothesis> | would be false if: <observation> | query: <what you will run>
```

Run it, then write `verdict: confirmed | ruled out | inconclusive` with the
number that decided it. Confirmed and it explains the symptom: that is the root
cause, stop. Ruled out: move to the next hypothesis or suspect. When a deploy
(image update or restart) is the suspect, compare the hour after it with the
hour before it instead of last week.

Read actual log lines (limit 10 to 20, `format: compact`) only for a suspect,
never during the wide pass.

## Step 4: report

Reply in the user's language (Spanish means Chilean or neutral Spanish, never
voseo). Shape:

```
## Homelab triage: <one-line verdict>
Window: <window> vs <baseline> | Queries: <n>/20 | Came in from: <starting point or "none">

### Root cause  (or "Best lead" if not confirmed, or "Nothing stands out")
<what, evidence with numbers, confidence: high/medium/low>

### Suspects
1. <service/host>: <signal vs baseline> -> <confirmed/ruled out/not tested> [link]

### Ruled out
- <thing>: <the number that ruled it out>

### Steady noise (not this incident)
- <service>: <signal that is high but the same as baseline>

### Next
- Query: `<exact query>` on <datasource>
- Suggested fix (not applied): <if root cause found>
```

Links: one `generate_deeplink` Explore link per suspect with the query that
decided its verdict, plus a dashboard link when `search_dashboards` (counts as
a query) found an obvious one. Keep the report under about 40 lines.

## Gotchas

- Loki rejects or slows on `{}`; use `{service_name=~".+"}` with an error line
  filter and `sum by (service_name)`, as an instant metric query.
- Tempo metrics range queries return hundreds of empty samples; use
  `search_tempo_traces` with `{ status = error }` for the wide pass.
- Watchtower logs failed updates at `level=info` ("Unable to update
  container"), so filter on the message, not the level.
- A recurring health-check job that errors every run (same count in the
  baseline) is steady noise, not the incident.
