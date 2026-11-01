# Infra Commands Reference

Comandos prontos para coleta de evidências. Copie, adapte variáveis entre `{}` e execute.

---

## Docker

```bash
# Status de containers
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Logs de um container (com timestamp)
docker logs {container} --timestamps --tail 200

# Logs no período do incidente
docker logs {container} --since "2026-11-01T14:00:00" --until "2026-11-01T15:00:00"

# Uso de recursos
docker stats --no-stream

# Inspecionar restart count e exit code
docker inspect {container} | python3 -c "import sys,json; c=json.load(sys.stdin)[0]; s=c['State']; print(f'Status: {s[\"Status\"]} | ExitCode: {s[\"ExitCode\"]} | Restarts: {c[\"RestartCount\"]}')"

# Entrar no container
docker exec -it {container} sh

# Ver variáveis de ambiente
docker inspect {container} | python3 -c "import sys,json; [print(e) for e in json.load(sys.stdin)[0]['Config']['Env']]"
```

---

## Kubernetes / AKS

```bash
# Pods com problema
kubectl get pods -A | grep -v "Running\|Completed"

# Detalhes do pod (ver Events, OOMKilled, reason)
kubectl describe pod {pod} -n {namespace}

# Logs atuais
kubectl logs {pod} -n {namespace} --tail 200

# Logs do container anterior (após crash)
kubectl logs {pod} -n {namespace} --previous

# Filtrar por erro nos logs
kubectl logs {pod} -n {namespace} --since=1h | grep -iE "error|exception|fatal|killed"

# Consumo de recursos
kubectl top pods -n {namespace}
kubectl top nodes

# Eventos recentes no namespace
kubectl get events -n {namespace} --sort-by='.lastTimestamp' | tail -20

# Rollout history (ver qual versão está rodando)
kubectl rollout history deployment/{deployment} -n {namespace}

# Rollback rápido
kubectl rollout undo deployment/{deployment} -n {namespace}

# Escalar para 0 e voltar (forçar restart)
kubectl scale deployment/{deployment} --replicas=0 -n {namespace}
kubectl scale deployment/{deployment} --replicas=3 -n {namespace}
```

---

## PostgreSQL / Dapper

```bash
# Entrar no psql
docker exec -it {postgres-container} psql -U {user} -d {database}
# ou via kubectl
kubectl exec -it {pod} -n {namespace} -- psql -U {user} -d {database}
```

```sql
-- Queries mais lentas (requer pg_stat_statements)
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 10;

-- Conexões ativas e o que estão fazendo
SELECT pid, usename, state, wait_event_type, wait_event,
       substring(query, 1, 80) AS query_snippet
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- Locks aguardando
SELECT pg_blocking_pids(pid) AS blocked_by, pid, query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;

-- Tamanho das tabelas
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC LIMIT 20;

-- Duplicatas em tabela
SELECT {coluna_id}, COUNT(*) AS cnt
FROM {tabela}
GROUP BY {coluna_id}
HAVING COUNT(*) > 1
ORDER BY cnt DESC LIMIT 20;

-- Últimas linhas inseridas
SELECT * FROM {tabela} ORDER BY created_at DESC LIMIT 20;

-- Verificar índices ausentes em tabela com seq scan alto
SELECT relname, seq_scan, idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan
ORDER BY seq_scan DESC LIMIT 10;
```

---

## Redis

```bash
# Verificar se chave existe e TTL
redis-cli -h {host} EXISTS {chave}
redis-cli -h {host} TTL {chave}
redis-cli -h {host} GET {chave}

# Info geral (memória, conexões, ops/s)
redis-cli -h {host} INFO stats | grep -E "instantaneous_ops|rejected_connections|evicted_keys"
redis-cli -h {host} INFO memory | grep -E "used_memory_human|maxmemory_human"

# Chaves que batem um padrão
redis-cli -h {host} KEYS "idempotency:*" | head -20
redis-cli -h {host} SCAN 0 MATCH "cache:*" COUNT 100

# Monitorar comandos em tempo real (cuidado em prod)
redis-cli -h {host} MONITOR | head -50

# Latência
redis-cli -h {host} --latency

# Flush de chave específica (cuidado)
redis-cli -h {host} DEL {chave}
```

---

## Kafka

