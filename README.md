# Shimano - App Vendedores

App web para que el equipo comercial de Shimano Argentina gestione zonas, clientes, prospectos, pedidos, campañas, visitas y dashboards de ventas. Funciona desde celular o computadora con un único link público (GitHub Pages), con persistencia en la nube (Firebase Firestore) y sincronización en tiempo real entre dispositivos.

**Link público (oficial):** https://shimano-arg.github.io/app-vendedores/
**Repo:** https://github.com/shimano-arg/app-vendedores
**Firebase project:** `app-vendedores-shimano`
**Admin bootstrap:** `bot.shimano.pesca@gmail.com`

---

## ¿Para qué sirve?

- Cada vendedor entra al link, se loguea con su cuenta Google y ve **solo la información de su zona**.
- Recorre **localidades, clientes y prospectos** sobre un mapa interactivo de Argentina y los marca como contactados.
- Arma **pedidos** filtrando por categoría/familia/subfamilia/SKU con búsqueda, los confirma con mes/año (Pendientes) y luego los confirma definitivamente (Confirmados).
- En cada paso recibe **recomendaciones de productos** que otras casas de pesca de la misma provincia ya pidieron (red global de vendedores).
- Cuando inicia un pedido le aparece un **alert recordando campañas activas** del mes.
- Registra **visitas a tiendas** con un formulario completo (campos obligatorios + opcionales + fotos).
- Tiene un **Dashboard** con ventas del mes/año (unidades + ARS + USD), top productos, top clientes, progreso contra targets en USD (Jul 2026, Jul-Dic 2026, 2027) y avance de campañas activas.
- El **gerente de ventas y los analistas** (admin/viewer) ven info agregada de los 6 vendedores, pueden filtrar por vendedor en el dashboard, crear campañas comerciales, gestionar roles de usuarios, vaciar el historial si lo necesitan y revisar un **log de operaciones** (cancelaciones, eliminaciones).
- **Exporta a Excel** en 3 formatos + **descarga ZIP de fotos** de visitas.

---

## Estructura del repo

```
shimano-arg/app-vendedores/
├── index.html         # App lista para abrir (~2.5 MB, datos embebidos + Firebase compat)
├── Shimano-Logo.png   # Logo Shimano (header + splash)
├── build_app.py       # Script Python que regenera index.html desde los masters
└── README.md          # Este archivo
```

---

## Historial de migración

Octubre 2026 – La app pasó del repo personal `MarianoErbino/app-vendedores` a la organización corporativa `shimano-arg/app-vendedores` para garantizar continuidad e independencia.

| Pieza | Antes | Después |
|---|---|---|
| Repo GitHub | `MarianoErbino/app-vendedores` | `shimano-arg/app-vendedores` |
| URL pública | `marianoerbino.github.io/app-vendedores/` | `shimano-arg.github.io/app-vendedores/` |
| Admin bootstrap | `erbinomariano@gmail.com` | `bot.shimano.pesca@gmail.com` |
| Firebase Owner | `erbinomariano@gmail.com` | `bot.shimano.pesca@gmail.com` (+ MarianoErbino como backup) |
| Billing | Pendiente (Blaze a activar) | Tarjeta corporativa Shimano (en curso) |

La URL vieja queda redirigiendo unas semanas para transición suave. Los datos en Firestore no se tocaron: vendedores, pedidos, visitas, campañas, todo intacto.

---

## Sistema de roles (9 usuarios)

| Rol | Quién | Qué ve | Qué puede modificar |
|---|---|---|---|
| **admin** | bot.shimano.pesca + Mariano (backup) | Todo (zonas, pedidos, contactados, campañas, visitas, usuarios, dashboard global con filtro por vendedor, log de operaciones) | Todo + crear campañas + asignar/eliminar roles + **borrar todo el historial** (zona de peligro con clave) |
| **vendedor** (×6) | Gonzalo, Federico, Boiero, Gil, Ioannis, Santiago | Solo su zona (selector bloqueado), solo sus propios pedidos y visitas | Crear pedidos / visitas en su zona |
| **viewer** (×2) | Diego, Santiago | Todo (igual que admin, dashboard con filtro por vendedor) | Nada — botones de escritura deshabilitados |

**Bootstrap del admin:** la primera vez que `bot.shimano.pesca@gmail.com` entra, la app le asigna `role=admin` automáticamente. El resto entra como `role=unassigned` y ve una pantalla "Acceso pendiente" hasta que el admin los habilite.

