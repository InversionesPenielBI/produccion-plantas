// ════════════════════════════════════════════════════════════════════════
// ENTORNO COMUN — Inversiones Peniel
// ════════════════════════════════════════════════════════════════════════
// Generado por herramientas/generar_compartido.py a partir de index.html.
// NO editar a mano: los cambios se hacen en index.html y se regenera.
//
// Cada pagina define window.APP ANTES de cargar este archivo:
//
//   window.APP = {
//     SHEET_ID:    '...',            // hoja de calculo de esa planta
//     CLIENT_ID:   '...',            // el mismo para todas
//     SCOPES:      '...',
//     SESSION_KEY: 'peniel_xxx_v1',  // distinta por pagina: no se pisan
//     TITULOS:     { vista: 'Titulo en la barra superior' },
//     CAMPOS_FECHA_HOY: ['id-de-input'],   // se rellenan con la fecha de hoy
//     alIniciar()        {},   // sesion lista: cargar datos y pintar
//     alCambiarUsuario() {},   // ya se sabe quien entro y con que rol
//     alCambiarVista(v)  {},   // el usuario cambio de seccion
//   };
// ════════════════════════════════════════════════════════════════════════

const APP = window.APP || {};
const SHEET_ID  = APP.SHEET_ID;
const CLIENT_ID = APP.CLIENT_ID;
const SCOPES    = APP.SCOPES ||
  'https://www.googleapis.com/auth/spreadsheets ' +
  'https://www.googleapis.com/auth/userinfo.profile ' +
  'https://www.googleapis.com/auth/userinfo.email';
const API_KEY = ''; // no se usa con OAuth

// Sesion: tiempo maximo de inactividad antes de cerrar sesion (8 horas).
// El usuario puede refrescar la pagina dentro de esa ventana sin perderla.
const INACTIVITY_TIMEOUT_MS = 8 * 60 * 60 * 1000;
const SESSION_STORAGE_KEY = APP.SESSION_KEY || 'peniel_session_v3';
let inactivityTimerId = null;

let accessToken = null;
let userInfo = null;
let tokenExpiresAt = 0;
let sheetIds = {};

// ----------------------------------------------------------------------
// Validacion de fechas
// ----------------------------------------------------------------------
const FECHA_MIN = '2025-01-01';
const FECHA_MAX = '2027-12-31';

function fechaFueraDeRango(v) {
  const s = String(v || '').trim();
  return s !== '' && (s < FECHA_MIN || s > FECHA_MAX);
}

// Devuelve las fechas fuera de rango dentro de un contenedor, con su etiqueta
function validarRangoFechas(contenedor) {
  const malas = [];
  if (!contenedor) return malas;
  contenedor.querySelectorAll('input[type="date"]').forEach(el => {
    if (!fechaFueraDeRango(el.value)) return;
    const lbl = el.closest('.field')?.querySelector('label')?.textContent.replace('*', '').trim()
             || el.getAttribute('title') || 'Fecha';
    malas.push({ el, lbl });
  });
  return malas;
}

function avisarFechasInvalidas(malas) {
  const nombres = malas.map(m => m.lbl);
  toast('El año debe estar entre 2025 y 2027. Revisa: '
    + nombres.slice(0, 3).join(', ')
    + (nombres.length > 3 ? ` y ${nombres.length - 3} más` : ''), 'error');
  const primera = malas[0].el;
  primera.closest('.card')?.classList.remove('collapsed');
  primera.focus();
}

// ----------------------------------------------------------------------
// Rueda del raton y aviso de fecha fuera de rango
// ----------------------------------------------------------------------
// La rueda del ratón sobre un campo numérico enfocado cambiaba el valor:
// se quita el foco antes de que el navegador lo modifique.
document.addEventListener('wheel', e => {
  const el = document.activeElement;
  if (el && el.tagName === 'INPUT' && el.type === 'number' && el === e.target) el.blur();
}, { passive: true });

// Marca en rojo la fecha mientras se escribe, sin esperar a guardar
document.addEventListener('input', e => {
  const el = e.target;
  if (el.tagName === 'INPUT' && el.type === 'date') {
    el.classList.toggle('fecha-invalida', fechaFueraDeRango(el.value));
  }
});

