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

## Prompt 2 — Medidas + visuais

Continuação direta do Prompt 1: o modelo **já está conectado** e o schema **já
foi exibido**. Este prompt não repete nenhum dos dois.

Versão completa também em [`prompts/slide17.md`](prompts/slide17.md).

```text
Você é especialista Power BI. Continuação direta do prompt anterior: já está
conectado ao modelo e o schema já foi exibido. NÃO liste conexões, NÃO rode
INFO.COLUMNS/INFO.TABLES, NÃO mostre tabela de schema de novo. Se precisar de
um nome de coluna, use o que já está no contexto.

Execute até o fim, sem pedir confirmação. Pare só se faltar informação
estrutural.

SKILLS OBRIGATÓRIAS
- Medidas / modelo → skill /semantic-model-authoring
- Visuais / relatório → skill /powerbi-report-authoring
Invoque a skill ANTES de agir na fase. Não improvise chamadas MCP fora do que
a skill manda.

REGRA DE FERRAMENTA
Permitido: connection_operations, measure_operations, dax_query_operations,
database_operations.
PROIBIDO: table_operations, column_operations (quebram nesse modelo).
Modelo (TMDL) e relatório (PBIR) são arquivos diferentes:
- Medida salva com database_operations → ExportToTmdlFolder
- Visual salva com a CLI powerbi-report-author
- Nunca use ExportToTmdlFolder para salvar visual.

FASE 1 — MEDIDAS (skill /semantic-model-authoring)
6 medidas a partir do schema já conhecido: 4 KPIs + 1 para barras + 1 para
série temporal. Justifique cada uma em 1 linha.
formatString conforme a natureza (não use um formato único):
  moeda → "R$ #,##0.00"   contagem → "#,##0"   ratio → "0.0%"
Para CADA medida, uma por vez:
  A) measure_operations → Create
     { tableName, name, expression, description, formatString }
  B) dax_query_operations → Validate com EVALUATE ROW("t",[<medida>])
     Erro → mostre o texto exato, corrija, revalide. Não avance com erro.
  C) database_operations → ExportToTmdlFolder
  D) Reporte: "Medida criada e salva. i/6."
DAX: SUM/AVERAGE/COUNT conforme o tipo. Nunca hardcode datas. Comparação
temporal: SAMEPERIODLASTYEAR. Ranking: TOPN + ALL.
Fim da fase: measure_operations → List confirmando as 6.

FASE 2 — VISUAIS (skill /powerbi-report-authoring)
Alvo: primeira página do relatório na pasta PBIP. Sem página → crie.
2+ relatórios → liste e use o primeiro, informando qual.
6 visuais nativos, UM POR VEZ. Após cada um: salve via CLI, valide o PBIR,
reporte "Visual criado e validado. n/6."
  1-4. Card nativo (cardVisual), um por KPI. 230x160px, lado a lado no topo.
  5. clusteredBarChart. Eixo: coluna de categoria (Top 10). Valor: medida de
     barras. Abaixo dos cards, à esquerda, 50% da largura x 300px.
  6. lineChart. Eixo X: coluna de data (formato "MMM YYYY"). Valor: medida
     temporal. À direita, 50% x 300px.

REGRAS DE FORMATAÇÃO PBIR — obrigatórias
1. visualContainerObjects fica DENTRO de "visual", irmão de "objects".
   Nunca na raiz do visual.json.
2. Só use propriedades que existem. NÃO invente. Nomes corretos:
     categoryAxis.fontColor  → categoryAxis.labelColor
     valueAxis.fontColor     → valueAxis.labelColor
     categoryAxis.lineColor  → categoryAxis.gridlineColor
     valueAxis.lineColor     → valueAxis.gridlineColor
     labels.fontColor        → labels.color
3. NÃO existem e não devem ser escritas: general.showTitle,
   padding.top/right/bottom/left (cardVisual), markers.show/size/shape
   (lineChart), border.transparency (VCO), categoryAxis.gridline,
   valueAxis.gridline, stylePreset dentro de objects.
   Título de visual: use visualContainerObjects.title.
4. Antes de escrever qualquer bloco de formatação novo, consulte a referência
   de propriedades da skill /powerbi-report-authoring. Se a propriedade não
   estiver documentada lá, não escreva.
5. Valide depois de CADA visual:
     powerbi-report-author validate <caminho>.Report --format text --no-schema
   --no-schema evita o warning PBIR_SCHEMA_UNREACHABLE (schema remoto
   offline). Qualquer erro → mostre o texto exato, corrija, revalide. Só
   então passe para o próximo visual.

TEMA
Aplicar via theme JSON da skill, UMA vez — não visual a visual. No theme JSON
valem as mesmas regras: sem "effects" em "*", sem categoryLabels/wordWrap em
cardVisual, sem header/items em advancedSlicerVisual, sem *.gridline.
  fundo card #000000 | valor #F0B90B | label #B0B0B0
  borda #F0B90B 40% | barras/linha #F0B90B
  fundo gráfico #1A1A1A | gridlines #333333 | Segoe UI

FINAL
database_operations → ExportToTmdlFolder (estado final do modelo).
Validação final: powerbi-report-author validate <caminho>.Report
--format text --no-schema → deve dar 0 error(s).
Confirme: "4 KPI cards + barras + série temporal criados e salvos. Abra no
Power BI Desktop para conferir."
```

Regra de ouro do fluxo: `validate → reload → screenshot → só então confirma`.

> As regras de formatação acima não são teoria: sem elas a execução gerou 71
> erros de validação PBIR — `visualContainerObjects` na raiz, nomes de
> propriedade inventados e `border.transparency`.
