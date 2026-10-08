**[English](README.md)** · Español

# Yapr

**Hablale a un bot de Telegram y recibí un `.md` ordenado.**

<p align="center">
  <img width="250" alt="1" src="https://github.com/user-attachments/assets/db41047c-ff4c-411b-973c-5408208d68bb" />
  &nbsp;&nbsp;&nbsp;&nbsp;
  <img width="250" alt="2" src="https://github.com/user-attachments/assets/48291ab7-7d22-412d-acb9-a1723f348323" />
</p>

Yapr convierte tus notas de voz en documentos Markdown estructurados. Salís de una reunión, una visita o simplemente tenés una idea dando vueltas: grabás un audio divagando todo lo que quieras, y el bot te devuelve un archivo `.md` con todo ordenado en secciones claras.

Es 100% selfhosted, BYOK (traés tu propia API key) y open source.

---

## Cómo funciona

```
Nota de voz (Telegram)
  → descarga del audio
  → transcripción (Whisper, vía Groq)
  → ordenado con un LLM según el modo elegido (gpt-oss-120b, vía Groq)
  → archivo .md de vuelta en Telegram
```

Todo corre como un único workflow de [n8n](https://n8n.io) en tu propia instancia.

## Modos

Cada modo es un system prompt distinto que define cómo se ordena la nota. Cada chat elige el suyo con `/modo`.

| Modo | Para qué sirve |
|---|---|
| `general` (por defecto) | Anota o ejecuta según lo que digas: listas, recordatorios, resúmenes y pedidos. Detecta si le estás dando una orden ("resumime X") o si solo querés anotar algo. |
| `negocios` | Notas de visitas a negocios y reuniones comerciales: cómo opera el negocio, dolores, oportunidades, objeciones y próximos pasos. Separa lo que dijo el cliente, lo que observaste y lo que pensás vos. |
| `ideas` | Ordena una idea, plan o proyecto: qué tenés definido, qué falta, contradicciones y preguntas para decidir. |

Todos los modos comparten las mismas reglas de fidelidad: no inventan datos, marcan con `(¿?)` lo que parece un error de transcripción y convierten fechas relativas ("el jueves") en fechas concretas.

## Comandos

| Comando | Qué hace |
|---|---|
| *(nota de voz)* | La procesa con el modo activo y te devuelve el `.md` |
| `/modos` | Lista los modos disponibles |
| `/modo <nombre>` | Cambia el modo activo de este chat (por ejemplo, `/modo negocios`) |

---

## Requisitos

- **n8n selfhosted.** Probado en n8n 2.29. Necesita la función de Data tables.
- **URL pública con HTTPS** para tu instancia de n8n. Telegram le avisa a n8n de cada mensaje por webhook, así que `localhost` no alcanza. Si corrés n8n en tu casa, un túnel (Cloudflare Tunnel, ngrok, etc.) resuelve esto.
- **Un bot de Telegram** (gratis, se crea con [@BotFather](https://t.me/BotFather)).
- **Una API key de [Groq](https://console.groq.com)**. El plan gratuito alcanza para uso personal.

## Instalación

### 1. Creá el bot de Telegram

Hablale a [@BotFather](https://t.me/BotFather), mandá `/newbot` y seguí los pasos. Te devuelve un **token**: guardalo.

### 2. Conseguí tu API key de Groq

Creala en [console.groq.com](https://console.groq.com). Empieza con `gsk_`.

### 3. Creá las data tables

En n8n, andá a **Data tables** y creá estas dos tablas, con estos nombres y columnas exactos (todas de tipo *String*):

**`modes`**: `name`, `description`, `prompt`

**`chat_settings`**: `chat_id`, `mode`

Después importá `modes.csv` en la tabla `modes`. Ahí vienen los tres modos base. `chat_settings` queda vacía y se llena sola cuando alguien usa `/modo` (también podés crearla importando `chat_settings.csv`).

### 4. Importá el workflow

En n8n: **Create workflow → ⋯ → Import from file** y elegí `workflow.json`.

### 5. Configurá las credenciales

Creá estas dos credenciales y asignalas en los nodos:

| Credencial | Tipo | Valores | Nodos |
|---|---|---|---|
| Telegram | Telegram API | el token de BotFather | todos los nodos de Telegram |
| `groq api` | Header Auth | **Name:** `Authorization` · **Value:** `Bearer gsk_tu_key` | `STT Request` y `HTTP Request` |

En la credencial de Groq, `Authorization` y la palabra `Bearer ` (con un espacio) van literalmente así.

### 6. Volvé a elegir las data tables en los nodos

Al importar un workflow, n8n pierde la referencia a las tablas. Abrí cada uno de estos nodos y elegí la tabla correcta en el desplegable:

| Nodo | Tabla |
|---|---|
| `Get all modes` | `modes` |
| `Get mode` | `modes` |
| `Get mode prompt` | `modes` |
| `Get chat mode` | `chat_settings` |
| `Upsert row(s)` | `chat_settings` |

### 7. Completá la configuración

Abrí el nodo **`Config`**:

| Campo | Qué poner |
|---|---|
| `allowed_ids` | Tu ID numérico de Telegram. Varios, separados por coma. |
| `default_mode` | El modo que se usa si un chat no eligió ninguno. Por defecto, `general`. |
| `language` | Código de idioma del audio y de la fecha (`es`, `en`, ...). |

**Si `allowed_ids` queda vacío, el bot no le responde a nadie.** Es a propósito: cualquiera que encuentre tu bot podría gastar tu API key. Si no sabés tu ID, mandale un mensaje al bot: te va a responder con tu ID y lo copiás de ahí.

### 8. Activá el workflow

Activalo y mandale una nota de voz a tu bot.

---

## Idioma

Yapr está pensado en español: los tres modos base y los mensajes del bot (confirmaciones, errores, lista de modos) están escritos en español. El campo `language` del nodo `Config` solo cambia el idioma que usa Whisper para transcribir y el formato de la fecha que se le pasa al modelo.

Para usarlo en otro idioma, traducí los prompts en la tabla `modes` y los textos de los nodos de Telegram (`Mode changed`, `No mode`, `No mode fallback`, `Unauthorized`, `Build modes list`, `Send .md file`). Cada prompt le indica al modelo que escriba en el idioma del audio, así que el resto debería adaptarse solo.

## Crear tus propios modos

Un modo es una fila en la tabla `modes`:

- **`name`**: lo que el usuario escribe en `/modo` (minúsculas, sin espacios).
- **`description`**: una línea que se muestra en `/modos`.
- **`prompt`**: el system prompt completo.

Para escribir un buen prompt, mirá los tres modos base. Lo que mejor funciona es explicarle al modelo **quién habla, en qué situación y para qué va a usar la nota**, no solo qué secciones poner.

¿Armaste un modo que le puede servir a otros? Abrí un pull request agregándolo a `modes.csv`.

## Privacidad

Los audios y sus transcripciones se envían a Groq para procesarse. Yapr no guarda los audios ni las notas: solo guarda qué modo eligió cada chat. Si eso no te sirve, podés cambiar `base_url` y `model` en los nodos `STT` y `Text IA` por cualquier proveedor compatible con la API de OpenAI (OpenAI, OpenRouter, Ollama local, etc.). Solo está probado con Groq.

## Limitaciones conocidas

- Telegram deja descargar archivos de hasta 20 MB y Groq acepta hasta 25 MB. Para notas de voz normales sobra, pero audios muy largos pueden fallar.
- Todavía no hay manejo de errores: si Groq falla o llegás al límite de la cuota gratuita, el bot no responde.
- `/start` y los comandos desconocidos responden con el mensaje de modo inexistente.
- Los mensajes del bot y los prompts base están en español (ver [Idioma](#idioma)).

## Licencia

[MIT](LICENSE). n8n se distribuye bajo su propia [Sustainable Use License](https://docs.n8n.io/sustainable-use-license/).
