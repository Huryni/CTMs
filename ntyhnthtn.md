---
DateCriation: "2026-08-18 13:43"
tags:
---

---
# Obsidian Query API — busca de preços via LLM local
API local que recebe uma pergunta em linguagem natural (ex.: "qual o preço do parafuso M8?"), usa um LLM rodando no próprio servidor para interpretar a pergunta, procura a resposta no vault do Obsidian (por título de arquivo e metadados de frontmatter) e devolve uma resposta curta e direta — sem enviar nenhum dado para fora da rede local.
Feito para rodar via Docker na máquina descrita abaixo, sem depender de GPU dedicada de forma obrigatória.
---
1. Visão geral
```
Usuário → POST /ask → API (FastAPI) ──┬─→ 1) busca no vault (título + frontmatter)
                                       └─→ 2) LLM local (Ollama) monta a resposta
                       ← resposta curta ──┘
```
O vault do Obsidian é montado como volume somente leitura dentro do container da API — o projeto nunca escreve no vault, só lê.
A busca não depende só da LLM: um passo de busca determinística (por nome de arquivo e por campos do frontmatter, tipo `preco:` ou `price:`) roda primeiro. A LLM entra em dois pontos: para extrair o termo de busca da pergunta do usuário, e para redigir a resposta final a partir do que foi encontrado. Isso evita o problema mais comum desse tipo de projeto — a LLM "alucinar" um preço que não está na nota.

