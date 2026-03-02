name: ultra-monitor
version: 2.1.0
maintainer: security-team@openclaw.org
description: AI-powered security monitoring and anomaly detection system
tags: [security, ai, automation, monitoring, threat-detection]
type: daemon
dependencies:
  - python3.9+
  - tensorflow>=2.8.0
  - scapy>=2.5.0
  - elasticsearch>=7.14.0
  - redis>=4.3.0
  - psutil>=5.9.0
required_env_vars:
  - ULTRAMONITOR_ES_HOST
  - ULTRAMONITOR_REDIS_HOST
  - ULTRAMONITOR_MODEL_PATH
  - ULTRAMONITOR_API_KEY
ports: [8080, 9090]
```

## Purpose

Ultra Monitor provides continuous AI-powered security monitoring for OpenClaw environments and attached infrastructure. It analyzes system logs, network traffic, and process behavior in real-time to detect anomalies, potential intrusions, and suspicious activities. The system uses pre-trained TensorFlow models combined with rule-based detection to provide multi-layered security coverage.

Real use cases:
- Detect brute-force attacks against SSH/RDP services within 60 seconds
- Identify data exfiltration attempts via unusual network flows
- Monitor for crypto-miner processes consuming excessive CPU
- Flag unauthorized database access patterns
- Alert on credential stuffing attempts from known malicious IPs
- Track lateral movement attempts within container networks
- Detect privilege escalation via sudo abuse patterns
- Monitor API endpoints for异常 request patterns

## Scope

Ultra Monitor operates with the following capabilities:

### Monitoring Commands
- `ultramonitor start --config=/path/to/config.yaml --daemon` - Start monitoring daemon
- `ultramonitor stop --graceful-timeout=30` - Stop with cleanup
- `ultramonitor status --detailed` - Show real-time status
- `ultramonitor analyze logs --type=auth --last=1h` - Historical log analysis
- `ultramonitor train --dataset=/data/events.parquet --epochs=50` - Retrain models

### Detection Commands
- `ultramonitor detect process --pid=12345 --profile=production` - Analyze specific process
- `ultramonitor detect network --interface=eth0 --threshold=0.85` - Live packet analysis
- `ultramonitor detect file --path=/etc/passwd --watch` - File integrity monitoring
- `ultramonitor detect user --user=root --behavior=anomalous` - User behavior analysis

### Alert Management
- `ultramonitor alerts list --severity=critical --since=10m` - List recent alerts
- `ultramonitor alerts acknowledge --id=ALERT-001 --comment="investigating"` - Acknowledge alert
- `ultramonitor alerts suppress --rule-id=xss-attempt --duration=1h` - Temporary suppression
- `ultramonitor alerts export --format=json --output=/tmp/alerts.json` - Export alerts

### System Commands
- `ultramonitor healthcheck --full` - Complete system health verification
- `ultramonitor model status` - Check AI model performance
- `ultramonitor config validate --strict` - Validate configuration
- `ultramonitor database cleanup --keep-days=90` - Elasticsearch maintenance

## Work Process

### 1. Initialization Phase
```
Load configuration from /etc/ultramonitor/config.yaml
Initialize Elasticsearch client (ULTRAMONITOR_ES_HOST)
Connect to Redis cache (ULTRAMONITOR_REDIS_HOST)
Load ML models from ULTRAMONITOR_MODEL_PATH
Establish baseline patterns from last 7 days of historical data
Start API server on port 8080 for integrations
```

### 2. Data Collection (Every 5 seconds)
```
Collect system metrics via psutil (CPU, memory, disk, network)
Read auth logs: /var/log/auth.log, /var/log/secure
Monitor process creation events via psutil.process_iter()
Capture network packets on monitored interfaces (with scapy)
Query auditd logs if available
```

### 3. Anomaly Detection Pipeline
```
For each event:
  Extract features (timestamp, user, process, network, etc.)
  Encode categorical features (user, binary, command)
  Normalize numerical features
  Run through ensemble of models:
    - Isolation Forest for rare events
    - LSTM autoencoder for sequential patterns
    - Random Forest for classification
  Combine scores (weighted average)
  If score > threshold (default 0.75):
    Generate alert with:
      - Severity (critical/high/medium/low)
      - Confidence score
      - Affected assets
      - MITRE ATT&CK mapping if available
      - Recommended actions
  Store in Elasticsearch index "ultramonitor-alerts-YYYY.MM"
  Push to Redis pub/sub for real-time dashboard
  Trigger webhook if configured
```

### 4. Alert Escalation
```
Critical alerts (>0.95):
  Immediate Slack/Teams notification
  Auto-create ticket in configured ticketing system
  Optionally trigger automated response (block IP, kill process)

High alerts (0.85-0.95):
  Hourly digest email
  Dashboard highlight

Medium/Low: Dashboard only
```

### 5. Continuous Learning
```
Nightly at 2 AM:
  Fetch last 24h of labeled data (confirmed true/false positives)
  Retrain models if drift detected (>5% accuracy drop)
  A/B test new models with 10% traffic
  Auto-promote if F1-score improves >2%
```

## Golden Rules

1. **Never run detection as root**: Ultra Monitor runs as dedicated user 'ultramonitor' with minimal privileges. All data collection uses sudoers rules for specific commands only.

2. **Always encrypt data in transit**: ES and Redis connections use TLS. API requires ULTRAMONITOR_API_KEY in headers. No plaintext credentials in configs.

3. **Retain logs for exactly 90 days**: Automatic Elasticsearch ILM policy. Manual export required for longer retention.

4. **Thresholds must be validated**: Before deploying new thresholds, run `ultramonitor detect dry-run --dataset=validation.json` and verify <5% false positive rate.

5. **Never suppress critical alerts**: Suppression rules cannot target severity=critical. Any attempt logs warning and fails.

6. **Model updates require approval**: Auto-training disabled by default. Set `auto_train: true` only in staging, then manually promote models using `ultramonitor model promote --version=2.1.0`.

7. **Always backup before config changes**: `ultramonitor config export --output=/backup/config-$(date +%Y%m%d).yaml` before any modification.

8. **False positives must be labeled**: Every suppressed alert requires `--reason` and `--ticket-id`. Unlabeled suppressions audit weekly.

9. **Monitor the monitor**: Ultra Monitor's own metrics (CPU, memory, alert latency) sent to separate ES cluster. Alert if latency >5s or errors >1%.

10. **Incident response coordination**: All automated actions (IP blocks, process kills) require pre-approval and create audit logs with operator attribution.

## Examples

### Example 1: Detect brute-force attack
```
$ ultramonitor detect logs --type=auth --last=30m --threshold=0.8

