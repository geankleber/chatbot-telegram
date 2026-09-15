# Chatbot Telegram — Weather Bot with N8N

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

Chatbot para Telegram que informa a temperatura atual de qualquer cidade do Brasil. O usuário envia o nome da cidade e o estado (ex.: `Curitiba, PR`), o workflow consulta a API da OpenWeather, processa a resposta e envia de volta uma mensagem curta e amigável com a temperatura. Opcionalmente, a mensagem final é reescrita pelo Google Gemini para um tom mais natural, com fallback automático para uma mensagem determinística caso o Gemini não esteja configurado.

## Tech Stack

- **[N8N](https://n8n.io/)** — workflow automation engine
- **[Telegram Bot API](https://core.telegram.org/bots/api)** — messaging interface
- **[OpenWeather API](https://openweathermap.org/api)** — weather data provider
- **[Google Gemini](https://ai.google.dev/)** — optional LLM rewriting
- **Docker Compose** — Postgres + Redis + N8N in queue mode with Traefik (HTTPS)

## Como funciona

1. **Telegram Trigger** recebe a mensagem enviada ao bot.
2. **Queue** normaliza o texto (remove espaços, acentos, deixa em minúsculas) e monta a query no formato `cidade,uf,br` esperado pela OpenWeather. Também extrai o `chat_id` do usuário.
3. **HTTP Request** consulta `https://api.openweathermap.org/data/2.5/weather` com os parâmetros `q`, `units=metric`, `lang=pt_br` e a API key da OpenWeather (via credencial).
4. **If** verifica se a resposta retornou erro (cidade não encontrada / status diferente de 200):
   - **Erro** → monta a mensagem `❌ Cidade não encontrada.`
   - **Sucesso** → monta a mensagem determinística `☀️ A temperatura em {cidade} é de {temperatura}°C`
5. *(Opcional)* **Basic LLM Chain + Google Gemini** reescreve a mensagem de sucesso de forma mais natural, retornando um JSON `{"message": "...", "ok": true}` validado por um Structured Output Parser.
   - Se o Gemini responder com sucesso, a mensagem reescrita é usada.
   - Se o Gemini falhar (sem credencial configurada, erro de API, etc.), o workflow usa automaticamente a mensagem determinística do passo 4 como fallback.
6. **Send a text message** envia a mensagem final ao usuário no Telegram.

## Pré-requisitos

- Instância do N8N (local via Docker ou cloud).
- Bot criado no Telegram via [@BotFather](https://t.me/BotFather).
- Conta na [OpenWeather](https://home.openweathermap.org/users/sign_up) com uma API key gerada.
- *(Opcional)* Conta no [Google AI Studio](https://aistudio.google.com/) com uma API key do Gemini.

## Quick Start

```bash
# 1. Clone o repositório
git clone https://github.com/geankleber/chatbot-telegram.git
cd chatbot-telegram

# 2. Configure o ambiente
cp .env.example .env
# Edite .env com seus valores (domínio, Postgres, chave de criptografia)

# 3. Suba os containers
docker compose up -d

# 4. Importe o workflow no N8N e configure as credenciais (veja abaixo)
```

## Importando o workflow

1. No N8N, vá em **Workflows** → **Add Workflow** → menu `...` → **Import from File**.
2. Selecione o arquivo `workflow-chatbot-telegram.json`.
3. Os nodes que usam credenciais ficarão sem credencial vinculada — siga a seção abaixo para configurá-las.
4. Após configurar todas as credenciais, publique/ative o workflow (botão **Publish** / toggle **Active**).

## Configurando as credenciais

O workflow usa três credenciais, nenhuma delas incluída no JSON exportado por questões de segurança.

### 1. Telegram (`Telegram account`)

Usada nos nodes **Telegram Trigger** e **Send a text message**.

1. Em **Credentials** → **Add Credential** → busque por **Telegram API**.
2. Em **Access Token**, cole o seu `TELEGRAM_BOT_TOKEN` (obtido com o @BotFather ao criar o bot via `/newbot`).
3. Salve e vincule essa credencial nos dois nodes Telegram do workflow.

### 2. OpenWeather (`Custom Auth account`)

Usada no node **HTTP Request**.

A OpenWeather espera a API key como query parameter (`appid`), então a credencial é do tipo **Custom Auth**:

1. Em **Credentials** → **Add Credential** → busque por **Custom Auth**.
2. No campo **JSON**, insira:
   ```json
   {
     "qs": {
       "appid": "SUA_OPENWEATHER_API_KEY"
     }
   }
   ```
3. Substitua `SUA_OPENWEATHER_API_KEY` pela chave gerada em [home.openweathermap.org/api_keys](https://home.openweathermap.org/api_keys).
4. Salve e vincule essa credencial no node **HTTP Request** (Authentication → Generic Credential Type → Custom Auth).

### 3. Google Gemini (`Google Gemini(PaLM) Api account`) — opcional

Usada no node **Google Gemini Chat Model**.

1. Em **Credentials** → **Add Credential** → busque por **Google Gemini (PaLM) Api**.
2. Cole sua API key gerada em [aistudio.google.com/apikey](https://aistudio.google.com/apikey).
3. Salve e vincule essa credencial no node **Google Gemini Chat Model**.

> Se essa credencial não for configurada, o node "Basic LLM Chain" falhará silenciosamente e o workflow usará automaticamente a mensagem determinística (fallback), sem custo adicional e sem interromper o funcionamento do bot.

## Variáveis de ambiente

| Variável | Onde é usada | Obrigatória |
|---|---|---|
| `N8N_HOST` | Domínio público do N8N (HTTPS via Traefik) | Sim |
| `POSTGRES_USER` | Usuário do banco de dados | Sim |
| `POSTGRES_PASSWORD` | Senha do banco de dados | Sim |
| `POSTGRES_DB` | Nome do banco de dados | Sim |
| `N8N_ENCRYPTION_KEY` | Chave de criptografia do N8N | Sim |

As credenciais de API (Telegram, OpenWeather, Gemini) são configuradas diretamente no N8N, não como variáveis de ambiente.

## Testando o chatbot

1. Após publicar o workflow, abra o Telegram e procure pelo bot que você criou.
2. Envie uma mensagem no formato `Cidade, UF`, por exemplo:
   - `Curitiba, PR`
   - `Belo Horizonte, MG`
   - `São Paulo, SP`

   **Resposta esperada** (com Gemini configurado, o texto pode variar):
   ```
   ☀️ A temperatura em Curitiba é de 18°C
   ```

3. Teste também com uma cidade inexistente:
   ```
   ❌ Cidade não encontrada.
   ```

## Estrutura do projeto

```
chatbot-telegram/
├── .env.example                    # Template de variáveis de ambiente
├── .gitignore
├── docker-compose.yml              # Postgres + Redis + N8N (queue mode) + Traefik
├── workflow-chatbot-telegram.json   # Workflow N8N exportado
├── LICENSE
└── README.md
```

## Observações

- O formato de entrada esperado é `Cidade, UF` (o workflow normaliza automaticamente espaços, acentos e maiúsculas/minúsculas).
- Apelidos populares de cidades que não correspondem à sigla oficial (ex.: "BH" em vez de "MG") podem não ser reconhecidos pela API da OpenWeather.
- Nenhum token ou chave de API está presente neste repositório. Configure todas as credenciais diretamente no N8N.

## License

This project is licensed under the Apache License 2.0 — see the [LICENSE](LICENSE) file for details.
