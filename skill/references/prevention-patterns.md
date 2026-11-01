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