**Activar un usuario nuevo:**
1. La persona entra al link, hace login con Google → la app crea `roles/{uid}` con `role=unassigned`.
2. El admin abre **Usuarios** desde el header, elige rol (`vendedor` + vendor asignado, `viewer` o `admin`) y guarda.
3. El usuario refresca y entra con su rol activo.

**Eliminar acceso:** botón rojo **Eliminar** en el panel Usuarios → confirma → borra el doc `roles/{uid}` de Firestore. El usuario pierde acceso al instante (su cuenta Google sigue existiendo, no se toca).

---

## Funcionalidades por sección

### 1) Mapa interactivo
- 24 polígonos provinciales + 527 polígonos de departamentos/partidos pintados según el vendedor asignado.
- 277 localidades con burbujas (visibles en desktop, ocultas en mobile para liberar pantalla).
- Selectores encadenados **Zona → Provincia → Localidad**.
- Vendedor con su zona bloqueada (no puede ver las demás).
- Click en una burbuja drillea (vendor → provincia → localidad).

### 2) Sidebar con 3 tabs

**Localidades**: tarjetas por localidad, click hace fly-to en el mapa.

**Clientes**: lista de clientes + prospectos del filtro activo.
- **Checkbox** = contactado. Marcar contactado pinta verde la burbuja del mapa y el polígono del departamento cuando el 100% de los clientes están contactados.
- **Click** (mobile) o **doble-click** (desktop) → modal con **historial de venta MercadoLibre** + recomendaciones por provincia.

**Pedidos** (3 sub-vistas con buscador por nombre de tienda):

- **Crear**: lista todos los clientes contactados. Click → pregunta confirmación → **si hay campañas activas muestra un alert recordando** objetivos vigentes → abre la ventana de pedido.
- **Pendientes**: pedidos confirmados por el vendedor (con mes), pendientes de confirmación definitiva. Click → modal con **solo la lista de sugeridos clickeables**. Al tocar un sugerido te pregunta cuántas unidades agregar. Footer con botones **Eliminar / Volver a borrador / Confirmar definitivo**.
- **Confirmados**: pedidos finales en modo solo lectura. Click → vista limpia con **solo los productos del pedido, precio y cantidad**. Nada más.

### 3) Ventana de pedido (Crear)
- **Vista única picker** ocupando todo el ancho: 3 filtros **encadenados** (Categoría → Familia → Subfamilia) + búsqueda por nombre/código.
- Cada producto agregado muestra la **cantidad** dentro del botón "+" (ej. tocar 3 veces convierte el "+" en un "3" celeste).
- Botón **"Confirmar pedido"** → abre el **modal de revisión** (celeste) con:
  - Cabecera con cliente + provincia + localidad.
  - Lista editable de productos con precio unitario, cantidad y eliminar.
  - Total al pie (productos + unidades + $).
  - **Volver a editar** o **Confirmar y enviar a Pendientes** → diálogo de mes/año → pasa a Pendientes.

### 4) Sección VISITA (botón violeta del header)
Formulario completo:
- **Localidad** y **Tienda de pesca** en cascada (solo clientes existentes de la zona).
- Tipo de tienda, Local (físico/ecommerce), Tamaño, Fidelidad.
- **Relevancia** (escala Likert 1-5).
- **POP** Sí/No → si Sí, pide Necesidad puntual.
- **Espacio**: hasta 8 fotos (cámara o galería, comprimidas a ~100KB c/u).
- Oportunidad, Lo más vendido Shimano, Lo que más preguntan, Ayuda a tienda (textareas).
- **Frente del local**: 1 foto obligatoria.
- Tipo de venta (Mostrado/Ecommerce/Ambos) → ponderación % si aplica.
- Competencia, Categoría del cliente.
- Antes de enviar pide confirmación: *"¿Seguro querés enviar el formulario de X del mes Y de Z?"*

**Sub-pestaña "Mis visitas"** con 4 filtros stacked:
- Vendedor (solo admin/viewer)
- Mes, Tienda, Año

Click en una visita → abre el formulario en **modo solo lectura** con todos los datos cargados + fotos.

### 5) Dashboard (botón celeste arriba a la derecha)
- **Filtro por vendedor** (admin/viewer): Todos o vendedor individual.
- **Mes en curso**: unidades + facturado ARS + USD.
- **Acumulado anual YTD**: ídem.
- **Top 5 productos** y **Top 5 clientes** del año.
- **3 cards de Targets en USD** con barra de progreso (Jul 2026, Jul-Dic 2026, 2027).
- **Campañas activas** con barra de progreso.

