# RCA Categories

Categorias de causa raiz para classificar o diagnóstico final.
Use para direcionar o prevention checklist correto.

---

## Categorias

| Código    | Nome               | Descrição                                                              |
|-----------|--------------------|------------------------------------------------------------------------|
| CODE      | Bug de Código      | Lógica incorreta, null reference, tipo errado, condição faltando       |
| INFRA     | Infraestrutura     | OOMKilled, disco cheio, rede instável, pod crashando                   |
| DATA      | Dados              | Migration incorreta, dado corrompido, constraint violada               |
| CONFIG    | Configuração       | Variável ausente, secret errado, feature flag incorreta                |
| EXTERNAL  | Dependência Externa| API terceira, DNS, CDN, serviço de pagamento instável                  |
| RACE      | Race Condition     | Concorrência, idempotência, duplicate processing, lost update          |
| SECURITY  | Segurança          | Auth bypass, IDOR, secret vazado, misconfiguration                     |
| PERF      | Performance        | Memory leak, GC pressure, thread saturation, CPU spike sem crash       |
| DEPLOY    | Regressão de Deploy| Bug introduzido por PR/deploy específico, funciona em staging          |
| QUEUE     | Fila / Mensageria  | Backlog Kafka/RabbitMQ, poison message, consumer group instável        |
| BUSINESS  | Lógica de Negócio  | Cálculo errado, regra violada, estado inválido, caso de borda faltando |

---

## Matriz de Frequência (referência)

Baseada em incidentes reais em sistemas distribuídos .NET/Node/Python em cloud:

| Categoria | Frequência | MTTR médio | Severidade típica |
|-----------|-----------|------------|-------------------|
| CODE      | ★★★★★     | 2–4h       | Alta              |
| DEPLOY    | ★★★★★     | 30min–2h   | Alta (rollback rápido) |
| RACE      | ★★★★☆     | 4–8h       | Crítica           |
| CONFIG    | ★★★★☆     | 30min      | Alta (fácil fix)  |
| INFRA     | ★★★☆☆     | 1–3h       | Alta              |
| EXTERNAL  | ★★★☆☆     | depende    | Variável          |
| PERF      | ★★★☆☆     | 2–6h       | Média → Crítica   |
| QUEUE     | ★★★☆☆     | 1–4h       | Alta              |
| BUSINESS  | ★★☆☆☆     | 4–12h      | Alta (impacto financeiro) |
| DATA      | ★★☆☆☆     | 6–24h      | Crítica           |
| SECURITY  | ★☆☆☆☆     | 2–48h      | Crítica           |

---

## Como usar na investigação

### Durante a FASE 1 (Hipóteses)
Classifique cada hipótese com a categoria candidata:
```
H1 [85%] Idempotência ausente → categoria: RACE
H2 [60%] Secret mal configurado → categoria: CONFIG
H3 [35%] Bug na serialização → categoria: CODE
```

### Durante a FASE 3 (Diagnóstico)
Declare a categoria definitiva:
```
Categoria RCA: RACE
→ Acionar prevention-patterns.md#RACE
```

### Após o incidente
Registre no postmortem e no ticket:
```
Root Cause Category: RACE
Prevention items: 6 actions added to backlog
Recurrence risk: HIGH (sem lock distribuído ainda)
```

---

## Sinais rápidos por categoria

**CODE** → stack trace aponta para linha específica do seu código
**DEPLOY** → bug começou exatamente após o último deploy; rollback resolve
**INFRA** → kubectl describe mostra OOMKilled, Evicted, ou node pressure
**DATA** → inconsistência existe no banco, não no código
**CONFIG** → funciona em local/staging, falha apenas em prod
**EXTERNAL** → error vem de serviço fora do seu controle
**RACE** → bug aparece sob carga ou em janelas de tempo específicas
**SECURITY** → acesso a dados de outros usuários, secret em log
**PERF** → memória cresce continuamente, GC Gen2 subindo, CPU sem crash
**QUEUE** → consumer lag crescendo, partição travada, mensagens acumulando
**BUSINESS** → valor calculado errado, estado de negócio inválido, caso de borda não tratado

---

## Árvore de Decisão Rápida

```
Problema começou após deploy? ────── SIM ──→ DEPLOY (verifique rollback primeiro)
        │
       NÃO
        │
Envolve cálculo / estado de negócio? → SIM ──→ BUSINESS
        │
       NÃO
        │
Memória crescendo ao longo do tempo? → SIM ──→ PERF
        │
       NÃO
        │
Kafka/RabbitMQ com lag? ─────────── SIM ──→ QUEUE
        │
       NÃO
        │
Erro intermitente sob carga? ────── SIM ──→ RACE
        │
       NÃO
        │
5xx / crash / exception? ──────────────────→ EXCEPTION + {CODE | INFRA | CONFIG}
        │
Lentidão sem crash? ───────────────────────→ SLOW + {CODE | INFRA | EXTERNAL}
```
