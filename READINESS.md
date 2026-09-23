# Production Readiness Evidence Pack & Sign-Off Case

**Service**: `production-readiness-lab`  
**Branch**: `feat/readiness-signoff`  
**Date**: September 23, 2026  
**Evaluator**: Site Reliability Engineering / DevOps  

---

## 1. Deployment Evidence

### Automated CI Pipeline Verification
The service includes a GitHub Actions CI workflow ([`.github/workflows/ci.yml`](file:///Users/isaacreji/Desktop/production-readiness-lab/.github/workflows/ci.yml)) that runs unit tests, linter checks, and builds the container image.

#### Local CI Execution Results
- **Unit Tests & ESLint**:
  ```bash
  $ cd app && npm run lint && npm test

  > production-readiness-lab-app@1.0.0 lint
  > eslint .

  > production-readiness-lab-app@1.0.0 test
  > node --test tests/*.test.js

  ✔ GET / returns service info (17.5ms)
  ✔ GET /health returns healthy (3.4ms)
  ✔ GET /metrics exposes prometheus metrics (3.7ms)
  ℹ tests 3 | pass 3 | fail 0 | duration_ms 106.7ms
  ```
- **Container Image Build**:
  ```bash
  $ docker build -t production-readiness-lab:ci .
  [+] Building 3.4s (10/10) FINISHED
  => naming to docker.io/library/production-readiness-lab:ci
  ```

### Container Stack Health (`docker compose ps`)
Running `docker compose up -d` brings up all 3 stack services cleanly:

```text
NAME             IMAGE                          COMMAND                  SERVICE      STATUS
prl-app          production-readiness-lab-app   "docker-entrypoint.s…"   app          Up (healthy)
prl-grafana      grafana/grafana:11.1.0         "/run.sh"                grafana      Up
prl-prometheus   prom/prometheus:v2.53.0        "/bin/prometheus --c…"   prometheus   Up
```

### Config Verification Matrix
Verification of service names, environment variables, exposed ports, and Prometheus scrape configs:

| Component | Container Name | Exposed Port | Environment Variables | Scrape Target / Config | Alignment Status |
|-----------|----------------|--------------|-----------------------|------------------------|------------------|
| **App Service** | `prl-app` | `8080:8080` | `PORT=8080`<br>`APP_VERSION=1.0.0` | Target `app:8080/metrics` | Verified & Healthy |
| **Prometheus** | `prl-prometheus` | `9090:9090` | Default | Mounts `./prometheus/prometheus.yml:ro` | Verified & Healthy |
| **Grafana** | `prl-grafana` | `3000:3000` | `GF_SECURITY_ADMIN_USER=admin`<br>`GF_SECURITY_ADMIN_PASSWORD=admin` | Provisioned datasource `uid: prometheus` | Verified & Healthy |

---

## 2. Monitoring Visibility

### Prometheus Target Status
Prometheus target discovery (`http://localhost:9090/targets`) confirms `app:8080` scrape target state:

- **Target URL**: `http://app:8080/metrics`
- **Job Name**: `app`
- **Health State**: `UP`
- **Scrape Interval**: `5s`
- **Prometheus Query Confirmation**: `up{job="app"} = 1`

```json
{
  "discoveredLabels": {
    "__address__": "app:8080",
    "job": "app"
  },
  "health": "up",
  "scrapeUrl": "http://app:8080/metrics"
}
```

### Grafana Golden Signals Dashboard
The automatically provisioned Grafana dashboard **Service Overview** (`http://localhost:3000/d/service-overview/service-overview`) renders live metrics:

1. **Request Rate**: `sum(rate(http_requests_total[1m]))` ~ `8.7 - 15.0 req/s`
2. **Error Rate**: `sum(rate(http_errors_total[1m]))` tracking 404/500 response rates
3. **Latency (p95)**: `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))` ~ `0.0047s` (`4.7ms`)
4. **Health Stat**: `up{job="app"}` displaying `UP` (value: `1`, color: green)

---

## 3. Recovery Readiness (Rollback Demonstration)

### Rollback Procedure & Live Proof
To prove operational recovery capability, a degraded release (`v1.1.0-broken`) was deployed and subsequently rolled back to `v1.0.0`.

#### Step 1: Deploying Faulty Release (`v1.1.0-broken`)
Simulated an application fault by modifying `/health` to return `HTTP 500 Internal Server Error`:
```bash
$ APP_VERSION=1.1.0-broken docker compose up -d --build
$ curl -i http://localhost:8080/health
HTTP/1.1 500 Internal Server Error
{"status":"unhealthy","error":"Simulated system degradation in v1.1.0-broken"}
```

#### Step 2: Automated Failure Detection
Docker's health check probe executed 3 failing attempts (30s interval):
```text
NAME             IMAGE                          STATUS
prl-app          production-readiness-lab-app   Up (unhealthy)
```

#### Step 3: Rolling Back to Last Working Version (`v1.0.0`)
Reverted changes to application source code and redeployed the container:
```bash
$ git checkout app/app.js
$ docker compose up -d --build
```

#### Step 4: Verification of Service Recovery
```bash
$ docker compose ps
NAME             IMAGE                          STATUS
prl-app          production-readiness-lab-app   Up (healthy)

$ curl -i http://localhost:8080/health
HTTP/1.1 200 OK
{"status":"healthy","uptimeSeconds":21}
```
Prometheus target status restored to `health: up` (`up{job="app"} = 1`).

---

## 4. Engineering Decisions

1. **HTTP Health Check Probe**:
   - *Implementation*: Added `wget --quiet --tries=1 --spider http://localhost:8080/health` inside `docker-compose.yml` with `interval: 30s`, `timeout: 3s`, and `retries: 3`.
   - *Reliability Impact*: Automates detection of application degradation before routing traffic and enables container orchestrators to restart unhealthy instances.

2. **Container Restart Policy**:
   - *Implementation*: Configured `restart: unless-stopped` across app, Prometheus, and Grafana containers.
   - *Reliability Impact*: Prevents service downtime caused by unexpected daemon restarts or transient process crashes.

3. **Metrics Integration & Granular Scrape Interval**:
   - *Implementation*: Configured Prometheus scrape interval to `5s` (`prometheus/prometheus.yml`) and instrumented `prom-client` middleware in `app/metrics.js`.
   - *Reliability Impact*: High-resolution metric sampling allows fast detection of latency spikes or error rate surges within seconds of a deployment.

4. **Zero-Config Dashboard Provisioning**:
   - *Implementation*: Configured explicit datasource `uid: prometheus` in `grafana/provisioning/datasources/datasource.yml` matching Grafana dashboard JSON targets.
   - *Reliability Impact*: Guarantees instant observability upon stack startup without requiring manual UI configuration during incident response.

---

## 5. Readiness Review & Verdict

### Strengths
- Fully containerized microservice stack with automated CI validation (`npm test`, `npm run lint`, `docker build`).
- Complete golden-signal telemetry coverage with automated Grafana dashboard provisioning.
- Built-in container health check probe enabling fast automated failure detection.
- Proven operational recovery through live container rollback.

### Unresolved Risks
1. **Single Instance Point of Failure (SPOF)**:
   - *Risk*: The application runs as a single container without horizontal redundancy or load balancing.
   - *Impact*: Node crash or host maintenance will cause 100% service outage.
2. **Plaintext Credentials in Configuration**:
   - *Risk*: Admin credentials (`GF_SECURITY_ADMIN_PASSWORD=admin`) are exposed in `docker-compose.yml`.
   - *Impact*: Security vulnerability if deployed in shared or public environments without secrets injection.
3. **No Automated Alerting Rules**:
   - *Risk*: Prometheus is collecting metrics but no Alertmanager rules or webhook notifications (Slack/PagerDuty) are defined.
   - *Impact*: SRE teams rely on manual dashboard inspection rather than proactive paging.
4. **Transient Local Metrics Storage**:
   - *Risk*: Prometheus data is stored in ephemeral container layers without persistent volumes.
   - *Impact*: Historical metric data is lost when containers are recreated.

### Readiness Verdict
**VERDICT: READY WITH CAVEATS**

*Justification*: The service exhibits strong software quality, passing CI test coverage, active golden-signal observability, container health checking, and verified rollback recovery. It is approved for staging and non-mission-critical environments. Production release to high-traffic environments requires adding horizontal replication, secret management, persistent volume storage, and automated alerting.

---

## 6. Cloud Release Mapping

This local Docker Compose architecture maps to Cloud Native GCP infrastructure as follows:

| Local Compose Layer | Cloud Release Equivalent | Migration Action & Tooling |
|---------------------|--------------------------|----------------------------|
| **Image Building** | GCP Artifact Registry | `docker build -t us-central1-docker.pkg.dev/$PROJECT_ID/prl/app:v1.0.0 .`<br>`docker push us-central1-docker.pkg.dev/$PROJECT_ID/prl/app:v1.0.0` |
| **App Runtime** | GCP Cloud Run / GKE | `gcloud run deploy prl-app --image=... --port=8080 --min-instances=2 --max-instances=10` |
| **Health Checks** | Cloud Run Liveness / Readiness Probes | HTTP Probe configured against `/health` (initial delay 5s, timeout 3s). |
| **Monitoring & Metrics** | Google Cloud Monitoring / Managed Prometheus | Deploy OpenTelemetry Collector or Prometheus Sidecar to ingest `/metrics` into Cloud Monitoring dashboard. |
| **Rollback Execution** | Cloud Run Traffic Splitting / GKE Rollout Undo | `gcloud run services update-traffic prl-app --to-revisions=prl-app-00001-v100=100`<br>or `kubectl rollout undo deployment/prl-app` |