### 6) Campañas comerciales (botón violeta admin)
Form en 3 pasos en cascada:
1. **Familia** → dropdown con las 28 familias del master.
2. **Subfamilia** → filtrada por la familia elegida.
3. **SKUs incluidos** → checkbox list con todos los SKUs de esa subfamilia (botones "Marcar todos" / "Desmarcar todos"). Contador al pie de cuántos van marcados.

Otros campos: nombre, objetivo (Unidades / Monto $ toggle) + monto, fechas desde/hasta.

Apenas se publica, todos los vendedores la ven en su Dashboard.

**Cuando un vendedor empieza un nuevo pedido, aparece un alert recordando todas las campañas vigentes** (con familia, subfamilia, cantidad de SKUs, objetivo y fecha de vigencia).

El progreso se mide matcheando el **código SKU** de cada línea de pedido confirmado contra la lista de SKUs de la campaña.

### 7) Targets en USD
- Cargados al build desde `MASTERFILES/ZONAS/TARGETS VENDEDORES-ZONAS.xlsx` → hoja **TARGET VENDEDORES**.
- Tres tipos por vendedor: Julio 2026, Julio-Diciembre 2026, 2027.
- Todo en USD. Tipo de cambio leído del mismo Excel (1500 ARS/USD).

### 8) Export a Excel (4 opciones)
Botón verde "Exportar a Excel" abre un diálogo: **¿Cómo querés descargar?**

| Formato | Cuándo | Hojas |
|---|---|---|
| **XLS Excel ejecutivo** | Mandar al gerente / equipo | Consolidado + una hoja por vendedor + Visitas + **Contactados** + **Log Operaciones** |
| **PBI Power BI** | Importar en Power BI Desktop como modelo estrella | Fact_Pedidos + Dim_Vendedor + Dim_Producto + Dim_Cliente + Dim_Calendario + Dim_Campania + Fact_Visitas + **Contactados** + **Log_Operaciones** + Parametros |
| **ML Python / IA / ML** | Análisis con pandas | `master_ml` (tabla larga con 24 columnas) + productos_catalogo + universo_clientes + targets_long + campanias + visitas + **contactados** + **log_operaciones** + parametros |
| **ZIP Fotos de visitas** | Auditar / archivar visualmente | ZIP organizado por `Vendedor / Tienda_FechaYYYYMMDD / { frente.jpg, espacio_1.jpg, espacio_2.jpg, ... }` |

### 9) Zona de peligro (solo admin)
- Dentro del modal **Usuarios**, sección con borde rojo: **"Borrar todo el historial"**.
- Click → prompt pidiendo contraseña (`1234`) → confirmación final → borra `pedidos`, `visits`, `campaigns`, `operations_log` y limpia `userData`. Los roles de usuarios se conservan.
- Antes de borrar, se registra un log final `PURGE_TOTAL_HISTORIAL` en `operations_log`.
- La app se recarga sola al terminar.

### 10) Log de operaciones (admin/viewer)
Cada operación destructiva se loggea automáticamente en Firestore (`operations_log`):

| Acción | Cuándo |
|---|---|
| `cancelar_borrador_pedido` | Vendedor cancela un pedido sin confirmar |
| `eliminar_pendiente` | Vendedor elimina un pedido pendiente |
| `volver_a_borrador` | Vendedor mueve un pendiente de vuelta a Crear |
| `eliminar_campaña` | Admin borra una campaña |
| `eliminar_usuario` | Admin borra acceso a un usuario |
| `PURGE_TOTAL_HISTORIAL` | Admin ejecuta la zona de peligro |

Cada entrada del log guarda: fecha exacta, email del usuario, rol, acción, nombre de la entidad afectada y detalles (qué tenía dentro). Solo admin/viewer ven el log en el export Excel.

---

## Stack técnico

- **HTML5 + Vanilla JS** (sin framework, sin bundler).
- **Leaflet 1.9.4** (CDN) para mapa y polígonos.
- **CartoDB Positron** como tile layer.
- **SheetJS xlsx 0.18.5** (CDN) para generar Excel en el navegador.
- **JSZip 3.10.1** (CDN) para empaquetar las fotos en un ZIP descargable.
- **Firebase 10.7.1 compat SDK** (CDN): Auth (Google) + Firestore + persistencia offline IndexedDB.
- **Python 3 + openpyxl** para el build offline (lee Excels y genera HTML autosuficiente).
- **localStorage** como cache local + buffer offline.

