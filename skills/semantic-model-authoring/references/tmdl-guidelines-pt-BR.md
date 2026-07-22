# Diretrizes TMDL (pt-BR)

Referencia unificada para Tabular Model Definition Language (TMDL) — regras de sintaxe, tipos de objetos, recursos avançados e boas práticas para criação de modelos semânticos do Power BI.

---

## Regras de Encoding de Arquivo (CRITICO)

Arquivos TMDL **devem** ser salvos com a codificação abaixo ou o Power BI Desktop **falhará** ao abrir o projeto.

| Propriedade | Valor obrigatório |
|-------------|-------------------|
| Codificação | UTF-8 **sem** BOM |
| Fim de linha | CRLF (`\r\n`) |
| Indentação | Tabs (não espaços) |

**Erros comuns que quebram o Desktop:**

| Escrito como | Erro no Desktop |
|-------------|-----------------|
| Com BOM (`EF BB BF`) | *"Only text with UTF8 encoding without BOM is supported"* |
| Windows-1252/Latin-1 (acentos `é`, `ç`, `ã` viram bytes únicos `E9`, `C7`, `C3`) | *"Cannot convert bytes [XX]"* |
| Apenas LF (`\n`) | Possíveis falhas de parsing |

**Como escrever corretamente em Python:**

```python
# Ler conteúdo existente (decodificar do encoding original)
with open(caminho, 'rb') as f:
    raw = f.read()

# Se o arquivo foi escrito em Windows-1252 (padrão do Windows pt-BR):
content = raw.decode('cp1252')

# Gravar como UTF-8 sem BOM, com CRLF:
with open(caminho, 'wb') as f:
    f.write(content.encode('utf-8').replace(b'\n', b'\r\n'))
```

**Como verificar:**

```python
with open(caminho, 'rb') as f:
    raw = f.read()
assert raw[:3] != b'\xef\xbb\xbf', "Arquivo tem BOM!"
assert raw.count(b'\n') == raw.count(b'\r\n'), "Arquivo tem LF-only!"
```

> **Nota:** Esta regra se aplica a TODOS os arquivos do projeto PBIP (`.tmdl`, `.json`, `.pbir`, `.pbism`). O Desktop escreve TMDL em UTF-8 com acentos (ex: `"Cabeçalhos promovidos"` no código M) e JSON em UTF-8 sem BOM.

---

## Regras de Sintaxe TMDL

- **TMDL usa indentação consistente para níveis aninhados** — use um estilo de indentação de forma consistente ao longo do arquivo; tabs são recomendadas para alinhar com o tooling do Power BI.
- Um objeto TMDL é declarado especificando o tipo do objeto TOM seguido do seu nome: `table Customer`, `column ProductId`, `measure 'Total Sales'`
- Objetos como partition ou measure têm propriedades padrão que podem ser atribuídas após o sinal de igual (`=`) que especificam a expressão PowerQuery ou DAX, respectivamente.
- Nomes com espaços ou caracteres especiais (`.`, `=`, `:`, `'`) devem ser envolvidos em **aspas simples**: `column 'Order Date'`
- Descrições usam `///` colocadas **acima** do objeto — não use a propriedade `description`:
  ```tmdl
  /// Receita por categoria de produto
  measure 'Total Sales' = SUM(Sales[Amount])
  ```
- Comentários `//` **não são suportados** no TMDL. Comentários podem estar dentro de expressões Power Query (M) ou blocos de código DAX.
- **Não adicione** a propriedade `lineageTag` ao criar novos objetos — Tags de linhagem são opcionais; ao criar novos objetos você pode omiti-las e o motor atribuirá GUIDs no primeiro salvamento.
- DAX multi-linha deve ser envolvido em crases triplas:
  ```tmdl
  measure 'Profit Margin' = ```
          DIVIDE(
              [Total Revenue] - [Total Cost],
              [Total Revenue]
          )
          ```
      formatString: 0.00%
  ```
- Coloque **medidas antes das colunas** nas definições de tabela
- `formatString` é obrigatório em toda medida
- Sempre aprenda com exemplos existentes e padrões no código (ex: convenções de nomenclatura existentes)

### Erros Comuns ao Inserir Medidas via Python (CRITICO)

**O bug mais comum:** ao usar Python para inserir medidas em um TMDL existente, o `///` commentário perde o tab de indentação e o Desktop rejeita o arquivo com *"Invalid indentation was detected!"*.

**Causa raiz:** usar string multiline com `\t` e depois inserir via `content.split('\n')` pode perder a indentação do primeiro elemento.

