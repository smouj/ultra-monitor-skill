---
name: ultra-monitor
version: 2.1.0
maintainer: security-team@openclaw.org
description: Sistema de monitoreo de seguridad y detección de anomalías con IA
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
---

# Ultra Monitor

Sistema de monitoreo de seguridad con IA que proporciona monitoreo continuo con IA para entornos OpenClaw y la infraestructura conectada. Analiza registros del sistema, tráfico de red y comportamiento de procesos en tiempo real para detectar anomalías, posibles intrusiones y actividades sospechosas. El sistema utiliza modelos TensorFlow pre-entrenados combinados con detección basada en reglas para proporcionar cobertura de seguridad multicapa.

Casos de uso reales:
- Detectar ataques de fuerza bruta contra servicios SSH/RDP en menos de 60 segundos
- Identificar intentos de exfiltración de datos a través de flujos de red inusuales
- Monitorear procesos de minería de criptomonedas que consumen CPU excesivo
- Marcar patrones de acceso a bases de datos no autorizados
- Alertar sobre intentos de relleno de credenciales desde IPs conocidas como maliciosas
- Rastrear intentos de movimiento lateral dentro de redes de contenedores
- Detectar escalación de privilegios a través de patrones de abuso de sudo
- Monitorear endpoints de API para patrones de solicitud anómalos

## Alcance

Ultra Monitor opera con las siguientes capacidades:

### Comandos de Monitoreo
- `ultramonitor start --config=/path/to/config.yaml --daemon` - Iniciar demonio de monitoreo
- `ultramonitor stop --graceful-timeout=30` - Detener con limpieza
- `ultramonitor status --detailed` - Mostrar estado en tiempo real
- `ultramonitor analyze logs --type=auth --last=1h` - Análisis histórico de registros
- `ultramonitor train --dataset=/data/events.parquet --epochs=50` - Reentrenar modelos

### Comandos de Detección
- `ultramonitor detect process --pid=12345 --profile=production` - Analizar proceso específico
- `ultramonitor detect network --interface=eth0 --threshold=0.85` - Análisis de paquetes en vivo
- `ultramonitor detect file --path=/etc/passwd --watch` - Monitoreo de integridad de archivos
- `ultramonitor detect user --user=root --behavior=anomalous` - Análisis de comportamiento de usuario

### Gestión de Alertas
- `ultramonitor alerts list --severity=critical --since=10m` - Listar alertas recientes
- `ultramonitor alerts acknowledge --id=ALERT-001 --comment="investigating"` - Reconocer alerta
- `ultramonitor alerts suppress --rule-id=xss-attack --duration=1h` - Supresión temporal
- `ultramonitor alerts export --format=json --output=/tmp/alerts.json` - Exportar alertas

### Comandos del Sistema
- `ultramonitor healthcheck --full` - Verificación completa de salud del sistema
- `ultramonitor model status` - Verificar rendimiento del modelo de IA
- `ultramonitor config validate --strict` - Validar configuración
- `ultramonitor database cleanup --keep-days=90` - Mantenimiento de Elasticsearch

## Proceso de Trabajo

### 1. Fase de Inicialización
```
Cargar configuración desde /etc/ultramonitor/config.yaml
Inicializar cliente de Elasticsearch (ULTRAMONITOR_ES_HOST)
Conectar a caché Redis (ULTRAMONITOR_REDIS_HOST)
Cargar modelos de ML desde ULTRAMONITOR_MODEL_PATH
Establecer patrones de referencia de los últimos 7 días de datos históricos
Iniciar servidor API en puerto 8080 para integraciones
```

### 2. Recopilación de Datos (Cada 5 segundos)
```
Recolectar métricas del sistema via psutil (CPU, memoria, disco, red)
Leer registros de autenticación: /var/log/auth.log, /var/log/secure
Monitorear eventos de creación de procesos via psutil.process_iter()
Capturar paquetes de red en interfaces monitoreadas (con scapy)
Consultar registros de auditd si está disponible
```

### 3. Pipeline de Detección de Anomalías
```
Para cada evento:
  Extraer características (timestamp, usuario, proceso, red, etc.)
  Codificar características categóricas (usuario, binario, comando)
  Normalizar características numéricas
  Ejecutar a través de conjunto de modelos:
    - Isolation Forest para eventos raros
    - Autoencoder LSTM para patrones secuenciales
    - Random Forest para clasificación
  Combinar puntuaciones (promedio ponderado)
  Si puntuación > umbral (predeterminado 0.75):
    Generar alerta con:
      - Severidad (crítica/alta/media/baja)
      - Puntuación de confianza
      - Activos afectados
      - Mapeo MITRE ATT&CK si está disponible
      - Acciones recomendadas
  Almacenar en índice de Elasticsearch "ultramonitor-alerts-YYYY.MM"
  Enviar a pub/sub de Redis para dashboard en tiempo real
  Disparar webhook si está configurado
```

