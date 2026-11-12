# Bot de Clima no Telegram com N8N

Chatbot do Telegram, construído no **n8n**, que informa a **temperatura atual de
qualquer cidade do Brasil**. A pessoa envia o nome da cidade (e, se quiser, a UF),
o workflow consulta a API gratuita da **OpenWeather** e responde com uma mensagem
curta e amigável:

```
Você:  Curitiba, PR
Bot:   🌤️ A temperatura em Curitiba é de 18°C.

Você:  Grifinória
Bot:   ❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).
```

Desafio prático da disciplina *Agentes e Automação* da Pós em IA e Automação
(Rocketseat).

## Arquivos

| Arquivo | Para que serve |
|---------|----------------|
| `workflow-chatbot-telegram.json` | Workflow exportado do n8n (sem credenciais nem tokens). |
| `docker-compose.yml` | Ambiente local do n8n usado nas aulas: editor + worker (modo fila), PostgreSQL, Redis e ngrok. |
| `.env.example` | Modelo das variáveis de ambiente. Copie para `.env` e preencha. |
| `ngrok.yml` | Arquivo montado no container do ngrok, como no curso. |

## Como o workflow funciona

```
Telegram Trigger
  → Formatar cidade (Set: variável "queue")
  → Consultar OpenWeather (HTTP Request)
  → Resposta válida? (IF)
       ├─ sim → Extrair temperatura (Set) → Mensagem fallback (Code)
       │          → Gemini - melhorar mensagem (opcional) → Enviar temperatura (Telegram)
       └─ não → Enviar erro (Telegram)
```

1. **Telegram Trigger**: recebe as mensagens de texto enviadas ao bot (`updates: message`).
2. **Formatar cidade** (Set): guarda o texto na variável `queue`, removendo espaços
   extras, acentos e maiúsculas, e completa com `,br` (país) quando não informado.
   A UF pode vir separada por vírgula, espaço, hífen, barra ou entre parênteses
   (só as 27 siglas oficiais são reconhecidas, para não cortar nomes de cidade):

   | Mensagem | `queue` |
   |----------|---------|
   | `São Paulo, SP` | `sao paulo,sp,br` |
   | `campo grande ms` | `campo grande,ms,br` |
   | `Campo Grande - MS` / `Campo Grande (MS)` | `campo grande,ms,br` |
   | `São Paulo/SP` | `sao paulo,sp,br` |
   | `Rio de Janeiro` | `rio de janeiro,br` |
   | `Ji-Paraná, RO` | `ji-parana,ro,br` |
3. **Consultar OpenWeather** (HTTP Request): `GET https://api.openweathermap.org/data/2.5/weather`
   com **Send Query Parameters** ativado:
   - `q` = `{{ $json.queue }}` (a mensagem formatada);
   - `units` = `metric` (graus Celsius);
   - `lang` = `pt_br` (português brasileiro);
   - `appid` = `{{ $env.OPENWEATHER_API_KEY }}` (chave lida da variável de ambiente).

   A resposta completa (status HTTP + corpo) é repassada e o nó não interrompe o
   fluxo em erros HTTP, para que o IF decida o caminho.
4. **Resposta válida?** (IF): segue pelo caminho *verdadeiro* só se o status HTTP
   for `200` **e** o JSON tiver `main.temp`. Cidade inexistente (404), mensagem vazia
   ou qualquer resposta inválida vão para o caminho *falso*.
5. **Extrair temperatura** (Set): pega o nome da cidade (`name`), a temperatura
   arredondada (`Math.round(main.temp)`) e a descrição do tempo.
6. **Mensagem fallback** (Code): monta, de forma determinística, a mensagem final
   `🌤️ A temperatura em <cidade> é de <temperatura>°C.` no formato
   `{"message": "...", "ok": true}`.
