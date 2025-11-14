# HAProxy Remediation Playbooks for Kolla-Ansible OpenStack

Comprehensive Ansible playbooks for diagnosing and remediating HAProxy issues in Kolla-Ansible OpenStack deployments running on Docker (2025).

## 📋 Overview

These playbooks are designed for **AutoSphere SRE Agent** integration and can also be run manually for HAProxy troubleshooting in production Kolla-Ansible environments.

## 🎯 Playbook Inventory

| Playbook | Purpose | Risk | Time | Auto-Approve |
|----------|---------|------|------|--------------|
| `diagnose-haproxy.yml` | Master diagnostic & orchestration | Variable | 60-180s | ✅ Yes (report-only) |
| `restart-haproxy.yml` | Graceful HAProxy restart | Medium | 45-60s | ⚠️ Requires HA |
| `fix-haproxy-backend.yml` | Backend health recovery | Low | 30-45s | ✅ Yes |
| `renew-haproxy-ssl.yml` | SSL certificate renewal | Medium | 60-90s | ⚠️ Manual review |
| `cleanup-haproxy-connections.yml` | Connection pool cleanup | Low | 20-30s | ✅ Yes |
| `optimize-haproxy-healthchecks.yml` | Health check tuning | Low | 45-60s | ✅ Yes |

## 🔧 Prerequisites

### System Requirements
- **Kolla-Ansible**: 2025.1+ (Docker-based deployment)
- **Ansible**: 2.9+
- **Python**: 3.8+
- **Docker**: 20.10+
- **Access**: Root/sudo on control nodes

### Python Dependencies
```bash
pip install ansible docker-py pyOpenSSL
```

### HAProxy Container
```bash
# Verify HAProxy container is running
docker ps | grep haproxy

# Verify socat socket access (required for stats)
docker exec haproxy which socat
```

### Network Access
- HAProxy stats socket: `/run/haproxy/admin.sock`
- API VIP: `10.0.0.100` (adjust in playbooks)
- Stats page: `http://<control-node>:1984`

## 🚀 Quick Start

### 1. Master Diagnostic (Recommended First Step)
```bash
# Full diagnostic with auto-remediation
ansible-playbook diagnose-haproxy.yml

# Report-only mode (no fixes)
ansible-playbook diagnose-haproxy.yml -e "auto_remediate=false"

# Target specific host
ansible-playbook diagnose-haproxy.yml -e "target_host=control01"
```

### 2. Specific Remediation Scenarios

#### Backend Down
```bash
# Auto-fix specific backend
ansible-playbook fix-haproxy-backend.yml \
  -e "target_backend=nova-api" \
  -e "backend_action=drain-enable"

# Fix all backends
ansible-playbook fix-haproxy-backend.yml -e "target_backend=nova-api"
```

#### High Connection Count
```bash
# Soft cleanup (recommended)
ansible-playbook cleanup-haproxy-connections.yml -e "action=soft"

# Aggressive cleanup
ansible-playbook cleanup-haproxy-connections.yml -e "action=aggressive"

# Full restart (last resort)
ansible-playbook cleanup-haproxy-connections.yml -e "action=restart"
```

#### SSL Certificate Issues
```bash
# Renew certificate (auto-detects expiry)
ansible-playbook renew-haproxy-ssl.yml

# Force renewal
ansible-playbook renew-haproxy-ssl.yml -e "force_cert_renewal=true"
```

#### Health Check Optimization
```bash
# Optimize all backends
ansible-playbook optimize-haproxy-healthchecks.yml

# Optimize specific backend
ansible-playbook optimize-haproxy-healthchecks.yml -e "target_backend=keystone"

# Custom health check settings
ansible-playbook optimize-haproxy-healthchecks.yml \
  -e "hc_interval=3s" \
  -e "hc_rise=2" \
  -e "hc_fall=3" \
  -e "timeout_server=90s"
```

#### Full HAProxy Restart
```bash
# Graceful restart (HA setup required)
ansible-playbook restart-haproxy.yml

# Force restart on specific node
ansible-playbook restart-haproxy.yml -e "target_host=control02"
```

## 🔗 AutoSphere Integration

### AWX Job Template Configuration

