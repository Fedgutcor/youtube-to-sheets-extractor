# Clase n8n — Extractor de Resultados de Loterías (YouTube → LLM → Auditor → Sheets)

## Qué hace el workflow

Recibe la URL de un video de resultados de chance/lotería de YouTube y produce un registro estructurado por cada sorteo, guardado en una hoja de cálculo.

```
Manual ─▶ Configurar ─▶ yt-dlp (transcript) ─▶ Preparar Fuentes ─▶ Extractor ─▶ Auditor ─▶ Expandir ─▶ Google Sheets
                                                                       │           │
                                                                  Ollama (modelo) + Esquema JSON
```

## El concepto central de la clase

Los números **no** vienen como texto limpio. El ASR de YouTube los transcribe tal como los dice la presentadora, dispersos:

```
Número ganador para el primer seco...
Siete
3
0
8 de la serie 711.
```

`Siete 3 0 8 de la serie 711` → `numero_ganador = "7308"`, `serie = "711"`.

Un regex se rompe con eso. **Por eso usamos un LLM**: normaliza voz→dígitos. Y por eso hay un **segundo agente auditor**: el primer modelo puede equivocarse al juntar los dígitos, así que el auditor re-verifica contra las fuentes originales. Ese es el patrón "extractor + verificador" que enseña la clase.

## Por qué NO se usa la YouTube Data API (ni OAuth, ni IAP)

- La **YouTube Data API** entrega título y descripción, **nunca** la transcripción.
- El número está en el **audio** → la fuente real es la **transcripción auto-generada**, que se obtiene con `yt-dlp`.
- Olvidá IAP / `gcloud auth login` / `iap web enable`: eso protege apps, no consume YouTube. No va para este caso.

---

## Paso 1 — Preparar el contenedor de n8n (self-hosted)

El nodo `Execute Command` corre `yt-dlp` **dentro** del contenedor de n8n. Hay que instalarlo. Solo funciona en self-hosted (en n8n Cloud `Execute Command` está deshabilitado).

### Opción A — instalar en el contenedor actual (rápido, para la clase)

```bash
# Entrar como root al contenedor
docker exec -u 0 -it n8n sh

# Dentro del contenedor (imagen oficial = Alpine)
apk add --no-cache ffmpeg python3 py3-pip
pip install --break-system-packages yt-dlp
yt-dlp --version   # verificar
exit
```

> Nota: al recrear el contenedor esto se pierde. Para algo permanente, usá la Opción B.

### Opción B — imagen propia (permanente)

`Dockerfile`:
```dockerfile
FROM n8nio/n8n:latest
USER root
RUN apk add --no-cache ffmpeg python3 py3-pip \
 && pip install --break-system-packages yt-dlp
USER node
```
```bash
docker build -t n8n-loterias .
# usar la imagen n8n-loterias en tu docker-compose/run
```

### Permitir Execute Command y el Code node
En el entorno de n8n (variables de entorno):
```
NODE_FUNCTION_ALLOW_EXTERNAL=*
# Execute Command viene habilitado en self-hosted por defecto.
```

---

## Paso 2 — Importar el workflow

1. n8n → **Workflows** → menú **⋮** → **Import from File**.
2. Elegí `workflow/loterias_v1.json`.

---

## Paso 3 — Configurar credenciales

