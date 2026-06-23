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

---

## 7. El modelo grande también miente (convincentemente)

**Síntoma:** con Groq `llama-3.3-70b` —un modelo bueno— procesamos tres videos nuevos. El JSON se veía impecable. Al **verificar a mano contra la transcripción**, dos de tres tenían errores:

- **Astro Luna:** puso `serie: "8159"`... pero `8159` era el **número de sorteo** ("sorteo 8159 de Super Astroluna"), no una serie. Y **se comió el signo zodiacal** ("signo Aries"), que es parte del resultado.
- **Caribeña:** `serie: "7748"` — otra vez, el número de sorteo ("sorteo 7748"), no una serie.

**El patrón:** cuando NO hay serie real en el audio, el modelo **inventa una tomando el número de "sorteo NNNN"**. Es plausible, es del mismo video, y suena bien. Pero es falso.

**Causa:** el prompt no distinguía "número de sorteo" (ID del evento) de "serie" (parte del billete). Y no contemplaba el signo del Astro.

**Solución (prompt):** dos reglas nuevas en el extractor y el auditor —
- *"El número de sorteo (`sorteo 8159`) es el ID del evento, NO la serie. Si no se dice 'de la serie N', serie = null."*
- *"En Astro, el signo zodiacal va en observaciones."*

Tras el fix: `serie: null` correcto en ambos, y `observaciones: "Signo: Aries"` capturado.

> **La lección completa:** esto **no** lo atrapás mirando el JSON —se ve perfecto—. Solo aparece al contrastar contra la fuente. Un modelo grande no te salva de verificar; solo hace los errores más convincentes. Esto es, literalmente, por qué existe el agente auditor. **Marco Aurelio no confía en que el cortesano sea elocuente; confía en lo que puede verificar.**
