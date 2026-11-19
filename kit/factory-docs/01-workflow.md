# Workflow da factory: da ideia ao PR mergeado

Este documento ensina o caminho completo de uma mudança neste projeto: o
alinhamento com você no chat (skills do Matt Pocock), a criação dos tickets no
GitHub e o fluxo autônomo da software factory até o PR.

```
 VOCÊ + CLAUDE (interativo)                         FACTORY (autônomo)
 ───────────────────────────                        ──────────────────────────────────────────
 1. /grill-with-docs   alinhar a mudança            6. /factory run
 2. /to-spec           1 issue  ─┐                     intake → spec → build → gate
    ou /to-tickets     N issues ─┤                     → reviews → approver
 3. conferir labels             ─┤  `factory` (livre) | `factory:blocked` (espera)    → merge  ou  factory:needs-human
 4. (/triage)          issues de terceiros ─┘       7. você decide o que escalou
                                                     8. depois: wiki, dívida, arquitetura
```

---

## 0. Antes de começar (uma vez por sessão de trabalho)

- Tudo o que os testes do projeto precisam rodando (Docker, banco...): sem isso o gate falha.
- Abra o Claude Code na raiz do projeto. Os plugins e MCPs carregam sozinhos:
  ponytail (regras de "senior preguiçoso"), TokenEconomizer (fatos aprendidos),
  o plugin LSP da linguagem (se instalado), CodeGraph, LLM Wiki e as skills do
  Matt Pocock.
- Opcional: `/tokeneconomizer:status` para ver o que ele já aprendeu.

---

## 1. Alinhar a mudança: `/grill-with-docs`

```
/grill-with-docs quero cadastrar categorias de despesa com nome e cor
```

O Claude te entrevista, uma rodada de perguntas por vez, até não sobrar decisão
em aberto: o que entra, o que fica de fora, regras de negócio, casos de borda.
Enquanto isso ele:

- cria/atualiza o **`GLOSSARY.md`** com os termos do domínio ("categoria",
  "lançamento", "orçamento"...), para que você, os agents e o código usem as
  mesmas palavras;
