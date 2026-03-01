# CLAUDE.md — Memória do Projeto

## Visão Geral

Stack Docker para automação de WhatsApp com agente de IA. Tudo é provisionado automaticamente no primeiro `docker compose up -d`.

**Serviços:** Evolution API · PostgreSQL 15 · Redis · n8n

---

## Arquivos Principais

| Arquivo | Função |
|---|---|
| `docker-compose.yml` | Orquestração de todos os serviços |
| `.env` | Todas as senhas e chaves (não commitar chaves reais) |
| `init-n8n.sh` | Script de boot do n8n: instala node, cria credenciais, importa e publica workflow, cria instância Evolution |
| `agente_whatsapp.json` | Workflow n8n do agente WhatsApp (exportado do n8n) |

---

## O que o init-n8n.sh faz (ordem de execução)

1. Instala `n8n-nodes-evolution-api` via npm
2. Sobe n8n em background, aguarda `/healthz`
3. Se banco limpo → cria owner; senão → faz login (cookie de sessão)
4. Cria credenciais: **Evolution API**, **OpenAI**, **Redis** — captura os IDs reais retornados
5. Grava marker `/home/node/.n8n/.credentials_initialized` (idempotência)
6. Lê `agente_whatsapp.json`, substitui IDs hardcoded pelos IDs reais, importa via `POST /rest/workflows` e **publica** via `POST /rest/workflows/{id}/activate` com `{ versionId }`
7. Aguarda Evolution API, cria instância `evo_n8n` com webhook → `http://n8n:5678/webhook/whatsapp/receberMensagem`

---

## n8n v2.x — API interna (autenticação por cookie)

| Ação | Endpoint correto |
|---|---|
| Login | `POST /rest/login` |
| Criar credencial | `POST /rest/credentials` |
| Criar workflow | `POST /rest/workflows` |
| **Publicar workflow** | `POST /rest/workflows/{id}/activate` com body `{ "versionId": "..." }` |
| Atualizar workflow | `PATCH /rest/workflows/{id}` |

> **Atenção n8n v2:** `POST .../activate` exige `versionId` no body — sem ele retorna HTTP 400.
> `PATCH` com `{ active: true }` retorna 200 mas **não ativa** o workflow de fato.
> O `versionId` vem na resposta da criação do workflow (`body.data.versionId`).

---

## Credenciais hardcoded no agente_whatsapp.json (IDs do ambiente de origem)

Esses IDs são substituídos automaticamente pelo `init-n8n.sh` pelos IDs reais do novo ambiente:

| Tipo | ID original |
|---|---|
| `evolutionApi` | `hDQN2JrpNKuCJ86H` |
| `openAiApi` | `2ckhdhCGpQ6pF26V` |
| `redis` | `x2XymCzQNbuNM2F0` |

---

## Workflow Agente Whatsapp

- Webhook: `POST /webhook/whatsapp/receberMensagem`
- Suporta: texto, áudio (Whisper), imagem (GPT-4o OCR)
- Debounce de mensagens via Redis (aguarda 13s antes de processar)
- Memória de conversa por contato via Redis Chat Memory
- Modelo: GPT-5.1 (alterar no node "OpenAI - Modelo LLM")
- Simula digitação proporcional ao tamanho da resposta

---

## Instância Evolution API

- Nome: `evo_n8n`
- Criada automaticamente pelo init se não existir
- Webhook configurado para `MESSAGES_UPSERT` com base64 ativado
- Após subir: escanear QR Code em `http://localhost:9090`

---

## Comandos Úteis

```bash
# Subir tudo
docker compose up -d

# Resetar SOMENTE o n8n (mantém Evolution, Postgres, Redis)
docker compose stop n8n && docker compose rm -f n8n && docker volume rm fast-n8n-evolutionapi-redis_n8n_data && docker compose up -d n8n

# Reforçar re-inicialização sem apagar dados
docker exec n8n rm -f /home/node/.n8n/.credentials_initialized && docker compose restart n8n

# Acompanhar logs de init
docker logs -f n8n

# Conferir estado de um workflow via API
curl -s -c /tmp/n8n_cookies.txt -X POST http://localhost:5678/rest/login \
  -H "Content-Type: application/json" \
  -d '{"emailOrLdapLoginId":"EMAIL","password":"SENHA"}' > /dev/null
curl -s -b /tmp/n8n_cookies.txt http://localhost:5678/rest/workflows/WORKFLOW_ID | \
  node -e "let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const w=(JSON.parse(d).data||JSON.parse(d));console.log('active:',w.active,'activeVersionId:',w.activeVersionId);})"
```

---

## Variáveis de Ambiente (.env)

```env
POSTGRES_USER=
POSTGRES_PASSWORD=
POSTGRES_DB=
AUTHENTICATION_API_KEY=   # chave da Evolution API
N8N_OWNER_EMAIL=
N8N_OWNER_PASSWORD=
OPENAI_API_KEY=           # obrigatório para o agente funcionar
```

---

## Convenções de Commit

Usar tags convencionais: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`
