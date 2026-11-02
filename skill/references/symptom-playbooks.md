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

---

## DEPLOY_REGRESSION
**Sinais:** bug apareceu após deploy específico, "funcionava ontem", staging sem problema

**Primeira pergunta:** "Qual foi o último deploy antes do problema? O rollback resolve?"

Hipóteses padrão:
- Breaking change em lógica de negócio no PR mergeado
- Variável de ambiente nova ausente em produção (funciona em staging com valor default)
- Migração de schema incompatível com a versão anterior do código
- Feature flag ligada para 100% dos usuários sem gradual rollout
- Dependência atualizada com comportamento diferente (package sem lock)

Comandos iniciais:
```bash
# Identificar o deploy problemático
git log --oneline -15 --stat
git log --oneline --since="24 hours ago"

# Ver o que mudou no deploy suspeito
git show {sha} --stat
git diff {sha-anterior}..{sha} -- src/

# Variáveis de ambiente presentes em prod?
kubectl exec -it {pod} -- env | sort | grep -E "FEATURE_|API_|CONNECTION_"

# Reverter rapidamente (se necessário)
kubectl rollout undo deployment/{deployment} -n {namespace}
kubectl rollout status deployment/{deployment} -n {namespace}
```

Estratégia:
1. Correlacione horário do incidente com horário do deploy no CI/CD
2. Analise diff do PR — procure alterações em: cálculos, validações, tratamento de null, migrations
3. Compare variáveis de ambiente entre staging e prod
4. Se feature flag: reduza para 0% e observe

---

## MEMORY_LEAK
**Sinais:** memória crescendo gradualmente, GC pressure, OOMKilled repetido, desempenho piorando ao longo do tempo

**Primeira pergunta:** "A memória cresce continuamente ou em picos? Após quanto tempo crashia?"

Hipóteses padrão:
- Event handlers não removidos (subscriber acumulando)
- Cache local crescendo sem eviction policy (Dictionary/List sem limite)
- Conexões não fechadas (HTTP client, DB connection, streams)
- Static collection acumulando objetos (logs, métricas em memória)
- Imagens ou arquivos grandes carregados em memória sem stream

Comandos iniciais:
```bash
# Memória do pod ao longo do tempo
kubectl top pod {pod} -n {namespace}  # monitorar por 5 min

# Histórico de OOMKilled
kubectl describe pod {pod} -n {namespace} | grep -A5 "OOMKilled\|Last State"

# .NET — counters de GC e memória
dotnet-counters monitor --process-id {pid} \
  --counters System.Runtime[gc-heap-size,gen-0-gc-count,gen-1-gc-count,gen-2-gc-count]

# Docker — uso de memória em tempo real
docker stats {container} --no-stream --format "{{.MemUsage}}"

# Dump de heap para análise offline
kubectl exec -it {pod} -- dotnet-dump collect --process-id 1 --output /tmp/heap.dmp
```

Estratégia:
1. Confirme o padrão de crescimento (linear = leak, dente de serra = normal)
2. Correlacione picos com tráfego ou operações específicas
3. Use dotnet-counters para ver qual geração do GC está crescendo (Gen2 = suspeito)
4. Procure no código: `static List/Dictionary`, `+=` em eventos, `IDisposable` sem `using`

---

## QUEUE_BACKLOG
**Sinais:** Kafka consumer lag crescendo, RabbitMQ com mensagens acumulando, processamento atrasado

**Primeira pergunta:** "O lag está crescendo ou estagnado? Consumers estão rodando?"

Hipóteses padrão:
- Consumer crashando/reiniciando (lag cresce porque não processa)
- Throughput do consumer menor que produção (scaling necessário)
- Mensagem poison: payload inválido causa reprocessamento infinito
- Downstream bloqueado: consumer processa mas escreve no DB lento
- Rebalance constante: consumers entrando e saindo → nenhum termina o commit

