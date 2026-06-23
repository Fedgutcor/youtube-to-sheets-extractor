# Correr el proyecto en n8n local (self-hosted, macOS)

Esta guía levanta n8n en tu Mac y deja el workflow funcionando con `yt-dlp` nativo. Es el camino **recomendado** para este proyecto (gratis, sin servicios externos). Para n8n Cloud, ver [`SETUP_CLOUD.md`](SETUP_CLOUD.md).

> Todo lo de acá lo aprendimos instalándolo de verdad. Los "⚠️ Gotcha" son trampas reales que aparecieron.

---

## Requisitos

| Herramienta | Para qué | Instalar |
|---|---|---|
| **Node.js 20+** | Correr n8n | `brew install node` (o nvm) |
| **n8n 1.70+** | El orquestador | ver abajo (la versión importa) |
| **yt-dlp** | Bajar transcripciones | `brew install yt-dlp` |
| **ffmpeg** | Dependencia de yt-dlp | `brew install ffmpeg` |

### ⚠️ Gotcha #9 — la versión de n8n importa

El workflow usa nodos modernos (**Groq Chat Model**, Google Sheets v4.5, Basic LLM Chain v1.5). Un n8n viejo (ej. 1.50) **no los reconoce** y tira:

```
This node is not currently installed. It is either from a newer version of n8n...
Cannot read properties of undefined (reading 'description')
```

La regla de compatibilidad **con Node 20**:
- **n8n 1.70.x** → funciona con Node 20 **y** ya trae el nodo de Groq. ✅ Recomendado.
- **n8n 2.x (última)** → exige **Node 22+**. Si querés la última, primero `brew install node@22`.

Instalar la 1.70:
```bash
npm install -g n8n@1.70.0
# Verificar:
n8n --version
```

---

## Arrancar n8n

```bash
n8n start
```
Queda escuchando en **http://localhost:5678**. Dejá esa terminal abierta (n8n corre ahí).

> **PATH:** el nodo *Execute Command* ejecuta `yt-dlp` usando el PATH del shell donde arrancaste n8n. Si da "command not found", arrancá así para forzar el PATH:
> ```bash
> PATH="/usr/local/bin:$PATH" n8n start
> ```

### ⚠️ Gotcha — cuenta vieja / "wrong username or password"

Si al abrir `localhost:5678` te pide **iniciar sesión** (en vez de **crear cuenta**) y ninguna clave funciona, hay una cuenta vieja en la base. Reseteá:
```bash
# detené n8n (Ctrl+C en su terminal), luego:
rm -f ~/.n8n/database.sqlite ~/.n8n/database.sqlite-shm ~/.n8n/database.sqlite-wal
n8n start
```
Refrescá el navegador → ahora sí aparece **"Set up owner account"**. Creá tu cuenta (la contraseña necesita 8+ caracteres, 1 mayúscula, 1 número).

---

## Importar y configurar

1. **Importar:** en el lienzo, menú **⋮** (arriba derecha) → **Import from File** → elegí `workflow/loterias_v1.json`.
2. **Groq (la IA):** doble clic en el nodo *Groq (principal)* → credencial → *Create New* → pegá tu API key de [console.groq.com/keys](https://console.groq.com/keys).
   - ⚠️ Si ves un **401 / Authorization failed**, la key está revocada → generá una nueva.
3. **Google Sheets:** doble clic en *Guardar en Hoja* → credencial *Google Sheets OAuth2*. En local necesitás tu **propio** cliente OAuth de Google Cloud (ver [`SETUP_LOCAL_GOOGLE.md` más abajo](#google-sheets-oauth-en-local)).
4. **Tu hoja:** en el nodo *Configurar*, pegá tu `sheet_id` en el campo correspondiente.
5. **Ejecutar:** botón **Test workflow** abajo. Mirá los nodos ponerse verdes.

---

## Google Sheets OAuth en local

A diferencia de n8n Cloud (que usa su propio OAuth), en self-hosted tenés que crear **tu** cliente OAuth de Google:

1. [Google Cloud Console](https://console.cloud.google.com) → **APIs y servicios → Biblioteca** → habilitá **Google Sheets API** y **Google Drive API**.
2. **Pantalla de consentimiento de OAuth** → tipo *Externo* → agregá tu email en *Usuarios de prueba*.
3. **Credenciales → Crear → ID de cliente OAuth → Aplicación web**.
4. En **URIs de redireccionamiento autorizados** pegá exactamente:
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
5. Copiá **Client ID** y **Client Secret** → pegalos en la credencial de Google Sheets en n8n → **Sign in with Google**.

> Es el paso más tedioso, pero se hace una sola vez. Si solo querés ver la extracción funcionar, salteá Sheets por ahora: ejecutá el flujo y mirá la salida JSON en el nodo *Agente Auditor*.

---

## Verificar que funciona sin la hoja

Para probar el núcleo (transcripción + IA) sin configurar Google Sheets todavía:
1. Ejecutá el workflow.
2. El nodo *Guardar en Hoja* se pondrá rojo (sin OAuth) — **es esperado**.
3. Hacé clic en *Agente Auditor* → pestaña **Output**: ahí ves el JSON con los sorteos extraídos. Si aparecen los números, **el motor funciona**. ✅