Alert ID: ALERT-20260302-154321
Severity: CRITICAL (0.94)
Source: auth.log
Pattern: Multiple failed SSH logins from 192.168.1.105
Details:
  - 45 failed attempts from IP 192.168.1.105 in 3 minutes
  - Targeted users: root, admin, deploy
  - Last success: none
  - MITRE: T1110.003 (Brute Force: Password Spraying)
Recommended Actions:
  1. Block IP at firewall: iptables -A INPUT -s 192.168.1.105 -j DROP
  2. Enable fail2ban for SSH
  3. Review user accounts for unauthorized access
```

### Example 2: Analyze suspicious process
```
$ ultramonitor detect process --pid=2847 --profile=production

Process: /tmp/.X11-unix/x eyes (PID 2847)
User: www-data (unexpected)
Anomaly Score: 0.89
Anomalies:
  - Process executed from /tmp (unusual location)
  - Network connection to 45.76.120.87:443 (known C2 server)
  - Cryptocurrency miner signature detected (xmrig)
  - CPU usage 450% sustained (4 cores)
Decision: MALICIOUS (confidence 0.91)
Action Taken: Process killed, IP blocked, container isolated
```

### Example 3: Configure new detection rule
```
$ cat > /etc/ultramonitor/rules/custom-api-abuse.yaml << 'EOF'
rule:
  name: "Excessive API Rate Limit Violations"
  description: "Detect clients exceeding API rate limits >10x"
  condition:
    all:
      - event.dataset: "nginx.access"
      - event.action: "rate_limited"
      - client.ip: "not_in: whitelist_ips"
  detection:
    model: "count_aggregation"
    timeframe: "5m"
    threshold: 10
  severity: "high"
  mitre: ["T1499"]
  response:
    - "block_ip:duration=1h"
    - "notify:slack=#security-team"
EOF

$ ultramonitor config reload
Rule loaded: custom-api-abuse (ID: rule-202503001)
```

### Example 4: Training a custom model
```
$ ultramonitor train \
  --dataset=/data/security-events-2026Q1.parquet \
  --model-type=lstm-autoencoder \
  --validation-split=0.2 \
  --epochs=100 \
  --batch-size=256 \
  --early-stopping

Training initiated (Job ID: train-20260302-1550)
Dataset: 2.1M events, 47 features
Architecture: LSTM(128) → Dropout(0.2) → Dense(47)
Expected duration: 45 minutes
Monitor: ultramonitor jobs status --job-id=train-20260302-1550
```

### Example 5: Investigating an alert
```
$ ultramonitor alerts list --severity=high --unacknowledged

ID              SEVERITY  SCORE  AGE     TITLE
ALERT-001       HIGH      0.87   12m     Suspicious DB query from dev server
ALERT-002       HIGH      0.82   45m     Impossible travel: user john.doe

$ ultramonitor alerts investigate --id=ALERT-001 --timeline

Timeline for ALERT-001:
14:32:01 Query executed from 10.0.2.15 (dev-web-01)
  SELECT * FROM users WHERE email LIKE '%@target.com'
14:32:02 Account 'developer' used (last login 3 days ago from different IP)
14:32:03 Data: 15,432 rows returned
14:32:05 Same query from 10.0.2.16 (dev-api-01)
Context:
  - No approved change ticket for this query
  - Data accessed: PII (email addresses)
  - Occurred outside business hours
IOC: IP 10.0.2.15, account 'developer'
Recommendation: Immediate credential rotation, access review
```

### Example 6: Generate compliance report
```
$ ultramonitor report compliance \
  --start-date=2026-02-01 \
  --end-date=2026-02-28 \
  --framework=iso27001 \
  --output=/reports/iso27001-feb2026.pdf

Generating compliance report...
Analyzing: 12,450 alerts, 1.2M events
Controls evaluated: 114
Pass rate: 96.5% (110/114)
Exceptions:
  - A.12.4.1: 3 false negatives in log review (medium risk)
  - A.14.2.1: 2 unpatched CVEs in non-production (low risk)
Report saved: /reports/iso27001-feb2026.pdf (4.2 MB)
```

## Rollback Commands

### Rollback model update
```
# If new model causes increased false positives:
$ ultramonitor model list --versions
Model versions:
  2.1.0 (active, deployed 2026-03-02 10:00)
  2.0.9 (previous, deployed 2026-02-28 14:00)

$ ultramonitor model rollback --version=2.0.9 --reason="FP rate 23%"
Rolling back from 2.1.0 to 2.0.9...
Model 2.0.9 loaded, reverting inference pipeline...
Validation: F1-score 0.94 (previous baseline)
Rollback complete. Alert volume stabilized.
```

### Rollback configuration change
```
# If config causes service disruption:
$ ultramonitor config history
Config versions:
  45 (active, /etc/ultramonitor/config.yaml)
  44 (2026-03-02 09:30, /backup/config-20260302-0930.yaml)
  43 (2026-03-01 22:00, /backup/config-20260301-2200.yaml)

$ ultramonitor config restore --backup=/backup/config-20260302-0930.yaml
Restoring configuration from 2026-03-02 09:30...
Validating schema... OK
Testing rule compilation... OK
Applying configuration...
Service reloaded without restart
Configuration restored successfully
```

### Rollback alert suppression
```
# Accidently suppressed critical rule:
$ ultramonitor alerts suppression list
Active suppressions:
  ID: sup-001, Rule: ssh-brute-force, Until: 2026-03-03 10:00, Reason: maintenance

$ ultramonitor alerts suppression remove --id=sup-001
Suppression sup-001 removed
Rule 'ssh-brute-force' re-enabled (was suppressed 2h 15m)
```

### Rollback automated response
```
# Blocked legitimate IP by mistake:
$ ultramonitor firewall list-blocked
Blocked IPs:
  192.168.1.200 (reason: ALERT-005, blocked 14:22)
  10.0.5.89 (reason: ALERT-012, blocked 15:45)

$ ultramonitor firewall unblock --ip=192.168.1.200 --reason="false positive, user approval #7890"
IP 192.168.1.200 unblocked from all firewalls
Ticket #7890 updated with unblock action
```

### Rollback database cleanup
```
# Deleted alerts needed for investigation:
$ ultramonitor database restore \
  --index=ultramonitor-alerts-2026.02 \
  --date=2026-02-15 \
  --output=/restore/feb15-alerts.json