```yaml
# Example AWX Job Template for AutoSphere
- name: OpenStack-HAProxy-Diagnose
  job_type: run
  inventory: production-openstack
  project: autosphere-playbooks
  playbook: diagnose-haproxy.yml
  extra_vars:
    auto_remediate: true
    severity_threshold: high
  
- name: OpenStack-HAProxy-Restart
  job_type: run
  inventory: production-openstack
  project: autosphere-playbooks
  playbook: restart-haproxy.yml
  limit: "{{ node_name }}"

- name: OpenStack-HAProxy-Backend-Fix
  job_type: run
  inventory: production-openstack
  project: autosphere-playbooks
  playbook: fix-haproxy-backend.yml
  extra_vars:
    target_backend: "{{ backend_name }}"
    backend_action: "drain-enable"
```

### AutoSphere Agent Runbook Mapping

```yaml
# healer/config/runbooks/haproxy_backend_down.yaml
id: haproxy_backend_down
name: "Fix HAProxy Backend Down"
description: "Drain and re-enable HAProxy backend servers"
applicable_alerts:
  - "HAProxyBackendDown"
  - "BackendHealthCheckFailed"
  - "HAProxyHighLatency"
awx_job_template: "OpenStack-HAProxy-Backend-Fix"
risk_level: low
estimated_duration: 45
auto_approve: true

parameters:
  target_backend:
    type: string
    required: true
    description: "Backend name (e.g., nova-api, neutron-server)"
  
  backend_action:
    type: string
    default: "drain-enable"
    enum: ["drain-enable", "disable-enable", "restart-backend"]

prerequisites:
  - condition: "HAProxy container is running"
  - condition: "Backend is configured in HAProxy"

validation:
  post_checks:
    - metric: 'haproxy_backend_active{backend="{{ target_backend }}"}'
      operator: ">="
      threshold: 1
      timeout: 45
```

### Alert Examples for AutoSphere

```yaml
# Prometheus alerting rules
groups:
  - name: haproxy_alerts
    interval: 30s
    rules:
      - alert: HAProxyBackendDown
        expr: haproxy_backend_up == 0
        for: 2m
        labels:
          severity: critical
          component: loadbalancer
          autosphere_priority: fast_path
        annotations:
          summary: "HAProxy backend {{ $labels.backend }} is DOWN"
          description: "Backend down on {{ $labels.instance }}"
          runbook: "haproxy_backend_down"
          
      - alert: HAProxyHighConnections
        expr: haproxy_current_sessions > 2000
        for: 5m
        labels:
          severity: warning
          component: loadbalancer
          autosphere_priority: fast_path
        annotations:
          summary: "High connection count on HAProxy"
          description: "{{ $value }} connections on {{ $labels.instance }}"
          runbook: "haproxy_connection_cleanup"
          
      - alert: HAProxyCertificateExpiring
        expr: (haproxy_ssl_certificate_expiry_seconds - time()) < 2592000
        for: 1h
        labels:
          severity: high
          component: loadbalancer
        annotations:
          summary: "HAProxy SSL certificate expiring soon"
          description: "Certificate expires in {{ $value | humanizeDuration }}"
          runbook: "haproxy_ssl_renewal"
```

## 📊 Monitoring Integration

### Grafana Dashboard Query Examples

```promql
# Backend health status
haproxy_backend_up{job="haproxy"}

# Connection count
haproxy_current_sessions{job="haproxy"}

# Backend response time
haproxy_backend_response_time_average_seconds{job="haproxy"}

# SSL certificate expiry
(haproxy_ssl_certificate_expiry_seconds - time()) / 86400
```

### OpenSearch Anomaly Detection

```json
{
  "name": "HAProxy Backend Latency Anomaly",
  "detector_type": "multi_entity",
  "indices": ["haproxy-metrics-*"],
  "feature_attributes": [
    {
      "feature_name": "backend_latency",
      "aggregation_query": {
        "avg_latency": {
          "avg": {
            "field": "haproxy_backend_response_time_seconds"
          }
        }
      }
    }
  ],
  "category_field": "backend_name",
  "detection_interval": { "period": { "interval": 5, "unit": "Minutes" } }
}
```

## 🛡️ Safety Features

### Pre-Flight Checks
All playbooks include:
- Container existence validation
- Service health verification
- Configuration backup
- HA setup detection (for restarts)

### Rollback Mechanisms
- Automatic configuration rollback on failure
- Container state snapshots (for restarts)
- Certificate backup before renewal
- Connection state preservation

### Blast Radius Control
```yaml
# High-risk operations require HA setup
- Single HAProxy node → Blocks restart (requires manual override)
- HA setup (2+ nodes) → Allows graceful restart
- Backend operations → No service interruption
```

