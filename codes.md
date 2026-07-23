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

> Só necessário pra **Opção 2 — MCP ao vivo**, lá embaixo. Se for usar só a
> **Opção 1 — Bridge (sem MCP)**, pule este passo.

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

## Preparar o ambiente (antes de qualquer prompt)

1. Abrir o Power BI Desktop
2. Obter dados → conectar a base (é a fonte que vira o modelo semântico)
3. Salvar como projeto `.pbip` — Arquivo → Salvar como → Power BI Project
4. Fechar o Desktop — obrigatório pra **Opção 1 (Bridge)**; pra **Opção 2
   (MCP)**, reabra antes do prompt 1 dela

Duas opções de fluxo a partir daqui — escolha uma. Os dois entregam o mesmo
resultado (1 card KPI + 1 gráfico de barras); a diferença é a ferramenta e
se o Desktop fica aberto ou fechado durante a execução.

---

## Opção 1 — Bridge (sem MCP)

Sem MCP: leitura e escrita direta dos arquivos `.tmdl`. Power BI Desktop
fica **fechado desde o primeiro prompt** — o Bridge só entra no fim, pra
recarregar e tirar print.

### Prompt 1 — Descoberta e plano

```text
Você é engenheiro Power BI. Execute até o fim sem pedir confirmação. Existe
um projeto .pbip nesta pasta, com modelo e dataset desconhecidos — descubra
tudo agora, não assuma nomes de nenhum outro projeto.

Invoque a skill /semantic-model-authoring, caminho TMDL direto (Tier 2) —
NUNCA use powerbi-modeling-mcp neste fluxo, mesmo que esteja registrado.
Power BI Desktop deve estar FECHADO: aberto, ele sobrescreve qualquer
edição de TMDL feita em disco.

Leia SOMENTE arquivos de definição (*.tmdl, *.json em
<nome>.SemanticModel/definition/ e <nome>.Report/definition/) — nunca CSV,
Excel, cache.abf ou qualquer engine de query. Dados de linha não entram no
contexto da IA.

Liste tabelas, colunas (com tipo), medidas existentes e relacionamentos.
Resuma tabela(s) fato, dimensões, cardinalidade. Use só nomes lidos dos
arquivos — não invente.

Proponha exatamente 2 visuais nativos — 1 card KPI e 1 gráfico de barras —
com as medidas necessárias: name, expression DAX, formatString conforme a
natureza (moeda "R$ #,##0.00" | contagem "#,##0" | ratio "0.0%"),
displayFolder. Nomes de medida em ASCII puro, sem acento — o TMDL vai ser
editado à mão no próximo prompt e acento com encoding errado corrompe o
arquivo. Se já existir medida equivalente, reuse.

Grave o plano em `_status/plano.md`: schema resumido, tabela e arquivo .tmdl
alvo, os 2 visuais e as medidas propostas. Mostre o plano e pare — a criação
vem no próximo prompt.
```

### Prompt 2 — Execução

Continuação: o plano já está em `_status/plano.md`. Power BI Desktop
continua **FECHADO** durante toda a execução.

```text
Você é engenheiro Power BI. Execute até o fim sem pedir confirmação. Leia
_status/plano.md (se não existir, pare e peça pra rodar o prompt 1).
Confirme que o Desktop está fechado antes de tocar em qualquer .tmdl.

FASE 1 — MEDIDAS (skill /semantic-model-authoring, TMDL direto)
Leia tmdl-guidelines-pt-BR.md antes de editar. Liste medidas existentes —
NÃO crie duplicata (Desktop rejeita: "TMDL objects cannot be merged").
Insira cada medida do plano ANTES das colunas:
  /// Descricao da medida (ASCII, sem acento)
  measure 'Nome' = EXPRESSAO_DAX
    formatString: <formato>
    displayFolder: <pasta>
Siga a indentação exata do arquivo existente.

ENCODING — obrigatório, evita corromper o arquivo
- NUNCA grave .tmdl via shell (Set-Content/Out-File/echo) — pode gravar em
  ANSI. Use só a ferramenta de edição de arquivo nativa.
- Após CADA edição, valide UTF-8 antes de continuar. Não avance sem
  confirmação de encoding correto.

FASE 2 — VISUAIS (skill /powerbi-report-authoring)
Alvo: primeira página do relatório (crie se não houver).
Card KPI: x=32 y=32 w=360 h=200. Barras: x=32 y=264 w=920 h=480 — baseline
zero, ordenado descendente. Títulos = pergunta analítica; eixos e labels em
linguagem natural, nunca nome técnico de coluna.

Antes de escrever visual.json, leia authoring.md § Erros Comuns e
formatting.md. Não repita:
  - field sem wrapper Column/Measure/Aggregation (Desktop não abre o
    relatório)
  - objects.* como objeto solto — sempre array [{ "properties": {...} }]
  - literal sem sufixo D/L ("14pt", "8" cru), cor como string crua
  - filterConfig dentro de "visual" (mora na raiz, irmão de "visual")
  - position embrulhado em Literal (usa número direto: "x": 32)
Gere cada literal com:
  powerbi-report-author expr encode <valor> --kind <bool|number|integer|string|color>

FECHAMENTO
powerbi-report-author validate — 0 erros. PBIR_SCHEMA_UNREACHABLE significa
que a checagem de schema foi PULADA, não é sucesso: aponte $schema
temporariamente pra última versão publicada, revalide, corrija, restaure.
Instrua o usuário: "Abra o .pbip no Power BI Desktop pra conferir."
```