- registra decisões difíceis como **ADRs** em `docs/adr/` (ex.: "valores em
  `BigDecimal` com escala 2").

Use sempre, mesmo para mudanças pequenas: é o que mais evita o agente construir
a coisa errada. Variações úteis:

| Comando | Quando |
|---|---|
| `/grill-me` | Alinhar algo que não é código (plano, decisão). |
| `/ask-matt` | Não sabe qual skill usar: ele indica o fluxo. |
| `/wiki-ask` / `/wiki-research` | Consultar o que a base `.wiki/` já sabe antes de decidir. |
| `/prototype` | Uma dúvida de design que só um protótipo descartável responde. |

---

## 2. Transformar em issues: `/to-spec` ou `/to-tickets`

Com a conversa alinhada, publique no GitHub. Os dois aplicam o papel
`ready-for-agent`, que **neste repo é a label `factory`**
(`docs/agents/triage-labels.md`). Ou seja: **a issue já nasce na fila da
factory.**

### `/to-spec`: uma issue
Sintetiza a conversa numa spec e publica **uma** issue. Bom para mudanças que
cabem num PR.

### `/to-tickets`: várias issues fatiadas
Quebra o plano em tickets "tracer bullet" (fatias verticais, cada uma
entregável sozinha), publica um por issue, em ordem de dependência, e cria as
dependências nativas do GitHub ("blocked by"). Bom para features maiores.

> **Tickets com dependências.** A factory em si não entende "blocked by": ela
> pega até 3 issues com `factory` em paralelo. Por isso `docs/agents/issue-tracker.md`
> instrui as skills a dar a label **`factory:blocked`** (e não `factory`) a todo
> ticket com bloqueio aberto, registrando os pais como dependências nativas do
> GitHub. Um ticket pode depender de vários pais.
>
> O script **`factory/bin/unblock`** troca `factory:blocked` por `factory`
> quando **todos** os pais estão fechados como concluídos. Ele roda:
> - no início de cada `/factory run` e depois de cada merge feito pela factory
>   (o filho liberado entra na mesma execução);
> - no GitHub Action `factory-unblock`, sempre que uma issue fecha (cobre o
>   caso de você mergear um PR escalado direto no GitHub).
>
> Se um pai for fechado como "not planned" ou duplicado, o filho **não** é
> liberado: o script avisa e a decisão é sua.

Para algo muito grande (mais do que uma sessão aguenta), use `/wayfinder`, que
cria um "mapa" de decisões no GitHub e as resolve uma a uma antes de virar
tickets.

---

## 3. Conferir o que entra na fila

```bash
gh issue list --label factory
```

- Ticket **bloqueado** com `factory` por engano? Troque:
  `gh issue edit <n> --remove-label factory --add-label factory:blocked`.
- Ver quem está esperando o quê: `factory/bin/unblock --dry-run`.
- Ticket grande demais para um PR? O intake da factory reduz o escopo sozinho,
  mas é melhor fatiar antes.
- Para começar com cautela: `concurrency: 1` em `factory/config.json`.

## 4. Issues que não vieram de você: `/triage`

Issues abertas por outras pessoas (ou suas, escritas às pressas) passam pelo
`/triage`, que as move pelos papéis `needs-triage` → `needs-info` /
`ready-for-agent` (= `factory`) / `ready-for-human` / `wontfix`. Quando falta
algo, ele te entrevista e escreve um "agent brief" na issue. Marcar como
`ready-for-agent` é o que manda para a factory.

Atalho sem skills: `/factory <descrição da feature>` cria a issue com `factory`
e já a executa.

---

## 5. O que é a label `factory`

É a fronteira de confiança: só issues com essa label são construídas, e só
quem tem acesso de escrita/triagem no repo pode aplicá-la.

| Label | Significa | Quem aplica |
|---|---|---|
| `factory` | Pronto: entra na fila | Você, `/to-spec`, `/to-tickets`, `/triage`, `unblock` |
| `factory:blocked` | Pronto, mas esperando pais | `/to-tickets` (ou você) |
| `factory:needs-info` | Intake perguntou algo ao autor | A factory |
| `factory:needs-human` | PR escalado para você | A factory |

---

## 6. Fluxo autônomo: `/factory run`

```
/factory run        # todas as issues abertas com `factory`, até 3 por vez
/factory run 12     # só a issue #12
```

A sessão que roda o comando vira o orquestrador. Para cada issue:

| Etapa | Quem (modelo) | O que acontece |
|---|---|---|
| **Intake** | `intake` (Fable) | Lê a issue, consulta o `AGENTS.md`, o código, o wiki e o CodeGraph. Decide **ACCEPT** (talvez com escopo reduzido), **NEEDS-INFO** (pergunta na issue) ou **REJECT**. Classifica **risco** e **superfícies** (`ui` = contrato REST/gRPC/CLI; `security`). |
| **Route** | script | Escolhe os revisores pelo risco e pelas superfícies (`factory/config.json`). |
| **Worktree** | script | Cria `.worktrees/<job>` no branch `factory/<job>` a partir de `origin/master`. |
| **Spec** | `spec-writer` (Opus) | Escreve `spec.md` com critérios de aceite verificáveis por teste ou `curl`/`grpcurl`. Usa o glossário (`domain-modeling`) e `codebase-design`. Mudança de schema vira migration nova. |
| **Build** | `builder` (Opus) | Lê os fatos aprendidos (`te microskills`), constrói cada critério com **TDD** (`tdd`), segue as regras do **ponytail** (reusar antes de escrever, nada além da spec), roda `spotlessApply`, commita, faz push e abre um **PR draft** "Closes #n". |
| **Gate** | script | Roda `factory/checks.sh`: as verificações do projeto (formatação, lint, tipos, testes, build). Também bloqueia mudança em arquivos protegidos (`protected_paths`: `checks.sh`, `config.json` e o que mais o projeto proteger). Falhou → volta ao builder com o log. |
| **Reviews** | em paralelo | `reviewer-code` sempre (Haiku no risco baixo, Sonnet no médio/alto), via `ponytail-review` + CodeGraph (chamadores, testes relacionados). `reviewer-security` (Opus) se a superfície é `security`. `reviewer-ux` (Sonnet) se é `ui` (contrato de API/CLI). Algum pede mudança → nova rodada (máx. 3). |
| **Approver** | `approver` (Opus) | Confere os critérios, mede o alcance com o CodeGraph. **APPROVE** → sincroniza com `master`, roda o gate de novo e faz o merge. **ESCALATE** → marca o PR como pronto, comenta a decisão que precisa de você e aplica `factory:needs-human`. |

O approver **sempre escala** quando a mudança mexe em: auth/permissões,
dinheiro, schema do banco, CI/deploy/hosting, dependências novas, e o que mais
a lista ESCALATE de `.claude/agents/approver.md` deste projeto definir.

### Acompanhar

```bash
cd factory && python3 -m http.server   # abra http://localhost:8000/dashboard.html
```

Ou pergunte no chat: `/factory como está o job 3?`, `/factory quanto custou?`.

---

## 7. Quando a factory precisa de você

```
/factory o que está esperando por mim?
```

| Situação | O que fazer |
|---|---|
| `unblock` diz que um ticket "stays blocked" porque um pai foi fechado sem ser feito | Decida: troque a label para `factory` (seguir assim mesmo), reabra o pai, ou feche o ticket. |
| **`factory:needs-info`**: o intake perguntou algo na issue | Responda na issue, ou `/factory responda a issue 12: <resposta>`. O job retoma. |
| **`factory:needs-human`**: PR escalado | Leia o comentário do approver no PR. Concorda → faça o **merge no GitHub**. Quer mudanças → `/factory no job 12, <o que mudar>` (nova rodada). |
| Não quer mais | `/factory rejeite o job 12, <motivo>`. Fecha o PR e tira as labels; a issue fica aberta. |
| Loop guard avisou que um turno seu está preso | Decida se interrompe; ele nunca para sozinho. |

---

## 8. Depois do merge

| Comando | Para quê |
|---|---|
| `/wiki-ingest <arquivo>` | Alimentar a base `.wiki/` com conhecimento (docs de domínio, decisões, docs de APIs externas). Os agents leem o wiki, mas **só você escreve** nele. |
| `/ponytail-debt` | Listar os `// shortcut:` que o builder deixou para depois. |
| `/ponytail-audit` | Auditoria do repo inteiro, priorizada. |
| `/improve-codebase-architecture` | Encontrar oportunidades de "módulos profundos"; rode a cada poucos dias. |
| `/diagnosing-bugs` | Bug difícil: loop disciplinado reproduzir → minimizar → hipótese → corrigir → teste de regressão. |
| `/retro` | Melhorias no ambiente dos agents depois de uma sessão. |
| `te toolprune` | Ver se o TokenEconomizer propõe remover ferramentas; aplique só as que **nenhum** agent em `.claude/agents/` usa. |

---

## Exemplo completo

```
você> /grill-with-docs quero categorias de despesa (nome e cor) com CRUD REST
claude> (10 perguntas: cor em hex? nome único por usuário? apagar com lançamentos?...)
        → GLOSSARY.md: "Categoria"; ADR 0002: categoria pertence ao usuário
você> /to-tickets
claude> #4 Criar Categoria (entity, migration V2, POST)        [factory]
        #5 Listar e editar Categorias (blocked by #4)            [factory:blocked]
        #6 Relatório por Categoria (blocked by #4 e #5)          [factory:blocked]
você> /factory run
factory> #4 intake ACCEPT (risk high: migration) → spec → build (3 testes) → gate PASS
         → reviewer-code PASS, reviewer-ux PASS → approver ESCALATE (migration V2)
você> (lê o PR, faz merge no GitHub → #4 fecha)
GitHub Action factory-unblock> #5 queued: #4 done
                               #6 stays blocked: waiting on #5
você> /factory run
factory> #5 ... APPROVE → merge → unblock: #6 queued: #4, #5 done → #6 ... na mesma execução
```
