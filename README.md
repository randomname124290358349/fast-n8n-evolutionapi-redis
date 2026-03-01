<div align="center">

# Evolution API + n8n

**Stack completa para automação de WhatsApp com agente de IA pronto para uso**

[![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![n8n](https://img.shields.io/badge/n8n-automation-EA4B71?logo=n8n&logoColor=white)](https://n8n.io/)
[![Evolution API](https://img.shields.io/badge/Evolution%20API-WhatsApp-25D366?logo=whatsapp&logoColor=white)](https://github.com/EvolutionAPI/evolution-api)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-cache-DC382D?logo=redis&logoColor=white)](https://redis.io/)

</div>

---

## O que é este projeto?

Este repositório contém um `docker-compose.yml` pronto para subir um ambiente completo de automação de WhatsApp com **zero configuração manual**. Na primeira inicialização, tudo é criado e publicado automaticamente.

### Serviços incluídos

| Serviço | Imagem | Descrição |
|---|---|---|
| **Evolution API** | `evoapicloud/evolution-api:latest` | Gerencia instâncias WhatsApp via API REST |
| **PostgreSQL 15** | `postgres:15` | Banco de dados da Evolution API |
| **Redis** | `redis:latest` | Cache e memória de conversas do agente |
| **n8n** | `docker.n8n.io/n8nio/n8n:latest` | Plataforma de automação de fluxos |

### O que é criado automaticamente no primeiro `docker compose up`?

O script `init-n8n.sh` executa uma única vez e faz tudo:

| O que | Detalhe |
|---|---|
| Community node | `n8n-nodes-evolution-api` instalado |
| Credencial **Evolution API** | Aponta para o container interno |
| Credencial **OpenAI** | Usa a chave definida no `.env` |
| Credencial **Redis** | Aponta para o container interno |
| Workflow **Agente Whatsapp** | Importado e **publicado automaticamente** |
| Instância **evo_n8n** | Criada na Evolution API com webhook configurado |

Após subir, só falta escanear o QR Code do WhatsApp!

---

## Requisitos

- [Docker](https://docs.docker.com/get-docker/) instalado
- [Docker Compose](https://docs.docker.com/compose/install/) (v2+)
- Portas **5678** (n8n) e **9090** (Evolution API) disponíveis na máquina
- Uma chave de API da [OpenAI](https://platform.openai.com/api-keys)

---

## Instalação

### 1. Clone o repositório

```bash
git clone https://github.com/randomname124290358349/fast-n8n-evolutionapi-redis.git
cd fast-n8n-evolutionapi-redis
```

### 2. Configure as variáveis de ambiente

Edite o arquivo `.env` na raiz do projeto:

```env
# PostgreSQL
POSTGRES_USER=admin
POSTGRES_PASSWORD=SuaSenhaSegura
POSTGRES_DB=evolution_api_db

# Evolution API — chave de autenticação da API
AUTHENTICATION_API_KEY=SuaChaveSegura

# n8n — credenciais do administrador
N8N_OWNER_EMAIL=seuemail@dominio.com
N8N_OWNER_PASSWORD=SuaSenhaSegura

# OpenAI — usada pelo agente de IA
OPENAI_API_KEY=sk-proj-SuaChaveOpenAI
```

> **Dica:** Você só precisa editar o `.env` — o `docker-compose.yml` propaga tudo automaticamente.

Se tiver um domínio, descomente e ajuste as linhas no `docker-compose.yml`:

```yaml
#WEBHOOK_URL: https://seu-dominio.com/
#N8N_EDITOR_BASE_URL: https://seu-dominio.com/
```

### 3. Suba os containers

```bash
docker compose up -d
```

### 4. Aguarde a inicialização

Na primeira execução, o script de setup pode levar alguns minutos. Acompanhe os logs do n8n:

```bash
docker logs -f n8n
```

Aguarde até ver as mensagens:

```
[init] ✔ Workflow "Agente Whatsapp" importado e publicado!
[init] ✔ Instância evo_n8n criada e configurada!
[init]  AÇÃO NECESSÁRIA: Conecte o WhatsApp manualmente.
[init]  Use o painel da Evolution API para escanear o QR Code.
```

### 5. Conecte o WhatsApp

Acesse o painel da Evolution API em **http://localhost:9090** e escaneie o QR Code da instância `evo_n8n`.

### 6. Acesse o n8n

Abra **http://localhost:5678** e faça login com as credenciais do `.env`.

O workflow **Agente Whatsapp** já estará publicado e recebendo mensagens!

---

## Workflow Agente Whatsapp

O arquivo `agente_whatsapp.json` contém um agente de IA completo que:

- Recebe mensagens de texto, áudio e imagem via WhatsApp
- Transcreve áudios com Whisper (OpenAI)
- Descreve imagens com GPT-4o (OCR incluso)
- Agrupa mensagens enviadas em sequência antes de responder (debounce com Redis)
- Mantém histórico de conversa por contato (memória com Redis)
- Simula tempo de digitação proporcional ao tamanho da resposta
- Usa o modelo **GPT-5.1** para geração de respostas

> Para trocar o modelo ou o prompt do agente, edite o node **"OpenAI - Modelo LLM"** e **"Agente IA"** diretamente no n8n.

---

## Arquitetura

```
┌─────────────────────────────────────────────────────┐
│                    evolution-net                    │
│                                                     │
│  ┌──────────────┐    ┌──────────┐    ┌───────────┐  │
│  │ Evolution API│◄───│  Redis   │    │ PostgreSQL│  │
│  │   :9090      │    │  :6379   │    │   :5432   │  │
│  └──────┬───────┘    └──────────┘    └─────┬─────┘  │
│         │ webhook                          │        │
│         └──────────────┬────────────────────┘        │
│                        │                            │
│                  ┌─────▼─────┐                      │
│                  │    n8n    │                      │
│                  │   :5678   │                      │
│                  └─────┬─────┘                      │
└────────────────────────┼────────────────────────────┘
                         │
                   http://localhost:5678
                   (seu navegador)
```

> A Evolution API fica acessível apenas localmente em `127.0.0.1:9090` — não é exposta diretamente para a internet por segurança.

---

## Comandos úteis

```bash
# Subir todos os containers
docker compose up -d

# Parar todos os containers (mantém os dados)
docker compose down

# Parar e remover todos os dados (volumes)
docker compose down -v

# Ver logs em tempo real
docker logs -f n8n
docker logs -f evolution_api

# Reiniciar apenas o n8n
docker compose restart n8n

# Ver status dos containers
docker compose ps
```

---

## Portas expostas

| Serviço | Porta | Acessível de |
|---|---|---|
| n8n | `5678` | `0.0.0.0` (todos os IPs) |
| Evolution API | `9090` | `127.0.0.1` (somente local) |
| Redis | interno | Somente dentro da rede Docker |
| PostgreSQL | interno | Somente dentro da rede Docker |

---

## Resetar o n8n (recriar tudo do zero)

Remove apenas o volume do n8n e recria o container, mantendo Evolution API, PostgreSQL e Redis intactos:

```bash
docker compose stop n8n && docker compose rm -f n8n && docker volume rm fast-n8n-evolutionapi-redis_n8n_data && docker compose up -d n8n
```

Se quiser apenas recriar as credenciais/workflow sem apagar os dados do n8n:

```bash
docker exec n8n rm -f /home/node/.n8n/.credentials_initialized
docker compose restart n8n
```

---

## Licença

Distribuído sob a licença MIT. Veja `LICENSE` para mais informações.

---

<div align="center">

Feito com Docker, n8n e Evolution API

</div>
