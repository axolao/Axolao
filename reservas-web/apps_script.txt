/**
 * AXOLAO — Backend de Reservas
 * Google Apps Script (gratis, corre en la cuenta de Google del restaurante).
 *
 * Endpoints (mismo URL del Web App):
 *   GET  ?action=disponibilidad   -> { totalesPorDia: {"2026-04-21": 12, ...} }
 *   POST { action:"reservar", nombre, email, fecha, hora, personas }
 *        -> { status:"ok"|"full"|"error", message?, restantes? }
 *
 * Configuración: edita el bloque CONFIG abajo. Luego: Implementar > Web App
 *   - Ejecutar como: Yo (tu cuenta)
 *   - Quién tiene acceso: Cualquier usuario (anónimo)
 *
 * El sheet debe tener una hoja llamada "Reservas" con encabezados:
 *   timestamp | nombre | email | fecha | hora | personas | estado
 * El script la crea automáticamente la primera vez si no existe.
 */

// ============== CONFIG ==============
const CONFIG = {
  CAPACIDAD_DIARIA: 30,                 // tope de personas por día
  HOJA_RESERVAS: "Reservas",            // nombre de la hoja dentro del Sheet
  NOMBRE_RESTAURANTE: "Axolao",
  // Si quieres recibir copia interna de cada reserva, pon aquí emails separados por coma:
  EMAIL_INTERNO: "",                    // ej: "equipo@axolao.com"
  // Asunto del correo de confirmación al cliente:
  ASUNTO_CONFIRMACION: "Tu reserva en Axolao está confirmada",
  // Cuántos días hacia adelante exponer en el endpoint de disponibilidad:
  DIAS_DISPONIBILIDAD: 90,
};
// ====================================

// ---------- ENDPOINTS ----------

function doGet(e) {
  try {
    const action = (e && e.parameter && e.parameter.action) || "disponibilidad";
    if (action === "disponibilidad") {
      return jsonResponse({ totalesPorDia: getTotalesPorDia() });
    }
    return jsonResponse({ status: "error", message: "Acción no soportada." });
  } catch (err) {
    return jsonResponse({ status: "error", message: String(err) });
  }
}

function doPost(e) {
  try {
    const body = JSON.parse(e.postData.contents);
    const action = body.action || "reservar";
    if (action !== "reservar") {
      return jsonResponse({ status: "error", message: "Acción no soportada." });
    }
    return registrarReserva(body);
  } catch (err) {
    return jsonResponse({ status: "error", message: String(err) });
  }
}

// ---------- LÓGICA ----------

function registrarReserva(data) {
  const nombre = sanitize(data.nombre);
  const email = sanitize(data.email);
  const fecha = sanitize(data.fecha);     // "YYYY-MM-DD"
  const hora = sanitize(data.hora);
  const personas = parseInt(data.personas, 10);

  if (!nombre || !email || !fecha || !hora || !personas || personas < 1) {
    return jsonResponse({ status: "error", message: "Datos incompletos." });
  }
  if (!/^\d{4}-\d{2}-\d{2}$/.test(fecha)) {
    return jsonResponse({ status: "error", message: "Formato de fecha inválido." });
  }
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    return jsonResponse({ status: "error", message: "Correo inválido." });
  }

  // Bloqueo para evitar condiciones de carrera (dos reservas que llegan al mismo tiempo)
  const lock = LockService.getScriptLock();
  lock.waitLock(20 * 1000);
  try {
    const sheet = getOrCreateHoja();
    const totalDia = sumarPersonasDelDia(sheet, fecha);

    if (totalDia >= CONFIG.CAPACIDAD_DIARIA) {
      return jsonResponse({
        status: "full",
        message: "Día completo.",
      });
    }
    if (totalDia + personas > CONFIG.CAPACIDAD_DIARIA) {
      return jsonResponse({
        status: "full",
        message: "Cupos insuficientes.",
        restantes: CONFIG.CAPACIDAD_DIARIA - totalDia,
      });
    }

    sheet.appendRow([
      new Date(),
      nombre,
      email,
      fecha,
      hora,
      personas,
      "confirmada",
    ]);

    enviarEmailConfirmacion({ nombre, email, fecha, hora, personas });
    if (CONFIG.EMAIL_INTERNO) {
      enviarEmailInterno({ nombre, email, fecha, hora, personas, totalDia: totalDia + personas });
    }

    return jsonResponse({
      status: "ok",
      restantes: CONFIG.CAPACIDAD_DIARIA - (totalDia + personas),
    });
  } finally {
    lock.releaseLock();
  }
}

function getTotalesPorDia() {
  const sheet = getOrCreateHoja();
  const last = sheet.getLastRow();
  if (last < 2) return {};
  const values = sheet.getRange(2, 1, last - 1, 7).getValues();
  // Columnas: 0 timestamp, 1 nombre, 2 email, 3 fecha, 4 hora, 5 personas, 6 estado
  const totales = {};
  const hoy = new Date();
  hoy.setHours(0, 0, 0, 0);
  const limite = new Date(hoy);
  limite.setDate(limite.getDate() + CONFIG.DIAS_DISPONIBILIDAD);

  values.forEach((row) => {
    const fecha = normalizarFecha(row[3]);
    const personas = parseInt(row[5], 10) || 0;
    const estado = String(row[6] || "").toLowerCase();
    if (!fecha || estado === "cancelada") return;
    const d = new Date(fecha + "T00:00:00");
    if (isNaN(d) || d < hoy || d > limite) return;
    totales[fecha] = (totales[fecha] || 0) + personas;
  });
  return totales;
}