Comandos iniciais:
```bash
# Kafka — lag de todos os consumer groups
kafka-consumer-groups.sh --bootstrap-server {host}:9092 \
  --describe --all-groups | sort -k6 -rn | head -20

# Kafka — tópico com mais lag
kafka-consumer-groups.sh --bootstrap-server {host}:9092 \
  --describe --group {group} | awk '{print $1, $5, $6}' | sort -k3 -rn

# Ver se consumers estão ativos
kafka-consumer-groups.sh --bootstrap-server {host}:9092 \
  --describe --group {group} | grep -E "CONSUMER-ID|HOST"

# RabbitMQ — filas com backlog
curl -s -u {user}:{pass} http://{host}:15672/api/queues | \
  python3 -c "import sys,json; [print(f'{q[\"name\"]}: {q[\"messages\"]}') for q in json.load(sys.stdin) if q['messages'] > 100]"
```

Estratégia:
1. Confirme que consumers estão rodando (não só existindo)
2. Verifique se lag é em todas as partições ou só uma (poison message = 1 partição travada)
3. Mensagem poison: `--reset-offsets --to-latest` na partição afetada + mova para DLQ
4. Se capacidade: escale consumers horizontalmente (até o número de partições)

---

## BUSINESS_LOGIC
**Sinais:** cálculo errado (preço, desconto, frete), regra de negócio violada, status de pedido inválido, relatório com número errado

**Primeira pergunta:** "Qual é o valor esperado vs o valor real? Para todos os usuários ou casos específicos?"

Hipóteses padrão:
- Condição de negócio faltando (caso de borda não tratado: desconto + cupom + frete grátis)
- Ordem de operações errada (aplica desconto antes do frete → resultado diferente)
- Arredondamento incorreto (float vs decimal em cálculo financeiro)
- Estado de transição inválido (pedido vai de CRIADO direto para ENTREGUE sem PAGO)
- Dado de referência desatualizado (tabela de preços stale, configuração de promoção vencida)

Comandos iniciais:
```bash
# Reproduzir o caso específico com o mesmo input
curl -X POST {endpoint} -d '{payload do caso com erro}'

# Ver o estado atual do registro no banco
SELECT * FROM orders WHERE id = '{order_id}';
SELECT * FROM order_items WHERE order_id = '{order_id}';

# Histórico de estados (se tiver audit log)
SELECT status, changed_at, changed_by FROM order_status_history
WHERE order_id = '{order_id}' ORDER BY changed_at;

# Commits que tocaram a lógica de negócio suspeita
git log --oneline -20 -- src/{Dominio}/{Serviço}/
git log -S "CalculateDiscount\|ApplyCoupon\|OrderTotal" --oneline -10
```

Estratégia:
1. Reproduza o caso com valores reais — não assuma nada
2. Trace o fluxo de dados passo a passo: input → transformações → output
3. Verifique os casos de borda mencionados no PR/ticket original
4. Escreva um teste que reproduza o bug antes de corrigir

---

## LOG_ANOMALY
**Sinais:** spike em logs de erro, novo tipo de exception antes inexistente, padrão anômalo (muitos 404, timeout em endpoint específico)

**Primeira pergunta:** "Quando o padrão apareceu? Tem correlação com deploy ou evento externo?"

Hipóteses padrão:
- Deploy introduziu novo caminho de código com logging diferente
- Usuário ou bot fazendo requisições anômalas (scraper, ataque, bug de front)
- Integração externa mudou formato de resposta (campo removido, tipo diferente)
- Certificado SSL expirado causando falhas silenciosas
- Rate limit atingido em API parceira → cascade de erros 429→500

Comandos iniciais:
```bash
# Frequência por tipo de erro
grep -oE "Exception: [^|]+" logs/app.log | sort | uniq -c | sort -rn | head -20
grep -oE "HTTP [0-9]+" logs/access.log | sort | uniq -c | sort -rn

# Primeiros logs do novo padrão (quando começou?)
grep "{novo_erro}" logs/ | head -5

# Correlação com IPs específicos
grep "4[0-9][0-9]" logs/access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10

# Correlação com horário do deploy
git log --oneline --since="{data_inicio_anomalia}" | head -5

# Loki (se disponível)
logcli query '{app="{service}"} |= "{novo_erro}"' --since=2h --limit=50
```

Estratégia:
1. Quantifique: qual é a taxa de ocorrência antes vs depois?
2. Identifique o primeiro log do padrão anômalo → correlacione com evento
3. Filtre por IP, user-agent, endpoint para isolar a origem
4. Se bot/scraper: adicione rate limiting; se bug de integração: compare response schema