// ----------------------------------------------------------------------
// Escape de HTML y anti-inyeccion de formulas
// ----------------------------------------------------------------------
function esc(v) {
  return String(v ?? '')
    .replaceAll('&','&amp;').replaceAll('<','&lt;').replaceAll('>','&gt;')
    .replaceAll('"','&quot;').replaceAll("'",'&#39;');
}
// -- Anti-inyeccion de formulas en Sheets (#3 auditoria) --
function safeText(v) {
  const s = String(v ?? '');
  return /^[=+\-@]/.test(s) ? "'" + s : s;
}

// ----------------------------------------------------------------------
// Letra de columna
// ----------------------------------------------------------------------
// Letra de columna a partir del índice (0 -> A). Evita rangos escritos a mano
// que se rompen al añadir columnas.
function colLetra(i) {
  let s = ''; i++;
  while (i > 0) { const r = (i - 1) % 26; s = String.fromCharCode(65 + r) + s; i = Math.floor((i - 1) / 26); }
  return s;
}

// ----------------------------------------------------------------------
// Fila <-> objeto
// ----------------------------------------------------------------------
function objToRow(o, cols) { return cols.map(c => (o[c] ?? '')); }
function rowToObj(r, cols) { return Object.fromEntries(cols.map((c, i) => [c, r[i] ?? ''])); }

// ----------------------------------------------------------------------
// Roles y acceso / logo de marca
// ----------------------------------------------------------------------
// ═══════════════════════════════════════════════════════════
// ROLES Y ACCESO
// ═══════════════════════════════════════════════════════════
// Para dar acceso: agregar email → 'ADMIN' o 'USUARIO'
// ADMIN : acceso completo (crear, editar, cerrar, reabrir y eliminar pedidos)
// USUARIO: puede crear pedidos y verlos; NO puede editar/eliminar pedidos
//          cerrados ni reabrirlos.
const USUARIOS_AUTORIZADOS = {
  'analista.datos@inversionespeniel.com': 'ADMIN',
  'hsuarez@inversionespeniel.com':        'ADMIN',
  'esantiago10325@gmail.com': 'USUARIO'
  // Agregar más usuarios abajo (copiar la línea y cambiar email y rol):
  // 'correo@dominio.com': 'ADMIN',
  // 'correo@dominio.com': 'USUARIO',
};

let currentUserRole = null;
function isAdmin()   { return currentUserRole === 'ADMIN'; }
function isUsuario() { return currentUserRole === 'USUARIO'; }

function verificarAcceso(email) {
  if (!email) return null;
  return USUARIOS_AUTORIZADOS[email] || USUARIOS_AUTORIZADOS[email.toLowerCase()] || null;
}

// ═══════════════════════════════════════════════════════════
// LOGO FALLBACK (si logo.png no existe, mostramos SVG inline)
// ═══════════════════════════════════════════════════════════
function mostrarLogoFallback() {
  const box = document.getElementById('auth-logo-box');
  if (!box) return;
  box.innerHTML = `
    <svg viewBox="0 0 400 460" xmlns="http://www.w3.org/2000/svg">
      <g fill="none" stroke="#D7DEE3" stroke-width="10" stroke-linejoin="round" stroke-linecap="round">
        <circle cx="270" cy="160" r="110"/>
        <path d="M30 340 L110 200 L140 240 L170 200 L210 250 L265 140 L325 240 L370 195 L380 340 Z"/>
      </g>
      <text x="200" y="395" text-anchor="middle" fill="#D7DEE3"
            font-family="Plus Jakarta Sans, sans-serif" font-size="26" letter-spacing="6">INVERSIONES</text>
      <text x="200" y="445" text-anchor="middle" fill="#D7DEE3"
            font-family="Plus Jakarta Sans, sans-serif" font-weight="700" font-size="58" letter-spacing="10">PENIEL</text>
    </svg>`;
}


// ----------------------------------------------------------------------
// Sesion de Google y API de Sheets
// ----------------------------------------------------------------------
// ═══════════════════════════════════════════════════════════
// AUTH — Google Identity Services Token Client + persistencia
// ═══════════════════════════════════════════════════════════
let tokenClient = null;
let silentReauthPending = false;

function iniciarLogin() {
  const errEl = document.getElementById('auth-error');
  errEl.style.display = 'none';
  if (!tokenClient) {
    errEl.textContent = 'Error: Google no cargó. Recarga la página.';
    errEl.style.display = 'block';
    return;
  }
  // Primera vez requiere consentimiento; luego solo refresh silencioso
  tokenClient.requestAccessToken({ prompt: 'consent' });
}

