# Power BI Project file (PBIP)

Power BI Project (PBIP) is the code-behind of a Power BI development. 

It can include a Semantic Model + Report or just a Report (live connect).

**PBIP Folder with both Semantic Model and Report folder:**

```text
PBIPFolder/
├── [Name].SemanticModel/
|   ├── /definition # The semantic model definition using TMDL language [REQUIRED]
│   ├── Copilot/ # Copilot tooling [OPTIONAL]
│   │   ├── Instructions/ # Contains the AI instructions configured for the semantic model, stored as a single markdown file.
│   │   │   ├── instructions.md 
│   │   ├── VerifiedAnswers/ # Contains the configured Verified answers for the semantic model, using PBIR format.
│   │   │   ├── definitions/
│   │   ├── schema.json # Contains the schema selection and field synonyms configured for the semantic model. 
│   │   ├── settings.json # Contains top level Copilot tooling settings. 
│   │   ├── examplePrompts.json # Contains zero prompts set up for the semantic model.
|   ├── definition.pbism # The semantic model definition file [REQUIRED]
|   ├── * # Other semantic model metadata files and folders
├── [Name].Report/        
|   ├── /definition # The report definition using PBIR format
|   ├── definition.pbir # The report definition file with a byPath relative reference to the semantic model folder. [REQUIRED]
|   ├── * # Other report metadata files and folders
└── [Name].pbip # A shortcut file to the report folder
```    

**PBIP Folder with Report folder:**

```text
PBIPFolder/
├── [Name].Report/        
|   ├── /definition # The report definition using PBIR format
|   ├──  definition.pbir # The report definition file with a byConnection reference to a semantic model in a Workspace. [REQUIRED]
|   ├── * # Other report metadata files and folders
└── [Name].pbip # A shortcut file to the report folder
```   

## definition.pbism file

No modifications are needed—just create the file exactly as shown in the example.

```json
{
    "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/semanticModel/definitionProperties/1.0.0/schema.json",
    "version": "4.2",
    "settings": {
        "qnaEnabled": true
    }
}
```
Refer to [JSON Schema](https://github.com/microsoft/json-schemas/blob/main/fabric/item/semanticModel/definitionProperties/1.0.0/schema.json) for more details.

## definition.pbir

Overall definition of the report and core settings. Also holds the reference to the semantic model of the report, it's possible to rebind the report to a different semantic model by updating this file.

This file can be opened by Power BI Desktop.

Example of `definition.pbir` file targeting a local semantic model folder:

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definitionProperties/2.0.0/schema.json",
  "version": "4.0",
  "datasetReference": {
    "byPath": {
      "path": "../Sales.SemanticModel"
    }
  }
}
```

Example of `definition.pbir` file targeting a semantic model in a workspace:

```json
{  
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definitionProperties/2.0.0/schema.json",
  "version": "4.0",
  "datasetReference": {
    "byConnection": {      
      "connectionString": "semanticmodelid=[SemanticModelId]"
    }
  }
}
```

Notes:
- Use forward slashes in byPath, only relative paths are supported.
- When using byConnection, Desktop does not open the model in edit mode.

Refer to [JSON Schema](https://github.com/microsoft/json-schemas/blob/main/fabric/item/report/definitionProperties/2.0.0/schema.json) for more details.

## .pbip file

Serves as a shortcut to a Power BI Report. 

Example of a `[name].pbip` file:

```json
{
    "$schema": "https://developer.microsoft.com/json-schemas/fabric/pbip/pbipProperties/1.0.0/schema.json",
    "version": "1.0",
    "artifacts": [
        {
          "report": {
              "path": "{Name of the Semantic Model}.Report"
          }
        }
    ],
    "settings": {
        "enableAutoRecovery": true
    }
}
```

Refer to [JSON Schema](https://github.com/microsoft/json-schemas/blob/main/fabric/pbip/pbipProperties/1.0.0/schema.json) for more details.

## Copilot/Instructions/instructions.md

```markdown
# AI Instructions for Semantic Model

