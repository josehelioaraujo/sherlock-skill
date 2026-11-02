# Prevention Patterns

Checklists de prevenção por categoria RCA.
Use após o diagnóstico (FASE 5) para evitar recorrência.

---

## CODE — Bug de Código

```
□ Adicionar teste unitário que reproduz o bug exato
□ Code review: checar casos de borda (null, empty, tipo errado)
□ Validação de input na entrada do endpoint/handler
□ Adicionar assertion/guard clause no ponto de falha
□ Aumentar cobertura de testes no módulo afetado
□ Analisar código similar no projeto (mesmo padrão, mesmo risco)
□ Considerar análise estática (SonarQube, Semgrep) no CI
```

**Exemplo de fix-pattern:**
```csharp
// Antes (sem guard)
var result = service.Process(order);

// Depois (com guard e log)
ArgumentNullException.ThrowIfNull(order, nameof(order));
if (order.Items.Count == 0)
    throw new InvalidOperationException($"Order {order.Id} has no items");
var result = service.Process(order);
```

---

## INFRA — Infraestrutura

```
□ Definir resource limits e requests no Kubernetes (CPU + memória)
□ Adicionar liveness e readiness probes no deployment
□ Configurar HPA (Horizontal Pod Autoscaler) para picos de carga
□ Alerta de memória: > 80% do limit por mais de 5 minutos
□ Alerta de restart count: > 2 restarts em 10 minutos
□ PodDisruptionBudget: garantir mínimo de réplicas durante updates
□ Testar comportamento do app ao atingir o memory limit (graceful degradation?)
□ Revisar tamanho de payload processado em memória
```

**Exemplo de resource limits:**
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

---

## DATA — Dados

```
□ Migration com rollback testado antes de aplicar em prod
□ Migration em janela de manutenção se altera tabela grande
□ Backup antes de migrations destrutivas (DROP, ALTER)
□ Constraint de unicidade no banco para dados críticos
□ Soft delete antes de hard delete (período de retenção)
□ Audit log para tabelas críticas (orders, payments, users)
□ Validação de dados na camada de aplicação E no banco (constraint)
□ Teste de migration: aplicar + reverter em ambiente clone de prod
```

**Exemplo de constraint de unicidade:**
```sql
-- Previne duplicata no nível do banco
ALTER TABLE fulfillment_event
ADD CONSTRAINT uq_fulfillment_order UNIQUE (order_id, event_type);
```

---

## CONFIG — Configuração

```
□ Validar secrets/variáveis obrigatórias na startup (fail-fast)
□ Health check que valida conectividade com dependências
□ Documentar todas as variáveis de ambiente obrigatórias
□ Usar secret manager (Azure Key Vault, AWS Secrets Manager)
□ Nunca commitar .env com valores reais — validar com git-secrets/gitleaks
□ Checklist de deploy: variáveis de prod vs staging
□ Smoke test pós-deploy que valida configuração crítica
```

**Exemplo de validação na startup (.NET):**
```csharp
// Program.cs — fail-fast em config ausente
var jwtSecret = builder.Configuration["Jwt:Secret"]
    ?? throw new InvalidOperationException("Jwt:Secret is required");

var connString = builder.Configuration.GetConnectionString("Postgres")
    ?? throw new InvalidOperationException("ConnectionStrings:Postgres is required");
```

---

## EXTERNAL — Dependência Externa

```
□ Timeout configurado em todas as chamadas externas (nunca infinito)
□ Circuit breaker: abrir após N falhas, fechar após M sucessos
□ Retry com backoff exponencial e jitter
□ Fallback: o que fazer quando a dependência está fora?
□ Alerta: error rate > 5% para a dependência X
□ Health check inclui dependências críticas
□ Testar comportamento do sistema com a dependência fora (chaos engineering)
□ SLA da dependência documentado — o nosso SLA não pode ser maior
```

**Exemplo com Polly (.NET):**
```csharp
services.AddHttpClient<IPaymentClient, StripeClient>()
    .AddPolicyHandler(Policy.WrapAsync(
        Policy.Handle<HttpRequestException>()
              .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)),
        Policy.Handle<HttpRequestException>()
              .WaitAndRetryAsync(3, i => TimeSpan.FromSeconds(Math.Pow(2, i)))
    ));
```