---
2. Requisitos
Item	Necessário
Docker + Docker Compose	v2+
Vault do Obsidian	pasta local acessível pelo Docker, com notas em Markdown usando frontmatter YAML
RAM livre	~6-8 GB (modelo 7-8B em Q4 + overhead do container)
GPU	opcional — funciona em CPU; ver seção 7
Cada nota de produto no vault precisa ter um frontmatter mínimo, por exemplo:
```markdown
---
tipo: produto
titulo: Parafuso M8 Inox
preco: 4.90
unidade: un
categoria: fixadores
---

Parafuso sextavado M8, inox A2, cabeça allen.
```
O título do arquivo (`Parafuso M8 Inox.md`) e os campos do frontmatter (`titulo`, `preco`, `categoria`) são exatamente o que o motor de busca usa — quanto mais consistente o frontmatter entre as notas, melhor a taxa de acerto.
---
3. Estrutura do projeto
```
obsidian-query-api/
├── docker-compose.yml
├── api/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app/
│       ├── main.py          # endpoint FastAPI
│       ├── vault_index.py   # varre o vault e indexa título + frontmatter
│       ├── search.py        # busca fuzzy por título/metadados
│       └── llm.py           # chamadas ao Ollama (extração + resposta)
```
---
4. `docker-compose.yml`
```yaml
version: "3.9"

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    volumes:
      - ollama_models:/root/.ollama
    ports:
      - "11434:11434"
    restart: unless-stopped
    # GPU (opcional, ver seção 7) — comentado por padrão, roda em CPU
    # devices:
    #   - /dev/dri:/dev/dri

  api:
    build: ./api
    container_name: obsidian-query-api
    depends_on:
      - ollama
    environment:
      - OLLAMA_HOST=http://ollama:11434
      - OLLAMA_MODEL=llama3.1:8b
      - VAULT_PATH=/vault
    volumes:
      - /caminho/no/host/para/o/vault:/vault:ro
    ports:
      - "8000:8000"
    restart: unless-stopped

volumes:
  ollama_models:
```
Ajuste `/caminho/no/host/para/o/vault` para o caminho real do vault na máquina.
---
5. Serviço da API (esqueleto)
`api/requirements.txt`
```
fastapi
uvicorn[standard]
python-frontmatter
rapidfuzz
httpx
```
`api/app/vault_index.py` — varre o vault uma vez (e recarrega periodicamente) e monta um índice em memória com título + metadados de cada nota:
```python
import frontmatter
from pathlib import Path

def build_index(vault_path: str) -> list[dict]:
    index = []
    for md_file in Path(vault_path).rglob("*.md"):
        try:
            post = frontmatter.load(md_file)
        except Exception:
            continue
        index.append({
            "titulo_arquivo": md_file.stem,
            "path": str(md_file),
            "metadata": post.metadata,   # dict com tudo do frontmatter
            "conteudo": post.content[:500],  # primeiras linhas, para contexto
        })
    return index
```
`api/app/search.py` — busca fuzzy por título e por campos do frontmatter:
```python
from rapidfuzz import fuzz

def search(query: str, index: list[dict], top_k: int = 3) -> list[dict]:
    scored = []
    for note in index:
        titulo = note["titulo_arquivo"]
        campos_texto = " ".join(str(v) for v in note["metadata"].values())
        score = max(
            fuzz.partial_ratio(query.lower(), titulo.lower()),
            fuzz.partial_ratio(query.lower(), campos_texto.lower()),
        )
        if score > 60:
            scored.append((score, note))
    scored.sort(key=lambda x: x[0], reverse=True)
    return [n for _, n in scored[:top_k]]
```
`api/app/llm.py` — duas chamadas curtas ao Ollama: uma para extrair o termo de busca, outra para redigir a resposta final:
```python
import httpx, os

OLLAMA_HOST = os.environ["OLLAMA_HOST"]
MODEL = os.environ["OLLAMA_MODEL"]

async def extrair_termo_busca(pergunta: str) -> str:
    prompt = (
        "Extraia apenas o nome do produto ou item mencionado na pergunta abaixo. "
        "Responda só com o termo de busca, sem mais nada.\n\n"
        f"Pergunta: {pergunta}"
    )
    return await _chamar_llm(prompt)

async def responder_com_contexto(pergunta: str, notas_encontradas: list[dict]) -> str:
    contexto = "\n---\n".join(
        f"Título: {n['titulo_arquivo']}\nMetadados: {n['metadata']}"
        for n in notas_encontradas
    )
    prompt = (
        "Responda a pergunta do usuário de forma curta e direta, usando APENAS "
        "as informações do contexto abaixo. Se o contexto não tiver a resposta, "
        "diga que não encontrou a informação no vault. Não invente valores.\n\n"
        f"Contexto:\n{contexto}\n\nPergunta: {pergunta}\nResposta:"
    )
    return await _chamar_llm(prompt)

async def _chamar_llm(prompt: str) -> str:
    async with httpx.AsyncClient(timeout=60) as client:
        r = await client.post(
            f"{OLLAMA_HOST}/api/generate",
            json={"model": MODEL, "prompt": prompt, "stream": False},
        )
        r.raise_for_status()
        return r.json()["response"].strip()
```
`api/app/main.py` — junta tudo no endpoint:
```python
from fastapi import FastAPI
from pydantic import BaseModel
import os
from . import vault_index, search, llm

app = FastAPI(title="Obsidian Query API")
INDEX = vault_index.build_index(os.environ["VAULT_PATH"])

class Pergunta(BaseModel):
    pergunta: str

@app.post("/ask")
async def ask(body: Pergunta):
    termo = await llm.extrair_termo_busca(body.pergunta)
    notas = search.search(termo, INDEX)
    if not notas:
        return {"resposta": "Não encontrei nenhuma nota correspondente no vault.", "fontes": []}
    resposta = await llm.responder_com_contexto(body.pergunta, notas)
    return {"resposta": resposta, "fontes": [n["titulo_arquivo"] for n in notas]}

@app.post("/reindex")
def reindex():
    global INDEX
    INDEX = vault_index.build_index(os.environ["VAULT_PATH"])
    return {"notas_indexadas": len(INDEX)}
```
`api/Dockerfile`
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app ./app
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```
---
6. Como usar
```bash
docker compose up -d
docker compose exec ollama ollama pull llama3.1:8b   # baixa o modelo na primeira vez
```
Exemplo de chamada:
```bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"pergunta": "qual o preço do parafuso M8?"}'
```
Resposta esperada:
```json
{
  "resposta": "O parafuso M8 Inox custa R$ 4,90 por unidade.",
  "fontes": ["Parafuso M8 Inox"]
}
```
Sempre que notas forem adicionadas ou editadas no vault, chame `POST /reindex` (ou automatize isso com um agendador simples) para o índice pegar as mudanças — o índice atual é montado só na inicialização do container.
---
7. Nota sobre a GPU (Radeon RX 570)
Essa GPU é AMD Polaris — a AMD não mantém mais suporte oficial a ela no ROCm, que é o stack que ferramentas como Ollama usam para acelerar em GPU AMD. Por isso o `docker-compose.yml` acima sobe o Ollama em modo CPU, que é o caminho confiável.
Com o Xeon E5-2640 v4 e um modelo 7-8B quantizado (Q4), espere respostas na casa de alguns segundos por pergunta — perfeitamente aceitável para uma consulta pontual como essa, mas não para volume alto simultâneo.
Se quiser tentar aproveitar a GPU mesmo assim, o caminho é usar o backend Vulkan do `llama.cpp` (funciona em GPUs AMD antigas sem depender do ROCm) em vez do Ollama padrão — é mais configuração e ganho não garantido, então vale tratar como otimização futura, não como parte do MVP.
---
8. Limitações conhecidas
A qualidade da resposta depende diretamente da qualidade do frontmatter das notas — sem um campo `preco` (ou equivalente) padronizado, a LLM não tem de onde tirar o valor e vai (corretamente) dizer que não encontrou. A busca fuzzy atual é simples (por título e metadados); para vaults muito grandes ou com nomes muito parecidos entre produtos, pode valer migrar para uma busca por embeddings mais adiante. O projeto não faz nenhuma escrita no vault — é só leitura.
---
9. Próximos passos sugeridos
Validar com um conjunto pequeno de notas reais (10-20 produtos) antes de indexar o vault inteiro. Medir o tempo de resposta na prática nesse hardware para calibrar expectativa de uso. Se o volume de perguntas crescer, considerar trocar a busca fuzzy por um índice de busca mais robusto (ex.: SQLite FTS5) sem precisar mudar o resto da arquitetura.