### 4. Escalamiento de Alertas
```
Alertas críticas (>0.95):
  Notificación inmediata por Slack/Teams
  Auto-crear ticket en el sistema de tickets configurado
  Opcionalmente disparar respuesta automatizada (bloquear IP, matar proceso)

Alertas altas (0.85-0.95):
  Resumen por correo cada hora
  Resaltar en dashboard

Medias/Bajas: Solo dashboard
```

### 5. Aprendizaje Continuo
```
Noche a las 2 AM:
  Obtener las últimas 24h de datos etiquetados (positivos verdaderos/falsos confirmados)
  Reentrenar modelos si se detecta deriva (>5% de caída en precisión)
  Hacer A/B test de nuevos modelos con 10% de tráfico
  Auto-promocionar si F1-score mejora >2%
```

## Reglas de Oro

1. **Nunca ejecutar detección como root**: Ultra Monitor se ejecuta como usuario dedicado 'ultramonitor' con privilegios mínimos. Toda la recopilación de datos usa reglas de sudoers para comandos específicos únicamente.

2. **Siempre cifrar datos en tránsito**: Conexiones a ES y Redis usan TLS. API requiere ULTRAMONITOR_API_KEY en encabezados. No credenciales en texto plano en configuraciones.

3. **Retener registros exactamente 90 días**: Política ILM automática de Elasticsearch. Exportación manual requerida para retención más larga.

4. **Los umbrales deben ser validados**: Antes de desplegar nuevos umbrales, ejecutar `ultramonitor detect dry-run --dataset=validation.json` y verificar tasa de falsos positivos <5%.

5. **Nunca suprimir alertas críticas**: Las reglas de supresión no pueden apuntar a severidad=crítica. Cualquier intento registra advertencia y falla.

6. **Las actualizaciones de modelo requieren aprobación**: Auto-entrenamiento deshabilitado por defecto. Establecer `auto_train: true` solo en staging, luego promover modelos manualmente usando `ultramonitor model promote --version=2.1.0`.

7. **Siempre hacer backup antes de cambios de configuración**: `ultramonitor config export --output=/backup/config-$(date +%Y%m%d).yaml` antes de cualquier modificación.

8. **Los falsos positivos deben ser etiquetados**: Toda alerta suprimida requiere `--reason` y `--ticket-id`. Supresiones no etiquetadas se auditan semanalmente.

9. **Monitorear el monitor**: Las propias métricas de Ultra Monitor (CPU, memoria, latencia de alerta) se envían a un clúster ES separado. Alertar si latencia >5s o errores >1%.

10. **Coordinación de respuesta a incidentes**: Todas las acciones automatizadas (bloqueos IP, kills de proceso) requieren pre-aprobación y crean registros de auditoría con atribución del operador.

## Ejemplos

### Ejemplo 1: Detectar ataque de fuerza bruta
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

### Ejemplo 2: Analizar proceso sospechoso
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

### Ejemplo 3: Configurar nueva regla de detección
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

### Ejemplo 4: Entrenar un modelo personalizado
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

### Ejemplo 5: Investigar una alerta
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

### Ejemplo 6: Generar reporte de cumplimiento
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

## Comandos de Rollback

### Reversión de actualización de modelo
```
# Si el nuevo modelo causa aumento de falsos positivos:
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

### Reversión de cambio de configuración
```
# Si la configuración causa interrupción del servicio:
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

### Reversión de supresión de alertas
```
# Accidentalmente se suprimió regla crítica:
$ ultramonitor alerts suppression list
Active suppressions:
  ID: sup-001, Rule: ssh-brute-force, Until: 2026-03-03 10:00, Reason: maintenance

$ ultramonitor alerts suppression remove --id=sup-001
Suppression sup-001 removed
Rule 'ssh-brute-force' re-enabled (was suppressed 2h 15m)
```

### Reversión de respuesta automática
```
# Bloqueó IP legítima por error:
$ ultramonitor firewall list-blocked
Blocked IPs:
  192.168.1.200 (reason: ALERT-005, blocked 14:22)
  10.0.5.89 (reason: ALERT-012, blocked 15:45)

$ ultramonitor firewall unblock --ip=192.168.1.200 --reason="false positive, user approval #7890"
IP 192.168.1.200 unblocked from all firewalls
Ticket #7890 updated with unblock action
```

### Reversión de limpieza de base de datos
```
# Se eliminaron alertas necesarias para investigación:
$ ultramonitor database restore \
  --index=ultramonitor-alerts-2026.02 \
  --date=2026-02-15 \
  --output=/restore/feb15-alerts.json

Restoring from Elasticsearch snapshot 'ultramonitor-snap-20260228'...
Found 1,247 alerts for 2026-02-15
Restored to /restore/feb15-alerts.json
Re-indexed into read-only cluster 'ultramonitor-forensics'
```