---

## RACE — Race Condition / Idempotência

```
□ Idempotency key em toda operação de consumer de fila
□ SETNX Redis com TTL antes de processar eventos críticos
□ INSERT ... ON CONFLICT DO NOTHING para operações únicas
□ enable.auto.commit=false em consumers Kafka + commitSync() explícito
□ Distributed lock (Redis SETNX) para operações críticas concorrentes
□ Teste de carga: simular N consumers processando o mesmo evento
□ Teste de caos: matar pod durante processamento de evento
□ Alerta: duplicate entries por {chave de negócio} nas últimas 24h
□ Revisão de código: checklist de idempotência em PRs de consumers/workers
```

**Exemplo de idempotência (.NET + Redis):**
```csharp
public async Task ProcessAsync(OrderCreatedEvent evt, CancellationToken ct)
{
    var key = $"idempotency:fulfillment:{evt.OrderId}";
    var acquired = await redis.SetAsync(key, "1", TimeSpan.FromHours(1),
                                        When.NotExists);
    if (!acquired)
    {
        logger.LogWarning("Duplicate event for OrderId={OrderId} — skipped", evt.OrderId);
        return;
    }

    await fulfillmentService.CreateAsync(evt.OrderId, ct);
}
```

---

## SECURITY — Segurança

```
□ Verificar ownership em todo acesso a recurso por ID (anti-IDOR)
□ Nunca logar dados sensíveis: senha, token, cartão, CPF
□ Rate limiting em endpoints de auth (login, register, forgot-password)
□ Security headers: CSP, HSTS, X-Frame-Options, X-Content-Type-Options
□ Dependências auditadas: dotnet list package --vulnerable / npm audit
□ SAST no CI: Semgrep, Gitleaks para secrets em código
□ Pentest periódico em endpoints de auth e admin
□ Revisar permissões de serviços: princípio do menor privilégio
```

**Exemplo de verificação de ownership:**
```csharp
// Verificar que o recurso pertence ao usuário autenticado
var order = await db.GetOrderAsync(orderId, ct);
if (order is null || order.CustomerId != authenticatedCustomerId)
    return Results.NotFound(); // nunca 403 — não confirmar existência
```

---

## PERF — Memory Leak / Performance

```
□ Configurar memory limit no Kubernetes e alerta em > 80%
□ dotnet-counters em CI: validar que GC Gen2 não cresce em carga
□ Revisar todo código com IDisposable: usar using ou DisposeAsync
□ Cache local: sempre definir tamanho máximo (MemoryCache com SizeLimit)
□ Event handlers: sempre desinscrever (−= ou WeakReference)
□ Static collections: nunca crescer indefinidamente — use bounded queue
□ HTTP clients: usar IHttpClientFactory (nunca new HttpClient() manual)
□ Streams e arquivos: sempre fechar com using — nunca ler tudo em memória
□ Teste de soak: rodar carga por 30+ minutos e monitorar memória
□ Alerta: GC Gen2 count crescendo por mais de 10 minutos
```

**Exemplo de bounded cache (.NET):**
```csharp
// MemoryCache com limite de tamanho
services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024; // máximo 1024 entradas
});

// Ao inserir — sempre com tamanho
cache.Set(key, value, new MemoryCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5),
    Size = 1
});
```

**Exemplo de HTTP client correto (.NET):**
```csharp
// ❌ Errado — socket exhaustion após ~60s
var client = new HttpClient();

// ✅ Correto — pool gerenciado pelo factory
services.AddHttpClient<IMyService, MyService>();
```

---

## DEPLOY — Regressão de Deploy

```
□ Feature flags para rollout gradual (0% → 10% → 50% → 100%)
□ Smoke tests automáticos pós-deploy antes de promover para prod
□ Canary deploy: rotear 5% do tráfego para nova versão antes de 100%
□ Rollback automático se error rate > X% nos primeiros 5 minutos após deploy
□ Checklist de deploy: migrations? variáveis novas? breaking changes de API?
□ Blue-green ou rolling update com maxUnavailable=0
□ Changelog legível: todo PR documenta o que mudou e o impacto esperado
□ Alertas sensíveis nos 15 minutos após cada deploy (janela de atenção)
□ Lock de deploy em horário de pico (ex: não deployar sexta 17h-20h)
□ Testes de contrato (contract tests) para quebras silenciosas de API
```