Restoring from Elasticsearch snapshot 'ultramonitor-snap-20260228'...
Found 1,247 alerts for 2026-02-15
Restored to /restore/feb15-alerts.json
Re-indexed into read-only cluster 'ultramonitor-forensics'
```

### Full system rollback
```
# Emergency: revert complete deployment:
$ ultramonitor system rollback --to-version=2.0.9 --components=all

Rolling back Ultra Monitor 2.1.0 → 2.0.9
Components:
  ✓ ML models (2.0.9)
  ✓ Configuration (/backup/config-20260228.yaml)
  ✓ Detection rules (/backup/rules-20260228/)
  ✓ Elasticsearch indices (snapshot restore)
  ✓ Redis cache (flush + repopulate from backup)
Service restart required? No (hot reload)
Rollback complete. System operating on 2.0.9.
```

## Dependencies & Requirements

### Runtime Dependencies
```
Python 3.9+ with packages:
  tensorflow==2.8.0 (or tensorflow-gpu for GPU acceleration)
  torch==1.11.0 (optional, for hybrid models)
  scapy==2.5.0
  elasticsearch==7.14.0
  redis==4.3.0
  psutil==5.9.0
  pyyaml==6.0
  pandas==1.4.0
  scikit-learn==1.1.0
  prometheus-client==0.16.0
```

### System Requirements
- 4+ CPU cores (8+ recommended)
- 16GB RAM minimum (32GB for large deployments)
- 100GB SSD for Elasticsearch data
- Network interfaces in promiscuous mode for packet capture
- Access to auth logs (/var/log/auth.log, /var/log/secure)

### Elasticsearch Requirements
```
Index templates:
  - ultramonitor-alerts-* (sharding: 1, replicas: 1)
  - ultramonitor-events-* (sharding: 3, replicas: 2)
  - ultramonitor-metrics-*
ILM policy: 90 days hot → delete
Snapshot repository: s3://ultramonitor-backups or NFS
```

### Redis Requirements
```
Version: 6.2+
Memory: 4GB minimum
Persistence: AOF everysec
Eviction: allkeys-lru
Keyspace notifications: enabled (notify-keyspace-events Ex)
```

### Environment Variables
```bash
export ULTRAMONITOR_ES_HOST="https://es-cluster.example.com:9200"
export ULTRAMONITOR_ES_CERT="/etc/ultramonitor/certs/es.pem"
export ULTRAMONITOR_ES_USER="ultramonitor"
export ULTRAMONITOR_ES_PASSWORD_FILE="/etc/ultramonitor/secrets/es-pass"

export ULTRAMONITOR_REDIS_HOST="redis-cluster.example.com:6379"
export ULTRAMONITOR_REDIS_PASSWORD_FILE="/etc/ultramonitor/secrets/redis-pass"
export ULTRAMONITOR_REDIS_SSL="true"

export ULTRAMONITOR_MODEL_PATH="/var/lib/ultramonitor/models/production"
export ULTRAMONITOR_API_KEY="rotated:master-key-here"
export ULTRAMONITOR_WEBHOOK_URL="https://slack.com/api/..."

export ULTRAMONITOR_LOG_LEVEL="INFO"
export ULTRAMONITOR_MAX_EVENT_RATE="10000"  # events/second
export ULTRAMONITOR_ALERT_COOLDOWN="300"   # seconds
```

## Verification Steps

### Post-installation verification
```
1. Check service is running:
   $ systemctl status ultramonitor
   $ ultramonitor healthcheck --full

2. Verify data collection:
   $ ultramonitor events count --last=5m
   Should return non-zero event count

3. Test detection:
   $ ultramonitor detect test --scenario=ssh-brute
   Should generate test alert (severity low)

4. Verify alert pipeline:
   $ ultramonitor alerts list --last=10m
   Should see test alert

5. Check dashboard connectivity:
   $ curl -H "X-API-Key: $ULTRAMONITOR_API_KEY" http://localhost:8080/api/v1/health
   Should return {"status": "healthy"}

6. Verify Elasticsearch indexing:
   $ curl -k -u ultramonitor:$PASS $ES_HOST/_cat/indices/ultramonitor-*
   Should see indices with document count

7. Test webhook (if configured):
   $ ultramonitor alerts trigger --test --webhook=slack
   Should receive test message in Slack
```

### Continuous verification
```
# Add to monitoring:
- ultramonitor_events_total (Prometheus)
- ultramonitor_alerts_generated_total (by severity)
- ultramonitor_processing_latency_seconds (p95 < 0.5s)
- ultramonitor_model_accuracy (if ground truth available)
- ultramonitor_errors_total

# Alert on:
- ultramonitor_processing_errors > 10/minute
- ultramonitor_events_dropped_total > 0
- Elasticsearch latency > 1s
- Model accuracy drop > 5%
```

## Troubleshooting

### Issue: High CPU usage (>90%) for >5 minutes
```bash
# Check model inference bottleneck:
$ ultramonitor model benchmark --duration=60
If inference time per event > 10ms, consider:
  - Scale horizontally (add more ultramonitor instances)
  - Use smaller model (ultramonitor model set --size=small)
  - Enable GPU acceleration (install tensorflow-gpu)

# Check event rate:
$ ultramonitor stats | grep events_per_second
If > 10,000 eps, increase sampling or filter sources:
$ ultramonitor config set --key=collection.sample_rate --value=0.5
```

### Issue: No alerts generated for known malicious activity
```bash
# Check threshold:
$ ultramonitor config get detection.threshold
If too high (0.9+), lower to 0.75.

# Verify model loaded:
$ ultramonitor model status
Should show "loaded: true" and "version: X.Y.Z"

# Check feature extraction:
$ ultramonitor debug event --raw='{"event": "test"}' --verbose
Ensure features extracted correctly.

# Verify training recency:
$ ultramonitor model list
If model >30 days old, retrain:
$ ultramonitor train --dataset=/data/labeled-events.parquet
```

### Issue: Elasticsearch indexing lag (>60 seconds)
```bash
# Check ES health:
$ curl -k -u user:pass $ES_HOST/_cluster/health?pretty
Status must be "green". If "yellow/red", add nodes or replicas.

# Check queue size:
$ ultramonitor metrics | grep es_queue
If queue > 10,000, increase batch size or number of workers:
$ ultramonitor config set --key=es.batch_size --value=5000