**❌ ERRADO — string multiline com `\t`:**
```python
measures = """
\t/// Minha descricao
\tmeasure 'Minha Medida' = SUM(Tabela[Coluna])
\t\tformatString: "#,##0"
"""
# Ao inserir, o \t do primeiro /// pode nao ser preservado
```

**✅ CORRETO — lista de linhas explicitas com tabs:**
```python
measure_lines = [
    '\t/// Minha descricao',
    "\tmeasure 'Minha Medida' = SUM(Tabela[Coluna])",
    '\t\tformatString: "#,##0"',
]
# Cada linha ja tem o tab correto como caractere literal
```

**Regra de ouro:** ao escrever TMDL via Python, use `'\t'` como caractere literal no início de cada linha, NUNCA confie em interpolação de string multiline para indentação.

**Verificação obrigatória após gravar:**
```python
with open(caminho, 'rb') as f:
    raw = f.read()
# Todas as linhas TMDL devem comecar com tab (exceto 'table ...' e linhas vazias)
for i, line in enumerate(raw.decode('utf-8').split('\n'), 1):
    if line.strip() and not line.startswith('table ') and not line.startswith('\t'):
        if not line.startswith('source =') and not line.startswith('\t\t'):
            print(f"AVISO: Linha {i} pode estar sem tab: {line[:40]}")
```

---

## Estrutura de Arquivos TMDL

TMDL usa uma estrutura de pastas onde alguns objetos são definidos em arquivos separados:

```
definition.pbism                        <- Configurações de conexão do modelo semântico
definition/database.tmdl                <- Propriedades do banco (compatibility level)
definition/model.tmdl                   <- Propriedades do modelo, refs de tabelas/roles/cultures
definition/relationships.tmdl           <- Relacionamentos nomeados/inativos
definition/functions.tmdl               <- Funções DAX
definition/tables/<NomeTabela>.tmdl     <- Tabelas: colunas, medidas, partições, hierarquias, calc groups
definition/roles/<NomeRole>.tmdl        <- Roles de segurança
definition/cultures/<locale>.tmdl       <- Traduções
definition/perspectives/<Nome>.tmdl     <- Perspectivas
```

### database.tmdl

O arquivo database **deve** começar com uma declaração de objeto `database` (GUID ou nome), não uma propriedade avulsa:

```tmdl
database 7124f8d8-6199-44fe-b35d-7f7f06b3e1c6
	compatibilityLevel: 1702
	compatibilityMode: powerBI
	language: 1033
```

> **Crítico**: Usar `compatibilityLevel:` avulso sem a declaração `database` causa erros `InvalidLineType: Property!`.

### model.tmdl

O arquivo model declara propriedades e referências para todas as tabelas, roles, perspectivas e cultures. Use declarações `ref` para que o engine descubra os arquivos correspondentes:

```tmdl
model Model
	culture: en-US
	defaultPowerBIDataSourceVersion: powerBI_V3
	sourceQueryCulture: en-US

ref table Sales
ref table Date

ref role RegionalManager

ref perspective 'Internet Sales'

ref cultureInfo en-US
ref cultureInfo fr-FR
```

> **Nota**: `defaultPowerBIDataSourceVersion: powerBI_V3` é obrigatório para modelos Import. Sem ele, a API retorna `Import from JSON supported for V3 models only`.

---

## Exemplos de Tabelas por Modo de Armazenamento

### Tabela Import

```tmdl
table Customer

	/// Número total de clientes
	measure '# Customers' = COUNTROWS(Customer)
		formatString: #,##0

	column CustomerId
		dataType: int64
		isHidden
		summarizeBy: none
		sourceColumn: CustomerId

	column 'Customer Name'
		dataType: string
		sourceColumn: CustomerName

	partition Customer = m
		mode: import
		source =
			let
				Source = Sql.Database(#"Server", #"Database"),
				Customer = Source{[Schema="dbo", Item="Customer"]}[Data]
			in
				Customer
```

### Tabela Direct Lake

```tmdl
expression DL_Lakehouse =
	let
		Source = AzureStorage.DataLake("https://onelake.dfs.fabric.microsoft.com/<WorkspaceId>/<LakehouseId>", [HierarchicalNavigation=true])
	in
		Source

table Sales

	/// Receita total
	measure 'Total Sales' = ```
			SUMX(
				Sales,
				Sales[Quantity] * Sales[UnitPrice]
			)
			``
		formatString: $ #,##0.00

	column SalesKey
		dataType: int64
		isHidden
		summarizeBy: none
		sourceColumn: sales_key

	column Quantity
		dataType: int64
		sourceColumn: quantity

	column UnitPrice
		dataType: decimal
		summarizeBy: none
		sourceColumn: unit_price

	partition Sales = entity
		mode: directLake
		source
			entityName: Sales
			schemaName: dbo
			expressionSource: DL_Lakehouse