## Business Context & Terminology
- You are a sales analyst for a retail company analyzing sales performance, customer behavior, and product trends
...
```

### Copilot/schema.json

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/semanticModel/copilot/schema/1.0.0/schema.json",
  "tables": [
    {
      "id": "[lineage tag]",
      "name": "[Table name 1]",
      "visibility": "Visible",
      "columns": [
        {
          "id": "[lineage tag]",
          "name": "[Column name 1]",
          "visibility": "Visible",
          "synonyms": ["[synonym 1]", "[synonym 2]"]
        },
        {
          "id": "[lineage tag]",
          "name": "[Column name 2]",
          "visibility": "Hidden",
          "synonyms": []
        }
      ],
      "measures": [
        {
          "id": "[lineage tag]",
          "name": "[measure name]",
          "visibility": "Visible",
          "synonyms": []
        }
      ],
      "hierarchies": [
        {
          "id": "[lineage tag]",
          "name": "[Hierarchy name]",
          "visibility": "Visible",
          "levels": [
            {
              "id": "[lineage tag]",
              "name": "[Level name]",
              "visibility": "Visible",
              "synonyms": []
            }
          ],
          "synonyms": []
        }
      ],
      "synonyms": []
    }
  ]
}
```

### Copilot/settings.json

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/semanticModel/copilot/settings/1.0.0/schema.json",
  "indexingEnabled": true
}
```

### Copilot/examplePrompts.json

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/semanticModel/copilot/examplePrompts/1.0.0/schema.json",
  "prompts": [
    "Top 5 customers by Sales Amount",
    "Margin % by product category"
  ]
}
```

## File Encoding (CRITICAL)

All files in a PBIP project (`.tmdl`, `.json`, `.pbir`, `.pbism`) **must** follow these rules or Power BI Desktop will fail to open the project:

| Property | Required value |
|----------|---------------|
| Encoding | UTF-8 **without** BOM |
| Line endings | CRLF (`\r\n`) |
| Indentation | Tabs for TMDL; spaces (2) for JSON |

**Common mistakes:**

| Written as | Desktop error |
|-----------|---------------|
| UTF-8 with BOM (`EF BB BF`) | *"Only text with UTF8 encoding without BOM is supported"* |
| Windows-1252/Latin-1 | *"Cannot convert bytes [XX]"* |
| LF-only line endings | Parsing failures |

**How to write correctly (Python):**

```python
# Write JSON or TMDL as UTF-8 without BOM, CRLF
with open(path, 'wb') as f:
    f.write(content.encode('utf-8').replace(b'\n', b'\r\n'))
```

**How to verify:**

```python
with open(path, 'rb') as f:
    raw = f.read()
assert raw[:3] != b'\xef\xbb\xbf', "File has BOM!"
assert raw.count(b'\n') == raw.count(b'\r\n'), "File has LF-only lines!"
```

> **Note:** The Desktop itself writes TMDL with UTF-8 accents (e.g. `"Cabeçalhos promovidos"` in M code) and JSON with UTF-8 accents (e.g. `"Página 1"` in page.json).

## PBIR Visual Structure (CRITICAL)

### Directory structure (MANDATORY)

Each visual MUST have its own subfolder under `definition/pages/{pageId}/visuals/`:

```
definition/pages/{pageId}/visuals/
  ├── cardKPI/
  │   └── visual.json
  └── barChartBrand/
        └── visual.json
```

**NEVER** place `.json` visual files loose in the page directory.
**NEVER** add a `visuals` array to `page.json` (not part of schema 2.1.0).