# Verify TLS/certificates:
$ openssl s_client -connect es-host:9200 -showcerts
Check certificate expiry and trust chain.
```

### Issue: False positives flood (>100/hour)
```bash
# Identify top false positive rules:
$ ultramonitor alerts metrics --group-by=rule_id --last=24h | sort -n
Top rules: rule-ssh-brute (45%), rule-api-anomaly (32%)

# Temporary suppression (with ticket):
$ ultramonitor alerts suppress --rule-id=rule-ssh-brute --duration=6h --ticket=SEC-1234

# Tune threshold specifically:
$ ultramonitor config set --path=rules.rule-ssh-brute.threshold --value=0.85

# Add whitelist:
$ ultramonitor whitelist add --type=ip --value=10.0.0.0/8 --reason="Internal network"
```

### Issue: Cannot connect to Redis
```bash
# Test connection manually:
$ redis-cli -h redis-host -p 6379 ping
Should return PONG.

# Check SSL mode:
$ ultramonitor config get redis.ssl
If true but Redis not configured for SSL, set false.

# Verify password:
$ cat /etc/ultramonitor/secrets/redis-pass | head -1
Test with:
$ redis-cli -h host -a 'password' ping

# Check Redis memory:
$ redis-cli -h host info memory | grep used_memory_human
If >80% used, increase memory or add Redis cluster.
```

### Issue: Model loading fails on startup
```bash
# Check model files:
$ ls -lh $ULTRAMONITOR_MODEL_PATH/
Should contain: model.savedmodel/ or model.pt

# Verify TensorFlow version:
$ python3 -c "import tensorflow as tf; print(tf.__version__)"
Must match training version (2.8.0). Upgrade/downgrade if mismatch.

# Check GPU availability (optional):
$ python3 -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
If empty and GPU expected, install CUDA drivers.

# Fallback to CPU:
$ ultramonitor config set --key=model.use_gpu --value=false
$ systemctl restart ultramonitor
```

### Issue: Auth log parsing fails
```bash
# Check log paths:
$ ultramonitor config get collection.log_paths
Verify paths exist: /var/log/auth.log, /var/log/secure

# Permissions:
$ ls -l /var/log/auth.log
Ensure 'ultramonitor' user can read (add to 'adm' group or set ACL):
$ setfacl -m u:ultramonitor:r /var/log/auth.log

# SELinux/AppArmor:
$ аудитctl -l  # if using auditd
Ensure ultramonitor can read audit logs.

# Test parsing manually:
$ ultramonitor debug parse-auth --file=/var/log/auth.log --lines=10
Should output JSON events.
```

### Issue: Containerized deployment connectivity
```bash
# Check Docker network:
$ docker network ls | grep ultramonitor
Container needs network access to ES and Redis hosts.

# Verify environment variables in container:
$ docker exec ultramonitor env | grep ULTRAMONITOR
All required vars must be set.

# Check time synchronization:
$ docker exec ultramonitor date
System time must be synced (NTP). Events require accurate timestamps.

# View container logs:
$ docker logs ultramonitor --tail=100
Look for stack traces or import errors.
```

### Issue: Alert not appearing in dashboard
```bash
# Verify alert stored in ES:
$ curl -k -u user:pass "$ES_HOST/ultramonitor-alerts-*/_search?q=ALERT-001" | jq .
Should find document.

# Check Redis pub/sub:
$ redis-cli -h redis-host psubscribe 'ultramonitor:alerts:*'
Generate test alert, should see message.

