# Query patterns

Generic patterns for a typical self-hosted stack: Prometheus with blackbox,
node_exporter and cadvisor; one Loki per host with a `service_name` label
(Promtail or Alloy Docker discovery); optionally Tempo. Adjust label names to
what the results show; never assume a job or host name.

`<W>` is the window length (`1h` by default). Run instant queries at the window
end. For a user-given window, set the query time to its end.

## W1: probes and targets

Prometheus, instant:

```promql
label_replace((probe_success == 0) or (up == 0), "period", "now", "", "")
or
label_replace((probe_success offset 1w == 0) or (up offset 1w == 0), "period", "baseline", "", "")
```

A target failing only `now` is new; failing in both is old breakage (report it,
rank it low). Empty result: everything is up, which is a finding.

## W2: host health

Prometheus, instant, one query for all hosts (group by `job` or `instance`,
whichever separates the hosts):

```promql
label_replace(max by (job) (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs"} / node_filesystem_size_bytes), "check", "disk_used", "", "")
or label_replace(max by (job) (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes), "check", "mem_used", "", "")
or label_replace(max by (job) (node_load5) / count by (job) (node_cpu_seconds_total{mode="idle"}), "check", "load_per_cpu", "", "")
```

Note: `max by (job) (node_load5) / count by (job) (...)`, not
`max by (job) (node_load5 / count ...)`: the inner division has mismatched
labels and returns nothing.

Thresholds: disk over 0.90, memory over 0.90, load per CPU over 1.0. Above a
threshold, check growth in the drill: the same expression with `offset 1w`, or
`predict_linear(node_filesystem_avail_bytes[6h], 86400) < 0` for a disk.

## W3: restarts

Prometheus, instant:

```promql
count by (job) (container_start_time_seconds{name!=""} > time() - <W seconds>)
or (count by (job) (container_last_seen{name!=""}) * 0)
```

The `or ... * 0` half makes every cadvisor job answer: `0` means it is scraped
and nothing restarted. A job missing from the result is a job cadvisor does not
report, which is a gap to note, not "no restarts".

If non-zero, the drill lists them:
`container_start_time_seconds{name!=""} > time() - <W seconds>`. Containers
missing entirely: `absent_over_time` per name is too expensive for the wide
pass; compare `count(container_last_seen{name!=""})` with `offset 1w` instead.

## W4: errors per service, per Loki

Loki, instant metric query, one per Loki datasource:

```logql
label_replace(sum by (service_name) (count_over_time({service_name=~".+"} |~ "(?i)(error|fatal|panic|exception|traceback)" [<W>])), "period", "now", "", "")
or
label_replace(sum by (service_name) (count_over_time({service_name=~".+"} |~ "(?i)(error|fatal|panic|exception|traceback)" [<W>] offset 1w)), "period", "baseline", "", "")
```

Set `limit` to 40 or more so no series is cut. Each service comes back with a
`period="now"` and/or `period="baseline"` row. A service with only a `now` row
is a new error source: a strong signal. Do not subtract the two sides
(`now - on(service_name) baseline`): the subtraction silently drops every
service present on only one side, which is exactly the one you want.

## W5: trace errors

Tempo, `search_tempo_traces`:

```traceql
{ status = error } | select(resource.service.name, name)
```

Read `serviceStats` and `rootTraceName`. A job that errors on every run with the
same span names is steady noise. Avoid `query_tempo_metrics` range queries in
the wide pass: they return a sample per step, mostly empty.

## W6: image updates

Loki, on the Loki of each host that runs an updater (watchtower, diun,
ouroboros), range query, limit 20:

```logql
{service_name="watchtower"} |~ "(?i)(found new|stopping|creating|unable to update|session done)"
```

`Found new ... image`, `Stopping` and `Creating` mean a container was replaced:
a deploy. `Unable to update` means the update failed and the old image is still
running. One query covers several hosts only if they share a Loki datasource.

## Drill patterns

- Log lines for a suspect: `{service_name="<svc>"} |~ "(?i)(error|fatal|panic)"`,
  limit 20, `format: compact`.
- Before and after a deploy at time T: the W4 expression scoped to the service,
  `[1h]` at T+1h versus at T.
- A probe failing: `probe_http_status_code{instance="<url>"}` and
  `probe_duration_seconds{instance="<url>"}` over the window (range, step 60).
- DNS: `probe_success{job=~".*dns.*"}`; if DNS probes fail, every HTTP probe
  failing is downstream of it.
- Container resources: `rate(container_cpu_usage_seconds_total{name="<c>"}[5m])`,
  `container_memory_working_set_bytes{name="<c>"}`, and
  `container_spec_memory_limit_bytes{name="<c>"}` for OOM risk.
