# Plano de Uso Otimizado do Hermes Agent

Objetivo: extrair melhores resultados do Hermes em quatro eixos — **qualidade das
respostas**, **autonomia** (ele trabalha sem você pedir), **custo por resultado**
e **ubiquidade** (disponível onde você está). O plano é incremental: cada fase
funciona sozinha e prepara a seguinte.

Princípio que atravessa tudo (vem do design do próprio Hermes): **o cache de
prompt por conversa é sagrado**. Conversas longas reutilizam prefixo cacheado a
cada turno; trocar toolset, recomeçar conversa sem necessidade ou inflar o
system prompt multiplica o custo. Quase toda recomendação abaixo deriva disso.

---

## Fase 0 — Diagnóstico (30 minutos)

Antes de otimizar, medir o ponto de partida:

```bash
hermes doctor            # sanidade da instalação
hermes portal info       # o que está roteado pelo Nous Portal (se usar)
```

Dentro de uma conversa:

- `/usage` — consumo da sessão atual
- `/insights --days 30` — padrão de uso do último mês (baseline de custo)

Anote: custo mensal, modelo em uso, quais ferramentas estão habilitadas
(`hermes tools`). É contra isso que as fases seguintes serão comparadas.

## Fase 1 — Fundação: provider, modelos e toolsets

1. **Provider unificado ou chaves próprias.** `hermes setup --portal` cobre
   modelo, busca web, geração de imagem, TTS e browser em nuvem com uma
   assinatura só. Alternativa: chaves próprias por backend (o gateway de
   ferramentas é por-backend, dá para misturar).
2. **Dois modelos, dois papéis.** Um modelo forte como padrão da conversa
   principal; um modelo barato para cron jobs e subagentes (tarefas mecânicas
   não precisam do modelo caro). Troca sem atrito com `/model provider:modelo`.
3. **Enxugar toolsets.** `hermes tools` e desabilitar o que não usa. Cada tool
   habilitada vai em **toda** chamada de API — toolset enxuto = menos tokens
   fixos por turno e modelo menos disperso. Importante: escolher o toolset
   **antes** de começar a conversa e não trocar no meio (invalida o cache).
4. **Higiene de contexto.** Preferir `/compress` a `/new` quando o histórico
   ainda importa; `/new` só quando o assunto realmente muda. `/undo` e `/retry`
   para corrigir um turno ruim em vez de re-explicar.

## Fase 2 — Memória e contexto persistente

O diferencial do Hermes é aprender entre sessões — mas só rende se for
alimentado e curado:

1. **SOUL.md** — persona e tom. Definir uma vez como o agente deve se
   comportar (idioma, estilo, o que nunca fazer).
2. **MEMORY.md / USER.md** — deixar os *nudges* periódicos de persistência
   ligados e, ao final de decisões importantes, pedir explicitamente:
   *"guarda isso na memória"*. O que não é registrado é re-explicado (e pago)
   de novo.
3. **Context files (AGENTS.md) por projeto** — cada diretório de trabalho com
   seu arquivo de contexto: o que é o projeto, convenções, comandos de
   build/teste. Isso molda toda conversa naquele diretório sem custo de
   repetição.
4. **Busca de sessões passadas** — o Hermes indexa conversas com FTS5 +
   sumarização. Usar ativamente: *"procura nas nossas conversas anteriores
   sobre X"* antes de re-resolver um problema já resolvido.
5. **Curadoria mensal** — revisar MEMORY.md/USER.md e apagar o que envelheceu.
   Memória suja degrada resposta tanto quanto memória vazia.

## Fase 3 — Skills: fechar o loop de aprendizado

Skills são memória procedural — o mecanismo pelo qual o agente melhora de
verdade com o uso:

1. **Capturar após tarefas complexas.** Quando uma tarefa de várias etapas der
   certo, pedir: *"cria uma skill disso"*. A criação autônoma também existe —
   deixá-la ligada e apenas revisar o resultado.
2. **Curar o catálogo.** `/skills` para ver o que existe; apagar skills ruins,
   pedir refinamento das boas (elas se auto-melhoram durante o uso, mas
   feedback explícito acelera).
3. **Não reinventar.** Antes de criar skill do zero, olhar o Skills Hub
   (agentskills.io) e o diretório `optional-skills/` do repositório —
   categorias prontas de creative, research, productivity, devops, email,
   finance etc. Instalar as 3–5 mais próximas do seu fluxo real.