function initTokenClient() {
  tokenClient = google.accounts.oauth2.initTokenClient({
    client_id: CLIENT_ID,
    scope: SCOPES,
    callback: (tokenResponse) => {
      if (tokenResponse.error) {
        if (silentReauthPending) {
          // Si falló el refresh silencioso, mostramos login
          silentReauthPending = false;
          forzarLogout('Tu sesión expiró. Inicia sesión nuevamente.');
          return;
        }
        document.getElementById('auth-error').textContent = 'Error: ' + tokenResponse.error;
        document.getElementById('auth-error').style.display = 'block';
        return;
      }
      silentReauthPending = false;
      accessToken = tokenResponse.access_token;
      // expires_in viene en segundos; restamos 60s de margen
      const expiresInSec = parseInt(tokenResponse.expires_in, 10) || 3600;
      tokenExpiresAt = Date.now() + (expiresInSec - 60) * 1000;

      // Si ya tenemos userInfo de una sesión persistida, no la pedimos otra vez
      if (userInfo) {
        // Re-verificar acceso (por si el rol cambió)
        if (!currentUserRole) {
          currentUserRole = verificarAcceso(userInfo.email);
        }
        if (!currentUserRole) {
          forzarLogout('No tienes acceso a esta aplicación. Contacta al administrador.');
          return;
        }
        actualizarUIUsuario(userInfo);
        guardarSesion();
        showApp();
        return;
      }
      // Get user info
      fetch('https://www.googleapis.com/oauth2/v3/userinfo', {
        headers: { Authorization: 'Bearer ' + accessToken }
      }).then(r => r.json()).then(info => {
        const role = verificarAcceso(info.email);
        if (!role) {
          forzarLogout('No tienes acceso a esta aplicación. Contacta al administrador.');
          return;
        }
        currentUserRole = role;
        userInfo = info;
        actualizarUIUsuario(info);
        guardarSesion();
      }).catch(() => {
        forzarLogout('No se pudo verificar tu cuenta. Inténtalo nuevamente.');
      }).finally(() => {
        // showApp se llama dentro de cada rama al verificar acceso
        if (userInfo && currentUserRole) showApp();
      });
    },
  });
}

// El logo sin letras se subió al repositorio; como no sabemos con qué nombre
// exacto quedó, se prueban las variantes más probables y se cae al logo
// completo (recortado por CSS) si ninguna existe.
// logosl.png es el logo sin letras. Si algún día cambia de nombre, basta con
// ponerlo aquí; logo.png queda como respaldo (se recorta por CSS).
const ARCHIVOS_MARCA = ['logosl.png','logo.png'];
document.addEventListener('DOMContentLoaded', () => cargarMarca(0));
function cargarMarca(i) {
  const img = document.getElementById('brand-img');
  if (!img || i >= ARCHIVOS_MARCA.length) return;
  const archivo = ARCHIVOS_MARCA[i];
  img.onerror = () => cargarMarca(i + 1);
  img.onload = () => { img.onerror = null; };
  img.classList.toggle('recortar', archivo === 'logo.png');
  img.src = encodeURI(archivo);
}

function actualizarUIUsuario(info) {
  const name = info.name || info.email || 'Usuario';
  document.getElementById('user-name').textContent = name.split(' ')[0];
  document.getElementById('user-avatar').textContent = (name[0] || 'U').toUpperCase();
  const roleEl = document.getElementById('user-role');
  if (roleEl) {
    if (currentUserRole === 'ADMIN') roleEl.textContent = 'Administrador';
    else if (currentUserRole === 'USUARIO') roleEl.textContent = 'Usuario';
    else roleEl.textContent = '';
  }
  // Cada página decide qué refrescar cuando ya se sabe quién entró
  if (typeof APP.alCambiarUsuario === 'function') APP.alCambiarUsuario();
}

function guardarSesion() {
  try {
    const payload = {
      accessToken: accessToken,
      tokenExpiresAt: tokenExpiresAt,
      userInfo: userInfo,
      currentUserRole: currentUserRole,
      lastActivity: Date.now(),
    };
    localStorage.setItem(SESSION_STORAGE_KEY, JSON.stringify(payload));
  } catch (e) { /* localStorage no disponible: ignorar */ }
}

