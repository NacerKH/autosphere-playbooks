# HAProxy Playbooks - Quick Reference

## 🚀 Fast Commands

### Instant Diagnostics
```bash
# Quick health check (30 seconds)
ansible-playbook test-haproxy-environment.yml

# Full diagnostic + auto-fix
ansible-playbook diagnose-haproxy.yml
```

### Common Fixes (< 60 seconds)
```bash
# Fix backend down
ansible-playbook fix-haproxy-backend.yml -e "target_backend=nova-api"

# Clean connections
ansible-playbook cleanup-haproxy-connections.yml

# Restart HAProxy
ansible-playbook restart-haproxy.yml
```

## 📋 Playbook Cheat Sheet

| Symptom | Command | Time |
|---------|---------|------|
| Backend DOWN | `fix-haproxy-backend.yml` | 30s |
| High connections | `cleanup-haproxy-connections.yml` | 20s |
| SSL expired | `renew-haproxy-ssl.yml` | 90s |
| Slow health checks | `optimize-haproxy-healthchecks.yml` | 60s |
| HAProxy hung | `restart-haproxy.yml` | 45s |
| Unknown issue | `diagnose-haproxy.yml` | 60s |

## 🎯 Target Selection

```bash
# All control nodes (default)
ansible-playbook <playbook>.yml

# Specific node
ansible-playbook <playbook>.yml -e "target_host=control01"

# Multiple nodes
ansible-playbook <playbook>.yml --limit "control01,control02"
```

## 🔧 Variable Overrides

### Backend Fix
```bash
-e "target_backend=nova-api"           # Specific backend
-e "backend_action=drain-enable"       # Graceful (default)
-e "backend_action=disable-enable"     # Hard reset
-e "backend_action=restart-backend"    # Restart service
```

### Connection Cleanup
```bash
-e "action=soft"        # Gentle drain (default)
-e "action=aggressive"  # Force close idle
-e "action=restart"     # Full HAProxy restart
```

### Health Check Tuning
```bash
-e "hc_interval=3s"        # Check every 3 seconds
-e "hc_rise=2"             # 2 successes = UP
-e "hc_fall=3"             # 3 failures = DOWN
-e "timeout_server=90s"    # 90 second timeout
```

### SSL Renewal
```bash
-e "force_cert_renewal=true"     # Force renewal
-e "cert_expiry_days=30"         # Renew if < 30 days
```

### Diagnostic Mode
```bash
-e "auto_remediate=false"        # Report only
-e "auto_remediate=true"         # Auto-fix (default)
-e "severity_threshold=high"     # Only fix high/critical
```

## 📊 Pre-Flight Checks

```bash
# Before any operation:
ansible-playbook test-haproxy-environment.yml

# Verify specific node:
ansible-playbook test-haproxy-environment.yml -e "target_host=control01"

# Check inventory:
ansible-inventory --list -i inventory.yml
```

## 🔍 Monitoring During Execution

### Terminal 1: Run Playbook
```bash
ansible-playbook restart-haproxy.yml -v
```

### Terminal 2: Watch HAProxy Logs
```bash
# On control node:
docker logs -f haproxy

# Remote:
ssh control01 'docker logs -f haproxy'
```

### Terminal 3: Monitor Backends
```bash
# Backend status:
watch -n 1 'echo "show stat" | docker exec -i haproxy socat stdio /run/haproxy/admin.sock | grep BACKEND'

# Connections:
watch -n 1 'docker exec haproxy ss -tan | grep -c ESTABLISHED'
```

### Terminal 4: API Health
```bash
# Keystone API:
watch -n 1 'curl -sk https://10.0.0.100:5000/v3 | jq -r ".version.id"'

# Nova API:
watch -n 1 'curl -sk https://10.0.0.100:8774/ | jq -r ".version"'
```

## 🚨 Emergency Procedures

### HAProxy Completely Down
```bash
# 1. Check container
docker ps -a | grep haproxy

# 2. Start if stopped
docker start haproxy

# 3. If start fails, recreate (requires kolla-ansible)
kolla-ansible -i /etc/kolla/inventory deploy --tags haproxy
```

### All Backends DOWN
```bash
# 1. Diagnose first
ansible-playbook diagnose-haproxy.yml -e "auto_remediate=false"

# 2. Check backend services
docker ps | grep -E "(nova|neutron|glance|cinder|keystone)"

# 3. Fix backends
ansible-playbook fix-haproxy-backend.yml -e "target_backend=all"

# 4. If still down, restart HAProxy
ansible-playbook restart-haproxy.yml
```