7. **Gemini - melhorar mensagem** (opcional): veja a seção [Google Gemini](#google-gemini-opcional).
8. **Enviar temperatura** / **Enviar erro** (Telegram → Send Message): respondem no
   mesmo chat (`message.chat.id`).

> **Sobre `queue` × `q`:** o enunciado pede a variável `queue`. Ela existe no nó
> *Formatar cidade*, mas o parâmetro enviado à OpenWeather se chama `q`, que é o
> nome aceito pela API (`queue` é ignorado por ela e a consulta sempre falharia).
>
> **Sobre a UF:** o formato `Cidade,UF,BR` é o pedido no desafio, mas a OpenWeather
> só usa o código de estado para cidades dos EUA. Para o Brasil ela considera o nome
> da cidade e o país, então a UF é informativa (ex.: `bom jesus,rs,br` retorna Bom
> Jesus/PI). Cidades homônimas podem trazer a de outro estado.

## Pré-requisitos

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (com Docker Compose).
- Conta gratuita no [ngrok](https://ngrok.com) com o **authtoken** e o **domínio estático**
  gratuito. O Telegram só envia mensagens para webhooks **HTTPS**, e o ngrok expõe o
  n8n local com essa URL.
- Um bot do Telegram criado no **@BotFather** (`/newbot`), com o token de acesso.
- Conta gratuita na [OpenWeather](https://home.openweathermap.org/users/sign_up) e a
  **API key** (em *API keys*). Chaves novas podem levar até ~2 horas para ativar.
- Opcional: API key do Gemini no [Google AI Studio](https://aistudio.google.com/apikey).

## Variáveis de ambiente

Copie o modelo e preencha os valores (o `.env` está no `.gitignore` e nunca vai
para o repositório):

```bash
cp .env.example .env
```

| Variável | Obrigatória | Onde é usada |
|----------|-------------|--------------|
| `OPENWEATHER_API_KEY` | sim | Lida no nó HTTP Request com `{{ $env.OPENWEATHER_API_KEY }}`. |
| `TELEGRAM_BOT_TOKEN` | sim | Token do @BotFather, usado na credencial **Telegram API** do n8n. |
| `NGROK_AUTHTOKEN` | sim | Autentica o container do ngrok. |
| `NGROK_DOMAIN` | sim | Domínio estático do ngrok, sem `https://`. Vira o `WEBHOOK_URL` do n8n. |
| `GEMINI_API_KEY` | não | API key do Google AI Studio, usada na credencial do Gemini. |

Para o n8n ler variáveis de ambiente nas expressões, o `docker-compose.yml` usa
`N8N_BLOCK_ENV_ACCESS_IN_NODE=false`, como nas aulas.

## Subindo o ambiente

```bash
docker compose up -d
docker compose ps
```

Serviços:

- `n8n-editor` (container `n8n-editor-ftr`): interface em http://localhost:5678 e
  na URL pública `https://<NGROK_DOMAIN>`;
- `worker`: executa os workflows (modo fila);
- `postgres` (porta `5434` no host; dentro da rede Docker é `postgres:5432`);
- `redis`: fila de execuções;
- `ngrok`: túnel HTTPS (painel em http://localhost:4040).

Na primeira vez, abra http://localhost:5678 e crie a conta de dono (owner) do n8n.

Para desligar: `docker compose down`. Os dados ficam nos volumes `n8n-data`,
`postgres-data` e `redis-data`.

## Importando o workflow

1. No n8n, clique em **Create Workflow** (ou abra um workflow vazio).
2. No menu **⋯** (canto superior direito), escolha **Import from File...**.
3. Selecione `workflow-chatbot-telegram.json` e salve.

## Credenciais no n8n

O JSON do workflow não traz nenhuma chave, só o nome das credenciais. Depois de importar:

1. **Telegram**: em **Credentials → Create Credential → Telegram API**, preencha
   *Access Token* com o token do bot. Duas formas:
   - colar o valor de `TELEGRAM_BOT_TOKEN` (como nas aulas); ou
   - usar a expressão `{{ $env.TELEGRAM_BOT_TOKEN }}`, que lê a variável do `.env`
     (é a forma usada neste projeto; funciona porque `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`).

   Selecione essa credencial nos nós **Telegram Trigger**, **Enviar temperatura** e
   **Enviar erro**.
2. **OpenWeather**: não precisa de credencial no n8n. O nó *Consultar OpenWeather*
   lê a chave da variável `OPENWEATHER_API_KEY` (definida no `.env`).
3. **Gemini** (opcional): veja a seção abaixo.

Por fim, **ative** o workflow (chave *Inactive/Active* no topo). Ao ativar, o n8n
registra o webhook do bot no Telegram usando a URL do ngrok (`WEBHOOK_URL`).

## Testando o chatbot

Abra o bot no Telegram e envie:

| Mensagem | Resposta esperada |
|----------|-------------------|
| `Curitiba, PR` | `🌤️ A temperatura em Curitiba é de <N>°C.` |
| `campo grande ms` | `🌤️ A temperatura em Campo Grande é de <N>°C.` |
| `São Paulo/SP` | `🌤️ A temperatura em São Paulo é de <N>°C.` |
| `Belo Horizonte` | `🌤️ A temperatura em Belo Horizonte é de <N>°C.` |
| `Grifinória` | `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).` |

Mensagens que não são cidade (como `/start` ou uma figurinha) também recebem a
mensagem de erro. As execuções aparecem em **Executions** no n8n.

Resultado dos testes feitos no ambiente local (n8n 1.122.4, 12/11/2026): Curitiba
(28°C), Campo Grande (32°C), São Paulo (27°C), Belo Horizonte (26°C), Itararé e
Itaberá responderam com a temperatura; `Grifinória` e `/start` receberam a
mensagem de erro. Todas as execuções terminaram com sucesso.

## Google Gemini (opcional)

- **Onde está:** nó **Gemini - melhorar mensagem** (Basic LLM Chain), logo depois de
  *Mensagem fallback*, com o sub-nó **Google Gemini Chat Model**
  (`models/gemini-2.5-flash`, temperatura `0.1`) e um **Structured Output Parser**
  que exige a saída `{"message": "...", "ok": true}`.
- **O que faz:** reescreve a mensagem de forma natural e amigável, em português,
  mantendo cidade e temperatura.
- **Como ativar:** gere a chave em https://aistudio.google.com/apikey e, no n8n, em
  **Credentials → Create Credential → Google Gemini(PaLM) Api**, preencha *API Key*
  com o valor de `GEMINI_API_KEY` (ou com a expressão `{{ $env.GEMINI_API_KEY }}`).
  Selecione a credencial no nó **Google Gemini Chat Model**.
- **Fallback:** o nó do Gemini está com *On Error → Continue (using error output)*.
  Se ele falhar (sem credencial, sem cota, projeto sem acesso ao nível gratuito,
  resposta inválida), o fluxo segue pela saída de erro e o Telegram envia a mensagem
  determinística do nó **Mensagem fallback**. Para rodar sem Gemini, também é
  possível desativar o nó *Gemini - melhorar mensagem* (selecione e tecle `D`).
- **No ambiente de teste** o projeto Google usado não tinha acesso ao nível gratuito
  (a API respondeu `403 Forbidden`), então todas as respostas vieram do fallback, o
  que confirmou o desvio pela saída de erro.

## Segurança

- Nenhum token ou chave está no JSON do workflow nem neste repositório.
- Os segredos ficam apenas no `.env` local (ignorado pelo git) e nas credenciais
  criptografadas do n8n.
