English · **[Español](README.es.md)**

# Yapr

**Talk to a Telegram bot, get a tidy `.md` back.**

Yapr turns your voice notes into structured Markdown documents. Walk out of a meeting, a site visit, or just have an idea rattling around: record a voice note, ramble as much as you want, and the bot sends back a `.md` file with everything organized into clear sections.

It's 100% self-hosted, BYOK (bring your own API key) and open source.

---

## How it works

```
Voice note (Telegram)
  → download the audio
  → transcription (Whisper, via Groq)
  → organized by an LLM according to the selected mode (gpt-oss-120b, via Groq)
  → .md file sent back on Telegram
```

Everything runs as a single [n8n](https://n8n.io) workflow on your own instance.

## Modes

Each mode is a different system prompt that defines how the note gets organized. Every chat picks its own with `/modo`.

| Mode | What it's for |
|---|---|
| `general` (default) | Takes notes or carries out requests, depending on what you say: lists, reminders, summaries, questions. It tells apart an order ("summarize X") from something you just want written down. |
| `negocios` | Notes from business visits and sales meetings: how the business operates, pain points, opportunities, objections and next steps. Keeps what the client said, what you observed and what you think apart. |
| `ideas` | Organizes an idea, plan or project: what's already defined, what's missing, contradictions and questions to decide. |

All modes share the same fidelity rules: they don't invent data, they flag likely transcription errors with `(¿?)`, and they turn relative dates ("on Thursday") into concrete ones.

## Commands

| Command | What it does |
|---|---|
| *(voice note)* | Processes it with the active mode and sends back the `.md` |
| `/modos` | Lists the available modes |
| `/modo <name>` | Switches this chat's active mode (for example, `/modo negocios`) |

---

## Requirements

- **Self-hosted n8n.** Tested on n8n 2.29. Needs the Data tables feature.
- **A public HTTPS URL** for your n8n instance. Telegram notifies n8n of every message through a webhook, so `localhost` won't work. If you run n8n at home, a tunnel (Cloudflare Tunnel, ngrok, etc.) solves this.
- **A Telegram bot** (free, created with [@BotFather](https://t.me/BotFather)).
- **A [Groq](https://console.groq.com) API key.** The free tier is enough for personal use.

## Setup

### 1. Create the Telegram bot

Message [@BotFather](https://t.me/BotFather), send `/newbot` and follow the steps. It gives you a **token**: keep it.

### 2. Get your Groq API key

Create one at [console.groq.com](https://console.groq.com). It starts with `gsk_`.

### 3. Create the data tables

In n8n, go to **Data tables** and create these two tables with these exact names and columns (all of type *String*):

**`modes`**: `name`, `description`, `prompt`

**`chat_settings`**: `chat_id`, `mode`

Then import `modes.csv` into the `modes` table. It contains the three base modes. `chat_settings` stays empty and fills itself in whenever someone uses `/modo` (you can also create it by importing `chat_settings.csv`).

### 4. Import the workflow

In n8n: **Create workflow → ⋯ → Import from file** and pick `workflow.json`.

### 5. Set up the credentials

Create these two credentials and assign them to the nodes:

| Credential | Type | Values | Nodes |
|---|---|---|---|
| Telegram | Telegram API | the BotFather token | all Telegram nodes |
| `groq api` | Header Auth | **Name:** `Authorization` · **Value:** `Bearer gsk_your_key` | `STT Request` and `HTTP Request` |

In the Groq credential, `Authorization` and the word `Bearer ` (followed by a space) go in exactly like that.

### 6. Re-select the data tables in the nodes

When you import a workflow, n8n loses the reference to the tables. Open each of these nodes and pick the right table from the dropdown:

| Node | Table |
|---|---|
| `Get all modes` | `modes` |
| `Get mode` | `modes` |
| `Get mode prompt` | `modes` |
| `Get chat mode` | `chat_settings` |
| `Upsert row(s)` | `chat_settings` |

### 7. Fill in the configuration

Open the **`Config`** node:

| Field | What to put |
|---|---|
| `allowed_ids` | Your numeric Telegram ID. Several IDs go separated by commas. |
| `default_mode` | The mode used when a chat hasn't picked one. `general` by default. |
| `language` | Language code for the audio and the date (`es`, `en`, ...). |

**If `allowed_ids` is left empty, the bot answers no one.** That's on purpose: anyone who finds your bot could burn through your API key. If you don't know your ID, send the bot a message and it will reply with your ID so you can copy it.

### 8. Activate the workflow

Activate it and send a voice note to your bot.

---

## Language

Yapr was built in Spanish: the three base modes and the bot's messages (confirmations, errors, the mode list) are written in Spanish. The `language` field in the `Config` node only changes the language Whisper transcribes in and the format of the date passed to the model.

To use it in another language, translate the prompts in the `modes` table and the texts of the Telegram nodes (`Mode changed`, `No mode`, `No mode fallback`, `Unauthorized`, `Build modes list`, `Send .md file`). Each prompt tells the model to write in the language of the audio, so the rest should adapt on its own.

## Create your own modes

A mode is a row in the `modes` table:

- **`name`**: what the user types after `/modo` (lowercase, no spaces).
- **`description`**: a one-line summary shown in `/modos`.
- **`prompt`**: the full system prompt.

To write a good prompt, look at the three base modes. What works best is telling the model **who is speaking, in what situation, and what the note will be used for**, not only which sections to include.

Built a mode that could help others? Open a pull request adding it to `modes.csv`.

## Privacy

Audio and transcriptions are sent to Groq for processing. Yapr doesn't store the audio or the notes: it only stores which mode each chat picked. If that doesn't work for you, you can change `base_url` and `model` in the `STT` and `Text IA` nodes to any OpenAI-compatible provider (OpenAI, OpenRouter, local Ollama, etc.). Only Groq has been tested.

## Known limitations

- Telegram lets bots download files up to 20 MB and Groq accepts up to 25 MB. That's plenty for normal voice notes, but very long recordings may fail.
- There's no error handling yet: if Groq fails or you hit the free tier limit, the bot stays silent.
- `/start` and unknown commands get the "that mode doesn't exist" reply.
- The bot's messages and the base prompts are in Spanish (see [Language](#language)).

## License

[MIT](LICENSE). n8n is distributed under its own [Sustainable Use License](https://docs.n8n.io/sustainable-use-license/).
