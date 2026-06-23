# Gotchas — los bugs que encontramos y cómo se resuelven

Cada uno de estos lo descubrimos **probando**, no leyéndolo en la documentación. Son la parte más transferible del proyecto: el tipo de problema que aparece en cualquier automatización real.

---

## 1. `--print` apaga la descarga de subtítulos

**Síntoma:** el comando devuelve título y descripción, pero el archivo `.vtt` nunca aparece y el `transcript` queda vacío.

**Causa:** en `yt-dlp`, la opción `--print` activa **modo simulación**. En ese modo, `--write-auto-subs` no descarga nada.

**Solución:** agregar `--no-simulate`, que fuerza la descarga aunque uses `--print` para la metadata en la misma llamada.

```bash
# MAL — transcript vacío
yt-dlp --skip-download --write-auto-subs --print "%(title)s" URL

# BIEN
yt-dlp --no-simulate --skip-download --write-auto-subs --print "%(title)s" URL
```

---

## 2. Subtítulos manuales vs. automáticos

**Síntoma:** un video que en `--list-subs` muestra subtítulos en español igual no descarga nada con `--write-auto-subs`.

**Causa:** `--write-auto-subs` solo baja subtítulos **automáticos** (ASR de YouTube). Si el canal subió subtítulos **manuales**, hay que pedirlos con `--write-subs`.

**Solución:** pedir ambos.

```bash
yt-dlp --no-simulate --write-subs --write-auto-subs --sub-langs "es.*" URL
```

---

## 3. Videos sin ningún subtítulo

**Síntoma:** `transcript` vacío incluso con todo bien configurado.

**Causa:** el video simplemente **no tiene subtítulos** (ni automáticos ni manuales). Nos pasó con el video del Cafeterito.

**Solución / diseño:** el flujo debe **degradar con elegancia** — si no hay transcript, el LLM trabaja solo con título y descripción (con menor confianza). Mejora futura: hacer ASR propio con Whisper sobre el audio.

> Esto no es un bug a "arreglar": es una condición del mundo real que el diseño debe contemplar.

---

## 4. Rate limit de YouTube (HTTP 429)

**Síntoma:** `ERROR: Unable to download video subtitles: HTTP Error 429: Too Many Requests`.

**Causa:** demasiadas peticiones a YouTube desde la misma IP en poco tiempo (nos pasó al probar varios videos seguidos).

**Solución:**
- Espaciar las peticiones: `--sleep-requests 2` y `--sleep-subtitles 2`.
- Instalar `curl_cffi` para habilitar *impersonation* (yt-dlp lo sugiere en un warning).
- En lotes grandes, procesar de a pocos con pausas.

---

## 5. Timeout de Node al llamar a Ollama (en pruebas locales)

**Síntoma:** `UND_ERR_HEADERS_TIMEOUT` al llamar a Ollama con un transcript largo.

**Causa:** sin streaming, Ollama no manda los headers HTTP hasta **terminar de generar**. En CPU, con un transcript largo, eso supera el timeout por defecto de Node.

**Solución:** usar **streaming** (`stream: true`) — los headers llegan de inmediato y se acumulan los tokens. *(Dentro de n8n este problema no aparece; fue solo en el script de prueba local.)*

---

## 6. El modelo chico alucina y se sobre-confía

**Síntoma:** `qwen2.5:3b` devolvía un solo registro para un sorteo de 25 premios, con `confianza: 100`, y a veces **copiaba textualmente los ejemplos del prompt**.

**Causa:** capacidad insuficiente del modelo. Un modelo de 3B no sostiene instrucciones de varias partes (granularidad + calibración + no-alucinar).

**Solución:** usar un modelo capaz — `llama-3.3-70b` (Groq) o, en local, mínimo `qwen2.5:7b`.

> **El gotcha más importante:** `confianza: 100` **no significa "correcto"**. Significa "el modelo está seguro". Un modelo malo está seguro de cosas falsas. Verificá la salida contra la fuente, siempre.
