# Códigos da apresentação

Todos os comandos e prompts usados na aula, prontos para copiar e colar — na mesma ordem dos slides.

---

## Passo 1 — Instalar o OpenClaude

```bash
npm install -g @gitlawb/openclaude@latest
```

Confirmar a instalação:

```bash
openclaude --version
```

> OpenClaude é um fork comunitário do Claude Code (não afiliado à Anthropic): [github.com/Gitlawb/openclaude](https://github.com/Gitlawb/openclaude). Se você usa o Claude Code oficial, troque `openclaude` por `claude` em todos os comandos.

---

## Passo 2 — Instalar o MCP do Power BI

```bash
openclaude mcp add powerbi-modeling-mcp -- npx -y @microsoft/powerbi-modeling-mcp@latest --start
```

Verificar (deve listar `powerbi-modeling-mcp ✓`):

```bash
openclaude mcp list
```

Já tinha instalado antes? Remova primeiro, no escopo em que estava:

```bash
# só este projeto
openclaude mcp remove powerbi-modeling-mcp -s local

# .mcp.json compartilhado do projeto
openclaude mcp remove powerbi-modeling-mcp -s project

# global, todas as sessões
openclaude mcp remove powerbi-modeling-mcp -s user
```

> Instalou pela extensão do VS Code? Desinstale a extensão, ou Command Palette → `MCP: Remove Server`, ou apague a entrada em `.vscode/mcp.json` e recarregue a janela — para não rodar duas instâncias do servidor.

---

## Passo 3 — Baixar as skills

```bash
git clone https://github.com/soutes/meetup_pbi.git
```

Copie as pastas desejadas de `skills/` para:

```text
%USERPROFILE%\.claude\skills\    (global — todas as sessões)
.claude\skills\                  (local — só neste projeto)
```

Depois, reinicie o OpenClaude.

---

## Passo 4 — Abrir o OpenClaude

```bash
openclaude --dangerously-skip-permissions
```

> **Atenção**: essa flag pula toda confirmação de leitura, escrita e execução — ação destrutiva roda sem chance de cancelar. Use só em ambiente controlado, com backup do projeto. Nunca em produção.

---

## Passo 5 — Configurar o provedor (OpenRouter)

Dentro do OpenClaude já aberto:

```text
/provider
```

1. Abra [openrouter.ai/keys](https://openrouter.ai/keys) e copie sua API key (formato `sk-or-...`)
2. No catálogo de modelos da OpenRouter, copie o *model slug* do modelo escolhido (ex.: `anthropic/claude-sonnet-4.6`)
3. No `/provider`, selecione OpenRouter e cole a key + o model slug quando pedido
4. Perfil salvo — as próximas sessões já abrem configuradas

---

## Prompt 1 — Conectar e analisar o modelo

```text
Use a skill /semantic-model-authoring pra conectar no arquivo .pbip aberto
no Power BI Desktop via powerbi-modeling-mcp (TOM ao vivo — não edite TMDL
na mão, o Desktop está aberto). Liste tabelas, colunas, medidas e
relacionamentos do modelo. Resuma os principais insights: tabelas
fato/dimensão, cardinalidade, medidas existentes.
```

---

## Prompt 2 — Time em ação (KPI + série temporal)

```text
Monte um time de 6 passos pra entregar só um KPI em HTML Content + um
visual de linha (série temporal), a partir do modelo já conectado:

1 — Lead, via powerbi-modeling-mcp direto (sem subagent, sem skill):
recomenda o indicador mais relevante pra evolução no tempo.

2 — Lead, via skill /powerbi-report-design: define o contrato visual —
paleta Binance (preto + dourado #F0B90B) pro card e pro gráfico de linha,
e o card HTML com flip 3D no hover: verso exibe uma dica de leitura do
indicador.

3 — Agente Power BI Visualization Expert Mode: revisa o contrato antes de
qualquer código — contraste, alinhamento, formatação do eixo de tempo.
Revisar aqui evita retrabalho: depois que o DAX nasce, mudança de contrato
custa caro.

4 — Agente Power BI DAX Expert Mode, via powerbi-modeling-mcp
(measure_operations + dax_query_operations): cria a medida base do
indicador e a medida HTML do KPI — DAX do valor + DAX do html/css com o
flip no hover e a dica do verso gerada a partir do próprio valor
(ex.: SWITCH por faixa), seguindo o contrato revisado no passo 3.

5 — Lead, via skill /powerbi-report-authoring: cria os dois visuais na
página ativa e conecta neles as medidas do passo 4.

6 — QA: Agente Power BI Visualization Expert Mode, agora conferindo o
resultado: valida o PBIR, recarrega o Desktop, tira screenshot e bate o
que renderizou contra o contrato — cor, contraste, eixo de tempo, card.
Divergiu? Volta pro passo 5. Nada é dado como pronto sem evidência visual.
```

Regra de ouro do fluxo: `validate → reload → screenshot → só então confirma`.
