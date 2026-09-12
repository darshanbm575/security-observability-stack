# Detection Rules

Details and instrumentation requirements for each rule in `prometheus/rules/security-alerts.yml`.

## HighAuthFailureRate

**Signal:** `rate(app_auth_failures_total[5m]) > 5`

**Requires:** your application to increment a `app_auth_failures_total` counter on every failed login attempt (ideally labeled by `method`, e.g. password vs. token). Most web frameworks make this a few lines in the auth middleware.

**Why 5/sec for 2m:** tuned to catch sustained brute-force or credential-stuffing traffic while tolerating normal background noise (users mistyping passwords). Adjust down for a low-traffic internal app, up for a high-traffic public one — the right threshold is empirical, watch it in `audit` for a week before trusting it to page anyone.

## ContainerCrashLoop

**Signal:** more than 3 container restarts in 10 minutes, via cAdvisor's `container_start_time_seconds`.

**Why it's a security rule, not just reliability:** a crash loop is often read as "just a bug," but a process crashing repeatedly can also mean something is actively probing it (malformed input causing a panic, an exploit attempt that OOMs the process instead of succeeding). Worth the same triage as an auth spike, not just an auto-restart-and-forget.

## UnexpectedPrivilegedProcess

**Signal:** `process_running_as_root{expected="false"} == 1`

**Requires:** a node-level exporter (or a Prometheus textfile collector cron job) that enumerates running processes and compares against an allow-list of "these are expected to run as root" (e.g., `sshd`), emitting `expected="false"` for anything not on that list running as root.

**Why it matters:** privilege escalation frequently manifests as an unexpected process suddenly running with root privileges. This is a cheap tripwire for that.

## HighRateOf4xxFromSingleSource

**Signal:** `sum by (source_ip) (rate(app_http_requests_total{status=~"401|403"}[5m])) > 10`

**Requires:** your app/reverse-proxy to export `app_http_requests_total` labeled by `status` and `source_ip` (careful with cardinality — bucket or sample source IPs if traffic is very high).

**Why it matters:** a burst of 401/403s from one source is the signature of endpoint enumeration or credential stuffing, distinct from the general background rate of legitimate auth failures.

## NodeExporterTargetDown

**Signal:** `up{job="node-exporter"} == 0`

**Why it's a security rule:** most of the time this is just an instance restarting. But an attacker who's gained a foothold and wants to operate unobserved will often try to kill or block the monitoring agent first — so a monitoring gap deserves the same "critical" severity as a confirmed incident until ruled out, not a shrug.

## Adding a rule

1. Add the alert to `prometheus/rules/security-alerts.yml` with a `for:` duration (avoid single-sample false positives) and a `severity` + `category` label.
2. Document the instrumentation it depends on here.
3. Route it in `alertmanager/alertmanager.yml` if it needs a different notification channel than the default.
