# 🔍 Sherlock — Distributed Systems Bug Investigator

> Uma skill para Claude que investiga bugs em sistemas distribuídos como um colega sênior — em rounds iterativos, com confidence score e prevention checklist.

**Autor:** Helio Andrade ([@josehelioaraujo](https://github.com/josehelioaraujo))
**Versão:** 1.0.0 | **Prazo TLC:** 06/11/2026

---

## Qual problema resolve

Você está num incidente às 2h da manhã. O sistema está fora. Você não sabe por onde começar.

Ou você tem um bug flaky que aparece uma vez por semana e some. Ou um pagamento que duplicou e você não sabe se foi o código, o Kafka, o Redis ou o deploy de ontem.

**Sherlock resolve isso.**

Você descreve o problema em linguagem natural — ou cola o log, ou passa o commit, ou a URL da PR — e o Sherlock começa a investigar como um colega experiente:
- Faz triagem automática do tipo de sintoma
- Gera hipóteses rankeadas por probabilidade com confidence score
- Pede evidências específicas em rounds iterativos
- Atualiza as hipóteses a cada dado novo
- Entrega: causa raiz exata, arquivo e linha, blast radius, timeline e prevention checklist

---

## Como usar

### 1. Instalação

**Claude Code (qualquer projeto):**
```bash
# Instalação rápida via npx
npx sherlock-skill

# Ou manualmente
mkdir -p .claude/skills/sherlock
curl -sSL https://raw.githubusercontent.com/josehelioaraujo/sherlock-skill/main/install.sh | bash
```

**Claude.ai Projects:**
Faça upload de `skill/SKILL.md` e da pasta `skill/references/` como documentos do projeto.

**Via API (integração em dashboard):**
Use `skill/SKILL.md` como system prompt para qualquer LLM (Claude, Groq, Ollama, GPT-4).

---

### 2. Exemplos de prompt

**Descrição livre:**
```
descubra a causa do erro
[cole o stack trace aqui]
```

**Com commit:**
```
o pagamento duplicado começou após o deploy de ontem, commit a3f2c1d
[log mostrando o duplicado]
```

**Com PR:**
```
identifique o que a PR #312 introduziu que causou a lentidão
```

**Código direto:**
```
identifique o bug nesse código:
[cole o trecho]
```

**P0 — sistema fora agora:**
```
--express: API completamente fora, started após deploy de 10min atrás
[último erro do log]
```

---

### 3. O que você recebe

```
🔍 Triagem         → tipo detectado, serviços, stack
📊 Hipóteses       → rankeadas por confidence score (%)
🔬 Rounds          → pedidos de evidência específicos
🎯 Diagnóstico     → causa raiz, arquivo:linha, blast radius
📅 Timeline        → alimenta o postmortem
✅ Checklist       → prevenção por categoria RCA
📋 Ticket Update   → texto pronto para Linear/Jira
```

---

### 4. Tipos de sintoma detectados automaticamente

| Tipo         | Quando usar                                          |
|--------------|------------------------------------------------------|
| EXCEPTION    | Stack trace, pod crashando, HTTP 5xx                 |
| SLOW         | Latência, timeout, CPU alta                          |
| DUPLICATE    | Registro duplicado, operação executada 2x            |
| WRONG_DATA   | Valor incorreto, status desatualizado                |
| CASCADE      | Múltiplos serviços falhando em sequência             |
| SECURITY     | Acesso não autorizado, dado de outro usuário         |
| FLAKY        | Erro intermitente, difícil de reproduzir             |

---

### 5. Stacks suportadas

.NET/C# · Node.js · Python · Go · Java
PostgreSQL · MySQL · MongoDB · Redis · Kafka · RabbitMQ
Docker · Kubernetes/AKS · AWS · Azure · GCP
Jaeger · Grafana · Prometheus · Loki · Datadog · New Relic

---

## Como evoluiu

### v0.1 — Prompt simples (não funcionou bem)
A primeira versão era um prompt que tentava resolver o bug em uma única resposta.
**Problema:** sem evidências reais, o Claude chutava. Confidence score era decorativo.

### v0.2 — Rounds mas sem estrutura
Adicionei rounds iterativos mas sem o ciclo de atualização de hipóteses.
**Problema:** o Claude perdia o fio da investigação entre rounds.

### v0.3 — Hipóteses explícitas com score
Introduzi o formato de hipóteses rankeadas e a atualização visual (tachado nas descartadas).
**O que mudou:** o Claude passou a raciocinar em voz alta, mostrando por que descartava cada hipótese.

### v1.0 — Protocolo completo (versão atual)
- **5 fases** bem delimitadas: Triagem → Hipóteses → Evidências → Diagnóstico → Timeline → Prevention
- **Modo --express** para P0/P1 (sistema fora agora — 3 perguntas, 1 ação imediata)
- **Blast radius** mapeado no diagnóstico
- **Output para Linear/Jira** — texto de update do ticket pronto
- **Integração com código:** detecção automática de commits/PRs e análise do diff
- **References modulares:** playbooks, comandos de infra, categorias RCA e prevention patterns como arquivos separados — fácil de atualizar sem mudar o protocolo principal

**O que não funcionou:**
- Tentar forçar uma categoria de sintoma antes de ver os dados — o Claude tentava encaixar o problema numa caixa antes de investigar
- Pedir muitas evidências de uma vez — o usuário se perdia. Round de 1-3 comandos funciona melhor
- Confidence score em percentual exato — depois mudei para faixas ("alto/médio/baixo") mas voltei ao percentual porque é mais informativo visualmente

---

## Demo ao vivo

Implementação real integrada ao **OpsWatch** do projeto [Comprai](https://github.com/josehelioaraujo/comprai):
- URL: `https://comprai.2.25.122.11.nip.io/opswatch/`
- Tab "🔍 Sherlock": textarea livre + investigação em rounds + postmortem export

Caso de uso completo documentado em `dump.md`.

---

## Estrutura do repositório

```
sherlock-skill/
├── skill/
│   ├── SKILL.md                         # protocolo principal (obrigatório)
│   └── references/
│       ├── symptom-playbooks.md         # 7 tipos de sintoma + hipóteses padrão
│       ├── infra-commands.md            # comandos prontos: Docker/K8s/PG/Redis/Kafka
│       ├── rca-categories.md            # 7 categorias + matriz de frequência
│       └── prevention-patterns.md       # checklists de prevenção por categoria
├── README.md
└── dump.md                              # investigação real no Comprai
```

---

## Instalação detalhada para integradores

Se quiser embedar Sherlock como API em seu próprio dashboard:

```csharp
// .NET — busca o SKILL.md do repo público e usa como system prompt
var skillUrl = "https://raw.githubusercontent.com/josehelioaraujo/sherlock-skill/main/skill/SKILL.md";
var systemPrompt = await httpClient.GetStringAsync(skillUrl); // cache no Redis por 1h

var response = await groqClient.ChatCompletion(new {
    model = "llama-3.3-70b-versatile",
    messages = new[] {
        new { role = "system", content = systemPrompt },
        new { role = "user",   content = userInput }
    },
    stream = true
});
```

Template completo disponível em: `integration-template/` (em breve no repo público).

---

## Licença

MIT — use, modifique e distribua livremente.
Se usar em produção, uma estrela no repo é sempre bem-vinda. ⭐
