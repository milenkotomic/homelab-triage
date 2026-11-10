# homelab-triage

A Claude Code skill that triages problems on a self-hosted stack through the
[Grafana MCP server](https://github.com/grafana/mcp-grafana), read-only.

It is built to avoid the usual failure of AI debugging: anchoring on the
service you mentioned and running queries until something looks guilty.

- **Wide pass first, always**: blackbox probes, scrape targets, host health,
  container restarts, error-looking log lines per service on every Loki, trace
  errors and image updates (watchtower and similar), each compared with the
  same window one week earlier.
- **Narrow** to at most five suspects, ranked by what changed against the
  baseline. Steady noise is reported separately.
- **Drill one hypothesis at a time**, each with what would prove it false and a
  verdict.
- **Budget of 20 data queries.** When it runs out, it reports the best lead,
  what was ruled out and the next query instead of guessing.
- **Read-only.** It never writes to Grafana or touches containers; it suggests
  the fix.

Nothing about your stack is hard-coded: services, hosts and endpoints are
discovered from query results on each run.

## Install

```
claude plugin marketplace add milenkotomic/claude-plugins
claude plugin install homelab-triage@milenkotomic
```

Requires the Grafana MCP server connected to your Grafana with at least a
Viewer service account. Works best with Prometheus (blackbox, node_exporter,
cadvisor), Loki with a `service_name` label, and optionally Tempo.

## Use

In Claude Code: "is anything broken on my homelab?", "immich is slow, check
it", "why did the downloads fail, last 2 hours", or in Spanish "revisa el
homelab", "se cayó X". Or `/homelab-triage:homelab-triage [starting point] [window]`.

## Layout

- `skills/homelab-triage/SKILL.md`: the procedure, budget and report format
- `skills/homelab-triage/references/debugging.md`: reasoning rules (root cause
  before fix, no anchoring, change is the signal, one hypothesis at a time)
- `skills/homelab-triage/references/queries.md`: PromQL, LogQL and TraceQL
  patterns for each step, with the pitfalls that were hit while building it
- `skills/homelab-triage/evals/`: three cases with graded assertions

## License

MIT