# Dashboard API:
$ curl -H "X-API-Key: $ULTRAMONITOR_API_KEY" http://localhost:8080/api/v1/alerts/recent
```markdown
---
name: ultra-monitor
description: AI-powered security monitoring and threat detection system
version: 2.4.1
author: Security Team <security@openclaw.io>
tags:
  - security
  - ai
  - automation
  - monitoring
  - threat-detection
type: daemon
category: security
dependencies:
  - python>=3.9
  - tensorflow>=2.12.0
  - scapy>=2.5.0
  - elasticsearch>=8.10.0
  - redis>=5.0.0
  - auditd>=3.0
  - fam>=0.5
system_dependencies:
  - auditd
  - libpcap-dev
  - python3-dev
  - redis-server
  - elasticsearch
ports:
  - 9400/tcp (API)
  - 9401/tcp (Metrics)
  - 9402/tcp (WebSocket)
environment:
  ULTRAMONITOR_MODEL_PATH: /var/lib/ultra-monitor/models/
  ULTRAMONITOR_CONFIG: /etc/ultra-monitor/config.yaml
  ULTRAMONITOR_LOG_LEVEL: INFO
  ULTRAMONITOR_THRESHOLD: 0.85
  ULTRAMONITOR_ES_HOST: localhost:9200
  ULTRAMONITOR_REDIS_HOST: localhost:6379
  ULTRAMONITOR_AUDIT_BACKLOG: 8192
  ULTRAMONITOR_INTERFACE: eth0
---

# Ultra Monitor

AI-powered security monitoring daemon that analyzes system events, network traffic, and audit logs in real-time to detect threats, anomalies, and compliance violations.

## Purpose

Ultra Monitor serves as an automated security operations center (SOC) that continuously analyzes:

1. **Intrusion Detection**: Real-time analysis of network packets using ML models trained on attack patterns (DDoS, port scans, exploit attempts)
2. **Insider Threat Detection**: File access monitoring (via auditd/fam) to detect unusual data exfiltration or unauthorized file modifications
3. **Compliance Auditing**: Automatic detection of PCI-DSS, HIPAA, SOC2 violations through log analysis
4. **Anomaly Detection**: Behavioral analysis of system calls, process trees, and user activity to identify compromised accounts
5. **Threat Intelligence**: Integration with OpenCTI/MISP to correlate local events with known IOCs
6. **Automated Response**: Configurable playbooks for containment (block IP, kill process, disable user)

Real use cases:
- Detect ransomware by monitoring mass file encryption patterns (>100 files/min with .encrypted/.locked extensions)
- Identify data exfiltration via DNS tunneling (base64-like queries >50 chars to external domains)
- Alert on privilege escalation attempts (sudo abuse, setuid binary modifications)
- Monitor for cryptocurrency mining (high CPU + specific process names + network connections to mining pools)

## Scope

### Commands

#### `ultra-monitor start [--config=<path>] [--interface=<iface>] [--daemon]`
Start the monitoring engine. Options:
- `--config`: Custom configuration file (default: /etc/ultra-monitor/config.yaml)
- `--interface`: Network interface to monitor (default: reads from config)
- `--daemon`: Run as background daemon (default: foreground)
- `--replay=<pcap_file>`: Replay offline traffic for analysis
- `--model=<model_name>`: Use specific ML model (default: ensemble_v3)

#### `ultra-monitor stop [--force] [--graceful=<seconds>]`
Stop monitoring. Options:
- `--force`: Kill immediately without cleanup
- `--graceful`: Wait N seconds for active analyses to complete

#### `ultra-monitor status [--json] [--metrics]`
Show monitoring status. Options:
- `--json`: Output machine-readable format
- `--metrics`: Show performance metrics (events/sec, false positive rate, CPU/memory usage)

#### `ultra-monitor analyze <log_file|pcap_file> [--format=<type>] [--output=<path>]`
Analyze historical data offline. Options:
- `--format`: Input format (audit, pcap, json, syslog)
- `--output`: Write report to file (default: stdout)
- `--time-range=<start>-<end>`: Filter by timestamp
- `--severity=<level>`: Minimum severity to report (high, medium, low)

#### `ultra-monitor rule [list|add|remove|test]`
Manage detection rules. Options:
- `list`: Show all active rules with IDs
- `add <rule_yaml>`: Add new detection rule
- `remove <rule_id>`: Disable rule by ID
- `test <rule_id> [--sample=<event>]`: Test rule against sample event

#### `ultra-monitor train [--dataset=<path>] [--model=<name>] [--epochs=<N>]`
Retrain ML models. Options:
- `--dataset`: Path to labeled training data (JSON/CSV)
- `--model`: Model type (lstm, cnn, transformer, ensemble)
- `--epochs`: Training iterations (default: 100)
- `--validation-split=<0.0-1.0>`: Fraction for validation

#### `ultra-monitor alert [list|ack|assign]`
Manage security alerts. Options:
- `list [--severity=<level>] [--status=open|closed|assigned]`: Filter alerts
- `ack <alert_id> [--comment=<text>]`: Acknowledge alert
- `assign <alert_id> <user|team>`: Assign to responder
- `export [--format=json|csv|splunk]`: Export alert database

#### `ultra-monitor report [--type=<report>] [--period=<range>] [--format=<fmt>]`
Generate security reports. Options:
- `--type`: daily, weekly, monthly, compliance, incident
- `--period`: Time range (e.g., "2024-01-01 to 2024-01-07")
- `--format`: html, pdf, json, csv
- `--dest`: Output file or email address

#### `ultra-monitor whitelist [add|remove|list] [--type=ip|process|user|file]`
Manage safe lists. Options:
- `add <entry> [--reason=<text>] [--expires=<date>]`
- `remove <entry>`
- `list [--type=<type>] [--expired]`

#### `ultra-monitor model [list|load|unload|benchmark]`
Manage ML models. Options:
- `list`: Show loaded models with performance stats
- `load <model_name> [--priority=<1-10>]`
- `unload <model_name>`
- `benchmark <model_name> [--dataset=<path>]`: Test accuracy/throughput

#### `ultra-monitor capture [start|stop|status] [--filter=<bpf>] [--rotate=<size>]`
Packet capture controls. Options:
- `start`: Begin capturing to /var/lib/ultra-monitor/pcap/
- `stop`: Terminate capture
- `status`: Show capture stats (packets, dropped, buffer)
- `--filter`: BPF filter expression (default: from config)
- `--rotate`: Rotate files at size (e.g., "100MB")

## Detailed Work Process

### Initial Setup

1. Install dependencies: `apt install auditd redis-server elasticsearch`
2. Deploy config: `cp configs/ultra-monitor.yaml /etc/ultra-monitor/config.yaml`
3. Edit configuration:
   ```yaml
   monitoring:
     network_interface: eth0
     auditd_enabled: true
     process_monitoring: true
   models:
     ensemble:
       anomaly_detector: models/anomaly_v3.h5
       intrusion_classifier: models/intrusion_v2.pb
       behavioral_analyzer: models/behavior_lstm.h5
   thresholds:
     anomaly_score: 0.85
     intrusion_confidence: 0.90
   outputs:
     elasticsearch: localhost:9200
     redis: localhost:6379
     slack_webhook: https://hooks.slack.com/...
   ```
4. Initialize database: `ultra-monitor db init`
5. Start service: `systemctl enable --now ultra-monitor`

### Normal Operation

1. Monitor status: `ultra-monitor status --metrics`
   Expected output:
   ```
   Status: RUNNING (PID 2451)
   Uptime: 3d 14h 22m
   Events processed: 12,456,789
   Detection rate: 2,145/sec
   Alerts (today): 47 (critical: 3, high: 12, medium: 22, low: 10)
   ML Models loaded: 3 (ensemble_v3)
   Memory usage: 1.2GB / 8GB
   Latest alert: 2024-01-15 14:32:11 - POTENTIAL_RANSOMWARE (id: ALT-789xyz)
   ```

2. Review alerts: `ultra-monitor alert list --severity=high --status=open`
   Sample output:
   ```
   ID          SEVERITY   TIMESTAMP               RULE                    DESCRIPTION
   ALT-789xyz  CRITICAL   2024-01-15 14:32:11     RANSOMWARE_ENCRYPTION   Mass encryption detected: /home/user (234 files in 45s)
   ALT-789ww  HIGH       2024-01-15 13:15:42     PRIVILEGE_ESCALATION    sudo -u root executed by user jsmith from 10.0.1.45
   ALT-789vv  HIGH       2024-01-15 11:03:19     DATA_EXFIL_HTTP         Unusual HTTP POST to external IP 45.76.120.33 (2.4GB transferred)
   ```

3. Investigate alert: `ultra-monitor analyze --format=json /var/lib/ultra-monitor/alerts/ALT-789xyz.json`
   Provides full context: affected processes, network connections, file modifications timeline.

4. Acknowledge and assign: `ultra-monitor ack ALT-789xyz --comment="Investigating with user"` && `ultra-monitor assign ALT-789xyz @soc-team`

5. Tune false positives: `ultra-monitor whitelist add 10.0.1.45 --type=ip --reason="Trusted dev workstation" --expires=2024-12-31`

### Incident Response

1. When critical alert triggers:
   - Check real-time dashboard: `curl http://localhost:9400/api/v1/alerts?severity=critical`
   - Get affected host details: `ultra-monitor host-info <hostname>`
   - Isolate host via playbook: `ultra-monitor playbook run isolate_host --target=<hostname>`
   - Collect evidence: `ultra-monitor capture start --filter="host <IP>"`
   - Generate incident report after resolution: `ultra-monitor report --type=incident --period="last 24h" --format=html`