Regra de ouro do fluxo: `validate → Desktop fechado até o fim → abrir e conferir`.

> As regras de encoding e de formatação acima não são teoria: escritas
> errado, elas já corromperam TMDL gravado com acento em ANSI e derrubaram o
> load do relatório inteiro no Desktop.

---

## Opção 2 — MCP ao vivo

Via `powerbi-modeling-mcp`: o Power BI Desktop precisa estar **aberto**, com
o `.pbip` já carregado — o agente fala direto com o modelo em memória.
Requer o Passo 2 (instalar o MCP) lá em cima.

### Prompt 1 — Conectar e analisar

```text
Você é engenheiro Power BI. Execute até o fim sem pedir confirmação. Existe
um .pbip aberto no Power BI Desktop nesta máquina, com modelo e dataset
desconhecidos — descubra tudo agora, não assuma nomes de nenhum outro
projeto.

Invoque a skill /semantic-model-authoring via powerbi-modeling-mcp (TOM ao
vivo — Desktop ABERTO com o .pbip carregado). Use só INFO.TABLES(),
INFO.COLUMNS(), INFO.MEASURES(), INFO.RELATIONSHIPS() — nunca EVALUATE
sobre tabela ou coluna bruta. Dados de linha não entram no contexto da IA.

Liste tabelas, colunas (com tipo), medidas existentes e relacionamentos.
Resuma tabela(s) fato, dimensões, cardinalidade. Use só nomes retornados —
não invente.

Proponha exatamente 2 visuais nativos — 1 card KPI e 1 gráfico de barras —
com as medidas necessárias: name, expression DAX, formatString conforme a
natureza (moeda "R$ #,##0.00" | contagem "#,##0" | ratio "0.0%"),
displayFolder. Se já existir medida equivalente, reuse.

Grave o plano em `_status/plano.md`: schema resumido, tabela alvo, os 2
visuais e as medidas propostas. Mostre o plano e pare — a criação vem no
próximo prompt.
```

### Prompt 2 — Medidas + visuais

Continuação: o plano já está em `_status/plano.md`. Power BI Desktop
continua **ABERTO** durante toda a execução.

```text
Você é engenheiro Power BI. Execute até o fim sem pedir confirmação. Leia
_status/plano.md (se não existir, pare e peça pra rodar o prompt 1).

FASE 1 — MEDIDAS (skill /semantic-model-authoring, via powerbi-modeling-mcp)
Liste as medidas existentes (INFO.MEASURES()). NÃO crie duplicata.
Para cada medida do plano que ainda não existe:
  measure_operations → Create
    { tableName, name, expression, formatString, displayFolder }
  dax_query_operations → EVALUATE ROW("t",[<medida>])
  Erro → mostre o texto exato, corrija, revalide. Não avance com erro.
Ao final de TODAS: database_operations → ExportToTmdlFolder
(uma vez só, não por medida — grava o TMDL em UTF-8 correto).

FASE 2 — VISUAIS (skill /powerbi-report-authoring)
Alvo: primeira página do relatório (crie se não houver).
Card KPI: x=32 y=32 w=360 h=200. Barras: x=32 y=264 w=920 h=480 — baseline
zero, ordenado descendente. Títulos = pergunta analítica; eixos e labels em
linguagem natural, nunca nome técnico de coluna.

Antes de escrever visual.json, leia authoring.md § Erros Comuns e
formatting.md. Não repita:
  - field sem wrapper Column/Measure/Aggregation (Desktop não abre o
    relatório)
  - objects.* como objeto solto — sempre array [{ "properties": {...} }]
  - literal sem sufixo D/L ("14pt", "8" cru), cor como string crua
  - filterConfig dentro de "visual" (mora na raiz, irmão de "visual")
Gere cada literal com:
  powerbi-report-author expr encode <valor> --kind <bool|number|integer|string|color>

FECHAMENTO
powerbi-report-author validate — 0 erros. PBIR_SCHEMA_UNREACHABLE significa
que a checagem de schema foi PULADA, não é sucesso: aponte $schema
temporariamente pra última versão publicada, revalide, corrija, restaure.
powerbi-desktop reload → screenshot da página.
```

Regra de ouro do fluxo: `validate → reload → screenshot → só então confirma`.

> Diferença-chave pra Opção 1: aqui a medida valida na hora (`EVALUATE ROW`)
> e o Desktop recarrega sozinho no fim — sem precisar fechar nem reabrir o
> arquivo.