## Modelo de datos en Firestore

```
roles/{uid}              { email, displayName, role, vendor, createdAt, assignedBy }
userData/{uid}           { email, displayName, contacted: [keys], orders: {key: lines}, lastSeen }
pedidos/{auto_id}        { ownerUid, ownerEmail, stage, key, tipo, province, locName, clientName,
                           month, monthIdx, year, confirmedAt, finalizedAt, lines: [...] }
campaigns/{auto_id}      { name, familia, subfamilia, skus: [...], filterType, filterValues,
                           targetType, targetAmount, startDate, endDate, createdBy, createdAt }
visits/{auto_id}         { ownerUid, ownerEmail, vendor, provincia, localidad, tienda, tipo, local,
                           tamano, fidelidad, relevancia, pop, necesidadPuntual, espacio: [base64],
                           oportunidad, masVendido, masPreguntan, ayudaTienda, frenteLocal: base64,
                           tipoVenta, ponderacionMostrado, ponderacionEcommerce, competencia,
                           categoriaCliente, fecha, mes, anio, createdAt }
operations_log/{auto_id} { userUid, userEmail, userRole, action, entityType, entityName,
                           details: {...}, timestamp }
```

**Reglas de seguridad** (publicadas en Firestore Console):
- `roles`: solo el propio uid lee el suyo, admin lee/escribe todo.
- `userData`: cada uid escribe el propio; admin/viewer pueden leer todos.
- `pedidos`: admin/viewer/vendor leen (para sugerencias cruzadas); admin escribe todo, vendor solo los propios.
- `campaigns`: todos los autenticados leen; solo admin crea/modifica/borra.
- `visits`: todos los autenticados leen; admin y vendor crean las suyas; admin puede modificar todas.
- `operations_log`: solo admin/viewer leen; cualquier autenticado puede crear su propio log; nadie puede editar/borrar (inmutable).

## Fuentes de datos del build

El script `build_app.py` lee y embebe en `index.html`:

- `MASTERFILES/PROSPECTOS/MAPAS/Mapa_Argentina_Shimano.html` → polígonos provinciales (24 provincias).
- `MASTERFILES/PROSPECTOS/MAPAS/Mapa_<Provincia>_Shimano.html` (25 archivos) → 277 localidades con lat/lon, clientes/prospectos + 527 polígonos de departamentos/partidos.
- `MASTERFILES/ZONAS/TARGETS VENDEDORES-ZONAS.xlsx` → 1) clientes y zonas para asignar locality→vendedor, 2) hoja `TARGET VENDEDORES` con targets USD y tipo de cambio.
- `FORECAST/DATOS_CRUDOS/Masterfile Shimano Venta Ult 365 Días.xlsx` → MercadoLibre últimos 365 días (~12.700 órdenes, 175 sellers) para el modal "Historial + recomendaciones".
- `MASTERFILES/PRODUCTO/MASTERFILE PRODUCTOS PESCA.xlsx` → 665 SKUs con código, descripción, categoría, familia, subfamilia.

Salida: `index.html` autosuficiente (~2.5 MB).

---

## Regenerar `index.html` cuando cambien los masters

```powershell
python "C:\Users\shimano.sandbox\Desktop\APP VENDEDORES\build_app.py"
Copy-Item "C:\Users\shimano.sandbox\Desktop\MASTERFILES\PROSPECTOS\MAPAS\Mapa_Argentina_Shimano_Zonas.html" "C:\Users\shimano.sandbox\Desktop\APP VENDEDORES\index.html"
cd "C:\Users\shimano.sandbox\Desktop\APP VENDEDORES"
git add index.html
git commit -m "data refresh"
git push
```

GitHub Pages republica en 30-90 segundos. Los datos en Firestore (contactados, pedidos, visitas, campañas, roles, log) **no se tocan** — solo se actualiza el HTML.

## Optimizaciones mobile

- Sin auto-zoom de iOS en inputs (font-size: 16px global + viewport meta + touch-action manipulation).
- **No se puede hacer zoom** en la app (intencional para evitar deformaciones).
- En la sección Pedidos / Visitas, los formularios y modales se acomodan en una columna.
- En Pedido Crear: filtros Categoría / Familia / Subfamilia / Buscar uno debajo del otro, todos del mismo ancho.
- Los doble-click se convierten en **single-tap** (más natural en celulares).
- Mapa oculto en mobile (libera espacio).
- Header en mobile: logo Shimano (90×32 px) a la izquierda + badge ADMIN a la derecha en la fila 1, botones del header en fila 2, título centrado en fila 3.
- Picker y Pedido actual con scroll interno controlado (max-height: 50vh cada uno) para evitar 3 niveles de scroll superpuestos.
- Splash de inicio (logo Shimano centrado, fade-in + zoom suave, fondo blanco).