### Model Retraining

Weekly: evaluate model performance. If false positive rate >15%:
1. Collect misclassified events: `ultra-monitor export-false-positives > fp_dataset.json`
2. Label new data and add to training set
3. Retrain: `ultra-monitor train --dataset=training_data_v4.csv --model=ensemble --epochs=150`
4. Validate: `ultra-monitor model benchmark ensemble_v4 --dataset=validation_set.csv`
5. Deploy if F1-score >0.95: `ultra-monitor model load ensemble_v4 --priority=10`

## Golden Rules

1. **Never disable auditd**: Ultra Monitor requires auditd for file/directory monitoring. If you must disable, switch to `file_monitoring: false` in config first, then restart.
2. **Threshold discipline**: Never set anomaly threshold below 0.7 without documented justification. Lower thresholds cause alert fatigue and SOC burnout.
3. **Model validation**: Before loading any new model, always benchmark against recent labeled data. Unvalidated models can blind you to real threats.
4. **Evidence preservation**: On critical alerts, immediately start packet capture (`ultra-monitor capture start`). Do not stop until incident is resolved.
5. **Whitelist hygiene**: Review expired whitelists weekly. Stale whitelists create security gaps: `ultra-monitor whitelist list --expired`.
6. **Rollback readiness**: Before config changes, backup: `cp /etc/ultra-monitor/config.yaml /etc/ultra-monitor/config.yaml.bak-$(date +%s)`. Never edit config without backup.
7. **Alert lifecycle**: All alerts must be acknowledged within 24h and assigned. Unassigned alerts >72h are escalated to SOC manager.
8. **Rule testing**: New rules must be tested against historical data before deployment: `ultra-monitor rule test <rule_id> --sample=events/last_week.json`.
9. **Model drift monitoring**: Track false positive rate daily. >20% increase triggers automatic model performance review.
10. **Security of the monitor**: Ultra Monitor itself must run as dedicated user `ultramon` with minimal privileges. Never run as root.

## Examples

### 1. Detect ransomware attack
```bash
# Setup: Enable file monitoring in config.yaml
$ ultra-monitor rule add rules/ransomware_encryption.yaml
$ ultra-monitor start --daemon

# Later, investigation triggered:
$ ultra-monitor alert list --severity=critical
ID: ALT-abc123  SEVERITY: CRITICAL  RULE: RANSOMWARE_ENCRYPTION
Description: User 'admin' encrypted 342 files in /var/www in 52 seconds

$ ultra-monitor analyze /var/lib/ultra-monitor/events/2024-01-15/abc123.json
{
  "timeline": [
    {"time": "14:32:01", "process": "bash", "cmd": "python encrypt.py --dir=/var/www --key=ABCDEF"},
    {"time": "14:32:02", "files_encrypted": 1},
    {"time": "14:32:10", "files_encrypted": 100},
    ...
  ],
  "network_connections": [
    {"dest": "185.220.101.48:443", "proto": "HTTPS", "data_out": "2.1MB"}
  ]
}

$ ultra-monitor whitelist add 185.220.101.48 --type=ip --reason="CDN backup service (temporary)" --expires=2024-01-20
```

### 2. Investigate privilege escalation
```bash
# Real-time monitoring query
$ curl -s http://localhost:9400/api/v1/search?q=rule:PRIVILEGE_ESCALATION&limit=10 | jq .

# Get detailed process tree for alert ALT-def456
$ ultra-monitor process-tree ALT-def456
Process tree:
PID 1234 (sshd) -> PID 5678 (bash) -> PID 9012 (sudo) -> PID 3456 (bash as root)

# Check what commands run as root:
$ ultra-monitor audit-query --user=jsmith --rule=setuid --last=1h
Timestamp: 2024-01-15 10:15:33
Executable: /usr/bin/sudo
Arguments: -u root /usr/bin/apt update
Exit code: 0

# Response: disable user and rotate credentials
$ ultra-monitor user disable jsmith --reason="Privilege abuse - incident ALT-def456"
$ ultra-monitor credential rotate --user=jsmith --affected=all
```

### 3. Monitor for data exfiltration via DNS
```bash
# Custom rule for DNS tunneling detection
$ cat > rules/dns_tunnel.yaml << 'EOF'
rule:
  id: DNS_TUNNELING
  name: "Potential Data Exfiltration via DNS"
  condition:
    all:
      - event: dns_query
      - query_length: >100
      - entropy(query_name): >4.5
      - not query_domain in whitelist.domains
  response:
    - alert: high
    - capture_pcap: true
    - block_ip: src_ip (temporary, 15m)
EOF

$ ultra-monitor rule add rules/dns_tunnel.yaml
$ ultra-monitor rule test DNS_TUNNELING --sample='{"event":"dns_query","query_name":"aGVsbG8gd29ybGQ=.example.com","src_ip":"10.5.2.12"}'
Test result: MATCH (confidence: 0.94)
```

### 4. Generate compliance report
```bash
$ ultra-monitor report --type=compliance --period="2024-01-01 to 2024-01-31" --format=pdf --dest="/reports/jan2024_compliance.pdf"
Generating PCI-DSS compliance report...
- Requirement 10 (audit trails): PASSED (100% coverage)
- Requirement 11 (vulnerability scanning): 3 failures (see section 4.2)
- Requirement 3 (access controls): PASSED
Report saved to /reports/jan2024_compliance.pdf (2.4MB)
```

### 5. Retrain model after false positives
```bash
# Export recent false positives
$ ultra-monitor alert export --status=closed --label=false_positive --format=json > false_positives.json

# Label true negatives and add to training set
$ ultra-monitor dataset augment false_positives.json --label=benign --output=training_set_v4.csv

# Retrain with new data
$ ultra-monitor train --dataset=training_set_v4.csv --model=ensemble_v4 --epochs=200 --validation-split=0.2
Training ensemble_v4...
Epoch 100/200: loss=0.234, accuracy=0.892, val_f1=0.87
Epoch 200/200: loss=0.156, accuracy=0.934, val_f1=0.93

# Benchmark before deploying
$ ultra-monitor model benchmark ensemble_v4 --dataset=test_set_2024Q1.csv
Model: ensemble_v4
Precision: 0.94, Recall: 0.91, F1-score: 0.925
Throughput: 15,234 events/sec
Previous model (ensemble_v3): F1=0.902

# Deploy new model
$ ultra-monitor model load ensemble_v4 --priority=10
Model ensemble_v4 loaded with priority 10 (active)
```

