# Tutorial paso a paso — Construir el extractor de loterías

Este documento reconstruye **el proceso real** que seguimos para llegar al workflow. No es la versión idealizada: incluye los desvíos y los errores, porque ahí está el aprendizaje. La idea es **recorrerlo en vivo con los estudiantes**, parando en cada decisión.

> **Filosofía del ejercicio:** a cada paso, antes de aceptar un resultado, preguntamos *"¿cómo sé que esto es verdad?"*. Verificamos contra la realidad, no contra lo que la herramienta dice de sí misma. (Ver la metáfora de Marco Aurelio en el [README](../README.md).)

---

## Módulo 0 — El punto de partida (y la primera trampa)

**Situación:** queremos extraer resultados de loterías de videos de YouTube usando IA, en n8n.

**La trampa:** al configurar el nodo de YouTube, n8n pide credenciales OAuth. Buscando cómo, aparecen instrucciones de Google Cloud que hablan de **IAP** (Identity-Aware Proxy), `gcloud auth login`, `gcloud iap web enable`...

**El momento de detenerse y verificar:**
- IAP sirve para *proteger* una aplicación tuya detrás de un login. **No** sirve para *consumir* YouTube.
- La YouTube Data API, además, **nunca entrega la transcripción** — solo título y descripción.
- El número ganador está en el **audio**. Conclusión: no necesitamos OAuth de YouTube ni IAP para nada.

> **Lección 1:** la primera instrucción que encontrás no siempre es la correcta. Antes de seguir un tutorial, verificá que resuelve *tu* problema. Mucha gente habría perdido una hora configurando IAP sin necesidad.

**Pregunta para los estudiantes:** *¿de dónde va a salir, entonces, el número ganador si la API oficial no lo da?*

---

## Módulo 1 — Conseguir la materia prima (la transcripción)

**Objetivo:** obtener lo que dice el video.

**Herramienta:** `yt-dlp` (descargador de YouTube con soporte de subtítulos).

**Verificación en vivo** — corré esto en una terminal con uno de los videos de prueba:

```bash
yt-dlp --no-check-certificates --skip-download \
  --write-auto-subs --sub-langs "es.*" --sub-format vtt \
  -o "%(id)s.%(ext)s" "https://www.youtube.com/watch?v=u109SuVRLuo"
```

Abrí el `.vtt` resultante. Vas a ver algo así:

```
Número ganador para el primer seco de 500 millones de pesos.
Siete
3
0
8 de la serie 711.
```

> **Lección 2 (el corazón del proyecto):** los números vienen **hablados y dispersos** — a veces palabra ("Siete"), a veces dígito ("3"), uno por línea. `Siete 3 0 8 de la serie 711` = número **7308**, serie **711**.
>
> Un `regex` se rompe con esto. **Por eso necesitamos un LLM:** para normalizar voz → dígitos. Esta es la justificación de toda la arquitectura.

**Pregunta:** *¿por qué no podemos resolver esto con una expresión regular?*

---

## Módulo 2 — Los bugs que aparecieron (verificar = encontrar)

Acá es donde el estilo "Marco Aurelio" paga. Cada bug lo encontramos **probando**, no asumiendo. Detalle completo en [`GOTCHAS.md`](GOTCHAS.md).

1. **`--print` apaga la descarga.** Al combinar metadata + subtítulos en un comando, `--print` activa modo simulación y **el `.vtt` no se descarga**. → Se arregla con `--no-simulate`.
2. **Subtítulos manuales vs automáticos.** Algunos videos usan subtítulos manuales. → `--write-subs --write-auto-subs` (ambos).
3. **Videos sin subtítulos.** El Cafeterito no tenía ninguno. → el flujo debe degradar a título/descripción.
4. **Rate limit (HTTP 429).** YouTube limita descargas masivas desde la misma IP. → espaciar peticiones.

> **Lección 3:** "funcionó una vez" no es "funciona". Probá con varios casos, incluidos los que esperás que fallen. El video sin subtítulos enseña más que el que funciona.

---

## Módulo 3 — El workflow en n8n (orquestar)

**Objetivo:** conectar todo en n8n.

Los nodos, en orden:

1. **Disparador Manual** — para ejecutar a mano en clase.
2. **Configurar** (Set) — define `video_url`, `sheet_id`, `sheet_name`.
3. **Descargar Transcripción** (Execute Command) — corre `yt-dlp`, emite `id<<<SEP>>>titulo<<<SEP>>>descripcion<<<SEP>>>transcript`.
4. **Preparar Fuentes** (Code) — parte ese string en campos.
5. **Agente Extractor** (Basic LLM Chain) — extrae los sorteos. Modelo + Structured Output Parser.
6. **Agente Auditor** (Basic LLM Chain) — verifica y corrige.
7. **Expandir Sorteos** (Code) — una fila por premio.
8. **Guardar en Hoja** (Google Sheets) — append.

