# Decision Records — PostHarvest

Structural decisions get numbered records here. Earlier decisions referenced
across the codebase (D1–D12 and O1/O4) predate this file and are documented
in the milestone plans under `plans/` and in `deploy/README.md`; the numbering
continues from D13 so those references stay valid.

---

## D13 — Prometheus + Grafana OSS, dashboards as code

**Date:** 2026-09-24  **Milestone:** `plans/monitoring.md`

We run Open Source Prometheus + Grafana as first-class compose citizens instead
of a SaaS or a fork. Datasources, dashboard JSONs, and alert rules are
provisioned as code from `docker/grafana/provisioning/` and
`docker/grafana/dashboards/` — everything lives in git, nothing is hand-made in
the UI.

Each service already exposes a Prometheus text-format `/metrics` (backend, go,
node); redis and nginx get exporters; the host and containers are covered by
`node_exporter` + `cadvisor`.

## D14 — Egress stays internal-only; prometheus is dual-homed

**Date:** 2026-09-24  **Milestone:** `plans/monitoring.md`

`prometheus` joins BOTH compose networks — `web` and `isolated`
(`internal: true`) — so one instance reaches every `/metrics` (backend/node/
go are dual-homed or on `isolated`; nginx/frontend/exporters on `web`) while
the stack still gets no route out beyond what prod services already have.
`grafana` joins `web` only; it talks to nothing but prometheus.

## D15 — No monitoring port is published; Grafana via SSH tunnel

**Date:** 2026-09-24  **Milestone:** `plans/monitoring.md`

No monitoring service publishes a host port. Grafana and every `/metrics`
endpoint are unreachable from outside the box; operators reach Grafana with
`ssh -L 3001:127.0.0.1:3001` and the browser on localhost. A public
basic-auth/Cloudflare route for Grafana is explicitly out of scope for this
milestone.

## D16 — /metrics is unauthenticated but unreachable; stub_status allow-listed

**Date:** 2026-09-24  **Milestone:** `plans/monitoring.md`

`/metrics` endpoints carry no auth — they are unexposable by construction:
nginx never proxies them, and no compose service publishes them. The only nginx
addition for monitoring is an internal-only `stub_status` listener on `:18080`
locked to the docker networks (`allow 172.16.0.0/12; allow 10.0.0.0/8; deny
all;`).

## D17 — Grafana-native alerting, webhook URL from env

**Date:** 2026-09-24  **Milestone:** `plans/monitoring.md`

Alerts are Grafana-managed rules provisioned as code (backend down, error-rate
spike, redis down / memory pressure, disk pressure, any scrape target down).
The notification channel is a generic webhook whose URL comes from
`GRAFANA_ALERT_WEBHOOK_URL` in `docker/.env`. When it is empty the rules still
exist and fire in the UI, but the contact point is inert — no notifications
until a channel is configured.