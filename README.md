# 🚀 Automação de Pedidos e Estoque via WhatsApp (Case Supfy)

MVP de digitalização de operação comercial para distribuidora de alimentos: recebimento de pedidos via WhatsApp com IA, validação de estoque em tempo real e alertas automáticos de preço/estoque.

**Documento completo (diagnóstico, priorização, arquitetura):** `Documento_case_tecnico.pdf`
**Vídeo de demonstração:** link no Drive (ver documento)

---

## 🧩 Arquitetura

```
Cliente (WhatsApp)
      │
      ▼
Evolution API ──(webhook)──► n8n
                                │
                  ┌─────────────┼─────────────┐
                  ▼             ▼             ▼
             Texto?        Foto?         Alteração
             (pedido/      (comprovante,  no NocoDB?
              dúvida)       simulação)    (preço/estoque)
                  │             │             │
                  ▼             ▼             ▼
          IA extrai item   IA valida     Redis (debounce)
          Redis (buffer)   e atualiza    → Alerta consolidado
          NocoDB (estoque)  NocoDB       no grupo WhatsApp
                  │
                  ▼
          Confirma pedido / avisa falta de estoque
```

Todos os serviços rodam em containers Docker isolados, conectados pela rede `automation_net`.

---

## 🛠️ Serviços

| Serviço | Porta | Função |
|---|---|---|
| **n8n** | 5678 | Orquestrador dos fluxos de automação e IA |
| **Evolution API** | 8080 | Ponte com o WhatsApp (webhooks de entrada/saída) |
| **Redis** | 6379 | Debounce de webhooks e buffer de mensagens picadas |
| **NocoDB** | 8081 | Banco de dados relacional (substitui as planilhas) |

---

## 📂 Fluxos n8n (pasta `/workflows`)

| Arquivo | O que faz |
|---|---|
| `Recebimento_de_pedidos___Duvidas.json` | Fluxo principal do MVP. Recebe mensagem no WhatsApp → bufferiza no Redis → IA extrai itens do pedido → valida estoque no NocoDB → confirma ou avisa falta → dá baixa no estoque. Se a mensagem não for pedido, cai no agente de dúvidas. Inclui fluxo de validação de comprovante por imagem (**simulação**). |
| `Movimentação_estoque.json` | Monitora alterações de preço/estoque no NocoDB via webhook. Usa Redis como lock para evitar alertas duplicados. Envia notificação consolidada no grupo de WhatsApp da distribuidora. |
| `Estoque_baixo.json` | Varre o estoque completo a cada alteração e classifica produtos em **crítico** (≤20) ou **atenção** (≤40), disparando alerta correspondente. |

**Faltando no MVP (descrito no documento, não implementado):** script Python de Web Scraping para atualização automática de preços via portal do fornecedor.

---

## ⚙️ Como rodar

### 1. Pré-requisitos
- Docker e Docker Compose instalados
- IP local da máquina host (ex: `192.168.x.x`) — necessário porque os containers não se comunicam via `localhost`

### 2. Subir a infraestrutura

```bash
cp .env.example .env
# edite o .env com seu IP local e chaves de API
docker compose up -d
```

Isso sobe os 4 serviços (n8n, Evolution API + Postgres, Redis, NocoDB) na mesma rede.

### 3. Evolution API (fork customizado)

Este projeto usa um fork da [Evolution API](https://github.com/EvolutionAPI/evolution-api) com alterações no código-fonte. Para buildar localmente:

```bash
git clone https://github.com/EvolutionAPI/evolution-api
# aplique as alterações deste repositório (pasta /evolution-custom)
docker build -t evolution-api-custom .
```

Referencie a imagem customizada no `docker-compose.yml` antes de subir.

### 4. Configurar credenciais no n8n

Acesse `http://<SEU_IP>:5678` e crie as credenciais:
- **NocoDB API Token** (gerado em Configurações → API Tokens no NocoDB)
- **Redis** (host: `redis`, porta `6379`)
- **OpenAI** (para os agentes de extração e dúvidas)

### 5. Importar os fluxos

No n8n: Workflows → Import from File → selecione cada `.json` da pasta `/workflows`.
Atualize os IDs de `workspaceId`/`projectId`/`table` do NocoDB e o número do grupo de WhatsApp para os seus.

### 6. Conectar o WhatsApp

Na Evolution API, crie uma instância e escaneie o QR Code para conectar o número de testes.

---

## ⚠️ Configuração de rede

Nunca use `localhost` nas URLs de webhook — os containers não se enxergam por esse endereço.

```
❌ http://localhost:8080/message/sendText/...
✅ http://192.168.x.x:8080/message/sendText/...
```

Substitua pelo IP local real da sua máquina em todas as variáveis `SERVER_URL`, `CACHE_REDIS_URI` e nos nodes HTTP Request dos fluxos.

---

## 🔐 Variáveis de ambiente (`.env.example`)

```env
# Rede
HOST_IP=192.168.x.x

# Evolution API
EVOLUTION_API_KEY=sua_chave_aqui
EVOLUTION_DB_USER=admin
EVOLUTION_DB_PASSWORD=troque_esta_senha
EVOLUTION_DB_NAME=evolution_db

# NocoDB (gerar após primeiro acesso)
NOCODB_API_TOKEN=seu_token_aqui
```

> As credenciais do `docker-compose.yaml` original eram valores de exemplo para ambiente local — troque antes de qualquer uso além de teste.

---

## 📊 Testes realizados

- Taxa de acerto da IA na extração de pedidos: ~95% (produtos regionais são o principal ponto de falha)
- Debounce de alertas: 0 duplicatas em testes com múltiplas edições simultâneas no NocoDB
- Validação de comprovante por imagem: fluxo desenhado e simulado, não conectado a um modelo de visão real

---

## 🔭 Próximos passos

- Implementar o Web Scraper em Python (Selenium/Playwright) para sincronizar preços do fornecedor
- Conectar a validação de comprovante a um modelo de visão real (GPT-4o Vision ou Claude Vision)
- Migrar NocoDB → PostgreSQL para maior robustez em produção
- Adicionar catálogo de sinônimos regionais no prompt de extração de pedidos