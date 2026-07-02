Aqui está o arquivo completo. Basta copiar tudo abaixo e salvar como `README.md` no seu repositório.

---

# 🚀 Arquitetura de Automação de Pedidos e Estoque (WhatsApp)

Este repositório contém a infraestrutura e os fluxos de automação para digitalização de processos de vendas e gestão de estoque via WhatsApp. O sistema orquestra a comunicação com clientes, gerencia o banco de dados em tempo real e utiliza Inteligência Artificial para extração de pedidos e atendimento.

Todos os códigos, fluxos e customizações estão disponíveis e detalhados no meu perfil do GitHub: [hendobooy](https://www.google.com/search?q=https://github.com/hendobooy).

---

## 🛠️ Tecnologias Utilizadas e Propósitos

A arquitetura é dividida em 4 serviços principais, cada um rodando de forma isolada em containers Docker para garantir estabilidade e escalabilidade.

* **n8n (Porta 5678):** É o cérebro orquestrador da operação. Utilizado para integrar as APIs, conectar-se com a OpenAI (para extração de dados JSON e agentes de conversação) e automatizar o roteamento de mensagens lógicas.
* **Evolution API (Porta 8080):** Responsável pela ponte direta com o WhatsApp. Utilizada para receber eventos e enviar mensagens via chamadas HTTP (Webhooks) para o n8n.
* **Redis (Porta 6379):** Banco de dados em memória. Utilizado exclusivamente para controle de concorrência de automação (*Locks*) e *Debounce*. Ele impede que gatilhos duplicados (como 3 webhooks disparados pelo banco de dados na mesma fração de segundo) gerem repetições de processos ou envio de spam no WhatsApp.
* **NocoDB (Porta: 8081):** Atua como o banco de dados relacional central, substituindo completamente planilhas manuais. Ele armazena o catálogo de produtos, o estoque (que sofre baixa em tempo real) e o registro de todos os pedidos efetuados e faturados.

---

## ⚠️ Configuração de Rede e Webhooks

Para que os serviços (n8n, Evolution API e NocoDB) se comuniquem corretamente na máquina hospedeira, **nunca utilize `localhost**` nas URLs de requisição HTTP ou configurações de Webhook.

Como cada serviço roda isolado em seu container Docker, o termo `localhost` fará o serviço tentar buscar a informação dentro do próprio container, resultando em erro. Utilize sempre o **IP local da sua máquina host seguido da porta do serviço**.

* ❌ *Incorreto:* `http://localhost:8080/message/sendText/...`
* ✅ *Correto:* `http://192.168.2.100:8080/message/sendText/...` (Substitua pelo seu IP de rede real).

---

## 🔧 Customização Open Source (Evolution API)

Para atender a requisitos específicos desta arquitetura e otimizar a leitura de payloads, não utilizamos a imagem padrão da Evolution API. Foi necessário fazer um clone (fork) de todo o repositório open-source original para realizar alterações profundas no código-fonte. Toda essa versão customizada, junto com suas variáveis de ambiente específicas, está configurada e versionada neste repositório.

### ⚙️ Configuração de Ambiente (.env) da Evolution API

O repositório possui um arquivo `.env.example` com todas as variáveis possíveis. No entanto, para que esta arquitetura de automação funcione corretamente integrada ao **n8n** e ao **Redis**, é **obrigatório** alterar as seguintes variáveis antes de subir o container:

**1. URL e Autenticação:**

* `SERVER_URL=http://<SEU_IP_LOCAL>:8080` (Nunca use localhost)
* `AUTHENTICATION_API_KEY=sua_chave_secreta_aqui` (Esta mesma chave deverá ser usada no Header `apikey` dos nós HTTP no n8n)

**2. Conexão com o Redis (Fundamental para o fluxo):**
Como estamos usando o Redis para gerenciar o *Debounce* e *Locks* de estoque, a Evolution também pode utilizá-lo para cache. Aponte para o IP correto da máquina host:

* `CACHE_REDIS_ENABLED=true`
* `CACHE_REDIS_URI=redis://<SEU_IP_LOCAL>:6379/6`

**3. Webhooks (Disparos para o n8n):**
Certifique-se de que os eventos de mensagens estão habilitados para que a API envie os dados ao n8n quando um cliente mandar mensagem:

* `WEBHOOK_EVENTS_MESSAGES_UPSERT=true`

> **Nota:** Renomeie o arquivo `.env.example` para `.env` na pasta da Evolution API e ajuste essas variáveis antes de executar o Docker Compose.

---

## 🐳 Como subir a infraestrutura (Docker Compose)

Para iniciar os serviços base de automação, crie os arquivos `docker-compose.yml` correspondentes (ou unifique-os) e execute o comando `docker compose up -d` no terminal da respectiva pasta.

### 1. Inicializando o Redis (Sistema de Buffer e Locks)

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

### 2. Inicializando o n8n (Motor de Automação)

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

*(A configuração Docker da Evolution API customizada e do NocoDB devem seguir os arquivos de ambiente detalhados nas documentações de seus respectivos diretórios).*

---

## 🔄 Fluxos de Automação (n8n Workflows)

Para ativar a lógica da operação, importe os arquivos `.json` disponíveis neste repositório diretamente na sua instância do n8n:

1. **Recebimento de Pedidos e Triagem de IA:** Recebe a mensagem do cliente via webhook e avalia a intenção (Switch). Se for uma dúvida comum, um Agente de IA (LangChain) responde naturalmente. Se for uma intenção de compra, uma IA extratora converte o texto livre em um array JSON estruturado, faz o cruzamento com o estoque no NocoDB, aprova itens disponíveis, rejeita itens faltantes e realiza o *Update* automático dando baixa na quantidade, finalizando com o aceite no WhatsApp.
2. **Notificação e Movimentação de Estoque:** Monitora alterações de preços ou volumes feitas diretamente no banco de dados. Utiliza o Redis para criar chaves temporárias exclusivas (`lock_movimentacao_ID`) e agrupar múltiplos disparos de Webhook (Debounce), garantindo que os clientes e a equipe de vendas recebam apenas uma mensagem limpa e consolidada sobre a alteração.