function leerSesion() {
  try {
    const raw = localStorage.getItem(SESSION_STORAGE_KEY);
    if (!raw) return null;
    return JSON.parse(raw);
  } catch (e) { return null; }
}

function limpiarSesion() {
  try { localStorage.removeItem(SESSION_STORAGE_KEY); } catch(e){}
  accessToken = null;
  userInfo = null;
  tokenExpiresAt = 0;
  currentUserRole = null;
}

function intentarRestaurarSesion() {
  const s = leerSesion();
  if (!s) return false;

  // ¿Pasó la ventana de inactividad?
  const inactivoMs = Date.now() - (s.lastActivity || 0);
  if (inactivoMs > INACTIVITY_TIMEOUT_MS) {
    limpiarSesion();
    return false;
  }

  // Sí hay sesión válida en el almacenamiento
  userInfo = s.userInfo || null;
  tokenExpiresAt = s.tokenExpiresAt || 0;
  accessToken = s.accessToken || null;

  // Restaurar rol (o derivarlo del email si la sesión es anterior al sistema de roles)
  currentUserRole = s.currentUserRole || null;
  if (!currentUserRole && userInfo) {
    currentUserRole = verificarAcceso(userInfo.email);
  }
  // Si el email no está autorizado, borrar sesión
  if (!currentUserRole) {
    limpiarSesion();
    return false;
  }

  // Si el token de Google sigue vigente: úsalo
  if (accessToken && Date.now() < tokenExpiresAt) {
    if (userInfo) actualizarUIUsuario(userInfo);
    guardarSesion(); // refresca lastActivity
    showApp();
    return true;
  }

  // Token expirado pero el usuario sigue activo: refresh silencioso
  if (tokenClient) {
    silentReauthPending = true;
    try {
      tokenClient.requestAccessToken({ prompt: '' });
      // Mientras llega el callback mostramos loading
      document.getElementById('loading').style.display = 'flex';
      document.getElementById('auth-screen').style.display = 'none';
      return true;
    } catch (e) {
      silentReauthPending = false;
      limpiarSesion();
      return false;
    }
  }
  return false;
}

function forzarLogout(motivo) {
  limpiarSesion();
  detenerTemporizadorInactividad();
  document.getElementById('app-root').style.display = 'none';
  document.getElementById('loading').style.display = 'none';
  document.getElementById('auth-screen').style.display = 'flex';
  if (motivo) {
    const errEl = document.getElementById('auth-error');
    errEl.textContent = motivo;
    errEl.style.display = 'block';
  }
}

function cerrarSesion() {
  if (!confirm('¿Cerrar sesión?')) return;
  // Revocar el token en Google (best effort)
  if (accessToken && window.google?.accounts?.oauth2?.revoke) {
    try { google.accounts.oauth2.revoke(accessToken, () => {}); } catch(e){}
  }
  forzarLogout(null);
}

function registrarActividad() {
  if (!accessToken) return;
  guardarSesion(); // actualiza lastActivity
  reiniciarTemporizadorInactividad();
}

function reiniciarTemporizadorInactividad() {
  detenerTemporizadorInactividad();
  inactivityTimerId = setTimeout(() => {
    forzarLogout('Tu sesión se cerró por inactividad. Inicia sesión nuevamente.');
  }, INACTIVITY_TIMEOUT_MS);
}

function detenerTemporizadorInactividad() {
  if (inactivityTimerId) { clearTimeout(inactivityTimerId); inactivityTimerId = null; }
}

function instalarListenersActividad() {
  // Registrar actividad ante interacción del usuario (throttle 30s)
  let lastSave = 0;
  const handler = () => {
    const now = Date.now();
    if (now - lastSave > 30000) { lastSave = now; registrarActividad(); }
  };
  ['click','keydown','mousemove','scroll','touchstart'].forEach(ev => {
    document.addEventListener(ev, handler, { passive: true });
  });
  // Cuando la pestaña vuelve a estar visible, refrescamos lastActivity
  document.addEventListener('visibilitychange', () => {
    if (!document.hidden) registrarActividad();
  });
}

function showApp() {
  document.getElementById('loading').style.display = 'none';
  document.getElementById('auth-screen').style.display = 'none';
  document.getElementById('app-root').style.display = 'flex';
  const today = new Date();
  document.getElementById('topbar-date').textContent = today.toLocaleDateString('es-CO', {
    weekday: 'long', day: 'numeric', month: 'long'
  });
  const hoyISO = today.toISOString().split('T')[0];
  (APP.CAMPOS_FECHA_HOY || []).forEach(id => {
    const el = document.getElementById(id);
    if (el && !el.value) el.value = hoyISO;
  });
  reiniciarTemporizadorInactividad();
  restaurarSidebar();
  if (typeof APP.alIniciar === 'function') APP.alIniciar();
}

