# RCA Categories

Categorias de causa raiz para classificar o diagnóstico final.
Use para direcionar o prevention checklist correto.

---

## Categorias

| Código    | Nome               | Descrição                                                      |
|-----------|--------------------|----------------------------------------------------------------|
| CODE      | Bug de Código      | Lógica incorreta, null reference, tipo errado, condição faltando |
| INFRA     | Infraestrutura     | OOMKilled, disco cheio, rede instável, pod crashando           |
| DATA      | Dados              | Migration incorreta, dado corrompido, constraint violada       |
| CONFIG    | Configuração       | Variável ausente, secret errado, feature flag incorreta        |
| EXTERNAL  | Dependência Externa| API terceira, DNS, CDN, serviço de pagamento instável          |
| RACE      | Race Condition     | Concorrência, idempotência, duplicate processing, lost update  |
| SECURITY  | Segurança          | Auth bypass, IDOR, secret vazado, misconfiguration             |

---

## Matriz de Frequência (referência)

Baseada em incidentes reais em sistemas distribuídos .NET/Node/Python em cloud:

| Categoria | Frequência | MTTR médio | Severidade típica |
|-----------|-----------|------------|-------------------|
| CODE      | ★★★★★     | 2–4h       | Alta              |
| RACE      | ★★★★☆     | 4–8h       | Crítica           |
| CONFIG    | ★★★★☆     | 30min      | Alta (fácil fix)  |
| INFRA     | ★★★☆☆     | 1–3h       | Alta              |
| EXTERNAL  | ★★★☆☆     | depende    | Variável          |
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
**INFRA** → kubectl describe mostra OOMKilled, Evicted, ou node pressure
**DATA** → inconsistência existe no banco, não no código
**CONFIG** → funciona em local/staging, falha apenas em prod
**EXTERNAL** → error vem de serviço fora do seu controle
**RACE** → bug aparece sob carga ou em janelas de tempo específicas
**SECURITY** → acesso a dados de outros usuários, secret em log