### Reversión completa del sistema
```
# Emergencia: revertir despliegue completo:
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

## Dependencias y Requisitos

### Dependencias en Tiempo de Ejecución
```
Python 3.9+ con paquetes:
  tensorflow==2.8.0 (o tensorflow-gpu para aceleración GPU)
  torch==1.11.0 (opcional, para modelos híbridos)
  scapy==2.5.0
  elasticsearch==7.14.0
  redis==4.3.0
  psutil==5.9.0
  pyyaml==6.0
  pandas==1.4.0
  scikit-learn==1.1.0
  prometheus-client==0.16.0
```

### Requisitos del Sistema
- 4+ núcleos de CPU (8+ recomendados)
- 16GB RAM mínimo (32GB para despliegues grandes)
- 100GB SSD para datos de Elasticsearch
- Interfaces de red en modo promiscuo para captura de paquetes
- Acceso a registros de autenticación (/var/log/auth.log, /var/log/secure)

### Requisitos de Elasticsearch
```
Plantillas de índice:
  - ultramonitor-alerts-* (sharding: 1, replicas: 1)
  - ultramonitor-events-* (sharding: 3, replicas: 2)
  - ultramonitor-metrics-*
Política ILM: 90 días hot → delete
Repositorio de snapshots: s3://ultramonitor-backups o NFS
```

### Requisitos de Redis
```
Versión: 6.2+
Memoria: 4GB mínimo
Persistencia: AOF everysec
Eviction: allkeys-lru
Notificaciones de keyspace: habilitadas (notify-keyspace-events Ex)
```

### Variables de Entorno
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

## Pasos de Verificación

### Verificación post-instalación
```
1. Verificar que el servicio esté corriendo:
   $ systemctl status ultramonitor
   $ ultramonitor healthcheck --full

2. Verificar recopilación de datos:
   $ ultramonitor events count --last=5m
   Debería retornar recuento de eventos no cero

3. Probar detección:
   $ ultramonitor detect test --scenario=ssh-brute
   Debería generar alerta de prueba (severidad baja)

4. Verificar pipeline de alertas:
   $ ultramonitor alerts list --last=10m
   Debería ver alerta de prueba

5. Verificar conectividad del dashboard:
   $ curl -H "X-API-Key: $ULTRAMONITOR_API_KEY" http://localhost:8080/api/v1/health
   Debería retornar {"status": "healthy"}

6. Verificar indexación en Elasticsearch:
   $ curl -k -u ultramonitor:$PASS $ES_HOST/_cat/indices/ultramonitor-*
   Debería ver índices con recuento de documentos

7. Probar webhook (si configurado):
   $ ultramonitor alerts trigger --test --webhook=slack
   Debería recibir mensaje de prueba en Slack
```

### Verificación continua
```
# Añadir a monitoreo:
- ultramonitor_events_total (Prometheus)
- ultramonitor_alerts_generated_total (by severity)
- ultramonitor_processing_latency_seconds (p95 < 0.5s)
- ultramonitor_model_accuracy (if ground truth available)
- ultramonitor_errors_total

# Alertar sobre:
- ultramonitor_processing_errors > 10/minute
- ultramonitor_events_dropped_total > 0
- Elasticsearch latency > 1s
- Model accuracy drop > 5%
```

## Solución de Problemas

### Problema: Uso alto de CPU (>90%) por >5 minutos
```bash
# Verificar cuello de botella de inferencia de modelo:
$ ultramonitor model benchmark --duration=60
Si tiempo de inferencia por evento > 10ms, considerar:
  - Escalar horizontalmente (añadir más instancias ultramonitor)
  - Usar modelo más pequeño (ultramonitor model set --size=small)
  - Habilitar aceleración GPU (instalar tensorflow-gpu)

# Verificar tasa de eventos:
$ ultramonitor stats | grep events_per_second
Si > 10,000 eps, aumentar muestreo o filtrar fuentes:
$ ultramonitor config set --key=collection.sample_rate --value=0.5
```

### Problema: No se generan alertas para actividad maliciosa conocida
```bash
# Verificar umbral:
$ ultramonitor config get detection.threshold
Si muy alto (0.9+), bajar a 0.75.

# Verificar modelo cargado:
$ ultramonitor model status
Debería mostrar "loaded: true" y "version: X.Y.Z"

# Verificar extracción de características:
$ ultramonitor debug event --raw='{"event": "test"}' --verbose
Asegurar que características se extraen correctamente.