```

---

## Relacionamentos

Declarados em `relationships.tmdl` ou inline nos arquivos de tabela.

```tmdl
relationship 'Sales to Date'
	fromColumn: Sales.'Order Date'
	toColumn: Date.Date

/// Inativo - use com USERELATIONSHIP() no DAX
relationship 'Sales - Ship Date to Date'
	isActive: false
	fromColumn: Sales.'Ship Date'
	toColumn: Date.Date
```

### Regras Principais

- Crie relacionamentos **antes** das medidas que dependem deles
- `fromColumn:` = lado many (fato); `toColumn:` = lado one (dimensão)
- Padrão: `crossFilteringBehavior: oneDirection`; adicione `bothDirections` apenas quando necessário
- `isActive: false` para dimensões role-playing; use `USERELATIONSHIP()` no DAX
- Prefira chaves inteiras sobre chaves string para performance
- Ambos os lados devem ter `dataType` correspondente
- Oculte chaves estrangeiras em tabelas fato (`isHidden: true`)
- Sem chaves compostas - use uma única chave inteira surrogada
- Sem chaves surrogadas em tabelas fato - use chaves naturais quando possível
- Não defina `isKey = true` na coluna chave primária de tabelas de dimensão para modelos não Direct Query.

---

## Hierarquias

Hierarquias são declaradas **dentro** da definição de uma tabela. Cada nível mapeia para uma coluna existente na mesma tabela.

```tmdl
table Geography

	column Continent
		dataType: string
		sourceColumn: Continent

	column Country
		dataType: string
		sourceColumn: Country

	column City
		dataType: string
		sourceColumn: City

	hierarchy 'Geography Hierarchy'

		level Continent
			column: Continent

		level Country
			column: Country

		level City
			column: City
```

### Regras Principais

- Uma tabela pode ter múltiplas hierarquias
- Níveis são ordenados de cima para baixo (mais grosseiro para mais fino)
- Cada `level` deve referenciar uma `column:` que existe na mesma tabela
- Nomes dos níveis podem diferir dos nomes das colunas
- Use descrições `///` **acima** do nível para documentação
- Não crie hierarquias com um único nível (sem valor de drill-down)

---

## Grupos de Cálculo (Calculation Groups)

Um grupo de cálculo é uma tabela especial com entradas `calculationGroup` e `calculationItem`. A tabela também deve ter uma coluna (a coluna do grupo de cálculo) e uma partição `calculationGroup`.

```tmdl
table 'Time Intelligence'

	calculationGroup

		calculationItem Current = SELECTEDMEASURE()

		calculationItem YTD = CALCULATE(SELECTEDMEASURE(), DATESYTD('Date'[Date]))

		calculationItem PY = CALCULATE(SELECTEDMEASURE(), SAMEPERIODLASTYEAR('Date'[Date]))

	column 'Time Intelligence'
		dataType: string

	partition 'Partition_Time Intelligence' = calculationGroup
```

### Regras Principais

- `calculationGroup` é declarado **sem nome** — apenas a palavra-chave indentada sob a tabela
- Cada `calculationItem <Nome> = <DAX>` é indentado sob `calculationGroup`
- Use `formatStringDefinition` (não `formatString`) para itens de cálculo que sobrepõem o formato da medida
- O nome da `column` tipicamente corresponde ao nome da tabela
- O tipo da partição deve ser `= calculationGroup` (não `= m` ou `= calculated`)

---

## Roles de Segurança

Cada role é um arquivo separado na pasta `roles/`. O arquivo declara o nível de acesso e expressões de filtro DAX por tabela.

### Arquivo: `roles/RegionalManager.tmdl`

```tmdl
/// Acesso restrito à região Leste
role RegionalManager
	modelPermission: read

	tablePermission Sales = [Region] = "East"
```

### Regras Principais

- `role <Nome>` é a declaração de nível superior
- `modelPermission:` é obrigatório — use `read` (mais comum) ou `readRefresh`
- `tablePermission <NomeTabela> = <filtro DAX>` — a expressão de filtro DAX restringe linhas
- Uma `tablePermission` por tabela; múltiplas tabelas podem ser filtradas na mesma role
- Em `model.tmdl`, adicione `ref role <Nome>` para cada role
- Ao criar novas roles, nunca inclua a anotação `PBI_Id`
- Se possível, analise os padrões de roles existentes primeiro