function showAuth() {
  document.getElementById('loading').style.display = 'none';
  document.getElementById('auth-screen').style.display = 'flex';
}

// ═══════════════════════════════════════════════════════════
// SHEETS API
// ═══════════════════════════════════════════════════════════
async function asegurarTokenVigente() {
  if (accessToken && Date.now() < tokenExpiresAt) return true;
  if (!tokenClient) return false;
  // Refresh silencioso
  return new Promise(resolve => {
    silentReauthPending = true;
    const prev = tokenClient.callback;
    tokenClient.callback = (resp) => {
      tokenClient.callback = prev; // restaurar callback original
      if (resp.error || !resp.access_token) {
        silentReauthPending = false;
        forzarLogout('Tu sesión expiró. Inicia sesión nuevamente.');
        resolve(false);
        return;
      }
      accessToken = resp.access_token;
      const expiresInSec = parseInt(resp.expires_in, 10) || 3600;
      tokenExpiresAt = Date.now() + (expiresInSec - 60) * 1000;
      silentReauthPending = false;
      guardarSesion();
      resolve(true);
    };
    try { tokenClient.requestAccessToken({ prompt: '' }); }
    catch (e) { tokenClient.callback = prev; silentReauthPending = false; resolve(false); }
  });
}

// Re-pide token sin interrumpir al usuario (prompt vacio) (#8 auditoria)
function pedirTokenSilencioso() {
  return new Promise((resolve) => {
    if (!tokenClient) return resolve(false);
    const prev = tokenClient.callback;
    tokenClient.callback = (resp) => {
      tokenClient.callback = prev;
      if (resp.error || !resp.access_token) return resolve(false);
      accessToken = resp.access_token;
      const expiresInSec = parseInt(resp.expires_in, 10) || 3600;
      tokenExpiresAt = Date.now() + (expiresInSec - 60) * 1000;
      guardarSesion();
      resolve(true);
    };
    try { tokenClient.requestAccessToken({ prompt: '' }); }
    catch (e) { tokenClient.callback = prev; resolve(false); }
  });
}

// fetch con Authorization automatico. Ante 401 renueva token y reintenta UNA vez,
// para que un token vencido a media jornada no rompa el guardado (#8 auditoria).
async function apiFetch(url, opts = {}, reintento = true) {
  await asegurarTokenVigente();
  opts.headers = { ...(opts.headers || {}), Authorization: 'Bearer ' + accessToken };
  let r = await fetch(url, opts);
  if (r.status === 401 && reintento) {
    const ok = await pedirTokenSilencioso();
    if (ok) {
      opts.headers.Authorization = 'Bearer ' + accessToken;
      return apiFetch(url, opts, false);
    }
  }
  return r;
}

async function sheetsGet(range) {
  const r = await apiFetch(
    `https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${encodeURIComponent(range)}`, {}
  );
  if (r.status === 401) { forzarLogout('Tu sesión expiró. Inicia sesión nuevamente.'); throw new Error('401'); }
  if (!r.ok) throw new Error('Error leyendo Sheets: ' + r.status);
  return r.json();
}

async function sheetsAppend(range, values) {
  const r = await apiFetch(
    `https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${encodeURIComponent(range)}:append?valueInputOption=USER_ENTERED&insertDataOption=INSERT_ROWS`,
    { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ values }) }
  );
  if (r.status === 401) { forzarLogout('Tu sesión expiró. Inicia sesión nuevamente.'); throw new Error('401'); }
  if (!r.ok) throw new Error('Error escribiendo Sheets: ' + r.status);
  return r.json();
}

async function sheetsUpdate(range, values) {
  const r = await apiFetch(
    `https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${encodeURIComponent(range)}?valueInputOption=USER_ENTERED`,
    { method: 'PUT', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ values }) }
  );
  if (r.status === 401) { forzarLogout('Tu sesión expiró. Inicia sesión nuevamente.'); throw new Error('401'); }
  if (!r.ok) throw new Error('Error actualizando Sheets: ' + r.status);
  return r.json();
}

