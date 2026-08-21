---
DateCriation: "{{Date}} {{Time}}"
tags:
  - geral
  - importante
tributo:
  - transversal
elemento:
  - base-calculo
tipo:
  - conceito
fonte:
  - doutrina
status: vigente
atualizado_em: "{{Date}}"
UF:
Municipio:
relacionados:
---
---
# Guia de padronização — direito tributário municipal

Este guia explica **qual template usar**, o **vocabulário controlado** das
propriedades e as **buscasmando do plugin *Templates*).

## 1. Qual template usar

| Conteúdo | Template | `tipo` |
| --- | --- | --- |
| Doutrina / definição de um conceito (eixo conceitual) | **Template - Conceito** | `conceito` |
| Regra ou procedimento de um município específico | **Template - Regra Municipal** | `procedimento` (ou `norma`) |
| Tabela de valores (BVT / PGV — valor do m² por setor/logradouro) | **Template - Tabela (BVT-PGV)** | `tabela` |
| Fatores / coeficientes de cálculo (cobertura, topografia, etc.) | **Template - Parametro** | `parametro` |

Regra dos eixos:

- **Eixo conceitual** (doutrina/definições, pasta *Conceitos Gerais*): documento
  **geral** → deixar `UF` e `Municipio` **vazios** e marcar **`geral`** em `tags`.
- **Eixo por município** (pasta *municípios/<cidade>*): preencher `UF` e
  `Municipio`.

## 2. Vocabulário controlado

Valores **sempre em minúsculas, sem acento, com hífen entre palavras**.
No YAML: propriedade **List** é bloco com hífens (`- iptu`); propriedade **Text**
é linha única sem hífen (`status: vigente`).

| Propriedade | Tipo | Valores permitidos |
| --- | --- | --- |
| `DateCriation` | Text (literal) | `{{Date}} {{Time}}` |
| `tags` | List | geral · revisar · duvida · importante |
| `tributo` | List | iptu · itbi · issqn · taxa · contrib-melhoria · transversal |
| `elemento` | List | hipotese-incidencia · base-calculo · aliquota · sujeito-passivo · sujeito-ativo · lancamento · pagamento · imunidade-isencao · infracao-penalidade |
| `tipo` | List | conceito · norma · tabela · procedimento · exemplo · parametro |
| `fonte` | List | cf88 · ctn · ctm · lei-especifica · decreto · jurisprudencia · doutrina |
| `status` | Text | vigente · revogado · em-revisao |
| `atualizado_em` | Date (literal) | `{{Date}}` |
| `UF` | List | ex.: MA · CE · PI |
| `Municipio` | List | ex.: Pinheiro · Fortaleza |
| `setor` | List | só no template de tabela; bairros/setores do BVT |
| `relacionados` | List | links `[[ ]]` |

> **Nota sobre os tokens:** `DateCriation` e `atualizado_em` ficam **entre aspas**
> no template (`"{{Date}} {{Time}}"` / `"{{Date}}"`). Isso é necessário porque
> `{{...}}` começando um valor YAML seria lido como mapa e quebraria as
> propriedades. O plugin *Templates* substitui o token normalmente; após a
> inserção o valor vira a data/hora reais. Wikilinks em `relacionados` também
> precisam de aspas: `- "[[Nota]]"`.

## 3. Regra sempre válida

**Sempre preencher `status` e `atualizado_em`** em toda nota.

- `status`: `vigente`, `revogado` ou `em-revisao`.
- `atualizado_em`: mantenha o token `{{Date}}` no template; ao revisar a nota,
  atualize para a data da revisão.

## 4. Buscas prontas

### Busca nativa do Obsidian

```
["tributo":"iptu"]
["elemento":"base-calculo"]
```

### Dataview — vigente em Pinheiro

````
```dataview
TABLE tipo, elemento, atualizado_em
WHERE contains(Municipio, "Pinheiro") AND status = "vigente"
SORT atualizado_em DESC
```
````

### Dataview — a revisar

````
```dataview
LIST
WHERE status = "em-revisao"
SORT atualizado_em ASC
```
````

### Dataview — tabelas

````
```dataview
TABLE Municipio, setor
WHERE contains(tipo, "tabela")
```
````