---

## Traduções / Cultures

Cada culture é um arquivo separado na pasta `cultures/`.

### Arquivo: `cultures/fr-FR.tmdl`

```tmdl
cultureInfo fr-FR
	translations
		model Model
			table Sales
				caption: Ventes
				column Amount
					caption: Montant
```

### Regras Principais

- `cultureInfo <locale>` é a declaração de nível superior (ex: `fr-FR`, `zh-CN`, `ja-JP`)
- `translations` -> `model Model` -> nesting de tabela/coluna/medida
- Use `caption:` para nome de exibição, `description:` para dicas de ferramentas
- Em `model.tmdl`, adicione `ref cultureInfo <locale>` para cada culture
- **Não inclua** `linguisticMetadata` — é gerenciado automaticamente

---

## Perspectivas

Cada perspectiva é um arquivo separado na pasta `perspectives/`.

### Arquivo: `perspectives/Internet Sales.tmdl`

```tmdl
perspective 'Internet Sales'

	perspectiveTable 'Internet Sales'
		includeAll

	perspectiveTable Customer
		perspectiveColumn 'Marital Status'
		perspectiveColumn Education

	perspectiveTable _Measures
		perspectiveMeasure 'Internet Sales Amount'
```

### Regras Principais

- `perspective <Nome>` é a declaração de nível superior
- `perspectiveTable <NomeTabela>` lista cada tabela incluída
- Use `includeAll` para incluir todas as colunas/medidas; caso contrário, liste `perspectiveColumn` / `perspectiveMeasure` específicos
- Em `model.tmdl`, adicione `ref perspective <Nome>` para cada perspectiva

---

## Funções DAX Definidas pelo Usuário (UDFs)

Funções DAX definidas pelo usuário são cálculos reutilizáveis declarados em `functions.tmdl`.

Exemplo de função DAX em `functions.tmdl`:

```tmdl
/// Retorna o dobro do valor de entrada
function DoubleValue = DoubleValue(Value) = Value * 2

/// Acumulado ano até hoje
function YTDTotal = ```
		YTDTotal(Measure) =
		CALCULATE(
		    Measure,
		    DATESYTD('Date'[Date])
		)
		``
```

### Regras Principais

- `function <Nome> = <Assinatura> = <DAX>` para linha única; crases triplas para multi-linha
- Declarado em `definition/functions.tmdl` (arquivo de nível superior, não dentro de uma tabela)

---

## Tabelas Calculadas

Tabelas podem ser originadas de expressões DAX usando uma partição calculada:

```tmdl
table _Measures

	measure 'Total Sales' = SUM(Sales[Amount])
		formatString: $#,##0.00

	column Dummy
		isHidden
		sourceColumn: [Dummy]

	partition _Measures = calculated
		mode: import
		source = ROW("Dummy", BLANK())
```

### Regras Principais

- Use `partition <Nome> = calculated` com `source = <DAX>`
- Partições calculadas usam `mode: import`
- Útil para tabelas somente de medidas (`ROW("Dummy", BLANK())`) e tabelas de datas computadas

---

## Tabela de Datas/Calendário

- Prefira uma tabela de datas existente na fonte sobre uma auto-gerada
- Garanta **intervalo de datas contíguo** sem lacunas
- Defina `dataCategory: Time` na tabela de datas
- Configure `sortByColumn` para colunas de nome de mês (ordene por número do mês)
- Desative tabelas de data auto se uma tabela de calendário adequada existir
- Crie tabela de data DAX calculada apenas se a fonte estiver indisponível

---

## Parâmetros

Use expressões nomeadas para parâmetros de conexão (Server, Database):

```tmdl
expression Server = "myserver.database.windows.net" meta [IsParameterQuery=true, Type="Text", IsParameterQueryRequired=true]

expression Database = "MyDatabase" meta [IsParameterQuery=true, Type="Text", IsParameterQueryRequired=true]
```

Referencie nas expressões M das partições via `#"Server"` e `#"Database"`.

---

## Anotações (Annotations)

Anotações armazenam metadados como pares chave-valor. Podem aparecer em tabelas, colunas, medidas, relacionamentos, roles e perspectivas.

Exemplos de anotações auto-geradas (somente leitura — não crie manualmente) `PBI_FormatHint`, `PBI_ResultType`:

```tmdl
table Sales

	column 'Unit Price'
		dataType: decimal
		sourceColumn: UnitPrice

		annotation SummarizationSetBy = Automatic
		annotation PBI_FormatHint = {"currencyCulture":"en-US"}

	partition Sales = m
		mode: import
		source = ...

	annotation PBI_ResultType = Table
```