function sumarPersonasDelDia(sheet, fechaStr) {
  const last = sheet.getLastRow();
  if (last < 2) return 0;
  const values = sheet.getRange(2, 4, last - 1, 4).getValues(); // fecha, hora, personas, estado
  let total = 0;
  values.forEach((row) => {
    const f = normalizarFecha(row[0]);
    const personas = parseInt(row[2], 10) || 0;
    const estado = String(row[3] || "").toLowerCase();
    if (f === fechaStr && estado !== "cancelada") total += personas;
  });
  return total;
}

// ---------- EMAIL ----------

function enviarEmailConfirmacion({ nombre, email, fecha, hora, personas }) {
  const fechaBonita = formatearFechaEsp(fecha);
  const cuerpo = [
    `¡Hola ${nombre}!`,
    ``,
    `Tu reserva en ${CONFIG.NOMBRE_RESTAURANTE} está confirmada.`,
    ``,
    `📅 Fecha: ${fechaBonita}`,
    `🕒 Hora: ${hora}`,
    `👥 Personas: ${personas}`,
    ``,
    `Una chingana chingona te espera. Si necesitas modificar o cancelar, respóndenos a este correo.`,
    ``,
    `— El equipo de ${CONFIG.NOMBRE_RESTAURANTE}`,
  ].join("\n");

  MailApp.sendEmail({
    to: email,
    subject: CONFIG.ASUNTO_CONFIRMACION,
    body: cuerpo,
    name: CONFIG.NOMBRE_RESTAURANTE,
  });
}

function enviarEmailInterno({ nombre, email, fecha, hora, personas, totalDia }) {
  const fechaBonita = formatearFechaEsp(fecha);
  const cuerpo = [
    `Nueva reserva:`,
    ``,
    `Nombre: ${nombre}`,
    `Email: ${email}`,
    `Fecha: ${fechaBonita}`,
    `Hora: ${hora}`,
    `Personas: ${personas}`,
    ``,
    `Total acumulado para ese día: ${totalDia} / ${CONFIG.CAPACIDAD_DIARIA}`,
  ].join("\n");

  MailApp.sendEmail({
    to: CONFIG.EMAIL_INTERNO,
    subject: `[Reserva] ${nombre} · ${fechaBonita} · ${hora} · ${personas}p`,
    body: cuerpo,
  });
}

// ---------- HELPERS ----------

function getOrCreateHoja() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(CONFIG.HOJA_RESERVAS);
  if (!sheet) {
    sheet = ss.insertSheet(CONFIG.HOJA_RESERVAS);
    sheet.appendRow([
      "timestamp", "nombre", "email", "fecha", "hora", "personas", "estado",
    ]);
    const header = sheet.getRange(1, 1, 1, 7);
    header.setFontWeight("bold");
    header.setBackground("#431848");
    header.setFontColor("#cac89d");
    sheet.setFrozenRows(1);
    sheet.setColumnWidths(1, 7, 130);
  }
  return sheet;
}

function normalizarFecha(v) {
  if (!v) return "";
  if (Object.prototype.toString.call(v) === "[object Date]") {
    const y = v.getFullYear();
    const m = String(v.getMonth() + 1).padStart(2, "0");
    const d = String(v.getDate()).padStart(2, "0");
    return `${y}-${m}-${d}`;
  }
  const s = String(v).trim();
  if (/^\d{4}-\d{2}-\d{2}$/.test(s)) return s;
  // Si viene "21/4/2026" o similar, lo intentamos parsear
  const d = new Date(s);
  if (!isNaN(d)) {
    const y = d.getFullYear();
    const m = String(d.getMonth() + 1).padStart(2, "0");
    const day = String(d.getDate()).padStart(2, "0");
    return `${y}-${m}-${day}`;
  }
  return "";
}

function formatearFechaEsp(yyyy_mm_dd) {
  const dias = ["domingo", "lunes", "martes", "miércoles", "jueves", "viernes", "sábado"];
  const meses = ["enero", "febrero", "marzo", "abril", "mayo", "junio",
                 "julio", "agosto", "septiembre", "octubre", "noviembre", "diciembre"];
  const d = new Date(yyyy_mm_dd + "T00:00:00");
  if (isNaN(d)) return yyyy_mm_dd;
  return `${dias[d.getDay()]} ${d.getDate()} de ${meses[d.getMonth()]} ${d.getFullYear()}`;
}

function sanitize(v) {
  return String(v == null ? "" : v).trim().slice(0, 200);
}

function jsonResponse(obj) {
  return ContentService
    .createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}

// ---------- TEST MANUAL (opcional) ----------
// Selecciona la función "testRegistro" en el editor y dale Ejecutar para probar.
function testRegistro() {
  const r = registrarReserva({
    nombre: "Prueba",
    email: "tu-correo@ejemplo.com",
    fecha: Utilities.formatDate(new Date(Date.now() + 86400000), "GMT", "yyyy-MM-dd"),
    hora: "20:00",
    personas: 2,
  });
  Logger.log(r.getContent());
}