async function sheetsFindRow(sheet, colIdx, value) {
  const data = await sheetsGet(`${sheet}!A:A`);
  const rows = data.values || [];
  for (let i = 1; i < rows.length; i++) {
    if (String(rows[i][colIdx] || '').trim() === String(value).trim()) return i + 1;
  }
  return -1;
}


// ----------------------------------------------------------------------
// Secciones colapsables y barra lateral
// ----------------------------------------------------------------------
// ═══════════════════════════════════════════════════════════
// SECCIONES COLAPSABLES
// ═══════════════════════════════════════════════════════════
const SIDEBAR_KEY = 'peniel_sidebar_contraida';

function toggleSidebar(forzar) {
  const app = document.getElementById('app-root');
  const btn = document.getElementById('btn-sidebar');
  const contraida = (forzar !== undefined) ? forzar : !app.classList.contains('sidebar-collapsed');
  app.classList.toggle('sidebar-collapsed', contraida);
  if (btn) {
    btn.setAttribute('aria-expanded', String(!contraida));
    btn.title = contraida ? 'Expandir el menú' : 'Contraer el menú';
  }
  // El tooltip solo aporta cuando no se ve el texto del item
  document.querySelectorAll('.nav-item').forEach(item => {
    if (contraida) item.title = item.querySelector('.nav-label')?.textContent.trim() || '';
    else item.removeAttribute('title');
  });
  try { localStorage.setItem(SIDEBAR_KEY, contraida ? '1' : '0'); } catch (e) {}
  // Las graficas del dashboard deben recalcular su ancho al terminar la animacion
  setTimeout(() => {
    Object.values(window.dashCharts || {}).forEach(c => { try { c.resize(); } catch (e) {} });
  }, 380);
}

function restaurarSidebar() {
  let contraida = false;
  try { contraida = localStorage.getItem(SIDEBAR_KEY) === '1'; } catch (e) {}
  if (contraida) toggleSidebar(true);
}

function toggleSection(headerEl) {
  headerEl.closest('.card').classList.toggle('collapsed');
}

// ----------------------------------------------------------------------
// Orden de tablas por encabezado
// ----------------------------------------------------------------------
// ═══════════════════════════════════════════════════════════
// ORDENAR TABLAS DESDE EL ENCABEZADO
// Un clic ordena de menor a mayor (o A→Z), otro clic invierte.
// Sin menús: solo una flecha en el encabezado activo.
// ═══════════════════════════════════════════════════════════
const ordenTablas = {};   // { clave: { campo, dir } }

function ordenarPorColumna(clave, campo, render) {
  const act = ordenTablas[clave];
  if (act && act.campo === campo) act.dir = act.dir === 'asc' ? 'desc' : 'asc';
  else ordenTablas[clave] = { campo, dir: 'asc' };
  render();
}

function limpiarOrdenColumna(clave) { delete ordenTablas[clave]; }

// getters: { campo: fila => valor }. Devuelve una copia ordenada.
function aplicarOrden(lista, clave, getters) {
  const o = ordenTablas[clave];
  if (!o || !getters[o.campo]) return lista;
  const get = getters[o.campo];
  const signo = o.dir === 'asc' ? 1 : -1;
  return lista.slice().sort((a, b) => {
    const va = get(a), vb = get(b);
    const na = typeof va === 'number', nb = typeof vb === 'number';
    if (na && nb) return (va - vb) * signo;
    return String(va ?? '').localeCompare(String(vb ?? ''), 'es', { numeric: true }) * signo;
  });
}

// Prepara los encabezados marcados con data-orden y refresca las flechas
function prepararOrden(selectorTabla, clave, render) {
  document.querySelectorAll(selectorTabla + ' thead th[data-orden]').forEach(th => {
    if (!th._ordenListo) {
      th._ordenListo = true;
      th.classList.add('th-ordenable');
      th.insertAdjacentHTML('beforeend', '<span class="sort-arrow"></span>');
      th.addEventListener('click', () => ordenarPorColumna(clave, th.dataset.orden, render));
    }
    const o = ordenTablas[clave];
    const activa = !!(o && o.campo === th.dataset.orden);
    th.classList.toggle('ordenada', activa);
    th.setAttribute('aria-sort', activa ? (o.dir === 'asc' ? 'ascending' : 'descending') : 'none');
    const flecha = th.querySelector('.sort-arrow');
    if (flecha) flecha.textContent = activa ? (o.dir === 'asc' ? '▲' : '▼') : '⇅';
  });
}

