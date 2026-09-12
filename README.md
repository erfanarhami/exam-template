# Monitoring Stack Deployment Documentation

## 1. Project Overview

This document describes the deployment, configuration, validation, and troubleshooting process of a monitoring stack based on:

* Prometheus
* Grafana
* Node Exporter
* Ansible automation

The goal of this implementation is to provide a complete monitoring solution capable of collecting infrastructure metrics and visualizing system performance through Grafana dashboards.

The deployment was performed using Ansible to ensure repeatability and infrastructure automation.

---

# 2. Architecture Overview

## Components

| Component     | Purpose                                     |
| ------------- | ------------------------------------------- |
| Prometheus    | Metrics collection and time-series database |
| Node Exporter | Linux host metrics exporter                 |
| Grafana       | Metrics visualization and dashboarding      |
| Ansible       | Configuration management and automation     |

## Data Flow

```
Node Exporter
      |
      |
      v
 Prometheus
      |
      |
      v
 Grafana
      |
      |
      v
 Monitoring Dashboard
```

---

# 3. Infrastructure Layout

## Monitoring Node

The monitoring services are deployed on:

```
mon-1
```

Running services:

```
grafana-server
prometheus
node_exporter
```

---

# 4. Deployment Process

## 4.1 Ansible Inventory Validation

The inventory was verified before deployment.

Example:

```bash
ansible -i inventory monitoring --list-hosts
```

Expected result:

```
mon-1
```

---

# 5. Grafana Deployment

## Service Validation

Grafana service status was checked:

```bash
systemctl status grafana-server
```

Final status:

```
Active: active (running)
```

Grafana restart after configuration changes:

```bash
systemctl restart grafana-server
```

---

# 6. Grafana Provisioning Configuration

Grafana provisioning directory:

```
/etc/grafana/provisioning/
```

Structure:

```
provisioning/
├── dashboards/
├── datasources/
├── plugins/
└── alerting/
```

Permissions were corrected to ensure Grafana could read provisioning files:

```bash
chown -R grafana:grafana /etc/grafana/provisioning
```

---

# 7. Prometheus Datasource Configuration

Datasource provisioning file:

```
/etc/grafana/provisioning/datasources/prometheus.yml
```

Configuration:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    uid: prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9090
    isDefault: true
    editable: false
```

After provisioning:

Datasource verification:

```bash
curl -u admin:admin \
http://localhost:3000/api/datasources
```

Result:

```json
[
 {
  "name":"Prometheus",
  "type":"prometheus",
  "url":"http://localhost:9090"
 }
]
```

---

# 8. Dashboard Provisioning

Dashboard provisioning path:

```
/etc/grafana/provisioning/dashboards/
```

Dashboard deployed:

```
System CPU & Memory
```

Dashboard UID:

```
system-cpu-memory
```

Validation:

```bash
curl -u admin:admin \
http://localhost:3000/api/search?type=dash-db
```

Result:

```
System CPU & Memory
```

---

# 9. Dashboard Panels

The dashboard contains system monitoring panels.

## CPU Usage Panel

PromQL query:

```promql
100 - (
 avg by (instance)
 (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
 )
 * 100
)
```

Purpose:

Shows CPU utilization percentage per instance.

---

## Memory Usage Panel

PromQL query:

```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

Purpose:

Shows used memory percentage.

---

# 10. Prometheus Validation

Prometheus API was tested directly.

## Check Prometheus Health

```bash
curl http://localhost:9090/api/v1/query?query=up
```

Expected result:

```json
{
 "status":"success"
}
```

Available targets:

```
prometheus
node_exporter
```

Both returned:

```
value: 1
```

Meaning:

* Prometheus is healthy
* Node Exporter metrics are available

---

# 11. Grafana Query Validation

Grafana datasource query API was tested:

```bash
curl -u admin:admin \
-X POST \
http://localhost:3000/api/ds/query
```

Test query:

```promql
up
```

Result:

```
Prometheus target = UP
Node exporter target = UP
```

---

# 12. Troubleshooting and Resolution

## 12.1 Grafana Datasource Provisioning Issue

During validation, the Prometheus datasource was recreated through provisioning.

The datasource was initially read-only because it was managed by provisioning.

Verification:

```bash
curl -u admin:admin \
http://localhost:3000/api/datasources/uid/prometheus
```

After correcting provisioning state and restarting Grafana, the datasource was correctly recreated.

---

## 12.2 Temporary No Data Issue

During dashboard validation, a temporary "No Data" situation was observed.

The investigation confirmed:

* Prometheus was collecting metrics correctly.
* Grafana datasource configuration was correct.
* Dashboard queries were valid.

The issue was caused by a time synchronization mismatch between the client environment and monitoring environment.

After correcting system time synchronization and refreshing the Grafana time range, metrics appeared correctly.

---

# 13. Important Validation Commands

## Grafana Service

```bash
systemctl status grafana-server
```

---

## Grafana Datasources

```bash
curl -u admin:admin \
http://localhost:3000/api/datasources
```

---

## Grafana Dashboards

```bash
curl -u admin:admin \
http://localhost:3000/api/search?type=dash-db
```

---

## Prometheus Metrics

```bash
curl \
http://localhost:9090/api/v1/query?query=up
```

---

## CPU Metrics

```bash
curl \
"http://localhost:9090/api/v1/query?query=node_cpu_seconds_total"
```

---

# 14. Final System Status

| Component              | Status       |
| ---------------------- | ------------ |
| Grafana Service        | ✅ Running    |
| Prometheus Service     | ✅ Running    |
| Node Exporter          | ✅ Running    |
| Grafana Provisioning   | ✅ Completed  |
| Prometheus Datasource  | ✅ Configured |
| Dashboard Provisioning | ✅ Completed  |
| Metrics Collection     | ✅ Working    |
| CPU Monitoring         | ✅ Working    |
| Memory Monitoring      | ✅ Working    |

---

# 15. Conclusion

The monitoring stack was successfully deployed and validated.

The final implementation provides:

* Automated deployment using Ansible
* Prometheus-based metrics collection
* Grafana visualization
* Provisioned datasource configuration
* Automated dashboard deployment
* Infrastructure monitoring capabilities

The system is ready for production-style monitoring and further expansion with additional exporters, alerts, and dashboards.