## 📈 Success Metrics

### Expected Outcomes

| Issue | Playbook | Success Rate | MTTR |
|-------|----------|--------------|------|
| Backend Down | `fix-haproxy-backend.yml` | 95% | 30s |
| High Connections | `cleanup-haproxy-connections.yml` | 90% | 20s |
| SSL Expiry | `renew-haproxy-ssl.yml` | 98% | 90s |
| Health Check Failures | `optimize-haproxy-healthchecks.yml` | 92% | 60s |
| HAProxy Unresponsive | `restart-haproxy.yml` | 97% | 45s |

### Post-Execution Validation
All playbooks verify:
- ✅ HAProxy process is running
- ✅ Backends are UP
- ✅ API endpoints respond (200/300/401)
- ✅ Stats socket is accessible
- ✅ No configuration syntax errors

## 🔍 Troubleshooting

### Common Issues

#### 1. HAProxy stats socket not accessible
```bash
# Verify socat is installed in container
docker exec haproxy which socat

# Install if missing (Kolla-Ansible 2025.1+)
docker exec haproxy apt-get update && apt-get install -y socat
```

#### 2. Certificate renewal fails
```bash
# Check certificate permissions
docker exec haproxy ls -la /var/lib/kolla/config_files/src/haproxy.pem

# Verify OpenSSL is available
docker exec haproxy openssl version
```

#### 3. Backend still DOWN after fix
```bash
# Check backend service logs
docker logs <backend-service-container> --tail 100

# Verify backend service is listening
docker exec <backend-service> netstat -tlnp | grep <port>
```

#### 4. Single HAProxy restart blocked
```bash
# Override safety check (USE WITH CAUTION)
ansible-playbook restart-haproxy.yml -e "force_restart=true" --skip-tags "safety-check"
```

## 📝 Logging & Audit Trail

All playbooks log to:
- **Syslog**: `logger -t autosphere-haproxy`
- **Local file**: `/var/log/autosphere/haproxy_*.log`
- **Diagnostic reports**: `/var/log/autosphere/haproxy_diagnostic_<timestamp>.txt`

View logs:
```bash
# Syslog entries
grep autosphere-haproxy /var/log/syslog

# AutoSphere logs
tail -f /var/log/autosphere/haproxy_remediation.log

# Latest diagnostic report
ls -lt /var/log/autosphere/haproxy_diagnostic_*.txt | head -1
```

## 🎓 Best Practices

### 1. Always Start with Diagnostics
```bash
ansible-playbook diagnose-haproxy.yml -e "auto_remediate=false"
```

### 2. Use Dry-Run Mode (if available)
```bash
ansible-playbook <playbook>.yml --check
```

### 3. Target Specific Hosts in Production
```bash
ansible-playbook restart-haproxy.yml -e "target_host=control01" --limit control01
```

### 4. Monitor During Execution
```bash
# Terminal 1: Run playbook
ansible-playbook restart-haproxy.yml

# Terminal 2: Watch HAProxy logs
docker logs -f haproxy

# Terminal 3: Monitor API health
watch -n 1 'curl -sk https://10.0.0.100:5000/v3 | jq .'
```

### 5. Verify After Every Remediation
```bash
# Quick health check
ansible-playbook diagnose-haproxy.yml -e "auto_remediate=false" -e "diagnostic_mode=report-only"
```

## 🔗 Related Playbooks

See also:
- `restart-rabbitmq.yml` - RabbitMQ remediation (often correlated with HAProxy issues)
- `restart-nova-compute.yml` - Nova compute service restart
- `recover-neutron-ovs.yml` - Neutron OVS bridge recovery

## 📚 References

- [Kolla-Ansible Documentation](https://docs.openstack.org/kolla-ansible/)
- [HAProxy Configuration Best Practices](https://www.haproxy.org/doc/latest/)
- [OpenStack API Endpoints](https://docs.openstack.org/api-quick-start/)
- [AutoSphere Architecture](../autosphereLangraph/ARCHITECTURE.md)

## 📧 Support

For issues or questions:
- **AutoSphere**: Check LangSmith traces at https://smith.langchain.com
- **Playbooks**: Review execution logs in `/var/log/autosphere/`
- **HAProxy**: Check container logs with `docker logs haproxy`

---

**Version**: 1.0.0  
**Last Updated**: November 2025  
**Tested On**: Kolla-Ansible 2025.1 (Docker), OpenStack 2025.1 (Dalmatian)
