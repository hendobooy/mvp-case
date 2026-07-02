# 🚀 Arquitetura de Automação de Pedidos e Estoque (WhatsApp)

Este repositório contém a infraestrutura e os fluxos de automação para digitalização de processos de vendas e gestão de estoque via WhatsApp. O sistema orquestra a comunicação com clientes, gerencia o banco de dados em tempo real e utiliza Inteligência Artificial para extração de pedidos e atendimento.

Todos os códigos, fluxos e customizações estão disponíveis e detalhados no meu perfil do GitHub: [hendobooy](https://www.google.com/search?q=https://github.com/hendobooy).

---

## 🛠️ Tecnologias Utilizadas e Propósitos

A arquitetura é dividida em 4 serviços principais, cada um rodando de forma isolada em containers Docker para garantir estabilidade e escalabilidade.

* **n8n (Porta 5678):** É o cérebro orquestrador da operação. Utilizado para integrar as APIs, conectar-se com a OpenAI (para extração de dados JSON e agentes de conversação) e automatizar o roteamento de mensagens lógicas.
* **Evolution API (Porta 8080):** Responsável pela ponte direta com o WhatsApp. Utilizada para receber eventos e enviar mensagens via chamadas HTTP (Webhooks) para o n8n.
* **Redis (Porta 6379):** Banco de dados em memória. Utilizado exclusivamente para controle de concorrência de automação (*Locks*) e *Debounce*.
* **NocoDB (Porta 8081):** Atua como o banco de dados relacional central, substituindo planilhas manuais.

---

## ⚠️ Configuração de Rede e Webhooks

Para que os serviços se comuniquem corretamente na máquina hospedeira, **nunca utilize `localhost**` nas URLs de requisição HTTP ou configurações de Webhook. Utilize sempre o **IP local da sua máquina host seguido da porta do serviço**.

* ❌ *Incorreto:* `http://localhost:8080/message/sendText/...`
* ✅ *Correto:* `[http://192.168.2.100:8080/message/sendText/](http://192.168.2.100:8080/message/sendText/)...` (Substitua pelo seu IP de rede real).

---

## 🔧 Customização e Instalação (Evolution API)

Para atender a requisitos específicos desta arquitetura, não utilizamos a imagem padrão da Evolution API. A versão utilizada neste projeto é um **fork customizado** com alterações no código-fonte.

**Instruções de Instalação:**
Para rodar a Evolution API, você deve clonar o repositório oficial da [Evolution API](https://github.com/EvolutionAPI/evolution-api), substituir o código-fonte pela versão customizada deste repositório (ou aplicar as alterações necessárias) e realizar o build da imagem localmente (`docker build`). Certifique-se de que o Docker esteja configurado para utilizar essa imagem customizada no seu `docker-compose.yml`.

### ⚙️ Configuração de Ambiente (.env)

Renomeie o `.env.example` para `.env` e ajuste as variáveis essenciais:

**1. URL e Autenticação:**

* `SERVER_URL=http://<SEU_IP_LOCAL>:8080`
* `AUTHENTICATION_API_KEY=sua_chave_secreta_aqui`

**2. Conexão com o Redis:**

* `CACHE_REDIS_ENABLED=true`
* `CACHE_REDIS_URI=redis://<SEU_IP_LOCAL>:6379/6`

**3. Webhooks:**

* `WEBHOOK_EVENTS_MESSAGES_UPSERT=true`

---

## 🐳 Como subir a infraestrutura (Docker Compose)

Execute o comando `docker compose up -d` na pasta onde os arquivos de serviço estão localizados.

### 1. Inicializando o Redis

```yaml
version: "3.8"
services:
  redis:
    container_name: redis_local
    image: redis:alpine
    restart: always
    ports:
      - "6379:6379"
    command: redis-server --save 60 1 --loglevel warning

```

### 2. Inicializando o n8n

```yaml
version: '3.8'
services:
  n8n:
    image: n8nio/n8n
    container_name: n8n_local
    ports:
      - "5678:5678"
    volumes:
      - ~/.n8n:/home/node/.n8n
    restart: unless-stopped

```

---

## 🔄 Fluxos de Automação (n8n Workflows)

Para ativar a lógica da operação, importe os arquivos `.json` disponíveis na pasta `end-to-end` deste repositório diretamente na sua instância do n8n:

1. **Recebimento de Pedidos e Triagem de IA:** Extrai dados de pedidos via IA, realiza o cruzamento com o estoque no NocoDB e processa a baixa em tempo real.
2. **Notificação e Movimentação de Estoque:** Monitora alterações de preços ou volumes, utilizando o Redis para controle de *Debounce*, garantindo o envio de alertas consolidados para a equipe de vendas.