### Certificate Expired (API Unreachable)
```bash
# Emergency renewal (bypasses expiry check)
ansible-playbook renew-haproxy-ssl.yml -e "force_cert_renewal=true"
```

## 📈 Performance Optimization

### Reduce Connection TIME_WAIT
```bash
ansible-playbook cleanup-haproxy-connections.yml -e "action=aggressive"
```

### Tune Health Checks for Speed
```bash
ansible-playbook optimize-haproxy-healthchecks.yml \
  -e "hc_interval=2s" \
  -e "hc_rise=2" \
  -e "hc_fall=2" \
  -e "timeout_server=30s"
```

### Tune Health Checks for Stability
```bash
ansible-playbook optimize-haproxy-healthchecks.yml \
  -e "hc_interval=10s" \
  -e "hc_rise=5" \
  -e "hc_fall=3" \
  -e "timeout_server=120s"
```

## 🔐 Security Considerations

### Run in Check Mode (Dry-Run)
```bash
ansible-playbook <playbook>.yml --check
```

### Limit to Specific Host
```bash
ansible-playbook restart-haproxy.yml --limit control01
```

### Require Manual Approval
```bash
ansible-playbook restart-haproxy.yml --step
```

### Review Changes Before Apply
```bash
ansible-playbook optimize-haproxy-healthchecks.yml --diff
```

## 📁 Log Locations

### Playbook Execution Logs
```bash
/var/log/autosphere/ansible-playbook.log
```

### Diagnostic Reports
```bash
/var/log/autosphere/haproxy_diagnostic_*.txt
```

### Remediation Logs
```bash
/var/log/autosphere/haproxy_remediation.log
```

### Certificate Renewals
```bash
/var/log/autosphere/certificate_renewal.log
```

### HAProxy Container Logs
```bash
docker logs haproxy --since 10m
```

### System Logs
```bash
grep autosphere-haproxy /var/log/syslog
```

## 🧪 Testing Playbooks

### Dry Run (No Changes)
```bash
ansible-playbook <playbook>.yml --check --diff
```

### Syntax Check
```bash
ansible-playbook <playbook>.yml --syntax-check
```

### Verbose Output
```bash
ansible-playbook <playbook>.yml -v    # Verbose
ansible-playbook <playbook>.yml -vv   # More verbose
ansible-playbook <playbook>.yml -vvv  # Very verbose
```

### Single Task Execution
```bash
ansible-playbook <playbook>.yml --start-at-task "Task name"
```

### Skip Tags
```bash
ansible-playbook restart-haproxy.yml --skip-tags "safety-check"
```

## 🔗 Integration Examples

### AWX/Tower Job Launch
```bash
# Using AWX CLI
awx jobs launch \
  --job-template "OpenStack-HAProxy-Backend-Fix" \
  --extra-vars '{"target_backend": "nova-api", "target_host": "control01"}'
```

### Curl to AutoSphere Webhook
```bash
curl -X POST http://localhost:8000/webhook/alert \
  -H "Content-Type: application/json" \
  -d '{
    "alert": "HAProxyBackendDown",
    "severity": "critical",
    "backend": "nova-api",
    "host": "control01"
  }'
```

### Prometheus Alertmanager
```yaml
route:
  receiver: autosphere-webhook
  routes:
    - match:
        alertname: HAProxyBackendDown
      receiver: autosphere-webhook

receivers:
  - name: autosphere-webhook
    webhook_configs:
      - url: 'http://autosphere:8000/webhook/alert'
```

## 📞 Troubleshooting

### Playbook Hangs
```bash
# Check SSH connectivity
ansible control -i inventory.yml -m ping

# Check if container is responsive
ansible control -i inventory.yml -m shell -a "docker ps"
```

### Permission Denied
```bash
# Verify SSH key access
ssh root@control01 'whoami'

# Test sudo
ansible control -i inventory.yml -m shell -a "sudo -n true" --become
```

### Container Not Found
```bash
# List all containers
ansible control -i inventory.yml -m shell -a "docker ps -a | grep haproxy"

# Verify container name
-e "haproxy_container=<actual-container-name>"
```

### Admin Socket Error
```bash
# Install socat
ansible control -i inventory.yml -m shell -a "docker exec haproxy apt-get update && apt-get install -y socat"
```

## 📚 Additional Resources

- Full documentation: [HAPROXY_README.md](./HAPROXY_README.md)
- Inventory setup: [inventory.yml](./inventory.yml)
- Environment test: `test-haproxy-environment.yml`
- AutoSphere integration: [AutoSphere Docs](../autosphereLangraph/)

---
**Quick Start**: `ansible-playbook test-haproxy-environment.yml && ansible-playbook diagnose-haproxy.yml`
