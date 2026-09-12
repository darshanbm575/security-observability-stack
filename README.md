# Security Observability Stack

Prometheus + Loki + Grafana + Alertmanager, defined entirely as code, with a set of custom detection rules aimed at security-relevant signals rather than only infrastructure health.

## Why this exists

Most observability stacks are tuned for "is the service up" — CPU, memory, request latency. Security-relevant signals (a spike in auth failures, a container restarting in a loop, an unexpected outbound connection pattern) need their own rules, or they get buried in the same noise as everything else. This repo wires those rules in from the start.

## Stack

```
┌────────────┐    scrape     ┌────────────┐   evaluate    ┌──────────────┐
│  targets    │ ────────────► │ Prometheus │ ────────────► │ Alertmanager │
│ (node/app)  │                │  + rules   │                └──────┬───────┘
└────────────┘                └─────┬──────┘                       │
                                     │                                ▼
┌────────────┐    tail logs   ┌────▼──────┐                   notification
│  log files  │ ─────────────► │  Promtail │──────► Loki            channel
└────────────┘                └───────────┘             │
                                                          ▼
                                                    ┌──────────┐
                                                    │ Grafana  │  dashboards +
                                                    │          │  log/metric correlation
                                                    └──────────┘
```

## Repo layout

```
docker-compose.yml
prometheus/prometheus.yml
prometheus/rules/security-alerts.yml   # the detection rules - see DETECTIONS.md
alertmanager/alertmanager.yml
loki/loki-config.yml
promtail/promtail-config.yml
grafana/provisioning/                  # datasources + dashboard provisioned automatically
DETECTIONS.md                          # what each rule catches and why it's tuned that way
```

## Running it

```bash
docker compose up -d
```

- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000 (admin/admin, change on first login)
- Alertmanager: http://localhost:9093

Grafana comes pre-provisioned with the Prometheus and Loki datasources and a starter security dashboard — no manual setup.

## Detection rules included

See [`DETECTIONS.md`](DETECTIONS.md) for the full rationale. Summary:

| Rule | Signal | Why it matters |
|---|---|---|
| `HighAuthFailureRate` | Spike in failed login attempts | Brute-force / credential-stuffing indicator |
| `ContainerCrashLoop` | Container restarting repeatedly in a short window | Could indicate exploitation attempts crashing the process, not just a bug |
| `UnexpectedPrivilegedProcess` | A process running as root where none is expected | Possible privilege escalation |
| `HighRateOf4xxFrom5xxHosts` | Sudden spike in 401/403 responses from a single source | Scanning / enumeration behavior |
| `NodeExporterTargetDown` | A monitored host stops reporting | Could be legitimate downtime, or an attacker disabling monitoring |

## Extending

Add new rules to `prometheus/rules/security-alerts.yml` following the existing pattern (a `for:` duration to avoid single-sample false positives, and a `severity` label so Alertmanager routes it correctly). Route severities to different notification channels in `alertmanager/alertmanager.yml`.