### 6. Real-time dashboard and API
```bash
# Check health endpoint
$ curl -s http://localhost:9400/health | jq .
{
  "status": "healthy",
  "uptime": 258732,
  "models_loaded": 3,
  "last_event": "2024-01-15T14:35:12Z",
  "alerts_today": 47
}

# Query recent high-severity events
$ curl -s "http://localhost:9400/api/v1/events?severity=high&last=1h&limit=50" | jq '.events[] | {time, rule, src_ip, description}'

# WebSocket stream for real-time alerts
$ wscat -c ws://localhost:9402/stream?level=critical
Connected (press CTRL+C to quit)
> {"type":"alert","id":"ALT-xyz987","timestamp":"2024-01-15T14:36:00Z","severity":"critical","rule":"DATA_EXFIL_HTTP","src_ip":"10.1.5.23","description":"2.3GB uploaded to external IP 103.21.5.12"}
```

## Rollback Commands

### Immediate Stop
```bash
# Graceful shutdown (waits for active analyses)
$ ultra-monitor stop --graceful=30

# Force kill if hung
$ ultra-monitor stop --force
# Verify stopped
$ ultra-monitor status
# Should return: "Status: STOPPED" or exit code 3

# If daemonized via systemd
$ systemctl stop ultra-monitor
$ systemctl disable ultra-monitor  # Prevent auto-start
```

### Configuration Rollback
```bash
# Restore previous config
$ cp /etc/ultra-monitor/config.yaml.bak-<timestamp> /etc/ultra-monitor/config.yaml
$ chmod 640 /etc/ultra-monitor/config.yaml
$ chown ultramon:ultramon /etc/ultra-monitor/config.yaml

# Reload config without restart (if supported)
$ ultra-monitor reload
# Or restart
$ systemctl restart ultra-monitor
```

### Model Rollback
```bash
# List loaded models and their load times
$ ultra-monitor model list
MODEL          VERSION  LOADED_AT          ACCURACY  ACTIVE
ensemble_v3    3.2.1    2024-01-10 08:00   0.892     yes
ensemble_v2    3.1.4    2024-01-03 10:30   0.876     no

# Unload problematic model
$ ultra-monitor model unload ensemble_v4

# Load known-good model
$ ultra-monitor model load ensemble_v3 --priority=10

# Verify fallback
$ ultra-monitor model list | grep ensemble_v3
ensemble_v3    3.2.1    active: true

# If models directory corrupted, restore from backup
$ cp /backup/ultra-monitor/models/ensemble_v3.h5 /var/lib/ultra-monitor/models/
$ chown ultramon:ultramon /var/lib/ultra-monitor/models/*.h5
```

### Rule Rollback
```bash
# Remove recently added rule causing false positives
$ ultra-monitor rule list --added-after="2024-01-15" | grep MALWARE_DETECTION
ID: MALWARE_DETECTION_2  Added: 2024-01-15 10:30:00

$ ultra-monitor rule remove MALWARE_DETECTION_2
Rule MALWARE_DETECTION_2 removed (will be disabled on config reload)

# Immediate reload to apply removals
$ ultra-monitor rule reload

# If rule broke parsing, restore rules directory from backup
$ cp -r /backup/ultra-monitor/rules/* /etc/ultra-monitor/rules/
$ ultra-monitor rule reload
```

### Whitelist Rollback
```bash
# List recent whitelist additions
$ ultra-monitor whitelist list --added-after="2024-01-14" --type=ip
Entry: 192.168.1.100  Added: 2024-01-14 16:20:00  Reason: "temp dev access"  Expires: never

# Remove suspect whitelist entry
$ ultra-monitor whitelist remove 192.168.1.100

# Bulk rollback: restore whitelist from backup
$ ultra-monitor whitelist clear --confirm  # Removes all entries (dangerous!)
$ ultra-monitor whitelist import /backup/ultra-monitor/whitelist_20240114.json

# Verify whitelist restored
$ ultra-monitor whitelist list | wc -l  # Should match backup count
```

### Data Rollback (Elasticsearch)
```bash
# If false positives flooded ES, restore from snapshot
# First, check available snapshots
$ curl -s http://localhost:9200/_snapshot/ultramonitor_backup/_all?pretty | jq '.snapshots[].snapshot'

# Restore specific snapshot (e.g., pre-incident)
$ curl -X POST http://localhost:9200/_snapshot/ultramonitor_backup/snapshot_20240115_0100/restore -H 'Content-Type: application/json' -d'
{
  "indices": "ultramonitor-events,ultramonitor-alerts",
  "rename_pattern": "ultramonitor-(.+)",
  "rename_replacement": "ultramonitor_restored_$1"
}'

# If ES corrupted and no snapshot, clear indices and restart collection
$ ultra-monitor stop
$ curl -X DELETE http://localhost:9200/ultramonitor-*
$ ultra-monitor start --reindex=true  # Rebuild from raw logs

# Verify reindexing
$ ultra-monitor status --metrics | grep "Events indexed"
```

### Capture Rollback
```bash
# Stop active packet capture
$ ultra-monitor capture status
Status: CAPTURING (PID 3245, 2.4GB captured, 0 dropped)
Interface: eth0, Filter: (default)

$ ultra-monitor capture stop
Capture stopped. Files saved in /var/lib/ultra-monitor/pcap/

# If capture filling disk, rotate/cleanup manually
$ ls -lh /var/lib/ultra-monitor/pcap/
-rw-r--r-- 1 ultramon ultramon 1.2G Jan 15 14:00 capture_20240115_1400.pcap
-rw-r--r-- 1 ultramon ultramon 950M Jan 15 14:15 capture_20240115_1415.pcap

# Remove old captures (keep last 24h)
$ find /var/lib/ultra-monitor/pcap/ -mtime +1 -delete
$ chown ultramon:ultramon /var/lib/ultra-monitor/pcap/*

# Adjust rotation policy in config.yaml
# monitoring:
#   packet_capture:
#     rotate_size: "500MB"
#     max_files: 48
```

