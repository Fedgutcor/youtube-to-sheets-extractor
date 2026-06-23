# Extractor de YouTube a Hojas de Datos (n8n + IA)

> Convierte **cualquier video de YouTube** en **datos estructurados** dentro de una hoja de cálculo. Toma la transcripción del video y la pasa por **dos agentes de IA** —un extractor y un auditor— que devuelven la información como filas listas para Google Sheets.

Este repositorio es, a la vez, una **plantilla reutilizable** y el **material de una clase** sobre automatización con n8n e IA. No documenta solo el resultado: documenta **el proceso completo** —incluidos los errores y cómo los descubrimos— para que se pueda reproducir y aprender paso a paso.

El ejemplo incluido extrae **resultados de loterías colombianas**, pero el patrón sirve para cualquier dato que viva en un video: precios, anuncios, datos deportivos, recetas, especificaciones, lo que sea.

---

## Cómo funciona (el patrón general)

```
YouTube  →  Transcripción  →  Agente Extractor  →  Agente Auditor  →  Hoja de cálculo
 (URL)       (yt-dlp)          (LLM + esquema)      (LLM verifica)      (Google Sheets)
```

1. **Transcripción** — `yt-dlp` baja los subtítulos automáticos del video (lo que se dijo).
2. **Extractor** — un LLM lee la transcripción y extrae los datos según un **esquema JSON** que vos definís.
3. **Auditor** — un segundo LLM verifica esos datos contra la fuente y corrige errores (no confiamos en la primera respuesta).
4. **Hoja** — cada registro se agrega como una fila en Google Sheets.