### visual.json schema

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/powerbi/report-visual-containers/visualContainer/2.10.0/schema.json",
  "name": "<visual-folder-name>",
  "displayName": "<human-readable title>",
  "position": {
    "x": 0, "y": 0, "z": 0,
    "height": 300, "width": 400,
    "tabOrder": 0
  },
  "visual": {
    "visualType": "<type>",
    "query": { "queryState": { ... } },
    "objects": { ... },
    "visualContainerObjects": { ... },
    "drillFilterOtherVisuals": true
  },
  "filterConfig": { "filters": [] }
}
```

**Root keys** (exactly these, no more, no less): `$schema`, `name`, `position`, `visual`, `filterConfig`

**`position`**: must include `height` and `width` (not a separate `size` object)

**`visual.objects`**: contains axis/dataPoint/legend/labels formatting
**`visual.visualContainerObjects`**: contains title/background/border/padding

### queryState structure (MANDATORY)

Each role MUST wrap projections in a `projections` array. **NEVER** use array directly on the role:

```json
"queryState": {
    "Category": {
        "projections": [
            {
                "field": {
                    "Column": {
                        "Expression": {
                            "SourceRef": {"Entity": "TableName"}
                        },
                        "Property": "columnName"
                    }
                },
                "queryRef": "TableName.columnName",
                "nativeQueryRef": "columnName"
            }
        ]
    },
    "Y": {
        "projections": [
            {
                "field": {
                    "Measure": {
                        "Expression": {
                            "SourceRef": {"Entity": "TableName"}
                        },
                        "Property": "MeasureName"
                    }
                },
                "queryRef": "MeasureName",
                "nativeQueryRef": "MeasureName"
            }
        ]
    }
}
```

**WRONG** (will fail validation):
```json
"Category": [ { "field": ... } ]  // MISSING projections wrapper
```

### Formatting properties — ALWAYS consult CLI first

**NEVER invent property names.** Always run BEFORE generating code:

```bash
powerbi-report-author catalog describe <visualType>
powerbi-report-author formatting describe-object <visualType> <objectName>
```

Known corrections (do NOT use the old names):

| Wrong (old) | Correct (new) | Object |
|-------------|---------------|--------|
| `categoryAxis.title` | `categoryAxis.showAxisTitle` + `categoryAxis.titleText` | categoryAxis |
| `categoryAxis.visible` | `categoryAxis.show` | categoryAxis |
| `valueAxis.title` | `valueAxis.showAxisTitle` + `valueAxis.titleText` | valueAxis |
| `valueAxis.visible` | `valueAxis.show` | valueAxis |
| `legend.visible` | `legend.show` | legend |
| `title.visible` (VCO) | `title.show` | visualContainerObjects.title |
| `categoryLabels` | Does NOT exist for `clusteredBarChart` | — |

### PBIR literal value formats (different from theme JSON)

| Value | PBIR expression |
|-------|----------------|
| Boolean | `"true"` / `"false"` |
| Number | `"14D"` (D suffix) |
| Integer | `"3L"` (L suffix) |
| String | `"'text'"` (single-quoted inside double) |
| Color | `"'#FFFFFF'"` (single-quoted inside double) |

### Mandatory workflow for PBIR visuals

1. Run `powerbi-report-author catalog describe <type>` to confirm roles
2. Run `powerbi-report-author formatting describe-object <type> <object>` for each formatting object
3. Build JSON with correct structure (projections wrapper, real property names)
4. Write with UTF-8 no-BOM + CRLF via Python
5. Run `powerbi-report-author validate` — **NEVER declare success before this**
6. Only `PBIR_SCHEMA_UNREACHABLE` warning is acceptable (pre-existing, does not block Desktop)

## References

**External references** (request markdown when possible):

- [PBIP docs](https://learn.microsoft.com/en-us/power-bi/developer/projects/projects-overview)
- [PBIP SemanticModel folder](https://learn.microsoft.com/en-us/power-bi/developer/projects/projects-dataset)
- [PBIP Report folder](https://learn.microsoft.com/en-us/power-bi/developer/projects/projects-report)