```bash
# Listar consumer groups
kafka-consumer-groups.sh --bootstrap-server {host}:9092 --list

# Ver lag do consumer group
kafka-consumer-groups.sh --bootstrap-server {host}:9092 \
  --describe --group {group-name}

# Mensagens de um tópico (últimas N)
kafka-console-consumer.sh --bootstrap-server {host}:9092 \
  --topic {topic} --from-beginning --max-messages 20

# Offset atual de cada partição
kafka-run-class.sh kafka.tools.GetOffsetShell \
  --broker-list {host}:9092 --topic {topic}

# Tópicos existentes
kafka-topics.sh --bootstrap-server {host}:9092 --list

# Detalhes do tópico (partições, replication factor)
kafka-topics.sh --bootstrap-server {host}:9092 \
  --describe --topic {topic}

# Reset de offset (use com cuidado — reprocessa mensagens)
kafka-consumer-groups.sh --bootstrap-server {host}:9092 \
  --group {group} --topic {topic} --reset-offsets --to-latest --execute
```

---

## RabbitMQ

```bash
# Via Management API
curl -s -u {user}:{pass} http://{host}:15672/api/queues | \
  python3 -c "import sys,json; [print(f'{q[\"name\"]}: {q[\"messages\"]} msgs, {q[\"consumers\"]} consumers') for q in json.load(sys.stdin)]"

# Filas com mensagens presas
curl -s -u {user}:{pass} http://{host}:15672/api/queues | \
  python3 -c "import sys,json; [print(q['name'], q['messages']) for q in json.load(sys.stdin) if q['messages'] > 0]"

# Via CLI (dentro do container)
docker exec -it {rabbitmq-container} rabbitmqctl list_queues name messages consumers
docker exec -it {rabbitmq-container} rabbitmqctl list_connections
```

---

## Jaeger / Distributed Tracing

```bash
# Buscar traces por serviço (via API)
curl "http://{jaeger-host}:16686/api/traces?service={service}&limit=20&lookback=1h" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); [print(t['traceID'], t['spans'][0]['operationName']) for t in d.get('data',[])]"

# Trace específico por ID
curl "http://{jaeger-host}:16686/api/traces/{traceId}"

# Serviços disponíveis
curl "http://{jaeger-host}:16686/api/services"
```

---

## Grafana / Prometheus / Loki

```bash
# Prometheus — query via API
curl "http://{prometheus}:9090/api/v1/query?query=rate(http_requests_total{job=\"{service}\"}[5m])"

# Verificar targets que estão down
curl "http://{prometheus}:9090/api/v1/targets" | \
  python3 -c "import sys,json; [print(t['labels']['job'], t['health']) for t in json.load(sys.stdin)['data']['activeTargets'] if t['health']!='up']"

# Loki — buscar logs por label e filtro
curl -G "http://{loki}:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={app="{service}"} |= "error"' \
  --data-urlencode "start=$(date -d '1 hour ago' +%s)000000000" \
  --data-urlencode "end=$(date +%s)000000000"
```

---

## .NET / C#

```bash
# Dump de memória em pod .NET
kubectl exec -it {pod} -n {namespace} -- \
  dotnet-dump collect --process-id 1 --output /tmp/dump.dmp

# Counters de runtime .NET
dotnet-counters monitor \
  --process-id {pid} \
  --counters System.Runtime,Microsoft.AspNetCore.Hosting

# Stack trace de threads travadas
dotnet-stack report --process-id {pid}

# Verificar versão do runtime
dotnet --info

# Testar endpoint internamente
kubectl exec -it {pod} -- curl -s http://localhost:{port}/health
kubectl exec -it {pod} -- curl -s http://localhost:{port}/metrics | grep -E "^comprai_|^aspnet"
```

---

## Git / GitHub

```bash
# Commits recentes com arquivos alterados
git log --oneline --stat -10

# Diff de commit específico
git show {commit-sha} --stat
git show {commit-sha} -- {arquivo}

# Quem alterou uma linha específica
git blame {arquivo} -L {linha-inicio},{linha-fim}

# Commits entre dois deploys
git log {sha-anterior}..{sha-atual} --oneline

# Buscar quando uma linha foi introduzida
git log -S "trecho do código" --oneline

# Ver PR no terminal (requer gh CLI)
gh pr view {numero} --json title,body,files
gh pr diff {numero}
```
