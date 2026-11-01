# Symptom Playbooks

Mapeamento rápido: sintoma detectado → estratégia de investigação + hipóteses padrão.

---

## EXCEPTION
**Sinais:** stack trace, unhandled exception, HTTP 5xx, pod crashando/reiniciando
**Primeira pergunta:** "O erro é reproduzível ou intermitente?"

Hipóteses padrão:
- Null reference / tipo inesperado no input (deserialização, tipo nullable)
- Dependência externa fora do ar (DB, API terceira, mensageria)
- Breaking change introduzida em deploy recente
- Exaustão de recurso: memória (OOMKilled), conexões (pool esgotado), threads
- Configuração incorreta em novo ambiente (secret ausente, variável de env)

Comandos iniciais:
```bash
kubectl logs {pod} --previous            # se pod reiniciou
kubectl describe pod {pod}               # ver reason do crash (OOMKilled, Error)
git log --oneline -10 --stat             # deploys recentes
grep -r "NullReferenceException\|StackOverflow" logs/
```

---

## SLOW
**Sinais:** latência aumentando, timeout, requests enfileirando, CPU/memória elevados

**Primeira pergunta:** "Lento para todos ou para endpoints/usuários específicos? Quando começou?"

Hipóteses padrão:
- Query sem índice ou N+1 queries (mais comum)
- Lock de banco de dados bloqueando transações
- Pool de conexões esgotado (DB, Redis, HTTP)
- Memory leak → GC pressure → pausa do runtime
- Dependência externa com latência crescente
- Thread pool saturation (ThreadPoolExhaustedException)

Comandos iniciais:
```bash
# PostgreSQL — queries lentas
SELECT query, mean_exec_time, calls FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;
SELECT pid, wait_event_type, wait_event, query FROM pg_stat_activity WHERE state != 'idle';

# Redis — latência
redis-cli --latency -h {host}
redis-cli INFO stats | grep -E "instantaneous_ops|rejected"

# .NET — thread pool
dotnet-counters monitor --process-id {pid} System.Runtime
```

---

## DUPLICATE
**Sinais:** registros duplicados, operação executada 2x, cobrança/email duplicado

**Primeira pergunta:** "O duplicado aparece imediatamente ou com delay? Após restart/deploy?"

Hipóteses padrão:
- Idempotência ausente em consumer de fila (Kafka offset não commitado)
- Retry sem deduplication key no lado do cliente
- Race condition em operação concorrente (INSERT sem ON CONFLICT)
- Rebalance de consumer group → reentrega de mensagem
- Webhook recebido 2x por timeout na confirmação

Comandos iniciais:
```bash
# Contar duplicatas
SELECT id, COUNT(*) FROM {tabela} GROUP BY id HAVING COUNT(*) > 1 LIMIT 20;

# Kafka — verificar consumer lag e rebalances
kafka-consumer-groups.sh --bootstrap-server {host} --describe --group {group}

# Timestamps dos dois processamentos
SELECT created_at, order_id FROM fulfillment_event WHERE order_id = '{id}' ORDER BY created_at;
```

---

## WRONG_DATA
**Sinais:** valor incorreto, status desatualizado, cálculo errado, dado faltando

**Primeira pergunta:** "O dado foi errado desde a criação ou mudou/sumiu depois?"

Hipóteses padrão:
- Cache stale (Redis com TTL longo ou sem invalidação)
- Race condition em update concorrente (lost update)
- Bug em cálculo ou transformação de dados
- Evento de atualização perdido na fila (mensagem dropada)
- Migration de dados incompleta ou com condição de race

Comandos iniciais:
```bash
# Comparar cache vs banco
redis-cli GET "cache:{chave}"
SELECT * FROM {tabela} WHERE id = '{id}';

# Histórico de mudanças (se audit log existir)
SELECT * FROM {tabela}_history WHERE record_id = '{id}' ORDER BY changed_at DESC;

# Eventos recentes na fila
kafka-console-consumer.sh --topic {topic} --from-beginning --max-messages 50
```

---

## CASCADE
**Sinais:** múltiplos serviços falhando em sequência, latência espalhando, circuit breaker aberto

**Primeira pergunta:** "Qual serviço apareceu nos alertas primeiro?"

Hipóteses padrão:
- Serviço upstream fora do ar sem timeout adequado (cascata por bloqueio)
- Thread pool / connection pool exhaustion se espalhando
- Thundering herd após recovery de dependência compartilhada
- Shared dependency saturada: DB, Redis, broker
- Retry storm: todos os clientes retentando ao mesmo tempo

Comandos iniciais:
```bash
# Pods com problema
kubectl get pods -A | grep -v Running | grep -v Completed

# Ordem dos alertas — qual foi primeiro
# (verificar no Grafana/Datadog timeline de alertas)

# Dependency graph: quem depende de quem
# usar architecture.json do projeto se disponível
```

---

## SECURITY
**Sinais:** acesso não autorizado, dado de outro usuário visível, comportamento anômalo de auth

**Primeira pergunta:** "O acesso foi autenticado (token válido com claim errado) ou bypass de auth?"

Hipóteses padrão:
- IDOR: ID de recurso sem validação de ownership
- JWT com claim incorreto (role, tenant, userId)
- Secret ou dado sensível vazando em log ou resposta de erro
- Misconfiguration de CORS ou auth middleware
- Rate limit ausente em endpoint de auth → brute force

Comandos iniciais:
```bash
# Logs de auth com IPs e user agents
grep -E "401|403|JWT|token" logs/app.log | tail -100

# Verificar se secrets aparecem em logs
grep -rE "password|secret|token|key" logs/ | grep -v "****"

# Rate limit hits
grep "rate limit\|too many requests\|429" logs/nginx.log | wc -l
```

---

## FLAKY
**Sinais:** erro intermitente, funciona às vezes, difícil de reproduzir, falha em CI mas não local

**Primeira pergunta:** "Tem padrão de horário, carga ou ambiente específico?"

Hipóteses padrão:
- Race condition timing-dependent (teste sem await, sleep insuficiente)
- Dependência externa instável (rate limit, timeout variável)
- Test pollution: estado compartilhado entre testes (DB não limpo, mock vazando)
- Diferença de configuração prod vs staging vs local
- Flaky por ordem de execução dos testes

Comandos iniciais:
```bash
# Frequência de falha
grep "FAILED\|ERROR" ci-logs/ | grep "{test-name}" | wc -l

# Correlação com horário (pico de tráfego?)
# verificar nos logs se as falhas coincidem com alta carga

# Diferenças de ambiente
diff .env.local .env.staging
env | sort > prod.env && diff local.env prod.env
```