4. **Meta de reuso.** Uma skill que nunca é reutilizada é ruído. Na revisão
   mensal, manter só o que dispara de verdade.

## Fase 4 — Automação com cron

Transformar tarefas recorrentes em jobs que rodam sozinhos, com entrega em
qualquer plataforma conectada:

```bash
hermes cron list / create / runs / status
```

Candidatos de maior retorno:

- **Brief diário** (manhã): agenda, pendências, resumo do que importa — entregue
  no Telegram.
- **Backup noturno** do estado do Hermes (`hermes backup`).
- **Auditoria semanal**: revisar memória e skills criadas na semana e propor
  limpeza (o próprio job pode fazer isso em linguagem natural).
- **Relatório de custo semanal**: rodar `/insights` e mandar o resumo.

Regra de custo: cron jobs rodam no **modelo barato** definido na Fase 1, salvo
os que exigem raciocínio pesado.

## Fase 5 — Ubiquidade: gateway de mensagens

O agente rende mais quando o custo de falar com ele é zero:

1. `hermes gateway setup` + `hermes gateway start` — começar pelo **Telegram**
   (setup mais simples, suporta voice memo com transcrição).
2. Continuidade entre plataformas é nativa: começar no desktop, continuar do
   celular na rua.
3. **Segurança primeiro**: DM pairing, allowlist de usuários e aprovação de
   comandos ligadas antes de expor o bot. `/status` e `/sethome` para
   administrar pela própria plataforma.

## Fase 6 — Delegação e paralelismo

Duas ferramentas que protegem o contexto (e o custo) da conversa principal:

1. **Subagentes** (`delegate_task`): trabalho exploratório, pesquisas longas ou
   frentes paralelas rodam isoladas e voltam só com a conclusão — o histórico
   principal não incha.
2. **Scripts com RPC de tools** (`execute_code`): pipelines de muitas etapas
   (ex.: baixar → transformar → salvar → notificar) escritos como um script
   Python que chama as tools via RPC — colapsa dezenas de turnos em um, com
   custo de contexto próximo de zero.

Hábito a adquirir: ao pedir algo grande, dizer explicitamente *"delega o que
der para subagentes"* ou *"resolve isso num script só"*.

## Fase 7 — Execução resiliente (opcional, maior alavancagem)

- **Isolamento**: backend Docker para o agente operar com liberdade sem risco à
  máquina pessoal.
- **Agente 24/7 barato**: Daytona ou Modal com persistência serverless — o
  ambiente hiberna quando ocioso e acorda sob demanda; combinado com o gateway
  (Fase 5), vira um agente sempre disponível por quase nada. Um VPS de $5
  cumpre o mesmo papel na versão sempre-ligada.
- Com isso, cron jobs (Fase 4) deixam de depender do laptop aberto.

---

## Rotina de melhoria contínua

| Frequência | Ação |
| ---------- | ---- |
| Semanal | `/insights`; revisar skills criadas na semana; conferir execuções de cron (`hermes cron runs`) |
| Mensal | Curar MEMORY.md/USER.md; podar skills sem reuso; `hermes update`; sincronizar o fork com o upstream (`NousResearch/hermes-agent`) |
| Contínuo | Registrar decisões na memória; capturar skills após tarefas complexas; usar busca de sessões antes de re-resolver |

**Métricas de sucesso** (comparar com o baseline da Fase 0):

- Custo por semana estável ou menor, com mais tarefas concluídas.
- Tarefas recorrentes migradas para cron (meta: 3+ jobs ativos).
- Skills com reuso real (meta: 5+ skills disparando regularmente).
- Menos re-explicação: o agente lembra contexto de projeto e preferências sem
  ser lembrado.

## Checklist de implantação

- [ ] Fase 0: baseline registrado (`hermes doctor`, `/insights --days 30`)
- [ ] Fase 1: provider definido; modelo forte + modelo barato; toolsets enxutos
- [ ] Fase 2: SOUL.md escrito; AGENTS.md nos projetos ativos; nudges de memória ligados
- [ ] Fase 3: 3–5 skills instaladas/criadas para o fluxo real
- [ ] Fase 4: brief diário + backup noturno + auditoria semanal no cron
- [ ] Fase 5: gateway no Telegram com pairing e allowlist
- [ ] Fase 6: hábito de delegar (subagentes) e de pedir pipelines em script
- [ ] Fase 7: backend isolado/serverless avaliado
- [ ] Rotina semanal/mensal no calendário (ou como cron job do próprio Hermes)