// ----------------------------------------------------------------------
// Detalle maestro-detalle y borrado de filas
// ----------------------------------------------------------------------
async function reemplazarDetalle(hoja, cols, idPedido, filas, now) {
  const data = await sheetsGet(`${hoja}!B:B`);   // columna B = id_pedido
  const rows = data.values || [];
  const nums = [];
  for (let i = 1; i < rows.length; i++) {
    if (String(rows[i][0] || '').trim() === idPedido) nums.push(i + 1);
  }
  if (nums.length) {
    const sid = await obtenerSheetId(hoja);
    await sheetsBatchUpdate(nums.sort((a, b) => b - a).map(row => ({
      deleteDimension: { range: { sheetId: sid, dimension: 'ROWS', startIndex: row - 1, endIndex: row } }
    })));
  }
  if (filas.length) {
    await sheetsAppend(`${hoja}!A:${colLetra(cols.length - 1)}`, filas.map(f => objToRow(f, cols)));
  }
}

async function obtenerSheetId(sheetName) {
  if (sheetIds[sheetName] !== undefined) return sheetIds[sheetName];
  const r = await apiFetch(
    `https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}?fields=sheets.properties`, {}
  );
  const data = await r.json();
  (data.sheets || []).forEach(sh => { sheetIds[sh.properties.title] = sh.properties.sheetId; });
  return sheetIds[sheetName];
}

// Alias con nombre honesto: este endpoint sirve para cualquier operación
// estructural, no solo para borrar filas.
async function sheetsBatchUpdate(requests) { return eliminarFilasBatch(requests); }

async function eliminarFilasBatch(requests) {
  const r = await apiFetch(
    `https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}:batchUpdate`,
    { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ requests }) }
  );
  if (!r.ok) throw new Error('Error al eliminar filas: ' + r.status);
  return r.json();
}


// ----------------------------------------------------------------------
// Renumerar filas repetibles
// ----------------------------------------------------------------------
function renumberContainer(containerId) {
  document.querySelectorAll('#' + containerId + ' .cinta-item').forEach((item, idx) => {
    const num = item.querySelector('.cinta-num');
    if (num) num.textContent = idx + 1;
  });
}

// ----------------------------------------------------------------------
// Multi-select
// ----------------------------------------------------------------------
// ═══════════════════════════════════════════════════════════
// MULTI-SELECT COMPONENT
// ═══════════════════════════════════════════════════════════
let _msOpenPanel = null;
document.addEventListener('click', () => {
  if (_msOpenPanel) {
    _msOpenPanel.style.display = 'none';
    const btn = _msOpenPanel.previousElementSibling;
    if (btn) btn.classList.remove('ms-active');
    _msOpenPanel = null;
  }
});

function buildMultiSelect(placeholder, options, selected, onChange) {
  const wrap = document.createElement('div');
  wrap.className = 'ms-wrap';
  wrap.addEventListener('click', e => e.stopPropagation());

  const btn = document.createElement('button');
  btn.type = 'button';
  btn.className = 'ms-btn';

  const panel = document.createElement('div');
  panel.className = 'ms-panel';

  function refreshBtn() {
    btn.innerHTML = selected.length
      ? `${placeholder} <span class="ms-count">${selected.length}</span>`
      : `${placeholder} <span class="ms-arrow">▾</span>`;
  }

  function buildOptions(opts) {
    panel.innerHTML = '';

    // "Todos" item
    const allLabel = document.createElement('label');
    allLabel.className = 'ms-item';
    const allChk = document.createElement('input');
    allChk.type = 'checkbox';
    allChk.checked = selected.length === 0;
    allLabel.appendChild(allChk);
    allLabel.appendChild(document.createTextNode(' Todos'));
    panel.appendChild(allLabel);

    const sep = document.createElement('hr');
    sep.className = 'ms-separator';
    panel.appendChild(sep);

    const itemEls = opts.map(opt => {
      const lbl = document.createElement('label');
      lbl.className = 'ms-item';
      const chk = document.createElement('input');
      chk.type = 'checkbox';
      chk.value = opt;
      chk.checked = selected.includes(opt);
      chk.addEventListener('change', () => {
        if (chk.checked) { if (!selected.includes(opt)) selected.push(opt); }
        else { const i = selected.indexOf(opt); if (i > -1) selected.splice(i, 1); }
        allChk.checked = selected.length === 0;
        refreshBtn();
        onChange();
      });
      lbl.appendChild(chk);
      lbl.appendChild(document.createTextNode(' ' + opt));
      panel.appendChild(lbl);
      return { chk, opt };
    });

    allChk.addEventListener('change', () => {
      selected.length = 0;
      itemEls.forEach(({chk}) => chk.checked = false);
      refreshBtn();
      onChange();
    });
  }

  buildOptions(options);

  btn.addEventListener('click', () => {
    const isOpen = panel.style.display === 'block';
    if (_msOpenPanel && _msOpenPanel !== panel) {
      _msOpenPanel.style.display = 'none';
      const prevBtn = _msOpenPanel.previousElementSibling;
      if (prevBtn) prevBtn.classList.remove('ms-active');
    }
    panel.style.display = isOpen ? 'none' : 'block';
    btn.classList.toggle('ms-active', !isOpen);
    _msOpenPanel = isOpen ? null : panel;
  });

  // Expose update method to refresh options without losing open state
  wrap.updateOptions = (newOpts) => {
    buildOptions(newOpts);
    refreshBtn();
  };

  refreshBtn();
  wrap.appendChild(btn);
  wrap.appendChild(panel);
  return wrap;
}