Lo que cambia entre un caso de uso y otro es **el prompt y el esquema** (ver [Adaptarlo a otro caso](#adaptarlo-a-otro-caso-de-uso)). La plomería es la misma.

---

## La idea central: hacer cosas con IA es como ser emperador romano

Tenés un poder enorme a tu disposición. Lo que define el resultado no es el poder — es **cómo lo gobernás**.

### 🏛️ Marco Aurelio — el que dirige

Marco Aurelio gobernó el imperio más grande de su tiempo con **disciplina, juicio y verificación**. No actuaba por impulso; contrastaba, dudaba, confirmaba antes de decidir.

> *"Si alguien puede demostrarme que algo que pienso o hago no es correcto, lo cambiaré con gusto. Porque busco la verdad."* — Marco Aurelio, *Meditaciones*

**Trabajar con IA al estilo Marco Aurelio** es lo que hicimos acá: la IA devolvió `confianza: 100` sobre números **equivocados**, y no le creímos — lo verificamos contra la transcripción. Encontramos bugs **probando**, no asumiendo.

> **TÚ diriges, la IA ejecuta.** La máquina propone; el juicio es tuyo.

### 🎭 Cómodo — el que se deja gobernar por la herramienta

Cómodo, hijo de Marco Aurelio, **heredó el mismo poder** y lo hundió por vanidad: confió en la apariencia, no en los hechos.

**Trabajar con IA al estilo Cómodo:** importás el workflow, lo corrés una vez, ves un JSON convincente y lo das por bueno. Confundís **fluidez con verdad**. Si el modelo dice `confianza: 100`, le creés.

> **La lección:** la IA te da un imperio. Marco Aurelio lo gobierna; Cómodo lo pierde. La diferencia es **verificar**. Por eso este proyecto tiene un agente auditor.

---

## Las herramientas y qué hace cada una

| Herramienta | Rol |
|---|---|
| **n8n** | El orquestador. Conecta todos los pasos en un flujo visual. Self-hosted vía Docker. |
| **yt-dlp** | Descarga la transcripción (subtítulos) y la metadata del video. Corre dentro del contenedor de n8n. |
| **Groq** | Modelo de IA en la nube (`llama-3.3-70b`), muy rápido. Es el modelo **principal**. |
| **Ollama** | Modelos de IA **locales** (gratis, privados). Alternativa. Requiere modelos de 7B+. |
| **Google Gemini** | Tercera opción de modelo. |
| **Structured Output Parser** (n8n) | Obliga a la IA a devolver JSON con la forma exacta de tu esquema. |
| **Google Sheets** | El destino: una fila por registro extraído. |

---

## El ejemplo incluido: resultados de loterías

Los canales de YouTube publican resultados de lotería en video. El número ganador casi nunca está escrito: lo **dice la presentadora**, dígito por dígito ("Siete. Tres. Cero. Ocho. De la serie 711"). No hay API que dé ese dato. El LLM normaliza esa voz dispersa a `7308`, serie `711`.

Es un caso ideal para enseñar porque combina: datos sucios (voz → dígitos), múltiples registros por video, y casos ambiguos donde **hay que bajar la confianza en vez de inventar**.

Videos de prueba y salidas verificadas: ver [`docs/GUIA_CLASE.md`](docs/GUIA_CLASE.md) y [`examples/`](examples/).

---

## Inicio rápido

Elegí dónde correrlo (la guía te lleva paso a paso):

- 🖥️ **Local (recomendado)** → [`docs/SETUP_LOCAL.md`](docs/SETUP_LOCAL.md) — `yt-dlp` nativo, gratis. Requiere **n8n 1.70+** (versiones viejas no reconocen el nodo de Groq).
- ☁️ **n8n Cloud** → [`docs/SETUP_CLOUD.md`](docs/SETUP_CLOUD.md) — Cloud **no puede correr `yt-dlp`**; hay que usar una API de transcripción externa. Más frágil.

Resumen del flujo, una vez levantado n8n:
1. **Importar** [`workflow/loterias_v1.json`](workflow/loterias_v1.json) (menú ⋮ → Import from File).
2. **Conectar Groq** (API key) y **Google Sheets OAuth**.
3. **Pegar tu `sheet_id`** en el nodo *Configurar*.
4. **Ejecutar** con una URL de prueba.

Para **ver el diagrama sin n8n**, abrí [`visor_workflow.html`](visor_workflow.html) en el navegador.

> **Sobre las credenciales:** hay **dos capas** que no se mezclan. El *ID de la hoja* (qué recurso) ya está cableado en el nodo Configurar. El *OAuth* (permiso para escribir) se conecta a mano en n8n, con tu cuenta de Google. n8n no puede escribir en la hoja hasta que conectes ese OAuth. Detalle en la guía.

---

## Adaptarlo a otro caso de uso

Para extraer **otra cosa** (no loterías), cambiás tres piezas:

1. **El esquema** (nodo *Esquema de Salida*) — definí los campos que querés extraer.
2. **Los prompts** (nodos *Agente Extractor* y *Agente Auditor*) — describí qué buscar y cómo verificarlo.
3. **Los encabezados de la hoja** y el mapeo del nodo *Expandir Sorteos*.

El resto —descarga de transcripción, los dos agentes, el guardado— no se toca. Eso es lo que lo hace un **extractor general**.

---

## Documentación

| Documento | Para qué |
|---|---|
| [`docs/TUTORIAL.md`](docs/TUTORIAL.md) | **Paso a paso del ejercicio** para desarrollar en vivo. Empezá acá. |
| [`docs/SETUP_LOCAL.md`](docs/SETUP_LOCAL.md) | Instalar y correr n8n local en macOS (con todos los gotchas de instalación). |
| [`docs/SETUP_CLOUD.md`](docs/SETUP_CLOUD.md) | Correr en n8n Cloud y por qué necesita una API de transcripción. |
| [`docs/GUIA_CLASE.md`](docs/GUIA_CLASE.md) | Setup detallado: credenciales, hoja, prueba. |
| [`docs/GOTCHAS.md`](docs/GOTCHAS.md) | Los 10 bugs que encontramos y cómo se resuelven. La parte más valiosa. |
| [`docs/PROMPT_NOTEBOOKLM_PPT.md`](docs/PROMPT_NOTEBOOKLM_PPT.md) | Prompt para generar la presentación con NotebookLM. |

---

## Aviso

Material educativo. Procesa contenido **público** de YouTube. Respetá los términos de servicio de YouTube y la legislación local aplicable.