## Detalles de UI / experiencia

- **Splash al iniciar**: logo Shimano centrado sobre fondo blanco con fade-in + zoom suave (~3 s).
- **Logo en el header**: click pregunta "¿Recargar la aplicación?" → refresca la página (útil para forzar resync).
- **Cartelito negro abajo-izquierda**: feedback de sincronización ("Conectado a la nube", "Pedido pendiente guardado", "Campaña creada", "Visita registrada", "Pedido confirmado definitivamente", "N fotos descargadas", etc.).
- **Paleta por zona**:

| Zona | Vendedor | Color |
|---|---|---|
| Z1 | Gonzalo De La Rosa | `#00A9E0` (celeste CIAM) |
| Z2 | Federico Castelanelli | `#003366` (navy) |
| Z4 | Martin Boiero | `#E83A2E` (rojo) |
| Z5 | Mauricio Gil | `#F97316` (naranja) |
| Z6 | Ioannis Palkoudakis | `#8E44AD` (violeta) |
| Z7 | Santiago Esteban | `#F39C12` (ámbar) |

- **Verde de cumplimiento** (`#10b981`): contactados, polígonos al 100%, barras de progreso al alcanzar target.

---

## Backups y seguridad

- **Datos persisten en Firestore** (server-side). Subir nuevas versiones del HTML no toca la base.
- **Cambios de código nunca eliminan datos** salvo migraciones de schema explícitas (siempre con código de migración).
- **GitHub guarda el historial de commits** → ante un bug, `git revert` vuelve al estado anterior sin perder datos.
- **localStorage funciona offline** y sincroniza al volver la conexión.
- El **Excel exportado** es backup manual paralelo (3 formatos + ZIP de fotos).
- **Log de operaciones** registra cualquier acción destructiva con email + timestamp → trazabilidad total.
- **Zona de peligro** del admin: borrado total con doble seguridad (clave `1234` + confirmación final) + log automático. Roles preservados.

## Costos operativos (pricing Firebase)

A volumen actual (6 vendedores × 15 pedidos/día):

| Período | Storage acumulado | Costo mensual |
|---|---|---|
| Mes 1-3 | < 0.5 GB | **USD 0** (free tier) |
| Mes 4-6 | < 1 GB | **USD 0** (free tier) |
| Mes 7-12 | 1-4 GB | **USD 1-3/mes** |
| Año 2 | 4-8 GB | **USD 3-6/mes** |
| Año 3+ | 8-20 GB | **USD 5-12/mes** |

> Mientras esté en Spark plan (free), nada que pagar. Cuando se acerque al límite de 1 GB de storage, el admin pasa a Blaze (Pay-as-you-go) con tarjeta corporativa y budget alert de USD 30/mes como red de seguridad.

## Roadmap (próximos pasos sugeridos)

- **Activar Blaze** con tarjeta corporativa Shimano + budget alert (pendiente, en evaluación con gerencia).
- **Pre-invitar por email** (admin carga un email + rol antes de que la persona se loguee).
- **Migrar fotos a Firebase Storage** para reducir costos de Firestore en el largo plazo (cuando la carga lo amerite).
- **PWA** (manifest + service worker) para instalar como app en celu y operar 100% offline.
- **Notificaciones push** cuando el gerente crea una campaña o cuando se acerca el deadline de un pedido pendiente.
- **Dominio personalizado** (ej. `app.shimano.com.ar`) en lugar de `shimano-arg.github.io`.
- **Edición de visitas** (hoy son solo lectura una vez enviadas).
- **Integración con SAP / ERP** para que pedidos confirmados se carguen automáticamente.

---

## Créditos y contacto

App desarrollada para Shimano Argentina (transición Baraldo → venta directa). Mantenida originalmente por Mariano Erbino (data scientist Shimano). Ownership migrado a la organización corporativa `shimano-arg` para garantizar continuidad. Para reportar bugs o sugerir mejoras: `bot.shimano.pesca@gmail.com`.

> Última actualización del README: octubre 2026 (post-migración a `shimano-arg`).
