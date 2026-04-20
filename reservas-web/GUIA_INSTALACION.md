# Guía de instalación — Reservas Axolao

Sistema **100% gratis** que reemplaza el flujo manual de WhatsApp por una landing page con la marca de Axolao, que registra reservas en un Google Sheet, bloquea automáticamente los días con 30 personas o más y envía un correo de confirmación al cliente.

Tiempo estimado de setup: **15–25 minutos**.

---

## Lo que vas a necesitar

- Una cuenta de Google (la del restaurante de preferencia).
- Estos 3 archivos de la carpeta `reservas-web/`:
  - `index.html` — la landing page
  - `apps_script.gs` — el backend
  - `logo.png` y `logo-morado.png` — los logos (versión beige y morada)

---

## Paso 1 — Crear el Google Sheet

1. Entra a [sheets.google.com](https://sheets.google.com) con la cuenta del restaurante.
2. Crea una hoja nueva en blanco. Llámala **"Axolao — Reservas"**.
3. No necesitas armar columnas a mano. El script las crea solito en una hoja llamada `Reservas` la primera vez que recibe una reserva.

---

## Paso 2 — Pegar el Apps Script

1. Dentro del Sheet, ve al menú **Extensiones → Apps Script**.
2. Se abre el editor. Borra todo el contenido del archivo `Code.gs` que viene por default.
3. Abre el archivo `apps_script.gs` (de la carpeta `reservas-web/`), copia **todo** y pégalo en el editor.
4. (Opcional) En el bloque `CONFIG` de arriba puedes ajustar:
   - `CAPACIDAD_DIARIA` — el tope diario (por default 30).
   - `EMAIL_INTERNO` — pon el correo del equipo si quieres recibir copia de cada reserva. Si lo dejas vacío, no manda nada al equipo.
   - `NOMBRE_RESTAURANTE` y `ASUNTO_CONFIRMACION` — texto de los correos al cliente.
5. Guarda con el ícono de disquete (o `Cmd/Ctrl + S`). Ponle nombre al proyecto: **"Reservas Axolao"**.

---

## Paso 3 — Desplegar como Web App

Esto es lo que convierte el script en una URL pública a la que la landing puede llamar.

1. Arriba a la derecha, click en **Implementar → Nueva implementación**.
2. En el ícono de tuerca (⚙) elige **Aplicación web**.
3. Configura:
   - **Descripción**: "v1 reservas"
   - **Ejecutar como**: *Yo* (tu cuenta de Google)
   - **Quién tiene acceso**: **Cualquier usuario** (anónimo). Esto es lo que permite que la landing funcione para tus clientes sin pedirles login.
4. Click en **Implementar**.
5. La primera vez te va a pedir **autorizar permisos**. Acepta:
   - Click en *Autorizar acceso*
   - Elige tu cuenta
   - En la pantalla de "Google no ha verificado esta app", click en *Configuración avanzada* → *Ir a Reservas Axolao (no seguro)*. Es **tu propio script** corriendo en tu Sheet, no es un riesgo, es la advertencia estándar de Google para proyectos personales.
   - Acepta los permisos (necesita acceso al Sheet y a Gmail para mandar la confirmación).
6. Te va a dar una **URL del Web App** que se ve así:
   ```
   https://script.google.com/macros/s/AKfycbxxxxxxxxxxxxxx/exec
   ```
   **Cópiala**. La vamos a pegar en el HTML.

---

## Paso 4 — Configurar el HTML

1. Abre `index.html` con cualquier editor de texto (incluso TextEdit o Notas).
2. Busca esta línea (cerca del inicio del bloque `<script>`, alrededor de la línea 320):
   ```js
   const APPS_SCRIPT_URL = "PEGA_AQUI_LA_URL_DE_TU_APPS_SCRIPT";
   ```
3. Reemplaza `"PEGA_AQUI_LA_URL_DE_TU_APPS_SCRIPT"` con la URL que copiaste en el paso anterior. Ejemplo:
   ```js
   const APPS_SCRIPT_URL = "https://script.google.com/macros/s/AKfycbx.../exec";
   ```
4. Guarda el archivo.

---

## Paso 5 — Probar localmente (opcional pero recomendado)

Antes de publicar al mundo, hagamos una prueba.

1. Abre `index.html` con doble click — se abre en tu navegador.
2. Llena el formulario con un nombre, **tu propio correo**, una fecha futura, hora y 2 personas.
3. Click en *Reservar mi mesa*.
4. Si todo está bien deberías ver:
   - Pantalla de confirmación con tu reserva.
   - Un correo en tu bandeja con la confirmación.
   - Una nueva fila en el Sheet (en la hoja `Reservas`).
5. Si algo falla, revisa la consola del navegador (`Cmd/Ctrl + Opción + I` → pestaña *Console*) y mira el error. Lo más común es:
   - Pegaste mal la URL del Apps Script.
   - El Apps Script quedó implementado con acceso restringido (debe ser *Cualquier usuario*).

---

## Paso 6 — Publicar la landing en internet (gratis)

Tienes 3 opciones gratis. Te recomiendo **Netlify Drop** porque es literalmente arrastrar y soltar.

### Opción A · Netlify Drop (más fácil, sin cuenta requerida al inicio) ⭐
1. Entra a [app.netlify.com/drop](https://app.netlify.com/drop)
2. Arrastra la carpeta `reservas-web/` completa (con el `index.html` y los 2 logos) al área indicada.
3. En segundos te da una URL tipo `https://reservas-axolao-xxx.netlify.app`.
4. Crea cuenta gratis para conservar la URL y editar el subdominio (puedes ponerle `axolao-reservas.netlify.app`).
5. Más adelante puedes apuntar tu propio dominio (ej. `reservas.axolao.com`).

### Opción B · GitHub Pages (si ya usas GitHub)
1. Crea un repo público en GitHub llamado `reservas-axolao`.
2. Sube los archivos de la carpeta `reservas-web/`.
3. En *Settings → Pages*, selecciona la rama `main` y carpeta `/root`.
4. En 1–2 minutos tendrás una URL: `https://tu-usuario.github.io/reservas-axolao/`.

### Opción C · Vercel
1. Entra a [vercel.com](https://vercel.com), crea cuenta gratis con GitHub.
2. Click en *New Project* → importa el repo (o sube los archivos).
3. Deploy en 30 segundos.

---

## Paso 7 — Compartir con los clientes

Pon la URL en:
- **Bio de Instagram** y stories
- **Estado y descripción de WhatsApp Business** del restaurante
- **Google My Business** como botón de reserva
- **Mensaje automático** del WhatsApp Business: cuando alguien escriba pidiendo reservar, responde con la URL ("¡Hola! Reserva tu mesa aquí: <url>")
- Cualquier flyer o material impreso (puedes generar un QR gratis en [qr-code-generator.com](https://www.qr-code-generator.com))

---

## Cómo funciona el control de cupos (importante)

- Cada vez que alguien abre la landing, el calendario consulta al Apps Script qué días ya están llenos. Esos días aparecen **tachados y deshabilitados** en el calendario — el cliente no puede ni intentar reservarlos.
- Cuando el cliente envía la reserva, **el Apps Script revuelve un candado** (`LockService`) y vuelve a contar las personas registradas para ese día. Si se pasa de 30, devuelve `"full"` y la landing muestra el mensaje:
  > *"Muchas gracias, no tenemos espacio para este día. Puedes acercarte directamente al restaurante si lo deseas, aceptaremos personas por orden de llegada."*
- Esto previene el caso del "doble cierre": dos personas reservando al mismo tiempo el último cupo.

---

## Operación diaria

- **Ver las reservas**: solo abre el Sheet "Axolao — Reservas". Cada fila es una reserva nueva con timestamp, contacto, fecha, hora y personas.
- **Cancelar una reserva**: cambia la columna `estado` de "confirmada" a "cancelada". Esto **libera automáticamente el cupo** de ese día.
- **Cerrar un día manualmente** (ej. evento privado): agrega una fila con cualquier nombre, la fecha bloqueada, hora `00:00`, y `personas = 30`. Estado "confirmada". El día queda lleno.
- **Cambiar el tope de 30**: edita `CAPACIDAD_DIARIA` en el Apps Script y vuelve a desplegar (Implementar → Administrar implementaciones → editar la actual → Nueva versión).

---

## Personalizar el contenido

Todos los textos visibles están en `index.html`. Algunos lugares útiles:

| Para cambiar | Busca en `index.html` |
|---|---|
| Título "Reserva tu mesa" | `<h1>Reserva` |
| Subtítulo "una chingana chingona…" | `<p>Una <strong>chingana` |
| Horarios disponibles | `<select class="select" id="hora"` |
| Máximo de personas por reserva | `<select class="select" id="personas"` |
| Mensaje de "día lleno" | `id="alertFull"` |

Los **colores** salen de la paleta oficial del brandbook (Pantone 2593C / 5875C / 3558C / 7416C / 2334C) y están definidos en las variables CSS al inicio de `<style>`.

---

## Notas técnicas

- **Tipografía**: la landing usa **DM Sans** para cuerpo (igual que el brandbook) y **Anton** como sustituto libre de **Compadre** para titulares — porque Compadre es una tipografía con licencia comercial. Si tienes Compadre licenciada y quieres usarla en producción, la puedes hostear como `@font-face` y reemplazar `"Anton"` por `"Compadre"` en el CSS.
- **Logo**: el HTML carga `logo.png` (versión beige sobre fondo morado). Para versión navideña, eventos, etc., basta con reemplazar el archivo `logo.png` manteniendo el mismo nombre.
- **Sin servidor propio**: el sistema completo corre en infraestructura de Google (Sheets + Apps Script) + un static host gratis (Netlify/Vercel/GitHub Pages). Costo recurrente: **$0**.
- **Límites de Gmail desde Apps Script**: 100 correos/día con cuenta personal, 1500/día con Google Workspace. Más que suficiente para reservas.

---

## ¿Algo no funciona?

| Problema | Solución |
|---|---|
| Calendario dice "⚠ Falta configurar APPS_SCRIPT_URL" | No reemplazaste la URL en `index.html`. Ver paso 4. |
| "Error de conexión" al enviar | La URL del Apps Script está mal o no se desplegó como "Cualquier usuario". Ver paso 3. |
| Las fechas llenas no se bloquean | Refresca la página (la disponibilidad se carga al abrir). Verifica que el Sheet no tenga las fechas en formato distinto a `YYYY-MM-DD`. |
| No llega el correo de confirmación | Revisa Spam. Si nada, ve al editor de Apps Script → menú *Ejecutar* → función `testRegistro` y mira los logs (*Ver registros*). |
| Quiero recuperar el sistema viejo | Solo deja la página caída. Las reservas que ya hayan entrado quedan en el Sheet. |

---

## Roadmap sugerido (si después quieres crecer)

- **Recordatorios automáticos** 24h antes (Apps Script puede correr en *triggers* programados).
- **Reseña post-visita**: email automático al día siguiente con un link a Google Reviews.
- **Lista de espera** cuando un día esté lleno (capturar datos y avisar si hay cancelaciones).
- **WhatsApp confirmation** vía API gratuita de [CallMeBot](https://www.callmebot.com) o Twilio (no es 100% gratis pero el primer tier es bajo costo).

¡Provecho! 🌶