**Exemplo de rollback automático (GitHub Actions):**
```yaml
- name: Verify deploy health
  run: |
    sleep 60
    ERROR_RATE=$(curl -s "$PROMETHEUS/api/v1/query" \
      --data-urlencode 'query=rate(http_requests_total{status=~"5.."}[2m])' | jq '.data.result[0].value[1]')
    if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
      echo "Error rate $ERROR_RATE > 5% — triggering rollback"
      kubectl rollout undo deployment/$SERVICE -n production
      exit 1
    fi
```

---

## QUEUE — Fila / Mensageria

```
□ Dead Letter Queue (DLQ) configurada para mensagens que falham N vezes
□ Alerta: consumer lag > threshold por mais de 5 minutos
□ Alerta: mensagens na DLQ > 0 (qualquer poison message é crítico)
□ Partition count = número máximo de consumers paralelos planejados
□ Monitore rebalances: muitos rebalances = consumers instáveis
□ enable.auto.commit=false + commit explícito APÓS processamento bem-sucedido
□ Max poll records baixo (50–200) para evitar timeout de sessão em processamento lento
□ Backpressure: se downstream lento, reduza poll records ou pause o consumer
□ Retry com backoff na mensagem antes de enviar para DLQ
□ Schema registry ou contrato de mensagem documentado para evitar breaking change
```

**Exemplo de consumer Kafka robusto (.NET + Confluent):**
```csharp
consumer.Subscribe(topic);
while (!ct.IsCancellationRequested)
{
    var msg = consumer.Consume(ct);
    try
    {
        await handler.HandleAsync(msg.Value, ct);
        consumer.Commit(msg);          // commit APÓS sucesso
    }
    catch (TransientException ex)
    {
        logger.LogWarning(ex, "Transient error — will retry");
        // não commita → Kafka reentregará
    }
    catch (PoisonMessageException ex)
    {
        logger.LogError(ex, "Poison message {Key} — sending to DLQ", msg.Key);
        await dlq.PublishAsync(msg.Value, ct);
        consumer.Commit(msg);          // commita para não travar a partição
    }
}
```

---

## BUSINESS — Lógica de Negócio

```
□ Testes de contrato para cada regra de negócio crítica (tabela de decisão)
□ Testes com valores reais de casos de borda: R$0, desconto > total, frete grátis + cupom
□ Decimal (não float/double) para todos os cálculos financeiros
□ State machine explícita para transições de status (nunca if-else solto)
□ Audit log em todas as tabelas financeiras (orders, payments, invoices)
□ Validação de estado válido antes de cada transição
□ Documentar regras de negócio no código (comment + link para spec)
□ Alerta: valor calculado fora de range esperado (pedido com total negativo, desconto > 100%)
□ Testes de regressão automáticos para cada bug de negócio corrigido
```

**Exemplo de state machine (.NET):**
```csharp
public static class OrderStatusTransitions
{
    private static readonly Dictionary<OrderStatus, HashSet<OrderStatus>> _allowed = new()
    {
        [OrderStatus.Created]    = [OrderStatus.Paid, OrderStatus.Cancelled],
        [OrderStatus.Paid]       = [OrderStatus.Processing, OrderStatus.Refunded],
        [OrderStatus.Processing] = [OrderStatus.Shipped],
        [OrderStatus.Shipped]    = [OrderStatus.Delivered],
    };

    public static void ValidateTransition(OrderStatus from, OrderStatus to)
    {
        if (!_allowed.TryGetValue(from, out var allowed) || !allowed.Contains(to))
            throw new InvalidOperationException(
                $"Invalid order status transition: {from} → {to}");
    }
}
```

**Cálculo financeiro correto:**
```csharp
// ❌ Errado — float acumula erro
float total = 0.1f + 0.2f; // = 0.30000001192092896

// ✅ Correto — decimal para dinheiro
decimal total = 0.1m + 0.2m; // = 0.3
decimal discounted = Math.Round(total * (1 - discountRate), 2, MidpointRounding.AwayFromZero);
```
