<div align="center">

# Evolution API + n8n

**Stack completa para automação de WhatsApp com criação automática de credenciais**

[![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![n8n](https://img.shields.io/badge/n8n-automation-EA4B71?logo=n8n&logoColor=white)](https://n8n.io/)
[![Evolution API](https://img.shields.io/badge/Evolution%20API-WhatsApp-25D366?logo=whatsapp&logoColor=white)](https://github.com/EvolutionAPI/evolution-api)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-cache-DC382D?logo=redis&logoColor=white)](https://redis.io/)

</div>

---

## O que é este projeto?

Este repositório contém um `docker-compose.yml` pronto para subir um ambiente completo de automação de WhatsApp com **zero configuração manual** de credenciais no n8n.

### Serviços incluídos

| Serviço | Imagem | Descrição |
|---|---|---|
| **Evolution API** | `evoapicloud/evolution-api:latest` | Gerencia instâncias WhatsApp via API REST |
| **PostgreSQL 15** | `postgres:15` | Banco de dados da Evolution API |
| **Redis** | `redis:latest` | Cache e fila de mensagens |
| **n8n** | `docker.n8n.io/n8nio/n8n:latest` | Plataforma de automação de fluxos |

### O que é criado automaticamente?

Ao subir os containers pela primeira vez, o script `init-n8n.sh` executa e:

- Instala o community node **n8n-nodes-evolution-api**
- Cria a credencial **"Evolution API"** no n8n apontando para o container interno
- Cria a credencial **"Redis"** no n8n apontando para o container interno

Tudo pronto para usar sem precisar configurar nada manualmente!

---

## Requisitos

- [Docker](https://docs.docker.com/get-docker/) instalado
- [Docker Compose](https://docs.docker.com/compose/install/) (v2+)
- Portas **5678** (n8n) e **9090** (Evolution API) disponíveis na máquina

---

## Instalação

### 1. Clone o repositório

```bash
git clone https://github.com/randomname124290358349/EvolutionN8n.git
cd EvolutionN8n
```

### 2. Configure as variáveis de ambiente

Antes de subir os containers, edite o `docker-compose.yml` e altere os seguintes valores:

```yaml
# PostgreSQL — escolha uma senha segura
POSTGRES_PASSWORD: SuaSenhaSegura

# Evolution API — chave de autenticação da API
AUTHENTICATION_API_KEY: SuaChaveSegura

# n8n — credenciais do administrador
N8N_OWNER_EMAIL: seuemail@dominio.com
N8N_OWNER_PASSWORD: "SuaSenhaSegura"

# (Opcional) Se tiver um domínio, descomente e configure:
# WEBHOOK_URL: https://seu-dominio.com/
# N8N_EDITOR_BASE_URL: https://seu-dominio.com/
```

> **Atenção:** Use a **mesma senha** em `POSTGRES_PASSWORD` e em `DATABASE_CONNECTION_URI`, e a **mesma chave** em `AUTHENTICATION_API_KEY` e `EVOLUTION_API_KEY`.

### 3. Suba os containers

```bash
docker compose up -d
```

### 4. Aguarde a inicialização

Na primeira execução, o script de setup pode levar alguns minutos. Acompanhe os logs do n8n:

```bash
docker logs -f n8n
```

Aguarde até ver a mensagem:

```
[init] Setup concluído com sucesso!
```

### 5. Acesse o n8n

Abra o navegador em: **http://localhost:5678**

Faça login com o e-mail e senha que você definiu no passo 2.

As credenciais **"Evolution API"** e **"Redis"** já estarão disponíveis para uso nos seus fluxos!

---

## Arquitetura

```
┌─────────────────────────────────────────────────────┐
│                    evolution-net                    │
│                                                     │
│  ┌──────────────┐    ┌──────────┐    ┌───────────┐ │
│  │ Evolution API│◄───│  Redis   │    │ PostgreSQL│ │
│  │   :9090      │    │  :6379   │    │   :5432   │ │
│  └──────┬───────┘    └──────────┘    └─────┬─────┘ │
│         │                                   │       │
│         └──────────────┬────────────────────┘       │
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

> A Evolution API fica acessível apenas localmente em `127.0.0.1:9090` — ela não é exposta diretamente para a internet por segurança.

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

## Resetar as credenciais do n8n

Se precisar recriar as credenciais automáticas, remova o marker de inicialização e reinicie o container:

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
