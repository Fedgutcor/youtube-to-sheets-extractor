# Correr el proyecto en n8n Cloud

> **Resumen honesto:** n8n Cloud **no puede correr `yt-dlp`** (el nodo *Execute Command* está deshabilitado por seguridad). Como `yt-dlp` es lo que baja la transcripción, en Cloud hay que reemplazar ese paso por un **servicio externo de transcripción** vía HTTP. Funciona, pero es más frágil y normalmente **de pago**. Para este proyecto, [local](SETUP_LOCAL.md) es mejor.

---

## Qué cambia respecto al workflow local

Todo el flujo es idéntico **excepto el primer paso** (obtener la transcripción):

| Paso | Local | Cloud |
|---|---|---|
| Transcripción | Nodo *Execute Command* → `yt-dlp` | Nodo *HTTP Request* → API de transcripción de terceros |
| Resto (Extractor, Auditor, Expandir, Sheets) | igual | igual |

---

## Opción A — API de transcripción (recomendada para Cloud)

Reemplazás el nodo **Descargar Transcripción (yt-dlp)** por un nodo **HTTP Request** que le pega a un servicio que devuelve la transcripción de un video de YouTube. Servicios usables (todos con free tier limitado):

- **Supadata** (`supadata.ai`) — API de transcript de YouTube.
- **youtube-transcript.io** — API REST.
- **Apify** — actor "YouTube Transcript Scraper".
- **RapidAPI** — varias APIs de "YouTube Transcript".

### Cómo armarlo

1. Creá cuenta en el servicio elegido y obtené tu **API key**.
2. En el workflow, **borrá** el nodo *Descargar Transcripción (yt-dlp)*.
3. Agregá un nodo **HTTP Request** en su lugar, conectado entre *Configurar* y *Preparar Fuentes*:
   - **Method/URL:** según la doc del servicio (GET o POST, con la URL del video).
   - **Auth:** header con tu API key (ej. `x-api-key: TU_KEY`).
   - **Query/Body:** la URL del video → `={{ $('Configurar').item.json.video_url }}`.
4. Ajustá el nodo **Preparar Fuentes** (Code) para leer la respuesta del servicio en vez del formato de yt-dlp. La respuesta es un JSON distinto según el proveedor; mapeá:
   ```js
   // Ejemplo genérico — adaptá los nombres de campo a tu servicio
   const r = $input.first().json;
   return [{ json: {
     video_url: $('Configurar').item.json.video_url,
     id: r.videoId ?? '',
     title: r.title ?? '',
     description: r.description ?? '',
     transcript: r.transcript ?? r.text ?? '',   // <- el campo varía por API
     ocr: null
   }}];
   ```

> ⚠️ **No hay un “pegá esto y anda”** universal: cada API de transcript devuelve un formato distinto. Hay que leer su doc y mapear el campo del texto. Por eso local (con `yt-dlp`) es más simple y robusto.

---

## Opción B — n8n híbrido (lo mejor de los dos mundos)

Si querés la comodidad de Cloud pero el poder de `yt-dlp`:

1. Corré un **mini-servicio propio** (un webhook que ejecuta `yt-dlp`) en tu Mac o un servidor barato.
2. Desde n8n Cloud, un nodo **HTTP Request** le pega a ese webhook y recibe la transcripción.

Es más setup, pero te da `yt-dlp` real sin depender de un tercero de pago. Para una clase, normalmente no vale la pena: con **local** ya tenés todo.

---

## Credenciales en Cloud (esto sí es más fácil que en local)

La parte buena de Cloud: **Google Sheets OAuth ya funciona** sin crear tu propio cliente de Google (n8n Cloud usa su OAuth gestionado, con redirect `oauth.n8n.cloud`). Solo conectás tu cuenta y listo. En local, en cambio, tenés que crear el cliente OAuth a mano (ver [`SETUP_LOCAL.md`](SETUP_LOCAL.md#google-sheets-oauth-en-local)).

---

## Recomendación final

| Si... | Usá |
|---|---|
| Querés que funcione gratis y robusto, y no te molesta instalar | **Local** ([SETUP_LOCAL.md](SETUP_LOCAL.md)) |
| Ya tenés Cloud y aceptás pagar/configurar una API de transcript | **Cloud, Opción A** |
| Querés Cloud pero con yt-dlp real | **Cloud, Opción B (híbrido)** |