### Complete System Rollback
```bash
# Emergency: Stop all monitoring
$ systemctl stop ultra-monitor
$ systemctl disable ultra-monitor

# Disable auditd (Ultra Monitor's primary data source)
$ systemctl stop auditd
$ systemctl disable auditd

# Restore entire configuration from yesterday's backup
$ tar -xzf /backup/ultra-monitor/backup_20240114.tar.gz -C /
$ chown -R ultramon:ultramon /var/lib/ultra-monitor/
$ chmod 640 /etc/ultra-monitor/config.yaml

# Restart auditd with original rules
$ systemctl enable --now auditd
$ augenrules --load

# Restart Ultra Monitor cleanly
$ systemctl enable --now ultra-monitor
$ ultra-monitor status
Status: RUNNING, Models: ensemble_v3, Events processed: 0 (fresh start)

# Verify no residual state
$ ultra-monitor alert list
# Should be empty or only show post-restart alerts

# If issues persist, check logs
$ journalctl -u ultra-monitor -n 100 --no-pager
$ tail -100 /var/log/ultra-monitor/error.log
```

## Troubleshooting

### High CPU/Memory Usage
```bash
# Check which model consuming resources
$ ultra-monitor model list --metrics
MODEL               CPU%  MEM%  QPS
ensemble_v3         45    12    8500
intrusion_v2        28    8     12000

# Temporarily unload heavy model
$ ultra-monitor model unload ensemble_v3
# Restart to apply: systemctl restart ultra-monitor

# If memory leak, check model load cycles
$ grep "model load" /var/log/ultra-monitor/debug.log | tail -20
# Repeated loads indicate config churn. Fix: enable model persistence in config.
```

### False Positives Flood
```bash
# Temporary suppress rule
$ ultra-monitor rule disable MALWARE_DETECTION --ttl=1h
# Rule disabled for 1 hour, auto-reenables

# Identify top false-positive rules
$ ultra-monitor alert list --status=closed --label=false_positive --group-by=rule | head -10
RULE                              FALSE_POSITIVES  PRECISION
MALWARE_DETECTION                 234             0.62
PORT_SCAN_DETECTION               189             0.71

# Adjust threshold for specific rule
$ ultra-monitor rule update MALWARE_DETECTION --threshold=0.92
# (Requires rule reload: ultra-monitor rule reload)

# Add context-based whitelist
$ ultra-monitor whitelist add 10.1.5.0/24 --type=ip_range --reason="Internal dev network" --context="malware_detection"
```

### No Events Processed
```bash
# 1. Check auditd is running and has rules
$ systemctl status auditd
$ auditctl -l
# Should see rules for /etc/ultra-monitor/audit.rules

# 2. Verify Ultra Monitor can read audit logs
$ ls -la /var/log/audit/audit.log
$ sudo -u ultramon tail -5 /var/log/audit/audit.log

# 3. Check interface monitoring
$ ultra-monitor capture status
# If "No packets captured", verify interface:
$ ip link show eth0
# And BPF filter:
$ ultra-monitor config get monitoring.packet_filter

# 4. Test with manual event injection
$ echo '{"event":"test","severity":"low"}' | nc localhost 9400
$ ultra-monitor alert list --last=1m
# Should show test event

# 5. Restart pipeline
$ ultra-monitor stop --graceful=10
$ sleep 5
$ ultra-monitor start --daemon
```

### Model Loading Fails
```bash
# Check model file integrity
$ ls -lh /var/lib/ultra-monitor/models/ensemble_v3.h5
$ file /var/lib/ultra-monitor/models/ensemble_v3.h5
# Should be: HDF5 data

# Verify TensorFlow can load it (as ultramon user)
$ sudo -u ultramon python3 -c "import tensorflow as tf; m=tf.keras.models.load_model('/var/lib/ultra-monitor/models/ensemble_v3.h5'); print(m.summary())"
# If CUDA errors but not needed, force CPU:
$ ultra-monitor config set models.ensemble.use_gpu false
$ ultra-monitor restart

# If corrupted, restore from backup or re-download
$ cp /backup/models/ensemble_v3.h5 /var/lib/ultra-monitor/models/
$ chown ultramon:ultramon /var/lib/ultra-monitor/models/ensemble_v3.h5
```

### Elasticsearch Connection Failed
```bash
# Check ES health
$ curl -s http://localhost:9200/_cluster/health?pretty
# Status should be: green or yellow

# Verify credentials in config match ES setup
$ grep elasticsearch /etc/ultra-monitor/config.yaml
# Should show: username/password if security enabled

# Test connectivity as ultramon user
$ sudo -u ultramon curl -u ultramon:password http://localhost:9200/_cat/indices?v

# If ES down, Ultra Monitor buffers to Redis (if configured)
$ redis-cli llen ultra-monitor:buffer
# > 12345 (events waiting)

# Restart ES first, then Ultra Monitor
$ systemctl restart elasticsearch
$ sleep 10
$ ultra-monitor restart

# If buffer overflowed, check for data loss
$ ultra-monitor status --metrics | grep "Events dropped"
# If >0, increase buffer sizes in config
```

### Web Dashboard Unavailable
```bash
# Check API health
$ curl -s http://localhost:9400/health | jq .status
# Should be "healthy"

# Check process listening
$ ss -tlnp | grep 9400
# LISTEN 0 128 *:9400 *:* users:(("python3",pid=2451,fd=5))

# If not listening, check Ultra Monitor logs
$ grep -i "dashboard\|api\|9400" /var/log/ultra-monitor/*.log

# Restart API component only (if supported)
$ ultra-monitor api restart
# Or full restart
$ systemctl restart ultra-monitor

# Check firewall
$ sudo iptables -L INPUT -n | grep 9400
# If blocked: sudo iptables -A INPUT -p tcp --dport 9400 -j ACCEPT
```

### Alert Fatigue
```bash
# Suppress noisy rule temporarily
$ ultra-monitor rule suppress MALWARE_DETECTION --reason="False positive burst" --duration=2h

# Tune threshold dynamically based on recent FP rate
$ ultra-monitor auto-tune MALWARE_DETECTION --target_precision=0.95

# Create rule exception for specific context
$ ultra-monitor rule add-exception MALWARE_DETECTION --condition='process_name in ["backup_tool", "encrypt_legit"] && user in ["admin", "backup_user"]'

# Implement rate limiting alerts
$ ultra-monitor config set alerting.rate_limit.max_alerts_per_rule_per_hour 20
$ ultra-monitor config set alerting.rate_limit.suppression_window "5m"
$ ultra-monitor reload
```
```