### Regras Principais

- `annotation <Chave> = <Valor>` — indentado sob o objeto que anota
- **Não adicione** anotações `PBI_*` manualmente — são metadados internos do Power BI
- Anotações customizadas podem ser usadas para documentação ou metadados de tooling

---

## Objetos de Calendário

Objetos de calendário fornecem metadados de inteligência de datas em uma tabela Date, declarados dentro da tabela após as hierarquias.

```tmdl
table Date

	calendar 'Gregorian Calendar'

		calendarColumnGroup = year
			primaryColumn: Year

		calendarColumnGroup = month
			primaryColumn: 'Year Month Number'
			associatedColumn: Month

		calendarColumnGroup = date
			primaryColumn: Date
			associatedColumn: Day
```

### Regras Principais

- `calendar '<Nome>'` é declarado dentro da definição da tabela
- `calendarColumnGroup = <granularidade>` mapeia níveis (year, quarter, month, week, date, monthOfYear, dayOfWeek)
- `primaryColumn:` = coluna de ordenação/numérica; `associatedColumn:` = colunas de exibição/texto
- Objetos de calendário são opcionais — melhoram a auto-detecção de inteligência temporal

---

## Scripts TMDL

Scripts TMDL são produzidos pela view TMDL e normalmente ficam sob a pasta `TMDLScripts` de um modelo semântico.

Um script TMDL sempre inclui um comando no topo seguido de um ou mais objetos com pelo menos um nível de indentação.

```tmdl
<Nome do Comando TMDL>
  <Objeto TMDL>

  <Objeto TMDL>
```
- A semântica da linguagem TMDL é aplicada aos objetos dentro do comando
- Scripts TMDL só suportam um comando hoje: `createOrReplace`

Exemplo de script TMDL usando `createOrReplace`:

```tmdl
createOrReplace

    table Product

        measure '# Products' = COUNTROWS('Product')
            formatString: #,##0

        column 'Product Name'
            dataType: string

        ...
```

---

## Tarefa: Definir Descrições em Objetos TMDL

**FAÇA:**
- O formato deve ser `/// Descrição` colocada logo acima de cada objeto como `table`, `column`, ou `measure` no código TMDL:
    ```tmdl
    /// Linha de descrição 1
    /// Linha de descrição 2
    measure 'Measure1' = [Expressão DAX]
        formatString: #,##0

    /// Linha de descrição 1
    column 'Column1'
        formatString: #,##0
        dataType: string
    ```
- Garanta que as descrições forneçam explicações claras das definições e propósito
- Melhore descrições existentes mas use-as como linha de base
- Use descrições concisas e significativas

**NÃO FAÇA:**
- Não use a propriedade `description` — use a sintaxe `///` em vez disso
- Não altere nenhuma outra propriedade ao inserir descrições

---

## Tarefa: Criar Medidas em TMDL

- Sempre inclua uma propriedade `formatString` apropriada para o tipo da medida
- Sempre inclua uma descrição seguindo as regras de `///` acima
- Não crie medidas para colunas não-agregáveis (chaves, descrições) a menos que especifiquem uma propriedade `summarizeBy` diferente de `none`
- Expressões DAX multi-linha devem ser envolvidas em crases triplas
- A expressão DAX deve aparecer após o nome da medida precedida pelo sinal `=`
- Se for uma expressão DAX de linha única, adicione-a imediatamente após o nome da medida e o sinal `=`
- Medidas devem ficar no topo do objeto tabela, antes de qualquer declaração de coluna

### Referência de Format String

| Tipo | Formato | Saída |
|------|---------|-------|
| Moeda | `$#,##0.00` | $1,234.56 |
| Porcentagem | `0.00%` | 45.67% |
| Inteiro | `#,##0` | 1,234 |
| Decimal | `#,##0.00` | 1,234.56 |
| Milhar | `#,##0,K` | 1,234K |
| Milhão | `#,##0,,M` | 1M |

---

## Tarefa: Definir Descrições no Código Power Query / M

- Insira um comentário acima do código explicando o que aquele trecho está fazendo
- Não comece o comentário com a palavra Step ou um número
- Não copie código para dentro do comentário
- Mantenha os comentários com no máximo 225 caracteres
- Atualize o nome do passo explicando o que ele está fazendo
- O nome do passo deve ser envolvido em aspas duplas e precedido pelo `#`
- O nome do passo deve sempre começar com um verbo no pretérito
- O nome do passo deve ter espaços entre as palavras
- Mantenha o nome do passo com no máximo 50 caracteres
