# Slide 17 — Medidas + Visuais (continuação do slide 16)

Você é especialista Power BI. Continuação direta do slide 16: **já está conectado ao modelo e o schema já foi
exibido**. NÃO liste conexões, NÃO rode INFO.COLUMNS/INFO.TABLES, NÃO mostre tabela de schema de novo.
Se precisar de um nome de coluna, use o que já está no contexto.

Execute até o fim, sem pedir confirmação. Pare só se faltar informação estrutural.

## SKILLS OBRIGATÓRIAS
- Medidas / modelo → Skill: `0.3.6:semantic-model-authoring`
- Visuais / relatório → Skill: `0.3.6:powerbi-report-authoring`

Invoque a skill ANTES de agir na fase. Não improvise chamadas MCP fora do que a skill manda.

## REGRA DE FERRAMENTA
Permitido: `connection_operations`, `measure_operations`, `dax_query_operations`, `database_operations`.
PROIBIDO: `table_operations`, `column_operations` (quebram nesse modelo).

Modelo (TMDL) e relatório (PBIR) são arquivos diferentes:
- Medida salva com `database_operations` → `ExportToTmdlFolder`
- Visual salva com a CLI `powerbi-report-author`
- Nunca use ExportToTmdlFolder para salvar visual.

---

## FASE 1 — MEDIDAS (Skill: semantic-model-authoring)

6 medidas a partir do schema já conhecido: 4 KPIs + 1 para barras + 1 para série temporal.
Justifique cada uma em 1 linha.

formatString conforme natureza (não use um formato único):
- moeda → `"R$ #,##0.00"`
- contagem → `"#,##0"`
- ratio → `"0.0%"`

Para CADA medida, uma por vez:
- A) `measure_operations` → `Create` { tableName, name, expression, description, formatString }
- B) `dax_query_operations` → `Validate` com `EVALUATE ROW("t",[<medida>])`
     Erro → mostre o texto exato, corrija, revalide. Não avance com erro.
- C) `database_operations` → `ExportToTmdlFolder`
- D) Reporte: "Medida criada e salva. i/6."

DAX: SUM/AVERAGE/COUNT conforme o tipo. Nunca hardcode datas. Comparação temporal: SAMEPERIODLASTYEAR.
Ranking: TOPN + ALL.

Fim da fase: `measure_operations` → `List` confirmando as 6.

---

## FASE 2 — VISUAIS (Skill: powerbi-report-authoring)

Alvo: primeira página do relatório na pasta PBIP. Sem página → crie. 2+ relatórios → liste e use o primeiro,
informando qual.

6 visuais nativos, UM POR VEZ. Após cada um: salve via CLI, valide o PBIR, reporte
"Visual criado e validado. n/6."

1-4. Card nativo (`cardVisual`), um por KPI. 230x160px, lado a lado no topo.
5. `clusteredBarChart`. Eixo: coluna de categoria (Top 10). Valor: medida de barras.
   Abaixo dos cards, à esquerda, 50% da largura x 300px.
6. `lineChart`. Eixo X: coluna de data (formato "MMM YYYY"). Valor: medida temporal.
   À direita, 50% x 300px.

### REGRAS DE FORMATAÇÃO PBIR — obrigatórias (erros reais da execução anterior: 71)

1. `visualContainerObjects` fica **dentro** de `visual`, irmão de `objects`.
   Nunca na raiz do `visual.json`. (erro `PBIR_VISUAL_VCO_AT_ROOT`, 6 ocorrências)

2. Só use propriedades que existem. NÃO invente. Nomes corretos:
   | Errado (quebrou antes)            | Correto             |
   |-----------------------------------|---------------------|
   | `categoryAxis.fontColor`          | `categoryAxis.labelColor`    |
   | `valueAxis.fontColor`             | `valueAxis.labelColor`       |
   | `categoryAxis.lineColor`          | `categoryAxis.gridlineColor` |
   | `valueAxis.lineColor`             | `valueAxis.gridlineColor`    |
   | `labels.fontColor`                | `labels.color`               |

3. NÃO existem e não devem ser escritas:
   `general.showTitle`, `padding.top/right/bottom/left` (cardVisual),
   `markers.show/size/shape` (lineChart), `border.transparency` (VCO),
   `categoryAxis.gridline`, `valueAxis.gridline`, `stylePreset` em objects.
   Título de visual: use `visualContainerObjects.title`, não `general.showTitle`.

4. Antes de escrever qualquer bloco de formatação novo, consulte a referência de propriedades da skill
   `powerbi-report-authoring`. Se a propriedade não estiver documentada lá, não escreva.

5. Valide com `powerbi-report-author validate <caminho>.Report --format text --no-schema` depois de CADA visual.
   `--no-schema` evita o warning `PBIR_SCHEMA_UNREACHABLE` (schema remoto offline).
   Qualquer erro → mostre o texto exato, corrija, revalide. Só então passe para o próximo visual.

### Tema
Aplicar via theme JSON da skill, UMA vez — não visual a visual.
No theme JSON valem as mesmas regras: sem `effects` em `*`, sem `categoryLabels`/`wordWrap` em `cardVisual`,
sem `header`/`items` em `advancedSlicerVisual`, sem `*.gridline`.

fundo card `#000000` | valor `#F0B90B` | label `#B0B0B0`
borda `#F0B90B` 40% | barras/linha `#F0B90B`
fundo gráfico `#1A1A1A` | gridlines `#333333` | Segoe UI

---

## FINAL
`database_operations` → `ExportToTmdlFolder` (estado final do modelo).
Validação final do PBIR: `powerbi-report-author validate <caminho>.Report --format text --no-schema`
→ deve dar `0 error(s)`.
Confirme: "4 KPI cards + barras + série temporal criados e salvos. Abra no Power BI Desktop para conferir."
