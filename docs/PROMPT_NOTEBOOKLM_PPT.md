# Prompt definitivo para generar la PPT con NotebookLM

**Antes de usarlo:** subí al notebook estas fuentes del repo → `README.md`, `docs/TUTORIAL.md`, `docs/GOTCHAS.md`, `examples/salidas_verificadas.json`. Luego pegá el prompt de abajo en el cuadro de generación / chat de NotebookLM.

---

## PROMPT (copiar y pegar)

```
Actúa como diseñador instruccional experto en charlas técnicas. Con base EXCLUSIVA en las fuentes cargadas (el repositorio de un ejercicio de clase sobre extracción de resultados de loterías con n8n e IA), generá el guion completo de una presentación de diapositivas en español, lista para armar en PowerPoint.

AUDIENCIA: estudiantes que están aprendiendo automatización con n8n e inteligencia artificial. Nivel intermedio. No asumas que conocen yt-dlp, LLMs locales ni agentes.

OBJETIVO DE LA CHARLA: que entiendan, paso a paso, cómo construimos el workflow Y —sobre todo— por qué la disciplina de VERIFICAR cada salida de la IA es lo que separa un buen resultado de un desastre convincente.

TONO: didáctico, directo y apasionado, como un mentor que quiere que el otro aprenda de verdad. Cero relleno corporativo. Usá ejemplos concretos de las fuentes (números reales, comandos reales).

FORMATO DE SALIDA: una diapositiva por bloque, numeradas, cada una con EXACTAMENTE esta estructura:
- TÍTULO: (máximo 8 palabras)
- VIÑETAS: (3 a 5, concisas, una idea cada una)
- NOTAS DEL ORADOR: (2 a 4 frases de lo que digo en voz alta, no leídas de la pantalla)
- VISUAL SUGERIDO: (qué imagen, diagrama o captura poner)

ESTRUCTURA OBLIGATORIA (15 diapositivas):
1. Portada: título de la charla + una frase gancho.
2. El problema: los resultados de lotería están en video; el número lo DICE la presentadora, no está escrito. Ninguna API lo entrega.
3. La primera trampa (IAP): la instrucción que parecía correcta (configurar IAP/gcloud) no resolvía nuestro problema. Lección: verificá que el tutorial sea para TU caso antes de seguirlo.
4. La materia prima: cómo conseguimos la transcripción con yt-dlp (subtítulos automáticos).
5. El corazón del problema: los números vienen hablados y dispersos ("Siete 3 0 8 de la serie 711" = 7308, serie 711). Por eso un regex NO sirve y necesitamos un LLM.
6. Las herramientas: n8n (orquestador), yt-dlp (transcripción), Ollama (IA local), Groq (IA en la nube), Google Sheets (destino). Una línea por herramienta.
7. El flujo: el diagrama de nodos de n8n, de la URL a la hoja de cálculo.
8. Dos agentes, no uno: extractor + auditor. Por qué una segunda pasada que verifica.
9. Los gotchas técnicos: los bugs que encontramos PROBANDO (--no-simulate, subtítulos manuales, videos sin subtítulos, rate limit 429). "Funcionó una vez" no es "funciona".
10. La prueba de la verdad: mismo código, dos modelos. qwen2.5:3b (1 registro basura, confianza 100, ~5 min) vs Groq llama-3.3-70b (35 registros correctos, ~13 s).
11. El modelo grande también miente: con un modelo bueno, el JSON se veía perfecto, pero etiquetaba el número de SORTEO como serie y se comía el signo del Astro. Solo verificar contra la fuente lo reveló.
12. La metáfora central: hacer cosas con IA es como ser emperador romano. Marco Aurelio (dirige con disciplina y verificación) vs Cómodo (hereda el mismo poder y lo hunde por confiar en la apariencia). confianza:100 no significa "correcto".
13. Las 6 lecciones: la tabla resumen del tutorial.
14. Manos a la obra: los ejercicios propuestos para los estudiantes (lote de URLs, fallback automático, rama de OCR, verificación manual).
15. Cierre: "Tú diriges, la IA ejecuta. La diferencia es verificar." + invitación a clonar el repo.

REGLAS:
- No inventes datos: usá solo lo que está en las fuentes. Si un número o comando no está, no lo pongas.
- Mantené los términos técnicos en su forma original (yt-dlp, LLM, JSON, workflow, prompt).
- Las viñetas son para la pantalla (cortas); las notas del orador son para hablar (completas).
- Al final, agregá una diapositiva extra "RECURSOS" con los nombres de archivo del repo donde está cada cosa.
```

---

## Variantes rápidas (opcional)

- **Más corta (10 slides):** agregá al final del prompt: *"Reducí a 10 diapositivas fusionando los gotchas (9) con la prueba de modelos (10-11). Prioridad: el problema, la solución, y la lección de verificar."*
- **Con foco en la metáfora:** agregá: *"Dedicá 3 diapositivas a desarrollar la analogía Marco Aurelio vs Cómodo, conectando cada decisión técnica con una virtud o vicio del gobernante."*
- **Para audio overview (podcast) en vez de slides:** cambiá la primera línea por *"Generá un guion de conversación de 8 minutos entre dos presentadores..."* y quitá el formato de diapositivas.
