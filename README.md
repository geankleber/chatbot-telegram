# Bot de Clima no Telegram com N8N

Chatbot para Telegram que informa a temperatura atual de qualquer cidade do Brasil. O usuário envia o nome da cidade e o estado (ex.: `Curitiba, PR`), o workflow consulta a API da OpenWeather, processa a resposta e envia de volta uma mensagem curta e amigável com a temperatura. Opcionalmente, a mensagem final é reescrita pelo Google Gemini para um tom mais natural, com fallback automático para uma mensagem determinística caso o Gemini não esteja configurado.

## Como funciona

1. **Telegram Trigger** recebe a mensagem enviada ao bot.
2. **queue** normaliza o texto (remove espaços, acentos, deixa em minúsculas) e monta a query no formato `cidade,uf,br` esperado pela OpenWeather. Também extrai o `chat_id` do usuário.
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
- Bot criado no Telegram via [@BotFather](https://t.me/BotFather) — bot: **ClimaTempo** (`@meuClimaTempo_bot`).
- Conta na [OpenWeather](https://home.openweathermap.org/users/sign_up) com uma API key gerada.
- *(Opcional)* Conta no [Google AI Studio](https://aistudio.google.com/) com uma API key do Gemini.

## Importando o workflow

1. No N8N, vá em **Workflows** → **Add Workflow** → menu `...` → **Import from File**.
2. Selecione o arquivo `workflow-chatbot-telegram.json`.
3. O workflow será importado, mas os nodes que usam credenciais ficarão sem credencial vinculada — siga a seção abaixo para configurá-las.
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

## Variáveis esperadas

| Variável | Onde é usada | Obrigatória |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | Credencial "Telegram account" | Sim |
| `OPENWEATHER_API_KEY` | Credencial "Custom Auth account" (campo `qs.appid`) | Sim |
| Gemini API Key | Credencial "Google Gemini(PaLM) Api account" | Não (opcional) |

Nenhuma dessas variáveis está presente no arquivo `workflow-chatbot-telegram.json` — todas devem ser configuradas manualmente como credenciais no N8N após a importação.

## Rodando o N8N com Docker (opcional)

Este reposit\u00f3rio inclui um `docker-compose.yml` de exemplo (Postgres + Redis + N8N em modo queue, com Traefik para HTTPS).

1. Copie `.env.example` para `.env` e preencha os valores (dom\u00ednio, credenciais do Postgres, chave de criptografia do N8N).
2. Garanta que a rede externa `traefik-public` j\u00e1 exista (ou ajuste o compose para seu cen\u00e1rio).
3. Suba os containers:
   ```bash
   docker compose up -d
   ```
4. Acesse `https://SEU_DOMINIO` configurado em `N8N_HOST`.

> O arquivo `.env` nunca deve ser commitado \u2014 ele j\u00e1 est\u00e1 listado no `.gitignore`.

## Testando o chatbot

1. Após publicar o workflow, abra o Telegram e procure por **@meuClimaTempo_bot**.
2. Envie uma mensagem no formato `Cidade, UF`, por exemplo:
   - `Curitiba, PR`
   - `Belo Horizonte, MG`
   - `São Paulo, SP`

   **Resposta esperada** (com Gemini configurado, o texto pode variar levemente):
   ```
   ☀️ A temperatura em Curitiba é de 18°C
   ```

3. Teste também com uma cidade inexistente, por exemplo:
   - `Cidade Inventada, XX`

   **Resposta esperada:**
   ```
   ❌ Cidade não encontrada.
   ```

## Observações

- O formato de entrada esperado é `Cidade,UF,BR` (o workflow normaliza automaticamente espaços, acentos e maiúsculas/minúsculas).
- Apelidos populares de cidades que não correspondem à sigla oficial do estado (ex.: "BH" em vez de "MG") podem não ser reconhecidos pela API da OpenWeather e retornarão a mensagem de erro — esse é um comportamento esperado da API, fora do controle do workflow.
- Nenhum token ou chave de API está presente neste repositório. Configure todas as credenciais diretamente no N8N conforme descrito acima.