// ----------------------------------------------------------------------
// Utilidades varias
// ----------------------------------------------------------------------
// ═══════════════════════════════════════════════════════════
// UTILS
// ═══════════════════════════════════════════════════════════
function fmtFecha(d) {
  if (!d) return '—';
  try {
    return new Date(d + 'T12:00:00').toLocaleDateString('es-CO', { day:'2-digit', month:'2-digit', year:'numeric' });
  } catch { return d; }
}

function toast(msg, type = 'success') {
  const el = document.getElementById('toast');
  el.textContent = msg;
  el.className = `toast toast-${type} show`;
  setTimeout(() => el.classList.remove('show'), 3000);
}

// ----------------------------------------------------------------------
// Texto en mayusculas al escribir
// ----------------------------------------------------------------------
// ═══════════════════════════════════════════════════════════
// INIT
// ═══════════════════════════════════════════════════════════
document.addEventListener('input', function(e) {
  const el = e.target;
  if (el.tagName === 'INPUT' && el.type !== 'date' && el.type !== 'number' && !el.readOnly) {
    const pos = el.selectionStart;
    el.value = el.value.toUpperCase();
    el.setSelectionRange(pos, pos);
  }
});


// ----------------------------------------------------------------------
// Arranque: esperar a Google y restaurar la sesion
// ----------------------------------------------------------------------
window.onload = () => {
  instalarListenersActividad();
  const waitForGoogle = setInterval(() => {
    if (window.google && google.accounts && google.accounts.oauth2) {
      clearInterval(waitForGoogle);
      initTokenClient();
      const restaurada = intentarRestaurarSesion();
      if (!restaurada) {
        document.getElementById('loading').style.display = 'none';
        document.getElementById('auth-screen').style.display = 'flex';
      }
    }
  }, 100);
  setTimeout(() => {
    clearInterval(waitForGoogle);
    if (!accessToken && document.getElementById('app-root').style.display === 'none') {
      document.getElementById('loading').style.display = 'none';
      document.getElementById('auth-screen').style.display = 'flex';
      if (!tokenClient) {
        document.getElementById('auth-error').textContent = 'No se pudo cargar Google. Verifica tu conexión y recarga.';
        document.getElementById('auth-error').style.display = 'block';
      }
    }
  }, 5000);
};

// ────────────────────────────────────────────────────────────────────────
// Navegacion entre vistas
// ────────────────────────────────────────────────────────────────────────
// Generica: los titulos y lo que hay que pintar los pone cada pagina.
function showView(v) {
  document.querySelectorAll('.view').forEach(el => el.classList.remove('active'));
  document.querySelectorAll('.nav-item').forEach(el => el.classList.remove('active'));
  document.getElementById('view-' + v)?.classList.add('active');
  document.getElementById('nav-' + v)?.classList.add('active');
  const titulo = document.getElementById('topbar-title');
  if (titulo) titulo.textContent = (APP.TITULOS || {})[v] || v;
  if (typeof APP.alCambiarVista === 'function') APP.alCambiarVista(v);
}