| Nodo | Credencial | Cómo |
|------|-----------|------|
| **Groq (principal)** | Groq API | Pegá tu API key de [console.groq.com/keys](https://console.groq.com/keys). Modelo: `llama-3.3-70b-versatile`. **Es el modelo conectado por defecto.** |
| **Ollama (alternativa local)** | Ollama API | Base URL: `http://host.docker.internal:11434` (Ollama en el host) o `http://ollama:11434` (otro contenedor). Modelo: `qwen2.5:7b` → `ollama pull qwen2.5:7b`. |
| **Gemini (alternativa)** | Google Gemini (PaLM) API | API key de Google AI Studio. Modelo: `models/gemini-2.0-flash`. |
| **Guardar en Hoja** | Google Sheets OAuth2 | *Acá* sí va OAuth de Google (este era el OAuth que estabas configurando). |

> **Por qué Groq por defecto (verificado en pruebas reales):** con el Extra de Medellín (25+ premios), Groq `llama-3.3-70b` produjo **35 registros correctos en ~13s**. El mismo caso con Ollama `qwen2.5:3b` dio **1 registro basura en ~5 min**. Para Ollama local usá **mínimo `qwen2.5:7b`** — los modelos de 3b no separan registros y alucinan campos (copian los ejemplos del prompt).

---

## Paso 4 — Preparar la Google Sheet

1. Creá una hoja de cálculo en Google Sheets con **estos 12 encabezados en la fila 1**:

```
procesado_en | video_url | video_id | titulo_video | loteria | fecha | numero_ganador | serie | premio | pais | observaciones | confianza
```

2. Copiá el **ID** de la hoja —lo que va entre `/d/` y `/edit` en la URL— y pegalo en el nodo **Configurar** → campo `sheet_id` (reemplazá `PEGA_AQUI_EL_ID_DE_TU_GOOGLE_SHEET`).

3. La pestaña la toma por `gid 0` (la primera), así que no importa cómo se llame el tab.

> **Las dos capas de credencial (no las confundas):**
> - **`sheet_id`** dice *qué* hoja → lo pegás en el nodo Configurar (paso 2).
> - **OAuth** da *permiso* para escribir → en el nodo **Guardar en Hoja**, *Create New Credential* → **Google Sheets OAuth2**, e iniciá sesión con la cuenta **dueña de la hoja**. Al ser la dueña, ya tiene permiso — no hay que compartir nada.
>
> n8n no puede escribir en la hoja hasta que conectes ese OAuth, aunque el `sheet_id` esté bien.

---

## Paso 5 — Probar

1. En **Configurar**, dejá `video_url` con uno de prueba:
   - `https://www.youtube.com/watch?v=u109SuVRLuo` (Extra de Medellín — 25+ premios)
   - `https://www.youtube.com/watch?v=9KNTZNg4Iew` (Paisita/Quinta/Suertudo — multi-juego)
   - `https://www.youtube.com/watch?v=CAH5PKTRNXw` (Cundinamarca — serie real explícita)
   - `https://www.youtube.com/watch?v=8cpPapXy7yI` (Astro Luna — número + signo zodiacal, sin serie)
   - `https://www.youtube.com/watch?v=SH2wEzIYQgE` (Caribeña Noche — mayor + la quinta)
   - `https://www.youtube.com/watch?v=UU789gTjfkQ` (Cafeterito — **sin subtítulos**, caso de degradación)
2. **Execute Workflow**.

> Las salidas reales y verificadas de varios de estos están en [`../examples/salidas_verificadas.json`](../examples/salidas_verificadas.json).
3. Revisá la salida de cada nodo: en **Preparar Fuentes** vas a ver el `transcript`; en **Agente Auditor**, el JSON final.

> **Ejemplo corrido de verdad:** procesar los 5 videos generó **41 filas** en la hoja (una por premio). El Extra de Medellín solo produjo 35. Así se ve poblada la salida — y fue corriendo el flujo completo donde apareció el [gotcha #8 de los ceros a la izquierda](GOTCHAS.md#8-google-sheets-se-come-los-ceros-a-la-izquierda).

---

## Elegir / cambiar el modelo (Groq ↔ Ollama ↔ Gemini)

Por defecto **Groq** alimenta a los dos agentes. Para usar **Ollama local** (gratis/privado) o **Gemini**: arrastrá la salida de ese nodo hacia el conector inferior (🧠) del **Agente Extractor** y/o **Agente Auditor**.

**Lección de clase (probada en vivo):** mostrá primero Ollama local con un modelo chico — falla al separar los premios. Después cambiá a Groq: 35 registros correctos en segundos. Esa comparación **es** la lección de por qué el tamaño del modelo importa.

**Fallback automático (tema avanzado):**
1. En **Agente Extractor** → Settings → activá **Continue On Fail**.
2. Conectá su salida de **error** a una copia del extractor que use **Ollama**.
3. Resultado: si la API de Groq falla o se acaba la cuota, el flujo sigue local sin cortarse.

---

## Cómo se obtiene el transcript (lo que hace el nodo yt-dlp)

```bash
yt-dlp --no-check-certificates --no-simulate --skip-download \
  --write-auto-subs --sub-langs "es.*" --sub-format vtt \
  -o "%(id)s.%(ext)s" \
  --print "%(id)s<<<SEP>>>%(title)s<<<SEP>>>%(description)s" "$URL"
```
- `--skip-download`: no baja el video, solo subtítulos y metadata (rápido y liviano).
- `--write-auto-subs --sub-langs "es.*"`: subtítulos auto-generados en español.
- El `.vtt` se limpia (se quitan timestamps y etiquetas) y se concatena en una sola línea.
- Se emite `id<<<SEP>>>titulo<<<SEP>>>descripcion<<<SEP>>>transcript`, que el nodo **Preparar Fuentes** parte en campos.

> **Gotcha clave (verificado):** `--print` activa modo *simulación* en yt-dlp y entonces **no** descarga el `.vtt`. Por eso va `--no-simulate`: fuerza la descarga del subtítulo aunque uses `--print` para la metadata en la misma llamada. Sin esa bandera, `transcript` sale vacío.

---

## Notas y límites (para responder en clase)

- **`--no-check-certificates`** evita el error SSL típico en algunos entornos. En producción conviene arreglar los certificados en vez de saltearlos.
- Si un video **no tiene** subtítulos, `transcript` queda vacío y el LLM usa solo título/descripción (menor confianza). Ahí entraría Whisper (ASR propio) como mejora futura.
- **OCR** (`ocr`) queda en `null`: estos videos dictan el número en audio. Si algún canal solo lo muestra en pantalla, se agregaría una rama con `ffmpeg` (extraer frames) + `tesseract`.
- Para **lotes**, reemplazá el Manual Trigger por un nodo que lea varias URLs (Google Sheets / RSS del canal) y dejá que el flujo itere.
```