# Verificar recientidad de entrenamiento:
$ ultramonitor model list
Si modelo >30 días, reentrenar:
$ ultramonitor train --dataset=/data/labeled-events.parquet
```

### Problema: Retraso de indexación en Elasticsearch (>60 segundos)
```bash
# Verificar salud de ES:
$ curl -k -u user:pass $ES_HOST/_cluster/health?pretty
Estado debe ser "green". Si "yellow/red", añadir nodos o réplicas.

# Verificar tamaño de cola:
$ ultramonitor metrics | grep es_queue
Si cola > 10,000, aumentar tamaño de lote o número de workers:
$ ultramonitor config set --key=es.batch_size --value=5000

# Verificar TLS/certificados:
$ openssl s_client -connect es-host:9200 -showcerts
Verificar expiración de certificado y cadena de confianza.
```

### Problema: Inundación de falsos positivos (>100/hora)
```bash
# Identificar reglas top de falsos positivos:
$ ultramonitor alerts metrics --group-by=rule_id --last=24h | sort -n
Top rules: rule-ssh-brute (45%), rule-api-anomaly (32%)

# Supresión temporal (con ticket):
$ ultramonitor alerts suppress --rule-id=rule-ssh-brute --duration=6h --ticket=SEC-1234

# Afinar umbral específicamente:
$ ultramonitor config set --path=rules.rule-ssh-brute.threshold --value=0.85

# Añadir whitelist:
$ ultramonitor whitelist add --type=ip --value=10.0.0.0/8 --reason="Internal network"
```

### Problema: No puede conectar a Redis
```bash
# Probar conexión manualmente:
$ redis-cli -h redis-host -p 6379 ping
Debería retornar PONG.

# Verificar modo SSL:
$ ultramonitor config get redis.ssl
Si true pero Redis no configurado para SSL, establecer false.

# Verificar contraseña:
$ cat /etc/ultramonitor/secrets/redis-pass | head -1
Probar con:
$ redis-cli -h host -a 'password' ping

# Verificar memoria de Redis:
$ redis-cli -h host info memory | grep used_memory_human
Si >80% usado, aumentar memoria o añadir clúster Redis.
```

### Problema: La carga del modelo falla al inicio
```bash
# Verificar archivos de modelo:
$ ls -lh $ULTRAMONITOR_MODEL_PATH/
Debería contener: model.savedmodel/ o model.pt

# Verificar versión de TensorFlow:
$ python3 -c "import tensorflow as tf; print(tf.__version__)"
Debe coincidir con versión de entrenamiento (2.8.0). Actualizar/downgrade si no coincide.

# Verificar disponibilidad de GPU (opcional):
$ python3 -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
Si vacío y GPU esperada, instalar controladores CUDA.

# Fallback a CPU:
$ ultramonitor config set --key=model.use_gpu --value=false
$ systemctl restart ultramonitor
```

### Problema: Fallo en parsing de registros de autenticación
```bash
# Verificar rutas de registros:
$ ultramonitor config get collection.log_paths
Verificar que rutas existen: /var/log/auth.log, /var/log/secure

# Permisos:
$ ls -l /var/log/auth.log
Asegurar que usuario 'ultramonitor' puede leer (añadir a grupo 'adm' o set ACL):
$ setfacl -m u:ultramonitor:r /var/log/auth.log

# SELinux/AppArmor:
$ auditctl -l  # si usa auditd
Asegurar que ultramonitor pueda leer registros de auditoría.

# Probar parsing manualmente:
$ ultramonitor debug parse-auth --file=/var/log/auth.log --lines=10
Debería output eventos JSON.
```

### Problema: Conectividad en despliegue containerizado
```bash
# Verificar red Docker:
$ docker network ls | grep ultramonitor
Container necesita acceso de red a hosts ES y Redis.

# Verificar variables de entorno en contenedor:
$ docker exec ultramonitor env | grep ULTRAMONITOR
Todas las vars requeridas deben estar seteadas.

# Verificar sincronización de tiempo:
$ docker exec ultramonitor date
Tiempo del sistema debe estar sincronizado (NTP). Eventos requieren timestamps precisos.

# Ver logs de contenedor:
$ docker logs ultramonitor --tail=100
Buscar trazas de pila o errores de import.
```

### Problema: Alerta no aparece en dashboard
```bash
# Verificar alerta almacenada en ES:
$ curl -k -u user:pass "$ES_HOST/ultramonitor-alerts-*/_search?q=ALERT-001" | jq .
Debería encontrar documento.

# Verificar Redis pub/sub:
$ redis-cli -h redis-host psubscribe 'ultramonitor:alerts:*'
Generar alerta de prueba, debería ver mensaje.

# API de dashboard:
$ curl -H "X-API-Key: $ULTRAMONITOR_API_KEY" http://localhost:8080/api/v1/alerts/recent
```

---

```markdown
---
name: ultra-monitor
description: Sistema de monitoreo de seguridad y detección de amenazas con IA
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
```