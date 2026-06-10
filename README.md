# Shimano App Vendedores — Documentación técnica completa

App web para el equipo comercial de **Shimano Argentina** durante la transición de Baraldo (distribuidor histórico) a venta directa. Cubre todo el ciclo: gestión territorial por zona, alta de clientes, visitas con GPS, armado de pedidos, rendiciones de gastos con OCR de tickets, rutas optimizadas, tareas entre usuarios e integración con SAP B1 vía DTW.

| | |
|---|---|
| **URL pública** | https://shimano-arg.github.io/app-vendedores/ |
| **Repo** | https://github.com/shimano-arg/app-vendedores |
| **Firebase project** | `app-vendedores-shimano` |
| **Admin bootstrap** | `bot.shimano.pesca@gmail.com` (auto-elevación al primer login) |
| **Admin backup** | `erbinomariano@gmail.com` (Mariano Erbino) |
| **Stack** | HTML5 + Vanilla JS + Firebase Firestore + Gemini API (OCR) |
| **Build pipeline** | Python (openpyxl) genera el HTML autosuficiente desde Excels master |

---

## Índice

1. [Resumen ejecutivo](#1-resumen-ejecutivo)
2. [Stack técnico](#2-stack-técnico)
3. [Estructura del repo](#3-estructura-del-repo)
4. [Pipeline de build](#4-pipeline-de-build)
5. [Sistema de autenticación y autorización](#5-sistema-de-autenticación-y-autorización)
6. [Roles y permisos](#6-roles-y-permisos)
7. [Modelo de datos Firestore](#7-modelo-de-datos-firestore)
8. [Firestore Security Rules](#8-firestore-security-rules)
9. [Estructura de la UI](#9-estructura-de-la-ui)
10. [Sección: Localidades / Clientes / Pedidos](#10-sección-localidades--clientes--pedidos)
11. [Sección: Rutas](#11-sección-rutas)
12. [Sección: Visita](#12-sección-visita)
13. [Sección: Dashboard](#13-sección-dashboard)
14. [Sección: Rendiciones](#14-sección-rendiciones)
15. [Sección: Alta Clientes](#15-sección-alta-clientes)
16. [Sección: Notificaciones (Alertas y tareas)](#16-sección-notificaciones-alertas-y-tareas)
17. [Sistema VDE-VDI (vendedor externo / interno)](#17-sistema-vde-vdi-vendedor-externo--interno)
18. [Campañas comerciales](#18-campañas-comerciales)
19. [Targets mensuales](#19-targets-mensuales)
20. [Panel Master Clientes (direcciones)](#20-panel-master-clientes-direcciones)
21. [Integración SAP B1](#21-integración-sap-b1)
22. [OCR de tickets con Gemini API](#22-ocr-de-tickets-con-gemini-api)
23. [PWA installable](#23-pwa-installable)
24. [Backup mensual](#24-backup-mensual)
25. [Exports a Excel / Power BI / ML](#25-exports-a-excel--power-bi--ml)
26. [Panel admin "Usuarios"](#26-panel-admin-usuarios)
27. [URLs externas e integraciones](#27-urls-externas-e-integraciones)
28. [Convenciones de código](#28-convenciones-de-código)
29. [Regenerar y deployar](#29-regenerar-y-deployar)
30. [Troubleshooting](#30-troubleshooting)
31. [Roadmap / pendientes](#31-roadmap--pendientes)

---

## 1) Resumen ejecutivo

### Para qué sirve

Shimano Argentina necesita gestionar la operación de 6 vendedores externos que recorren tiendas de pesca en todo el país después de la salida del distribuidor histórico (Baraldo). La app cubre:

- **Mapa interactivo** de Argentina con 24 provincias + 527 departamentos pintados por zona de vendedor.
- **941 tiendas** pre-cargadas (master + prospectos) con sistema de habilitación.
- **665 SKUs** del master de productos (pesca).
- **6 zonas** (Z1 Gonzalo, Z2 Federico, Z4 Martín, Z5 Mauricio, Z6 Ioannis, Z7 Santiago).
- **Rutas mensuales** auto-generadas por proximidad (10-15 tiendas por ruta).
- **Visitas a tiendas** con formulario completo + foto + GPS doble-check.
- **Pedidos** confirmados → exportables como ZIP DTW para SAP B1.
- **Rendiciones de gastos** con OCR de tickets (Gemini) y aprobación por gerente.
- **Alta de clientes nuevos** con doble aprobación + link público para que el cliente se cargue solo.
- **Notificaciones y tareas** entre usuarios con imágenes.
- **Dashboard** comparativo del equipo + por vendedor individual.
- **Campañas comerciales** vigentes con tracking de SKUs.
- **Targets mensuales en ARS** cargados por gerente.
- **PWA installable** en celular como app nativa.
- **Backup mensual** con eliminación automática de fotos viejas para no inflar Firestore.

### Filosofía de diseño

- **App estática en GitHub Pages** (sin backend propio). Toda la lógica corre en el navegador.
- **Firebase Firestore** como backend real (auth + DB).
- **Sin framework** (JS vanilla): un solo `index.html` ~2.9 MB con todo embebido.
- **Build offline en Python** genera el HTML desde Excels master cuando hay que actualizar datos estáticos.
- **Datos vivos** (pedidos, visitas, usuarios) viven 100% en Firestore.
- **Tiempo real** via `onSnapshot` listeners.
- **Offline-friendly**: persistencia IndexedDB activada en Firestore.

---

## 2) Stack técnico

| Capa | Tecnología | Versión / detalle |
|---|---|---|
| Frontend | HTML5 + CSS3 + Vanilla JavaScript | — |
| Mapa | Leaflet | 1.9.4 (CDN unpkg) |
| Tiles | CartoDB Positron | OSM stack |
| Excel | SheetJS (xlsx) | 0.18.5 (CDN jsdelivr) |
| Excel con fotos embebidas | ExcelJS | 4.4.0 (lazy load, solo al exportar) |
| ZIP | JSZip | 3.10.1 (CDN cloudflare) |
| Auth + DB | Firebase compat SDK | 10.7.1 (auth + firestore) |
| OCR | Google Gemini API | `gemini-2.5-flash` (REST) |
| Hosting | GitHub Pages | rama `main` |
| Build offline | Python 3 + openpyxl | genera HTML desde Excels |
| Storage local | localStorage + IndexedDB | persistencia y cache |

**Sin Node, sin Webpack, sin TypeScript, sin React.** Una elección deliberada para minimizar fricciones de mantenimiento: cualquier persona con conocimientos de JS y HTML puede editar el código.

---

## 3) Estructura del repo

```
shimano-arg/app-vendedores/
├── index.html                # App completa (~2.9 MB - todo embebido)
├── alta-cliente.html         # Formulario público standalone (link compartible)
├── manifest.json             # PWA manifest
├── sw.js                     # Service Worker
├── Shimano-Logo.png          # Logo (header + splash)
├── icon-180-v3.png           # PWA icon iOS 180×180
├── icon-192-v3.png           # PWA icon Android 192×192
├── icon-512-v3.png           # PWA icon 512×512 (any)
├── icon-512-maskable-v3.png  # PWA icon 512×512 (maskable Android adaptive)
└── README.md                 # Este archivo
```

El código fuente que genera `index.html` está en otra carpeta del disco del mantenedor (no en el repo público):

```
C:\Users\shimano.sandbox\Desktop\MASTERFILES\PROSPECTOS\MAPAS\
└── _build_argentina_zonas_v2.py    # Python ~12.000 líneas
                                    # Output: Mapa_Argentina_Shimano_Zonas.html
                                    # Se copia a APP VENDEDORES/index.html
```

El script de build:
1. Lee 25 archivos `Mapa_<Provincia>_Shimano.html` con polígonos pre-procesados.
2. Lee `MASTERFILES/ZONAS/TARGETS VENDEDORES-ZONAS.xlsx` para clientes/prospectos + targets USD.
3. Lee `MASTERFILES/PRODUCTO/MASTERFILE PRODUCTOS PESCA.xlsx` (665 SKUs).
4. Lee `FORECAST/DATOS_CRUDOS/Masterfile Shimano Venta Ult 365 Días.xlsx` (~12.700 órdenes MELI).
5. Embebe todo como constantes JSON dentro del template HTML.
6. Output: un `.html` autosuficiente que se sirve estático.

---

## 4) Pipeline de build

### Cuándo regenerar
- Cambian los masters (productos, tiendas, vendedores, targets).
- Se modifica el código fuente del template (que vive dentro del Python script).

### Comando de build + deploy
```powershell
cd "C:\Users\shimano.sandbox\Desktop\MASTERFILES\PROSPECTOS\MAPAS"
python _build_argentina_zonas_v2.py
Copy-Item Mapa_Argentina_Shimano_Zonas.html "C:\Users\shimano.sandbox\Desktop\APP VENDEDORES\index.html"
cd "C:\Users\shimano.sandbox\Desktop\APP VENDEDORES"
git add index.html
git commit -m "data refresh"
git push
```

GitHub Pages propaga en 30-90 segundos. Los datos en Firestore **NO se tocan**: solo se reemplaza el HTML estático.

### Cuándo no regenerar
Cualquier cambio de **datos dinámicos** (pedidos, visitas, rutas, rendiciones, targets, campañas, etc.) **NO requiere regenerar el HTML**: se modifica desde la propia app y persiste en Firestore.

---

## 5) Sistema de autenticación y autorización

### Login
- Solo **Google OAuth** (firebase-auth-compat).
- Botón único "Continuar con Google" → popup nativo de Google.
- Forzado `prompt: 'select_account'` para mostrar el selector siempre.

### Whitelist de emails (4 fuentes de aprobación)

Un usuario tiene acceso si cumple **al menos UNA** de estas condiciones:

1. **Email en hardcoded `ALLOWED_EMAILS`** (admins/bots históricos):
   - `bot.shimano.pesca@gmail.com`
   - `erbinomariano@gmail.com`
   - `srb90284@gmail.com`
   - `quilgym@gmail.com`
2. **Dominio en hardcoded `ALLOWED_EMAIL_DOMAINS`**:
   - `shimano.com.ar`
   - `shimano-arg.com.ar`
3. **Ya tiene rol asignado** en `/roles/{uid}` distinto de `unassigned` (admin ya lo aprobó previamente).
4. **Email cargado en collection `allowed_emails`** (gestionable desde el Panel Usuarios).

Si NO cumple ninguna, se hace `signOut()` inmediato antes de crear el doc en `/roles`, evitando que la collection se ensucie con cuentas no autorizadas.

### Bootstrap del admin
La primera vez que `bot.shimano.pesca@gmail.com` loguea, la app le asigna `role='admin'` automáticamente (`fetchAndApplyRole`).

### Login fallido
Si la cuenta no pasa el whitelist, se muestra en el splash de auth:
> El email `X@gmail.com` no está autorizado. Solo se permite iniciar sesión con una cuenta @shimano.com.ar o un email previamente autorizado por el admin.

---

## 6) Roles y permisos

| Rol | Quiénes | Capacidades |
|---|---|---|
| `admin` | bot.shimano.pesca + Mariano Erbino | Todo: gestión de roles, campañas, targets, panel SAP, master clientes, integración SAP, zona de peligro, backup mensual, todos los exports, dashboard global. |
| `vendedor` | 6 vendedores externos (Z1-Z7 sin Z3) | Solo su zona, sus pedidos, sus visitas, sus rendiciones. Puede armar pedidos, cargar visitas, cargar rendiciones, ver dashboard de su zona, generar rutas, derivar tiendas al VDI, exportar su ZIP DTW. |
| `interno` (VDI) | Vendedor interno de oficina | Recibe derivaciones de tiendas que el externo encontró cerradas. Carga visitas remotas (sin foto del frente). Ve sus propias notificaciones. Puede aprobar/rechazar solicitudes de alta de clientes. |
| `gerente` | Gerente de ventas | Carga targets mensuales. Aprueba/rechaza rendiciones de los vendedores que lo tengan como responsable. Ve dashboard consolidado. |
| `viewer` | Solo lectura (analistas) | Lee todo (pedidos, visitas, dashboard) pero no escribe. Acceso al panel SAP en modo lectura. |
| `unassigned` | Default al primer login | Sin acceso. Ve pantalla "Acceso pendiente — tu usuario aún no tiene rol asignado". |

### Sub-asignaciones por vendedor (configurables en Panel Usuarios)

- **`vendor`** (string, solo para `role=vendedor`): clave de zona (`GONZALO DE LA ROSA`, `FEDERICO CASTELANELLI`, etc.).
- **`internalPartnerUid`**: UID del vendedor interno (VDI) que es su pareja para derivaciones de tiendas cerradas.
- **`whatsapp`**: número de WhatsApp del usuario en formato `wa.me` (5491126762031). Se usa al enviar ruta por WhatsApp — el mensaje le llega al propio número del vendedor.
- **`rendicionesApproverUid`**: UID de quien aprueba sus rendiciones (gerente / interno / admin). Si no está asignado, la rendición no se puede enviar.

---

## 7) Modelo de datos Firestore

Listado exhaustivo de las collections y la forma de los docs. La numeración refleja el orden en que aparecen en las rules.

### `roles/{uid}` — Rol y configuración por usuario
```js
{
  email: string,
  displayName: string,
  role: 'admin' | 'vendedor' | 'interno' | 'gerente' | 'viewer' | 'unassigned',
  vendor: string | null,                    // vendor key cuando role=vendedor
  internalPartnerUid: string | null,        // uid del VDI pareja
  whatsapp: string | null,                  // formato wa.me sin + ni espacios
  rendicionesApproverUid: string | null,    // uid de quien aprueba sus rendiciones
  assignedBy: string,
  assignedAt: serverTimestamp,
  lastLogin: serverTimestamp
}
```

### `userData/{uid}` — Estado local + drafts del vendedor
```js
{
  email: string,
  displayName: string,
  contacted: Set<string>,                   // claves de clientes "habilitados"
  canceled: Set<string>,                    // claves de clientes "cancelados"
  orders: { [key]: lines[] },               // pedidos borrador (no enviados)
  lastSeen: serverTimestamp
}
```

`key` para un cliente: `<tipo>|<provincia>|<localidad>|<nombre>` (ej. `C|BUENOS AIRES|San Pedro|MARIANO-PESCA`). Tipo `C` = cliente master, `P` = prospecto.

### `pedidos/{auto_id}` — Pedidos confirmados
```js
{
  ownerUid: string,
  ownerEmail: string,
  stage: 'confirmed',                       // único valor en producción
  key: string,                              // misma key que userData.orders
  tipo: 'C' | 'P',
  province: string,
  locName: string,
  clientName: string,
  month: string,                            // 'JUNIO 2026' (uppercase)
  monthIdx: number,                         // 0-11
  year: number,
  confirmedAt: ISOString,
  finalizedAt: ISOString,
  lines: [
    { code: 'XYZ', desc: '...', cat: '...', fam: '...', sub: '...', qty: number, precio: number }
  ],
  transferidoSAP: {                         // se completa cuando se marca como transferido
    transferredAt: ISOString,
    transferredBy: string,
    batchId: string,                        // YYYYMMDDHHMMSS
    sapDocRange: string                     // ej. '5000123-5000147'
  } | null
}
```

### `campaigns/{auto_id}` — Campañas comerciales
```js
{
  name: string,
  familia: string,
  subfamilia: string,
  skus: string[],                           // SKUs incluidos en la campaña
  filterType: string,                       // legacy
  filterValues: string[],                   // legacy
  targetType: 'units' | 'money',
  targetAmount: number,
  startDate: 'YYYY-MM-DD',
  endDate: 'YYYY-MM-DD',
  scope: 'all' | 'province' | 'vendor',     // alcance
  scopeValues: string[],                    // provincias o vendor keys según scope
  createdBy: string,
  createdAt: serverTimestamp
}
```

### `visits/{auto_id}` — Visitas a tiendas
```js
{
  ownerUid: string,
  ownerEmail: string,
  vendor: string,
  provincia: string,
  localidad: string,
  tienda: string,
  tipo: string,                             // tipo de tienda
  local: 'FISICO' | 'ECOMMERCE',
  tamano: 'GRANDE' | 'MEDIANA' | 'CHICA',
  fidelidad: 'ALTA' | 'MEDIA' | 'BAJA',
  relevancia: 1 | 2 | 3 | 4 | 5,
  pop: 'SI' | 'NO',
  necesidadPuntual: string | null,
  espacio: string[],                        // hasta 8 fotos base64
  oportunidad: string,
  masVendido: string,
  masPreguntan: string,
  ayudaTienda: string,
  frenteLocal: string | null,               // base64 (null para internos)
  tipoVenta: 'MOSTRADO' | 'ECOMMERCE' | 'AMBOS',
  ponderacionMostrado: number | null,
  ponderacionEcommerce: number | null,
  competencia: string,
  fecha: 'YYYY-MM-DD',
  mes: 'JUNIO',                             // SIEMPRE UPPERCASE
  anio: number,
  // GPS doble-check
  gpsStatus: 'confirmed' | 'near' | 'far' | 'first' | 'denied' | 'no_reference' | 'unavailable' | 'timeout' | 'error',
  gpsLat: number | null,
  gpsLon: number | null,
  gpsAccuracy: number | null,
  gpsCapturedAt: ISOString | null,
  gpsDistanceM: number | null,
  gpsRefLat: number | null,                 // ubicación de referencia de la tienda
  gpsRefLon: number | null,
  gpsRefSource: 'auto' | null,
  gpsError: string | null,
  // Backup mensual
  photosDeletedAt: serverTimestamp | null,
  photosDeletedBy: string | null,
  createdAt: serverTimestamp
}
```

### `operations_log/{auto_id}` — Log de auditoría (inmutable)
```js
{
  action: string,                           // 'cancelar_borrador_pedido', 'PURGE_TOTAL_HISTORIAL', 'backup_mensual_run', etc.
  entityType: string,
  entityName: string,
  details: object,
  userUid: string,
  userEmail: string,
  userRole: string,
  timestamp: serverTimestamp
}
```

### `sap_clients/{docId}` — Mapping Tienda app → CardCode SAP
```js
{
  clientName: string,
  sapCode: string,                          // CardCode (ej. C123456789)
  sapName: string | null,                   // nombre como lo trae SAP
  updatedBy: string,
  updatedAt: serverTimestamp,
  source: 'manual' | 'sap_integration_v1'
}
```
`docId` = `sapNorm(clientName).replace(/[^A-Z0-9]/g, '_')`.

### `sap_products/{docId}` — Mapping SKU app → Material SAP
```js
{
  productCode: string,                      // código en la app
  sapMaterial: string,                      // ItemCode en SAP
  sapName: string | null,
  updatedBy: string,
  updatedAt: serverTimestamp,
  source: 'manual' | 'sap_integration_v1'
}
```

### `sap_vendors/{docId}` — Mapping vendedor app → SlpCode SAP
```js
{
  vendorKey: string,                        // 'GONZALO DE LA ROSA'
  slpCode: string,                          // ej. '12'
  slpName: string,                          // como lo creó SAP
  zone: string,                             // 'Z1'
  updatedBy: string,
  updatedAt: serverTimestamp,
  source: 'sap_integration_v1'
}
```

### `route_overrides/{auto_id}` — Reagendamientos y derivaciones
```js
{
  vendor: string,
  mes: string,                              // 'Junio' (capitalización normal en este caso)
  monthIdx: number,
  anio: number,
  tienda: string,
  localidad: string,
  provincia: string,
  action: 'derivada' | 'reagendada',
  // Si derivada:
  derivedToUid: string,
  derivedToEmail: string,
  // Si reagendada:
  targetDate: 'YYYYMMDD',
  createdByUid: string,
  createdByEmail: string,
  createdAt: serverTimestamp
}
```

### `notifications/{auto_id}` — Alertas + tareas + ACKs
```js
{
  type: 'derivacion' | 'task' | 'task_ack' | 'client_approval' | 'client_approval_ack' | 'rendicion_approval' | 'rendicion_approval_ack',
  fromUid: string,
  fromEmail: string,
  fromName: string,
  targetUid: string,
  targetEmail: string,
  targetName: string,                       // para tasks
  status: 'unread' | 'read' | 'done',
  createdAt: serverTimestamp,
  readAt: serverTimestamp | null,
  doneAt: serverTimestamp | null,

  // según el type:
  // derivacion:
  tienda, localidad, provincia, vendor, message,
  // task:
  title, description, images: string[],
  // task_ack:
  title, message, relatedTaskId,
  // client_approval / client_approval_ack:
  title, description | message, applicationId,
  // rendicion_approval / rendicion_approval_ack:
  title, description | message, rendicionId
}
```

### `targets/{docId}` — Targets mensuales en ARS
```js
{
  sellerId: string,                         // vendor key
  year: number,
  month: number,                            // 0-11
  targetArs: number,
  updatedBy: string,
  updatedByEmail: string,
  updatedAt: serverTimestamp
}
```
`docId` = `tgtNormKey(vendorKey)_<year>_<MM>` (ej. `GONZALO_DE_LA_ROSA_2026_06`).

### `client_locations/{docId}` — GPS preciso de cada tienda
Auto-aprendido en la primera visita con GPS válido. Después se usa para doble-check de visitas y para optimizar rutas.
```js
{
  provincia: string,
  localidad: string,
  tienda: string,
  lat: number,
  lon: number,
  accuracy: number,
  source: 'auto' | 'manual',
  setBy: string,
  setByUid: string,
  setAt: serverTimestamp
}
```
`docId` = `clientLocId(prov, loc, tienda)` = `<provNorm>__<locNorm>__<nombreNorm>`.

### `client_master/{docId}` — Direcciones exactas (Master Clientes)
```js
{
  clientName: string,
  provincia: string,
  localidad: string,
  vendor: string,
  address: string,                          // 'Av. Belgrano 123, Barrio Norte'
  updatedBy: string,
  updatedAt: serverTimestamp
}
```
Mismo `docId` que `client_locations`. Usado por el link de WhatsApp de rutas: si hay GPS preciso lo usa; sino usa `address` para geocoding; sino fallback al nombre.

### `allowed_emails/{docId}` — Whitelist gestionable por admin
```js
{
  email: string,
  note: string,                             // 'Vendedor Z3 Diego'
  addedBy: string,
  addedByUid: string,
  addedAt: serverTimestamp
}
```
`docId` = email normalizado (lowercase + `[^a-z0-9]+` → `_`).

### `client_applications/{auto_id}` — Solicitudes de alta de cliente
```js
{
  // Datos comercio:
  email, comercio, fantasia, cuit, condicionFiscal,
  calle, numero, localidad, provincia, cp,
  telefono, web, redes,
  // Contacto:
  contactoNombre, contactoTelParticular, contactoWhatsapp, contactoEmail,
  tipoComercio: 'PESCA' | 'OUTDOOR' | 'CAZA' | 'MULTIRUBRO',
  tiendaOnline: 'MERCADO LIBRE' | 'PÁGINA WEB PROPIA' | 'OTROS' | 'NO TIENE',
  // Documentos (base64):
  constanciaArca: string,
  constanciaIIBB: string,
  fotosLocal: string[],                     // hasta 5
  // Metadata:
  ownerUid: string,
  ownerEmail: string,
  ownerName: string,
  vendor: string | null,
  submittedByPublicForm: boolean,           // true si vino del form público compartido
  status: 'pending_approval' | 'approved' | 'rejected',
  approvals: { [uid]: { approvedAt, email, name } },  // hasta 2
  approvedAt: serverTimestamp | null,
  rejectedBy, rejectedByEmail, rejectedReason, rejectedAt,
  createdAt: serverTimestamp
}
```

### `rendiciones/{auto_id}` — Rendiciones y solicitudes de anticipo
```js
{
  tipo: 'solicitud' | 'gasto',
  ownerUid, ownerEmail, ownerName,
  vendor: string | null,
  approverUid: string,                      // snapshot al momento de crear
  approverEmail: string,
  status: 'pending_approval' | 'approved' | 'rejected',
  approvedBy, approvedByEmail, approvedAt,
  rejectedBy, rejectedByEmail, rejectedReason, rejectedAt,
  createdAt: serverTimestamp,

  // Si tipo='solicitud':
  solicitadoPor: string,
  motivo: string,
  tipoOperacion: 'ANTICIPO DE EFECTIVO' | 'RENDICION DE GASTO' | 'RECARGA',
  importe: number,
  moneda: 'PESOS ARGENTINOS' | 'DOLARES' | 'OTRAS MONEDAS',
  observaciones: string,
  estadoSolicitud: 'ABIERTO' | 'CERRADO',
  adjunto: { name, type, data } | null,     // base64 (Excel/PDF/imagen)

  // Si tipo='gasto':
  numeroTicket: string,
  descripcion: 'COMBUSTIBLE' | 'COMIDA' | 'HOSPEDAJE' | 'PEAJE' | 'TRASLADO' | 'OTROS',
  modoPago: 'RECARGABLE' | 'CORPORATIVA' | 'EFECTIVO',
  moneda: 'PESOS' | 'DOLARES' | 'OTRAS MONEDAS',
  tipoGasto: 'GASTO CON COMPROBANTE' | 'GASTO SIN COMPROBANTE' | 'FACTURA A',
  importe: number,
  importeUsd: number,
  divisionGasto: 'GASTO LOCAL' | 'GASTO REGIONAL',
  observaciones: string,
  fotoTicket: string                        // base64 jpeg
}
```

### `app_config/{docId}` — Configuración sensible
```js
// app_config/gemini
{
  apiKey: string,                           // hardcoded en Firestore (NO en repo)
  updatedBy, updatedByUid, updatedAt
}
```

---

## 8) Firestore Security Rules

Rules completas (publicar en Firebase Console → Firestore → Rules):

```javascript
rules_version = "2";
service cloud.firestore {
  match /databases/{database}/documents {

    function role() {
      return get(/databases/$(database)/documents/roles/$(request.auth.uid)).data.role;
    }
    function isAdmin()   { return request.auth != null && role() == 'admin'; }
    function isViewer()  { return request.auth != null && role() == 'viewer'; }
    function isVendor()  { return request.auth != null && role() == 'vendedor'; }
    function isInterno() { return request.auth != null && role() == 'interno'; }
    function isGerente() { return request.auth != null && role() == 'gerente'; }
    function isReader()  { return isAdmin() || isViewer() || isVendor() || isInterno() || isGerente(); }

    match /roles/{uid} {
      allow read: if request.auth != null && (request.auth.uid == uid || isAdmin());
      allow create: if request.auth != null && request.auth.uid == uid
                     && request.resource.data.role in ['unassigned', 'admin'];
      allow update, delete: if isAdmin();
    }

    match /userData/{uid} {
      allow read: if request.auth != null && (request.auth.uid == uid || isAdmin() || isViewer());
      allow write: if request.auth != null && (request.auth.uid == uid || isAdmin());
    }

    match /pedidos/{pedidoId} {
      allow read: if isReader();
      allow create: if isAdmin() || (isVendor() && request.resource.data.ownerUid == request.auth.uid);
      allow update, delete: if isAdmin() || (isVendor() && resource.data.ownerUid == request.auth.uid);
    }

    match /campaigns/{campaignId} {
      allow read: if isReader();
      allow create, update, delete: if isAdmin();
    }

    match /visits/{visitId} {
      allow read: if isReader();
      allow create: if (isAdmin() || isVendor() || isInterno())
                     && request.resource.data.ownerUid == request.auth.uid;
      allow update, delete: if isAdmin()
                     || ((isVendor() || isInterno()) && resource.data.ownerUid == request.auth.uid);
    }

    match /operations_log/{logId} {
      allow read: if isAdmin() || isViewer();
      allow create: if request.auth != null && request.resource.data.userUid == request.auth.uid;
      allow update, delete: if false;
    }

    match /sap_clients/{docId}  { allow read: if isReader(); allow write: if isAdmin(); }
    match /sap_products/{docId} { allow read: if isReader(); allow write: if isAdmin(); }
    match /sap_vendors/{docId}  { allow read: if isReader(); allow write: if isAdmin(); }

    match /route_overrides/{docId} {
      allow read: if isReader();
      allow create: if (isAdmin() || isVendor()) && request.resource.data.createdByUid == request.auth.uid;
      allow update, delete: if isAdmin();
    }

    match /notifications/{docId} {
      allow read: if request.auth != null && (
        resource.data.targetUid == request.auth.uid
        || resource.data.fromUid == request.auth.uid
        || isAdmin()
      );
      allow create: if request.auth != null && request.resource.data.fromUid == request.auth.uid;
      allow update: if request.auth != null && resource.data.targetUid == request.auth.uid;
      allow delete: if isAdmin();
    }

    match /targets/{docId} {
      allow read: if isReader();
      allow create, update: if isAdmin() || isGerente();
      allow delete: if isAdmin();
    }

    match /client_locations/{docId} {
      allow read: if isReader();
      allow create: if isAdmin() || isVendor() || isInterno();
      allow update, delete: if isAdmin();
    }

    match /client_master/{docId} { allow read: if isReader(); allow write: if isAdmin(); }

    match /allowed_emails/{docId} {
      // Read sin isReader (la verificacion corre ANTES de tener rol asignado)
      allow read: if request.auth != null;
      allow write: if isAdmin();
    }

    match /client_applications/{docId} {
      allow read: if isReader();
      allow create: if (request.auth != null && request.resource.data.ownerUid == request.auth.uid)
                 || (request.resource.data.submittedByPublicForm == true
                     && request.resource.data.comercio is string
                     && request.resource.data.cuit is string
                     && request.resource.data.status == 'pending_approval');
      allow update: if isAdmin() || isGerente() || isInterno();
      allow delete: if isAdmin();
    }

    match /rendiciones/{docId} {
      allow read: if isReader();
      allow create: if request.auth != null && request.resource.data.ownerUid == request.auth.uid;
      allow update: if isAdmin() || isGerente();
      allow delete: if isAdmin();
    }

    match /app_config/{docId} {
      allow read: if isReader();
      allow write: if isAdmin();
    }
  }
}
```

---

## 9) Estructura de la UI

### Header (siempre visible arriba)
- Logo Shimano + título "Market Scan - Argentina por Zonas" + badge ADMIN/VENDEDOR
- Botones admin: **TARGETS, CAMPAÑAS, SAP, MASTER CLIENTES, USUARIOS, SALIR**
- Botón **CAMPAÑAS ACTIVAS** (amarillo): muestra campañas vigentes en un modal con SKUs incluidos
- Botón **EXPORTAR PARA ANÁLISIS** (PIN 1235): formatos avanzados
- Botón **EXPORTAR A EXCEL** (verde): export rápido

### Sidebar derecho (3 filas de 3 tabs c/u)
```
Fila 1: 🟥 LOCALIDADES    🟩 CLIENTES        🟦 PEDIDOS
Fila 2: 🟨 RUTAS          🟪 VISITA          🟦 DASHBOARD
Fila 3: 🌸 RENDICIONES    🩵 ALTA CLIENTES   🟥 NOTIFICACIONES
```

Las tabs ROJA / VERDE / AZUL son las que muestran contenido en el panel sidebar (Localidades, Clientes, Pedidos, Rutas, Rendiciones, Alta Clientes). El resto son "actions" que abren modal (Visita, Dashboard, Notificaciones).

Los contadores numéricos al lado del nombre están **ocultos por preferencia del usuario** (regla CSS `.tabs .tab-count { display:none }`).

### Mapa (centro)
Leaflet con polígonos provinciales/departamentales pintados según vendedor. Burbujas (markers) en cada localidad con conteo de tiendas. Click para drillear.

### Filtros superiores
- **Zona** (vendedor): bloqueado para `role=vendedor`.
- **Provincia**: dependiente de zona.
- **Localidad**: dependiente de provincia.
- **Stats** (cards): 277 localidades · 8 habilitados · 932 pendientes · 941 tiendas.

---

## 10) Sección: Localidades / Clientes / Pedidos

### Pestaña LOCALIDADES
Tarjetas con cada localidad de la zona del vendedor. Búsqueda por nombre. Click → fly-to en el mapa.

### Pestaña CLIENTES
Lista de tiendas (clientes + prospectos del master). Cada tienda tiene **3 estados**:
- `pendiente` (default): sin contactar todavía.
- `habilitado`: el vendedor confirmó que es un cliente activo. Pinta verde. Solo los habilitados aparecen en CREAR PEDIDO.
- `cancelado`: el vendedor confirmó que NO es cliente (cerró, no le interesa, etc.).

Búsqueda por nombre. Filtros: TODOS / CONFIRMADOS / NO CONFIRMADOS.

Click en una tienda → modal con:
- Datos: tipo, provincia, localidad.
- Botones de estado: HABILITAR / CANCELAR.
- Sub-sección **"Compras anteriores"** si hay pedidos confirmados.
- Sub-sección **"Recomendados"**: SKUs que otras tiendas de la misma provincia compraron y esta tienda NO tiene (cross-selling).

### Pestaña PEDIDOS (3 sub-vistas)
- **CREAR**: lista de clientes habilitados. Click en uno → modal intermedio con 4 opciones:
  - **Cargar visita** → abre form de visita.
  - **Ver compras anteriores** → muestra historial.
  - **Ver recomendados** → SKUs sugeridos.
  - **Crear pedido** → abre el picker de productos.
- **PENDIENTES**: pedidos en draft (no confirmados definitivos). Click → revisión + sugeridos clickeables.
- **CONFIRMADOS**: solo lectura.

### Picker de productos (Crear pedido)
- Filtros encadenados: Categoría → Familia → Subfamilia.
- Buscador por código/descripción.
- Cada producto: nombre + categoría + botón "+".
- **Cuando ya hay qty > 0**: el botón "+" se convierte en un mini-stepper `[−][qty editable][+]`.
- **Highlight amarillo** si el SKU está en una campaña activa aplicable + badge `★ CAMP`.

### Modal de revisión de pedido
- Header: cliente + provincia + localidad.
- Lista de líneas: SKU, descripción, qty (editable), precio (editable), eliminar.
- Total al pie.
- Botones: Volver a editar / Confirmar y enviar a Pendientes.

### Confirmación final
- Botón "Confirmar definitivo" en el modal de Pendientes.
- Pasa el pedido a `stage='confirmed'` con `confirmedAt` y `finalizedAt`.
- A partir de ese momento es exportable a SAP via DTW.

---

## 11) Sección: Rutas

### Generación automática
Algoritmo `generarRutasVendor(vendor, monthIdx, year)`:
1. Toma todas las tiendas **habilitadas** de la zona del vendedor.
2. Las agrupa por **proximidad** usando lat/lon de la localidad (haversine clustering).
3. Forma rutas de **10-15 tiendas** (target 12).
4. Asigna **fechas hábiles** del mes a cada ruta, distribuyendo equitativamente.
5. Aplica **route_overrides**:
   - `derivada`: la tienda queda marcada en amarillo "Esperando contacto del VDI", no se elimina de la ruta.
   - `reagendada`: la tienda se mueve a la ruta de la fecha objetivo (puede crear ruta nueva si no existía).
6. Re-numera y ordena por fecha.

### Vista detalle de una ruta
Card por cada tienda:
- 🟢 **Verde** = visitada (hay visita en `visits` para esa tienda+mes+año por cualquier vendor, incluido VDI)
- 🟡 **Amarillo mostaza** (derivada) = derivada al VDI, esperando que cargue visita
- 🟡 **Amarillo ámbar** (reagendada) = movida a fecha futura
- ⚪ **Gris** = pendiente sin tocar

### Click en "Cargar visita" en una tienda
Abre un mini-modal con 2 opciones:
- **La tienda está abierta** → abre form de visita con tienda pre-seleccionada.
- **La tienda está cerrada** → 2 opciones:
  - **Derivar al VDI**: crea `route_overrides` con `action='derivada'` + manda notif al VDI.
  - **Reagendar**: muestra picker de días hábiles (próximos 7) + confirma.

### Enviar ruta por WhatsApp
Botón verde en cada ruta. Al tocarlo:
1. Pide GPS del vendedor (`navigator.geolocation.getCurrentPosition`).
2. Si lo otorga, ordena las tiendas con **Nearest Neighbor** desde su posición.
3. Si la ruta tiene > 10 tiendas, las divide en **tramos balanceados** (Google Maps URL soporta max 9 waypoints + 1 destino).
4. Para cada tienda usa, en orden de prioridad:
   - GPS preciso de `client_locations` (lat,lon)
   - Dirección de `client_master` (geocoding)
   - Nombre + localidad + provincia (fallback)
5. Genera links Google Maps `https://www.google.com/maps/dir/?api=1&origin=...&destination=...&waypoints=...|...&travelmode=driving`
6. Compone mensaje WhatsApp con todos los tramos y abre `wa.me/<myWhatsappNumber>?text=...`

El número destino es el propio número del vendedor (configurado en Panel Usuarios) → se manda a sí mismo como notas.

### Histórico
Toggle "Histórico" para ver rutas de meses anteriores (solo lectura).

---

## 12) Sección: Visita

Formulario completo de visita a tienda. Pestañas internas: **Nueva visita** / **Mis visitas**.

### Campos del form (todos obligatorios salvo aclarado)
- **Localidad** (cascada con `vf-tienda`)
- **Tienda** (filtrada por localidad y vendedor; incluye clients + prospects)
- **Tipo de tienda** (texto libre)
- **Local**: `FISICO` | `ECOMMERCE`
- **Tamaño**: `GRANDE` | `MEDIANA` | `CHICA`
- **Fidelidad**: `ALTA` | `MEDIA` | `BAJA`
- **Relevancia**: Likert 1-5
- **POP**: SI / NO. Si SI, pide **Necesidad puntual** (CAÑERO/CARTEL/MOSTRADO/OTROS)
- **Espacio**: hasta 8 fotos (cámara o galería)
- **Oportunidad** (textarea, opcional)
- **Lo más vendido Shimano** (textarea, opcional)
- **Lo que más preguntan** (textarea, opcional)
- **Ayuda a tienda** (textarea, opcional)
- **Frente del local**: 1 foto. **OBLIGATORIA salvo `role=interno`** (el VDI carga visitas desde la oficina, no puede sacar foto).
- **Tipo de venta**: `MOSTRADO` | `ECOMMERCE` | `AMBOS`. Si AMBOS, pide ponderación % (debe sumar 100).
- **Competencia** (texto, opcional)

### GPS doble-check al enviar
1. Al tocar "Enviar formulario", pide GPS del navegador.
2. Si lo otorga, busca la tienda en `client_locations`.
3. Si NO existe ubicación de referencia → la primera visita la auto-confirma.
4. Si existe → calcula distancia haversine.
5. Estados:
   - `confirmed`: ≤ 300 m
   - `near`: 300 m – 1 km
   - `far`: > 1 km (pide confirmación extra)
   - `first`: primera visita, define la referencia
   - `denied`/`unavailable`/`timeout`/`error`: sin GPS válido

### Mis visitas (sub-tab)
Lista con filtros: Mes, Tienda, Año, Vendedor (solo admin/viewer). Click → ver visita en modo solo lectura con foto del frente, GPS info, link a Google Maps.

---

## 13) Sección: Dashboard

### Vista admin con filtro "Todos los vendedores"
Cuando admin/viewer abre el Dashboard sin filtrar a un vendedor específico, ve la **vista consolidada**:

#### Card "Resumen equipo"
Fondo azul Shimano. Totales del mes:
- Facturado ARS total del equipo
- # Pedidos
- # Visitas
- % Cumplimiento global (suma facturado / suma targets)
- Línea info: cuántos vendedores tienen target asignado.

#### Card "Ranking de vendedores"
Los 6 vendedores apilados, ordenados por **% cumplimiento descendente**. Cada card:
- 🟢 #1 (verde) · 🏆 LÍDER DEL MES (si pct > 0)
- 🔵 #2 (azul)
- 🟠 #3 (ámbar)
- ⚪ #4-6 (gris)

Cada card muestra: nombre + zona, % cumplimiento grande, barra de progreso, facturado / target, contadores (pedidos / unidades / visitas), o badge "SIN TARGET" si no tiene asignado.

### Vista por vendedor específico
Cuando se filtra a un vendedor:
- Card "Mes en curso" con unidades + facturado + target + % cumplimiento + barra
- Card "Acumulado anual YTD" con totales del año
- Card "Target Jul-Dic 2026" del semestre completo
- Card "Campañas activas" del vendedor

Los targets vienen de la collection `targets` (cargados por gerente).

---

## 14) Sección: Rendiciones

3 sub-tabs:

### Sub-tab "Solicitar recarga"
Formulario simple:
- Solicitado por (texto)
- Motivo o evento (textarea)
- Tipo de operación: ANTICIPO DE EFECTIVO / RENDICION DE GASTO / RECARGA
- Importe (number)
- Moneda: PESOS ARGENTINOS / DOLARES / OTRAS MONEDAS
- Observaciones (textarea, obligatorio)
- Estado: ABIERTO / CERRADO
- Adjunto opcional (Excel/PDF/imagen)

### Sub-tab "Rendir gasto"
Form para cada ticket. Foto del ticket **obligatoria** con 2 botones:
- 📷 **SACAR FOTO** (cámara trasera, `capture="environment"`)
- 📂 **ELEGIR DE GALERÍA** (file picker normal)

Cuando se sube la foto, **automáticamente** se llama a Gemini API para extraer los datos (OCR). Si el OCR funciona, los siguientes campos quedan autocompletados (revisables antes de enviar):

- Número de ticket
- Descripción: COMBUSTIBLE / COMIDA / HOSPEDAJE / PEAJE / TRASLADO / OTROS
- Modo de pago: RECARGABLE / CORPORATIVA / EFECTIVO
- Moneda: PESOS / DOLARES / OTRAS MONEDAS
- Tipo de gasto (categoría tributaria): GASTO CON COMPROBANTE / GASTO SIN COMPROBANTE / FACTURA A
- Importe (number)
- Importe USD (opcional)
- División gasto: GASTO LOCAL / GASTO REGIONAL
- Observaciones (texto, obligatorio)

Banner amarillo después del OCR: "Revisá los campos antes de enviar. La IA puede equivocarse, sobre todo en montos, número de ticket y descripción." + botón "Re-analizar".

### Sub-tab "Mis rendiciones"
Lista de todas las rendiciones del vendedor con tags por estado:
- 🟡 Pendiente de aprobación
- 🟢 Aprobada (sale en el Excel)
- 🔴 Rechazada (con motivo)

Botón **"Exportar Excel (solo aprobadas)"**: descarga un xlsx con formato **Shimano oficial**:
- Hoja "RENDICIÓN" con título "PLANILLA RENDICIÓN" en fila 2, subtítulo "gastos visitas DD-MM-AAAA" en fila 4, headers en fila 8.
- Columnas: FECHA / NUMERO DE TICKET / DESCRIPCIÓN / MODO DE PAGO / MONEDA / TIPO DE GASTO / IMPORTE / DIVISIÓN GASTO / OBSERVACIONES.
- Totales por moneda + cantidad de tickets al final.
- Hoja "Solicitudes" con los anticipos aprobados (si hubo).
- Hoja "Desplegable" con las listas de validación.
- Nombre archivo: `RENDICION DE GASTOS DD DE MES AAAA.xlsx`.

### Flujo de aprobación
- Cuando vendedor envía → `status='pending_approval'`.
- Se determina el aprobador leyendo `myRendicionesApproverUid` del rol del vendedor.
- Se manda notif tipo `rendicion_approval` al aprobador (SOLO a él, no broadcast).
- Aprobador ve la notif en su pestaña Notificaciones → "Ver detalle y decidir" → modal con todos los datos + foto del ticket ampliable → Aprobar / Rechazar (rechazo pide motivo obligatorio).
- Vendedor recibe `rendicion_approval_ack` con el resultado.
- Solo las APROBADAS aparecen en el Excel.

---

## 15) Sección: Alta Clientes

2 sub-tabs:

### Sub-tab "Nueva solicitud"
Panel cyan arriba: **"🔗 Compartir formulario con el cliente"** con 2 botones:
- 📋 **Copiar link**: copia al portapapeles `https://shimano-arg.github.io/app-vendedores/alta-cliente.html?vendor=<uid>&vendorName=<nombre>&vendorEmail=<email>`
- 📱 **Enviar por WhatsApp**: abre wa.me con mensaje pre-armado

Debajo, el formulario completo con 3 secciones:

#### Sección "Datos del comercio" (13 campos)
E-mail, Nombre del comercio, Nombre de fantasía, CUIT, Condición fiscal (RI/Monotributo/Exento/CF/RNI), Calle, Número, Localidad, Provincia, CP, Teléfono, Web, Redes (IG/FB).

#### Sección "Datos de contacto" (6 campos)
Nombre y apellido, Tel particular, WhatsApp/celular, E-mail contacto, Tipo de comercio (PESCA/OUTDOOR/CAZA/MULTIRUBRO), Tienda online (MELI/Web propia/Otros/No tiene).

#### Sección "Documentación a adjuntar"
- Constancia ARCA (1 imagen/PDF) — obligatorio
- Constancia IIBB (1 imagen/PDF) — obligatorio
- Fotos del local (hasta 5, frente e interior) — obligatorio

### Sub-tab "Mis solicitudes"
Lista de solicitudes del vendedor con tags por estado: Pendiente (X/2 aprobaciones) / Aprobada / Rechazada con motivo.

### Flujo de doble aprobación
- Al enviar → `status='pending_approval'`, `approvals={}`.
- Se manda notif `client_approval` a los **2 aprobadores hardcoded** (`CLIENT_APPLICATION_APPROVER_EMAILS`):
  - `srb90284@gmail.com` (Santiago)
  - `quilgym@gmail.com` (Diego)
- Cada uno entra al notif → "Ver detalle y decidir" → modal con todos los datos + thumbnails de docs clickeables.
- Aprobar → `approvals[uid] = { approvedAt, email, name }`.
- Cuando el doc tiene 2 entradas en `approvals` → `status='approved'` automáticamente.
- Se manda `client_approval_ack` al vendedor con el resultado.
- Si alguno rechaza primero → `status='rejected'` + motivo + ACK al vendedor.

### Página standalone `alta-cliente.html`
Formulario público (sin login) que vive en el mismo repo. Conecta al mismo proyecto Firebase sin auth (las rules permiten create cuando `submittedByPublicForm=true` con validaciones mínimas: comercio + cuit + status=pending_approval).

Recibe en query string: `vendor`, `vendorName`, `vendorEmail`. Muestra "Enviada a través de: NOMBRE" como atribución visual. Al enviar, crea el doc en `client_applications` con `submittedByPublicForm: true`, `ownerUid: vendorUid`, `ownerName: vendorName`. Después sigue el flujo de aprobación normal.

---

## 16) Sección: Notificaciones (Alertas y tareas)

Modal con **4 tabs** (en mobile: grid 2×2):

### Tab "Recibidas"
Lista de las notifs con `status='unread'` dirigidas al usuario. Cada item se renderiza según el `type`:

| Type | Render | Acciones |
|---|---|---|
| `derivacion` | Tienda + mensaje + ubicación | "Contactar" (abre form visita con tienda pre-cargada) / "Solo marcar leída" |
| `task` | Título + descripción + thumbnails de imágenes (click → fullscreen) | "Marcar como completada" → status='done' + manda task_ack al emisor |
| `task_ack` | Mensaje "X marcó como completada tu tarea" | "Marcar leída" |
| `client_approval` | Resumen de la solicitud + Vendedor + CUIT + Localidad | "Ver detalle y decidir" (Aprobar / Rechazar) |
| `client_approval_ack` | Aprobada/Rechazada + motivo si aplica | "Marcar leída" |
| `rendicion_approval` | Tipo + importe + moneda + ticket | "Ver detalle y decidir" |
| `rendicion_approval_ack` | Aprobada/Rechazada | "Marcar leída" |

### Tab "Realizadas"
Notifs con `status='read'` o `status='done'`. Solo lectura, sin botones.

### Tab "Crear tarea"
Formulario:
- Destinatario: dropdown con todos los usuarios con rol asignado (excluye unassigned y self).
- Título corto (max 100 chars)
- Descripción (textarea)
- Imágenes: hasta 5 fotos comprimidas a 1200px@75% calidad.
- Botón "Enviar tarea" → crea notif type='task'.

### Tab "Enviadas"
Tareas que el usuario creó. Listener `where('fromUid', '==', currentUser.uid)` y filtra `type='task'` en cliente (evita índice compuesto). Muestra estado actualizado en tiempo real (cambia a "✓ Completada" cuando el destinatario marca done).

### Badge rojo en la pestaña
Cuenta solo las **pendientes** (status ≠ read y ≠ done). Se actualiza vía `updateNotifsBadge` cada vez que el listener `myNotifications` se dispara.

### Image viewer fullscreen
Click en cualquier thumbnail abre overlay oscuro fullscreen con la imagen ampliada. Click-out cierra.

---

## 17) Sistema VDE-VDI (vendedor externo / interno)

VDE = vendedor externo (recorre el campo). VDI = vendedor interno (oficina). Cada VDE tiene un **VDI pareja** asignado en Panel Usuarios (`internalPartnerUid`).

### Caso de uso típico
1. Gonzalo (VDE Z1) recorre San Pedro. Visita "MARIANO-PESCA" pero la encuentra cerrada.
2. Gonzalo abre la ruta del día → toca la tienda → "La tienda está cerrada" → "Derivar al VDI".
3. Se crea `route_overrides` con `action='derivada'`, `derivedToUid: quilgymUid`.
4. Se crea notif `derivacion` para `quilgym@gmail.com` (Diego, el VDI).
5. La card de "MARIANO-PESCA" en la ruta de Gonzalo queda 🟡 amarilla "Derivada VDI".
6. Diego abre la app, ve la notif → "Contactar" → form de visita con tienda pre-cargada.
7. Diego completa la visita (sin foto del frente, porque está en la oficina). Envía.
8. Listener `unsubVisitsPartner` de Gonzalo detecta la visita del partner (filter `ownerUid == myInternalPartnerUid`) y actualiza el cache.
9. La card de Gonzalo pasa automáticamente de 🟡 amarillo a 🟢 verde sin recargar.

### Visitas del VDI
- Campo "Frente del local" no obligatorio.
- El asterisco rojo del label se oculta.
- La hint dice "(opcional - 1 foto)".

---

## 18) Campañas comerciales

Solo admin puede crear/editar/borrar.

### Crear campaña (modal violeta)
Form en cascada:
1. **Familia** (dropdown).
2. **Subfamilia** (filtrada por familia).
3. **SKUs incluidos** (checkbox list con todos los SKUs de esa subfamilia).
4. **Nombre** de la campaña.
5. **Objetivo**: Unidades / Monto $ (toggle).
6. **Fechas** desde/hasta.
7. **Alcance**: TODAS LAS ZONAS / POR PROVINCIA / POR VENDEDOR.

### Botón "Campañas activas" (header amarillo)
Disponible para todos los roles. Modal con las campañas vigentes (`startDate <= today <= endDate`):
- Para vendedores: filtradas por `isCampaignApplicableToVendor()` (su provincia / su vendedor key).
- Para admin/viewer: todas.
- Cada card: nombre + fechas + objetivo + ámbito + SKUs (colapsable).

### Highlight en picker de productos
Cuando un vendedor está armando un pedido, los SKUs que pertenecen a una campaña activa aplicable se renderizan con **fondo amarillo claro + borde naranja + badge ★ CAMP**.

---

## 19) Targets mensuales

Solo `admin` o `gerente` pueden cargar (función `canManageTargets`). Botón **TARGETS** en el header (oculto para los demás).

### Panel Targets
Modal con selector de Vendedor + Año + tabla de 12 meses (Enero a Diciembre). Por cada mes:
- Input numérico de Target ARS.
- Tag de estado: Asignado (verde) / Sin cargar (gris).
- Highlight del mes actual.

Al guardar:
- Si el valor está vacío → borra el doc.
- Si tiene valor → upsert en `targets/{vendorKey}_<year>_<MM>`.

El dashboard usa estos targets para calcular % cumplimiento.

---

## 20) Panel Master Clientes (direcciones)

Solo admin. Botón **MASTER CLIENTES** en el header.

Modal con:
- **Filtros**: Vendedor, Provincia, Localidad (dependiente), Estado (con/sin dirección), Buscador por nombre.
- **Stats arriba**: Con dirección X / Sin dirección Y / Total filtrado Z / Cambios sin guardar.
- **Tabla**: cada fila = una tienda (clients + prospects). Columnas: Tienda + tag, Localidad, Provincia, Vendedor, Input dirección, Botón Guardar.

El botón Guardar se vuelve ámbar y pulsa cuando hay cambios sin guardar en esa fila.

### Cómo lo usa el WhatsApp de rutas
`buildStopRef(tienda)` consulta en orden de prioridad:
1. GPS de `client_locations` (coordenadas).
2. Dirección de `client_master` (string con dirección).
3. Fallback al nombre.

Cargar direcciones mejora la precisión del link Google Maps que se manda por WhatsApp.

---

## 21) Integración SAP B1

Panel **SAP** del header (admin + viewer + vendedor; cada vendedor ve solo sus pedidos).

### 4 tabs (solo admin ve las 2 últimas)
- **Pendientes de SAP**: pedidos confirmed sin transferir, separados en "Listos" (con CardCode asignado) y "Bloqueados" (sin alta SAP del cliente).
- **Ya transferidos**: histórico, modo lectura.
- **Mapeo Clientes** (solo admin): tabla con todas las tiendas + input CardCode SAP + bulk import (TAB/coma).
- **Mapeo Productos** (solo admin): tabla con todos los SKUs + input Material SAP + bulk import.

### Exportar ZIP DTW
Botón "Exportar ZIP DTW (Listos)" genera un ZIP con:
- `ORDR - Documents.csv`: cabeceras de Sales Order (2 filas header API+Internal + datos)
- `RDR1 - Document_Lines.csv`: líneas de pedido
- `LEEME.txt`: instrucciones paso a paso para DTW

Campos que completa al CSV:
- `CardCode` ← `sap_clients[clientName]`
- `ItemCode` ← `sap_products[code]` (fallback al code app)
- `DocDate`, `DocDueDate` (+30 días), `TaxDate`
- `WarehouseCode` = `07` (PESCA hardcoded)
- `Comments` = `"AppShimano | <Tienda> | <Mes> | <Email vendedor>"`
- `NumAtCard` = ID Firestore del pedido (trazabilidad)
- `SalesPersonCode` ← `sap_vendors[vendorKey].slpCode`
- `DiscountPercent` = 0

### Marcar lote como transferido
Después de subir el CSV a DTW, el vendedor (o admin) tilda los pedidos y toca "Marcar lote como transferido (N)" → modal pide opcional el rango de DocNum SAP creado → actualiza cada pedido con `transferidoSAP: { transferredAt, transferredBy, batchId, sapDocRange }`.

### Panel "Integración SAP" (solo Mariano)
Dentro de "Exportar para Análisis" (PIN 1235) → opción "Integración SAP" → modal con 3 inputs file:
- **Business Partners**: Excel/CSV con `CardCode, CardName` (acepta variaciones de nombre de columna).
- **Items**: `ItemCode, ItemName`.
- **Sales Employees**: `SlpCode, SlpName`.

Al procesar:
- **Match exacto normalizado** (NFD + lowercase + sin "S.A./SRL/LTDA/CIA").
- Productos: match por código case-insensitive.
- Vendedores: match `SlpName` contra `VENDORS.key`.

Resultados en 3 cards: Matches OK / Ambiguos / Sin match. Botón "Descargar reporte CSV" + "Aplicar a Firestore" (escribe a `sap_clients`, `sap_products`, `sap_vendors` con `source: 'sap_integration_v1'`).

### UDFs a crear en SAP (pendiente SEIDOR)
En la tabla **ORDR** (Sales Order header):

| UDF | Tipo | Valores |
|---|---|---|
| `U_AppOrigen` | Alfanumérico 20 | Valid Values: `APP / SAP_MAN / SAP_TEL` (default `SAP_MAN`) |
| `U_AppOrderId` | Alfanumérico 50 | Texto libre (ID Firestore) |
| `U_AppBatchId` | Alfanumérico 30 | Texto libre (timestamp YYYYMMDDHHMMSS) |

Además se pidió crear:
- **Serie "APP"** para Sales Order Document Numbering (numeración separada de manual).
- **6 Sales Employees** (los 6 vendedores externos) → se mapean en `sap_vendors`.

---

## 22) OCR de tickets con Gemini API

### Configuración
- **Provider**: Google Gemini API.
- **Modelo**: `gemini-2.5-flash` (alta calidad multimodal, free tier 3M tokens/día).
- **Endpoint**: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`.
- **API key**: vive en **Firestore** (`app_config/gemini.apiKey`), NO en el código fuente del repo público.
- **Carga de key**: Panel Usuarios → sección violeta "Gemini API Key" → botón "Cargar/Cambiar key" → prompt para pegarla.
- **Lectura desde la app**: `getGeminiApiKey()` async lee del cache local o de Firestore.

### Flujo
1. Vendedor en sub-tab "Rendir gasto" sube foto del ticket (cámara o galería).
2. Auto-llamada a `extractTicketDataWithGemini(dataUrl)`.
3. Body del request:
   ```json
   {
     "contents": [{
       "parts": [
         { "text": "<prompt con esquema JSON>" },
         { "inline_data": { "mime_type": "image/jpeg", "data": "<base64>" } }
       ]
     }],
     "generationConfig": {
       "response_mime_type": "application/json",
       "temperature": 0.1
     }
   }
   ```
4. El prompt fuerza a Gemini a devolver JSON con 10 campos válidos (dropdowns hardcoded del sistema Shimano).
5. `fillRendGastoFormFromOcr(json)` autocompleta los inputs/selects del form. Para los selects, hace match case-insensitive contra las opciones válidas.
6. Banner amarillo después: "Revisá los campos antes de enviar".

### Restricciones de seguridad (recomendadas)
- En Google Cloud Console → Credentials → editar la key:
  - **API restrictions**: solo "Generative Language API".
  - HTTP referrer (si la key no está vinculada a Service Account): `shimano-arg.github.io/*`.
- Si se filtra: el peor escenario es que alguien agote tu cuota gratis. Mitigación: rotar la key desde el Panel Usuarios.

---

## 23) PWA installable

La app puede instalarse en el home del celular como app standalone.

### Archivos PWA en el repo
- `manifest.json`: nombre, íconos, theme color (#00A9E0), background color (#ffffff), display=standalone, start_url=./
- `sw.js`: service worker mínimo:
  - **HTML (index.html)**: network-first con fallback a cache. Garantiza que el usuario siempre ve la última versión.
  - **Assets locales** (manifest, íconos, logo): cache-first. No cambian salvo deploy.
  - **CDNs externos** (firebase, leaflet, sheetjs, openstreetmap tiles): NO se interceptan. Pasan directo a la red.
- Íconos: `icon-180-v3.png` (iOS), `icon-192-v3.png` (Android), `icon-512-v3.png`, `icon-512-maskable-v3.png` (Android adaptive icons).

### Versionado de cache
Constante `CACHE_VERSION` en `sw.js` (actualmente `'v3'`). Bumpearla invalida el cache viejo. Los íconos tienen `-v3` en el nombre para forzar refresh en iOS (que cachea agresivamente).

### Instalación en celular
- **Android Chrome**: prompt automático "Agregar a pantalla principal" o desde menú "Instalar app".
- **iPhone Safari**: Compartir → Agregar a pantalla de inicio. (Chrome iOS no permite instalar PWAs por restricción de Apple.)

### Notas
- El **theme color** se aplica al status bar Android.
- El **background color** es el splash screen al abrir la PWA.
- Cuando se cambian íconos hay que **bumpear versión** y renombrar archivos para invalidar el cache de iOS.

---

## 24) Backup mensual

Acceso: admin → "Exportar para Análisis" (PIN 1235) → opción "Backup mensual + limpiar fotos" (borde naranja, badge ADMIN).

### Flujo
1. Selector mes / año (default: mes pasado).
2. Toca "Generar backup" → en 30-90 segundos descarga 4 archivos:
   - **`Shimano_Fotos_YYYY-MM.zip`**: organizado por `Vendedor/Tienda_YYYYMMDD/{frente.jpg, espacio_N.jpg}`.
   - **`Shimano_Visitas_YYYY-MM.xlsx`**: metadata de cada visita del mes (sin fotos).
   - **`Shimano_Pedidos_YYYY-MM.xlsx`**: líneas de pedido de los confirmados del mes.
   - **`Shimano_Rutas_YYYY-MM.xlsx`**: 2 hojas (Cumplimiento por vendedor + Detalle rutas con # asignadas vs visitadas y %).
3. Después de verificar que los archivos están OK, botón rojo **"Borrar fotos del mes (N)"** → pide PIN 1234 → para cada visita del mes hace:
   ```js
   visits/{id}.update({
     frenteLocal: FieldValue.delete(),
     espacio: FieldValue.delete(),
     photosDeletedAt: serverTimestamp,
     photosDeletedBy: currentUser.email
   })
   ```
4. Las visitas quedan intactas, solo se eliminan las imágenes. Esto controla el crecimiento de Firestore.

### Envío automático mensual (NO implementado)
La app no manda email automático (es estática, sin backend). Alternativas para automatizar:
- **Firebase Cloud Functions** + Cloud Scheduler + SendGrid (Blaze plan).
- **GitHub Actions** con cron + service account + nodemailer (gratis).

---

## 25) Exports a Excel / Power BI / ML

Botón "EXPORTAR PARA ANÁLISIS" (PIN 1235) abre modal con 5+ opciones:

| Opción | Descripción |
|---|---|
| **Power BI** | Tabla de hechos + dimensiones (vendedor, producto, cliente, calendario, campaña). Modelo estrella importable directo. |
| **Python / ML** | Una tabla larga (`master_ml`) con 24 columnas por línea de pedido. Para pandas/scikit-learn. |
| **Fotos de visitas (ZIP)** | ZIP con todas las fotos organizadas por vendedor/tienda/fecha. |
| **Excel con fotos embebidas** | XLSX donde cada visita lleva la foto del frente DENTRO de la celda (usa ExcelJS 4.4.0 lazy load). |
| **Integración SAP** (solo Mariano) | Cruce de masters SAP con la app (descrito arriba). |
| **Backup mensual + limpiar fotos** (solo admin) | Descrito arriba. |

Botón "EXPORTAR A EXCEL" (verde, sin PIN) abre otro modal con:
- **Excel ejecutivo**: consolidado + hoja por vendedor + visitas + contactados + log.
- **Excel de visitas**: solo visitas.

---

## 26) Panel admin "Usuarios"

Botón "USUARIOS" en el header (solo admin). Modal con 3 secciones:

### 1. Emails pre-autorizados (sección azul)
Lista de chips con los emails de `allowed_emails`. Botón "+ Agregar email" → prompt email + nota. Cada chip tiene botón X para quitar.

### 2. Gemini API Key (sección violeta)
Muestra la key enmascarada + quien la cargó + cuándo. Botones "Cambiar key" / "Borrar".

### 3. Tabla de usuarios
Una fila por cada doc en `/roles`. Columnas:
- Email + tags (`(VOS)` si self, 🔒 PROTEGIDO si admin protegido)
- Nombre
- **Rol** (dropdown)
- **Vendedor** (dropdown VENDOR keys)
- **Pareja interno** (dropdown internos disponibles)
- **WhatsApp** (input tel, se limpia a solo dígitos al guardar)
- **Resp. rendiciones** (dropdown admin/gerente/interno)
- **Acciones**: Eliminar (rojo, no aparece para protegidos ni self) + Guardar (verde)

En mobile la tabla se rinde como cards apilados.

### Admins protegidos
- `bot.shimano.pesca@gmail.com`
- `erbinomariano@gmail.com`

No pueden eliminarse desde la UI.

### Zona de peligro
Sección con borde rojo abajo: **"Borrar todo el historial"**.
Click → prompt pidiendo password `1234` → confirmación final → itera collections `pedidos, visits, campaigns, sap_clients, sap_products, route_overrides, notifications` y borra todo. Limpia `userData`. Crea log `PURGE_TOTAL_HISTORIAL` antes.

Los roles, allowed_emails y app_config se preservan.

---

## 27) URLs externas e integraciones

| Servicio | URL | Uso |
|---|---|---|
| GitHub Pages | https://shimano-arg.github.io/app-vendedores/ | Hosting del HTML |
| GitHub repo | https://github.com/shimano-arg/app-vendedores | Source code |
| Firebase project | app-vendedores-shimano.firebaseapp.com | Auth + DB |
| Firebase Console | https://console.firebase.google.com/project/app-vendedores-shimano | Mgmt |
| Google AI Studio | https://aistudio.google.com/app/apikey | Generar API key Gemini |
| Google Cloud Console | https://console.cloud.google.com/apis/credentials?project=gen-lang-client-0633054372 | Restringir API key |
| Leaflet | https://unpkg.com/leaflet@1.9.4/ | Mapa (CDN) |
| OpenStreetMap CartoDB | tiles via cartocdn.com | Tile layer |
| SheetJS | https://cdn.jsdelivr.net/npm/xlsx@0.18.5 | Excel (CDN) |
| ExcelJS | https://cdn.jsdelivr.net/npm/exceljs@4.4.0 | Excel con fotos (lazy) |
| JSZip | https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1 | ZIP (CDN) |
| Firebase compat SDK | https://www.gstatic.com/firebasejs/10.7.1/ | App+Auth+Firestore |
| Gemini API | https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent | OCR |
| WhatsApp | https://wa.me/ | Compartir rutas y links de alta cliente |
| Google Maps Directions | https://www.google.com/maps/dir/?api=1 | Rutas WhatsApp |

### Origins permitidos en Firebase Auth
- `app-vendedores-shimano.firebaseapp.com`
- `shimano-arg.github.io`
- localhost (para testing)

---

## 28) Convenciones de código

### Idioma
Comentarios y mensajes al usuario en **español rioplatense informal**. Identificadores en inglés cuando son técnicos (uid, status, etc.).

### Nomenclatura
- Funciones globales expuestas en `window.X` para que `onclick="X()"` del HTML las encuentre.
- Listeners de Firestore: `unsubXxx` (función para desuscribir) + `ensureXxxListener()` (idempotente).
- Cache local: `xxxCache` (Map o array).
- Constantes UPPER_SNAKE_CASE.

### Manejo de estados
- Pedidos: `stage='confirmed'` (único valor real). Los borradores viven en `userData.orders`.
- Visitas: `mes` UPPERCASE (`'JUNIO'`). Cuando se compara contra `MESES[idx]` (capitalización normal), siempre hacer `.toUpperCase()` en ambos lados.
- Aprobaciones: `status='pending_approval' | 'approved' | 'rejected'`.
- Tareas: `status='unread' | 'read' | 'done'`.

### Helpers comunes
- `titleCase(s)`: primera letra mayúscula del primer token.
- `escapeHtml(s)`: para usar en innerHTML.
- `escapeAttr(s)`: para usar en atributos.
- `fmtMoney(n)`: `$1.234.567` (es-AR).
- `fmtNum(n)`: con separadores de miles.
- `fmtUSD(n)`: `USD 1,234`.
- `arsToUsd(ars)`: `ars / EXCHANGE_RATE`.
- `haversineKm(lat1, lon1, lat2, lon2)`.
- `compressImage(file, maxWidth, quality)`: redimensiona y comprime a JPEG base64.
- `sapNorm(s)`: normaliza nombres (NFD + uppercase + sin acentos + sin "S.A./SRL").
- `clientLocId(prov, loc, tienda)`: docId determinístico para `client_locations` y `client_master`.
- `emailToDocId(email)`: para `allowed_emails`.

### Patrón listener
```js
let myCache = [];
let unsubMy = null;
function ensureMyListener(){
  if (unsubMy || !currentUser || !fbDb) return;
  unsubMy = fbDb.collection('xxx').where(...).onSnapshot(qs => {
    myCache = [];
    qs.forEach(d => myCache.push({...}));
    // re-render condicional
  }, err => console.warn('xxx listener', err));
}
```

### Listeners centrales
Activados en `applyRolePermissions` cuando `userRole !== 'unassigned'`:
- `ensureNotifsListener`
- `ensureRouteOverridesListener`
- `ensureTargetsListener`
- `ensureClientLocsListener`
- `ensureSapVendorsListener`
- `ensureClientMasterListener`
- `ensureAltaCliListener`
- `ensureRendicionesListener`

---

## 29) Regenerar y deployar

### Workflow normal (cambios en código JS dentro del template Python)
```powershell
cd "C:\Users\shimano.sandbox\Desktop\MASTERFILES\PROSPECTOS\MAPAS"
python _build_argentina_zonas_v2.py
Copy-Item Mapa_Argentina_Shimano_Zonas.html "C:\Users\shimano.sandbox\Desktop\APP VENDEDORES\index.html"
cd "C:\Users\shimano.sandbox\Desktop\APP VENDEDORES"
git add index.html
git commit -m "descripción del cambio"
git push
```

### Cambios sin regenerar (manifest, sw, alta-cliente.html, README)
```powershell
cd "C:\Users\shimano.sandbox\Desktop\APP VENDEDORES"
git add <archivo>
git commit -m "..."
git push
```

### Tiempo de propagación
GitHub Pages: 30-90 segundos.

### Forzar refresh en clientes
Para invalidar cache del Service Worker:
1. Editar `sw.js`: bumpear `CACHE_VERSION` a `vN+1`.
2. Bumpear referencias a íconos si cambiaron (v3 → v4) y renombrar archivos.

Usuarios PWA en iOS pueden necesitar quitar la app del home + reinstalar para ver los íconos nuevos (iOS cachea agresivamente).

---

## 30) Troubleshooting

### "Missing or insufficient permissions"
Falta publicar las rules. Ver sección 8 y copiar todo a Firebase Console.

### Login rechazado a un email autorizado
- Verificar que el email esté en `ALLOWED_EMAILS` hardcoded, `ALLOWED_EMAIL_DOMAINS`, o en collection `allowed_emails`.
- Si tiene rol asignado previo en `/roles` ≠ `unassigned`, también debe pasar (condición 3 del whitelist).

### Las visitas no se pintan verdes en las rutas
- Verificar que `v.mes` sea UPPERCASE en el doc.
- El comparador case-sensitive era un bug; ahora todos los matches hacen `.toUpperCase()` en ambos lados.

### Las tareas "Enviadas" no se actualizan al ser completadas
- Antes había problema de índice compuesto (`where fromUid + where type`).
- Ahora el listener filtra solo por `fromUid` y aplica `type==='task'` en cliente.

### OCR Gemini devuelve 429
- Quota exceeded. Cambiar de modelo (`gemini-2.5-flash-lite` tiene más cuota).
- O la key está vinculada a Service Account sin billing → crear key nueva desde AI Studio sin SA.

### Tab "Notificaciones" tiene contador raro
- `updateNotifsBadge` cuenta solo `status !== 'read' && status !== 'done'`.
- Si querés ocultar el contador globalmente, el CSS `.tabs .tab-count { display:none }` ya lo hace.

### El form Alta Cliente público no envía
- Verificar rule de `client_applications` que permita create cuando `submittedByPublicForm == true`.
- Browser console de la página standalone para ver error real.

### Push protection de GitHub rechaza commits
GitHub detecta secretos (API keys, credenciales) y bloquea push. Soluciones:
- Mover el secret a Firestore (caso Gemini key).
- Si es falso positivo: bypass desde la web (no recomendado).
- Nunca usar ofuscación (split de string) — el guardrail del agente también lo bloquea.

---

## 31) Roadmap / pendientes

### Para SEIDOR (integración SAP)
- Crear 3 UDFs en ORDR: `U_AppOrigen`, `U_AppOrderId`, `U_AppBatchId` (ver sección 21).
- Crear serie "APP" para Sales Order numbering.
- Crear 6 Sales Employees (David Carballo).

### Mejoras planeadas
- **Filtrar clientes CANCELADOS de rutas** (hoy aparecen).
- **Alertas de tiendas atrasadas**: tiendas no visitadas hace > 30 días, por vendedor.
- **Integración total client_applications aprobadas → POINTS**: hoy las solicitudes aprobadas no aparecen automáticamente en la pestaña Clientes para hacerles pedidos. Requiere decisión sobre si sincronizar con master Python o cache paralelo.
- **Notificaciones push** (FCM) para tareas y rendiciones pendientes.
- **Cloud Functions** para envío automático del backup mensual por email.
- **Dominio personalizado** (`app.shimano.com.ar`).
- **Dashboard del admin con vista consolidada anual** (no solo del mes).

### Pendientes operativos
- Mariano: actualizar Gemini API key cuando se rote.
- Mariano: configurar `internalPartnerUid` para cada vendedor cuando entren VDIs definitivos.
- Mariano: configurar `rendicionesApproverUid` para cada vendedor.
- David Carballo: enviar dump OITM con datos de productos para integración SAP.
- David Carballo: dump OCRD con BPs.
- Eliana: confirmar creación de UDFs + serie.

---

## Apéndice A: Variables globales clave

```js
let currentUser = null;                    // Firebase Auth user
let userRole = null;                       // 'admin' | 'vendedor' | etc.
let assignedVendor = null;                 // vendor key cuando role=vendedor
let myWhatsappNumber = null;
let myRendicionesApproverUid = null;
let myInternalPartnerUid = null;           // VDI pareja

// Caches
let myNotifications = [];
let mySentTasks = [];
let routeOverrides = [];
let visitsCache = [];
let visitsCachePartner = [];               // visitas del VDI pareja
let targetsCache = new Map();
let clientLocsCache = new Map();
let clientMasterCache = new Map();
let sapVendorsCache = new Map();
let altaCliMine = [];
let misRendiciones = [];
let allowedEmailsCache = new Set();        // implícito en collection check
let geminiApiKeyCache = null;
let usersCache = [];                       // para panel admin
let globalPedidos = [];                    // todos los pedidos (admin/viewer ven todos)
let confirmed = {};                        // organizado por key
let orders = {};                           // drafts locales

// Constantes
const ANALISIS_PIN = '1235';
const WHATSAPP_TEST_NUMBER = '5491126762031';
const MAPS_MAX_WAYPOINTS = 9;
const RUTA_MIN = 10, RUTA_TARGET = 12, RUTA_MAX = 15;
const ALTA_CLI_MAX_FOTOS = 5;
const CLIENT_APPLICATION_APPROVER_EMAILS = ['srb90284@gmail.com', 'quilgym@gmail.com'];
const GEMINI_MODEL = 'gemini-2.5-flash';
const ADMIN_BOOTSTRAP_EMAIL = 'bot.shimano.pesca@gmail.com';
const PROTECTED_ADMIN_EMAILS = ['bot.shimano.pesca@gmail.com', 'erbinomariano@gmail.com'];
```

---

## Apéndice B: Datos master embebidos en el build

| Constante | Origen | Tamaño |
|---|---|---|
| `POINTS` | 25 archivos `Mapa_<Prov>_Shimano.html` | 277 localidades |
| `PRODUCTS` | `MASTERFILE PRODUCTOS PESCA.xlsx` | 665 SKUs |
| `VENDORS` | hardcoded | 6 vendedores externos (Z1, Z2, Z4, Z5, Z6, Z7) |
| `TARGETS_BY_VENDOR` | `TARGETS VENDEDORES-ZONAS.xlsx` (hoja TARGET VENDEDORES) | Targets USD por vendedor (legacy, ahora se usa collection targets) |
| `EXCHANGE_RATE` | Excel de targets | 1500 ARS/USD aprox |
| `CLIENT_SALES` | MELI últimos 365 días | ~12.700 órdenes, 74 clientes matcheados |
| `PROV_TOP_PRODUCTS` | MELI agregado por provincia | Top SKUs por provincia |
| `DTW_DOC_API`, `DTW_DOC_INT`, `DTW_LIN_API`, `DTW_LIN_INT` | Plantillas DTW de SEIDOR | Headers oficiales |

---

> Última actualización del README: 10-junio-2026. Documento mantenido por Mariano Erbino (data scientist Shimano Argentina). Para reportar bugs o sugerir mejoras: bot.shimano.pesca@gmail.com / erbinomariano@gmail.com.