> **Lección 4 — "workflow as code":** todo esto es un archivo JSON. Se versiona, se comparte, se revisa en un PR. Cambiar un prompt es un commit. No hay capturas de pantalla.

Importá [`workflow/loterias_v1.json`](../workflow/loterias_v1.json) y recorré los nodos en pantalla. Para mostrar el diagrama sin n8n: abrí [`visor_workflow.html`](../visor_workflow.html).

---

## Módulo 4 — Dos agentes: extractor + auditor (no confiar a ciegas)

**Por qué dos agentes y no uno:**

- El **extractor** puede equivocarse al juntar los dígitos hablados, o alucinar un campo.
- El **auditor** recibe la extracción *y* las fuentes originales, y verifica: ¿el número existe de verdad en la transcripción? ¿la serie es de ese premio o de otro? ¿la fecha cuadra?

> **Lección 5:** el auditor es el Marco Aurelio del pipeline. Su trabajo es **dudar de la primera respuesta**. En las pruebas, el extractor dio `confianza: 100` sobre datos equivocados; el auditor la bajó. Un solo agente no se cuestiona a sí mismo.

---

## Módulo 5 — La prueba de la verdad (el modelo importa)

Acá viene el experimento más didáctico. Corrimos **el mismo workflow, los mismos prompts**, con dos modelos distintos.

### Con `qwen2.5:3b` (Ollama local, modelo chico)

Caso difícil (Extra de Medellín, 25+ premios):

```json
{ "loteria": "Extra", "numero_ganador": "549", "serie": "06",
  "premio": "20 millones", "confianza": 100 }
```

- ❌ Un solo registro (se comió 24 premios).
- ❌ Agarró un seco menor al azar; perdió el premio mayor.
- ❌ `confianza: 100` sobre datos equivocados.
- ❌ En otra prueba, **copió literal un ejemplo del prompt** ("Seco 1 de 500 millones" donde no había secos).

### Con `llama-3.3-70b` (Groq, modelo grande)

Mismo caso:

- ✅ **35 registros**, uno por premio.
- ✅ Primer seco `7308` serie `711` — exacto a la transcripción.
- ✅ Donde el audio no capturó el número: `numero_ganador: null`, `confianza: 20`, `"Número no especificado"`.
- ✅ ~13 segundos (vs ~5 minutos del 3B en CPU).

> **Lección 6:** el código era el mismo. La diferencia fue el modelo. Un modelo demasiado chico **no separa, alucina y se sobre-confía**. Y `confianza: 100` no significa "correcto" — significa "el modelo está seguro", que no es lo mismo. **Cómodo le habría creído al 3B. Marco Aurelio verificó.**

**Decisión del proyecto:** Groq como modelo principal; Ollama local queda como "el concepto" y para mostrar el contraste.

---

## Módulo 6 — Guardar y cerrar

El nodo **Expandir Sorteos** convierte el array `sorteos` en una fila por premio, con metadata (URL, fecha de proceso), y **Google Sheets** las agrega.

Encabezados de la hoja:

```
procesado_en | video_url | video_id | titulo_video | loteria | fecha | numero_ganador | serie | premio | pais | observaciones | confianza
```

---

## Ejercicios propuestos para los estudiantes

1. **Fácil:** agregá una columna `link_video` o cambiá el destino a otra hoja.
2. **Medio:** procesá una lista de URLs en lote (reemplazar el Disparador Manual por un nodo que lea varias URLs).
3. **Medio:** activá el fallback automático (Continue On Fail → rama Ollama) y probá qué pasa si Groq falla.
4. **Difícil:** para un video que solo muestra el número en pantalla (sin audio), agregá una rama de OCR (`ffmpeg` extrae frames + `tesseract`).
5. **Reflexión:** tomá un resultado del modelo y verificalo a mano contra el video. ¿Coincide? ¿Dónde bajó la confianza y por qué? *(Esto es ser Marco Aurelio.)*

---

## Resumen de las lecciones

| # | Lección |
|---|---------|
| 1 | La primera instrucción no siempre resuelve tu problema. Verificá antes de seguirla. |
| 2 | Los datos del mundo real son sucios (números hablados y dispersos). Por eso usamos IA. |
| 3 | "Funcionó una vez" ≠ "funciona". Probá los casos que esperás que fallen. |
| 4 | El workflow es código: se versiona, se comparte, se revisa. |
| 5 | No confíes en la primera respuesta de la IA. Un segundo agente la audita. |
| 6 | El tamaño del modelo importa. `confianza: 100` no es "correcto". Verificá siempre. |
