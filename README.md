# Extractor de Resultados de Loterías con n8n + IA

> Un workflow de **n8n** que toma un video de YouTube de resultados de chance/lotería colombiana, le saca la transcripción, y usa **dos agentes de IA** (un extractor y un auditor) para devolver los números ganadores como datos estructurados, listos para una hoja de cálculo.

Este repositorio es el material de una clase sobre **automatización con n8n e IA**. No es solo el resultado final: documenta **el proceso completo** —incluidos los errores y cómo los descubrimos— para que se pueda reproducir y aprender paso a paso.

---

## Qué problema resuelve

Los canales de YouTube publican los resultados de loterías en video. El número ganador casi nunca está escrito: lo **dice la presentadora**, dígito por dígito ("Siete. Tres. Cero. Ocho. De la serie 711"). No hay una API que te dé ese dato. Hay que:

1. **Obtener lo que se dijo** → transcripción automática del video.
2. **Entender los números hablados y dispersos** → un LLM que normaliza "Siete Tres Cero Ocho" a `7308`.
3. **No confiar a ciegas** → un segundo agente que verifica el resultado contra la fuente.
4. **Guardarlo estructurado** → una fila por premio en una hoja de cálculo.

---

## La idea central: hacer cosas con IA es como ser emperador romano

Tenés un poder enorme a tu disposición. Lo que define el resultado no es el poder — es **cómo lo gobernás**.

### 🏛️ Marco Aurelio — el que dirige

Marco Aurelio gobernó el imperio más grande de su tiempo con **disciplina, juicio y verificación**. No actuaba por impulso; contrastaba, dudaba, confirmaba antes de decidir. Su poder estaba **al servicio de la realidad**, no de su ego.

> *"Si alguien puede demostrarme que algo que pienso o hago no es correcto, lo cambiaré con gusto. Porque busco la verdad, y la verdad nunca le hizo daño a nadie."* — Marco Aurelio, *Meditaciones*

**Trabajar con IA al estilo Marco Aurelio** es lo que hicimos en este proyecto:
- La IA nos devolvió `confianza: 100` sobre un número **equivocado**. No le creímos. Lo verificamos contra la transcripción real y descubrimos que era basura.
- Encontramos un bug (`--print` apagaba la descarga de subtítulos) **probando**, no asumiendo.
- Solo confiamos en el resultado de Groq **después** de contrastarlo con la fuente.

> **TÚ diriges, la IA ejecuta.** La máquina propone; el juicio es tuyo.

### 🎭 Cómodo — el que se deja gobernar por la herramienta

Cómodo, hijo de Marco Aurelio, **heredó exactamente el mismo poder** y lo hundió. Gobernó por vanidad y espectáculo, creyó su propio mito, confió en la apariencia y no en los hechos. Mismo imperio, disciplina opuesta.

**Trabajar con IA al estilo Cómodo** es el anti-patrón:
- Importás el workflow, lo corrés una vez, ves un JSON que *suena* convincente, y lo das por bueno.
- Confundís **fluidez con verdad**. Si el modelo dice `confianza: 100`, le creés.
- Despliegas a producción sin verificar. (El equivalente romano: Nerón tocando la lira mientras Roma ardía.)

El modelo de 3B de este proyecto es el cortesano de Cómodo: te dice con total seguridad un número que se inventó.

> **La lección de la clase:** la IA te da un imperio. Marco Aurelio lo gobierna; Cómodo lo pierde. La diferencia es **verificar**.

---

## Las herramientas y qué hace cada una

| Herramienta | Rol en el proyecto |
|---|---|
| **n8n** | El orquestador. Conecta todos los pasos en un flujo visual (nodos). Self-hosted vía Docker. |
| **yt-dlp** | Descarga la **transcripción** (subtítulos auto-generados) y la metadata del video de YouTube. Corre dentro del contenedor de n8n. |
| **Ollama** | Servidor de modelos de IA **locales** (gratis, privados). En la clase es "la opción local". Requiere modelos de 7B+ para esta tarea. |
| **Groq** | API de inferencia **en la nube**, muy rápida. Modelo `llama-3.3-70b`. Es el modelo **principal** del workflow: 35 registros correctos en ~13s. |
| **Google Gemini** | Tercera opción de modelo (alternativa/fallback). |
| **Structured Output Parser** (n8n) | Obliga a la IA a devolver JSON con la forma exacta del esquema. |
| **Google Sheets** | El destino: una fila por premio extraído. |

### El flujo

```
Disparador → Configurar → yt-dlp → Preparar Fuentes → Agente Extractor → Agente Auditor → Expandir → Google Sheets
                                                              │                  │
                                                         Groq / Ollama / Gemini  +  Esquema JSON
```

- **Agente Extractor**: lee título + descripción + transcripción y extrae los sorteos.
- **Agente Auditor** (segunda pasada): verifica cada número contra la fuente, corrige errores, ajusta la confianza. *Este es el Marco Aurelio del pipeline.*

---

## Inicio rápido

1. **Tener n8n self-hosted** con `yt-dlp` instalado → ver [`docker/Dockerfile`](docker/Dockerfile).
2. **Importar** [`workflow/loterias_v1.json`](workflow/loterias_v1.json) en n8n.
3. **Configurar credenciales** (Groq, Google Sheets) → ver [`docs/GUIA_CLASE.md`](docs/GUIA_CLASE.md).
4. **Ejecutar** con una URL de prueba.

Para **ver el diagrama sin n8n**, abrí [`visor_workflow.html`](visor_workflow.html) en el navegador.

---

## Documentación

| Documento | Para qué |
|---|---|
| [`docs/TUTORIAL.md`](docs/TUTORIAL.md) | **Paso a paso del ejercicio** para desarrollar en vivo con los estudiantes. Empezá acá. |
| [`docs/GUIA_CLASE.md`](docs/GUIA_CLASE.md) | Setup detallado: contenedor, credenciales, prueba. |
| [`docs/GOTCHAS.md`](docs/GOTCHAS.md) | Los 4 bugs/trampas que encontramos y cómo se resuelven. La parte más valiosa. |

---

## Aviso

Material educativo. Procesa contenido **público** de YouTube. Respetá los términos de servicio de YouTube y la legislación local sobre datos de juegos de azar.
