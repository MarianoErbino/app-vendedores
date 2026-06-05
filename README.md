# Shimano - App Vendedores

App web para que el equipo comercial de Shimano Argentina gestione zonas, clientes, prospectos, pedidos, campañas, visitas y dashboards de ventas. Funciona desde celular o computadora con un único link público (GitHub Pages), con persistencia en la nube (Firebase Firestore) y sincronización en tiempo real entre dispositivos.

**Link público:** https://marianoerbino.github.io/app-vendedores/

---

## ¿Para qué sirve?

- Cada vendedor entra al link, se loguea con su cuenta Google y ve **solo la información de su zona**.
- Recorre **localidades, clientes y prospectos** sobre un mapa interactivo de Argentina y los marca como contactados.
- Arma **pedidos** filtrando por categoría/familia/subfamilia/SKU con búsqueda, los confirma con mes/año (pasan a Pendientes) y luego los confirma definitivamente (pasan a Confirmados).
- En cada paso recibe **recomendaciones de productos** que otras casas de pesca de la misma provincia ya pidieron (red global de vendedores).
- Registra **visitas a tiendas** con un formulario completo (campos obligatorios + opcionales + fotos).
- Tiene un **Dashboard** con ventas del mes/año (unidades + ARS + USD), top productos, top clientes, progreso contra los targets en USD (Jul 2026, Jul-Dic 2026, 2027) y avance de campañas activas.
- El **gerente de ventas y los analistas** (admin/viewer) ven la información agregada de los 6 vendedores, pueden filtrar por vendedor en el dashboard, crear campañas comerciales, gestionar roles de usuarios y vaciar el historial si lo necesitan.
- **Exporta a Excel** en 3 formatos según el uso (ejecutivo / Power BI / Python-ML).

---

## Estructura del repo

```
APP VENDEDORES/
├── index.html         # App lista para abrir (~2.5 MB, datos embebidos + Firebase compat)
├── Shimano-Logo.png   # Logo Shimano (header + splash)
├── build_app.py       # Script Python que regenera index.html desde los masters
└── README.md          # Este archivo
```

---

## Sistema de roles (9 usuarios)

| Rol | Quién | Qué ve | Qué puede modificar |
|---|---|---|---|
| **admin** | Mariano | Todo (zonas, pedidos, contactados, campañas, visitas, usuarios, dashboard global con filtro por vendedor) | Todo + crear campañas + asignar/eliminar roles + **borrar todo el historial** (zona de peligro con clave) |
| **vendedor** (×6) | Gonzalo, Federico, Boiero, Gil, Ioannis, Santiago | Solo su zona (selector bloqueado), solo sus propios pedidos y visitas | Crear pedidos / visitas en su zona |
| **viewer** (×2) | Diego, Santiago | Todo (igual que admin, dashboard con filtro por vendedor) | Nada — botones de escritura deshabilitados |

**Bootstrap del admin:** la primera vez que `erbinomariano@gmail.com` entre, la app le asigna `role=admin` automáticamente. El resto entra como `role=unassigned` y ve una pantalla "Acceso pendiente" hasta que el admin los habilite desde el panel "Usuarios".

**Activar un usuario nuevo:**
1. La persona entra al link, hace login con Google → la app crea `roles/{uid}` con `role=unassigned`.
2. Mariano (admin) abre **Usuarios** desde el header, elige rol (`vendedor` + vendor asignado, `viewer` o `admin`) y guarda.
3. El usuario refresca y entra con su rol activo.

**Eliminar acceso:** botón rojo **Eliminar** al lado de cada usuario en el panel Usuarios → confirma → borra el doc `roles/{uid}` de Firestore. El usuario pierde acceso al instante (su cuenta Google sigue existiendo, no se toca).

---

## Funcionalidades por sección

### 1) Mapa interactivo
- 24 polígonos provinciales + 527 polígonos de departamentos/partidos pintados según el vendedor asignado.
- 277 localidades con burbujas (visibles en desktop, ocultas en mobile para liberar pantalla).
- Selectores encadenados **Zona → Provincia → Localidad**.
- Vendedor con su zona bloqueada (no puede ver las demás).
- Click en una burbuja drillea (vendor → provincia → localidad).

### 2) Sidebar con 3 tabs (Localidades / Clientes / Pedidos)

**Localidades**: tarjetas por localidad, click hace fly-to en el mapa.

**Clientes**: lista de clientes + prospectos del filtro activo.
- **Checkbox** = contactado. Marcar contactado pinta verde la burbuja del mapa y el polígono del departamento cuando el 100% de los clientes están contactados.
- **Click** (mobile) o **doble-click** (desktop) en una tarjeta → modal con **historial de venta MercadoLibre** + recomendaciones por provincia.
- Botón "Limpiar" desmarca todos los contactados de la selección actual.

**Pedidos** (3 sub-vistas con buscador por nombre de tienda):

- **Crear**: lista todos los clientes contactados. Click (mobile) o doble-click (desktop) → pregunta *"¿Querés crear un pedido para X?"* → si hay **campañas activas** muestra un alert recordando objetivos vigentes → abre la ventana de pedido.
- **Pendientes**: pedidos confirmados por el vendedor (con mes), pendientes de confirmación definitiva. Click → modal con la **lista de sugeridos** clickeables. Al tocar un sugerido te pregunta cuántas unidades agregar. Footer con botones **Eliminar / Volver a borrador / Confirmar definitivo**.
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
Formulario de visita a tienda con:
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

**Sub-pestaña "Mis visitas"** con 4 filtros stacked (Vendedor [admin/viewer], Mes, Tienda, Año).
- Click en una visita → abre el formulario en **modo solo lectura** con todos los datos cargados + fotos.

### 5) Dashboard (botón celeste arriba a la derecha)
- **Filtro por vendedor** (admin/viewer): Todos o vendedor individual.
- **Mes en curso**: unidades + facturado ARS + USD.
- **Acumulado anual YTD**: ídem.
- **Top 5 productos** y **Top 5 clientes** del año.
- **3 cards de Targets en USD** con barra de progreso (Jul 2026, Jul-Dic 2026, 2027).
- **Campañas activas** con barra de progreso.

### 6) Campañas comerciales (botón violeta admin)
- Admin crea campañas con:
  - Nombre (ej. "POWER PRO")
  - Filtro: Familia / Subfamilia / Categoría + valor concreto del catálogo
  - Objetivo: toggle de 2 botones **Unidades** / **Monto $** + monto
  - Rango: desde / hasta
- Apenas se publica, todos los vendedores la ven en su Dashboard.
- **Cuando un vendedor empieza un nuevo pedido, aparece un alert recordando todas las campañas vigentes** (con filtro, objetivo y fecha de vigencia).

### 7) Targets en USD
- Cargados al build desde `MASTERFILES/ZONAS/TARGETS VENDEDORES-ZONAS.xlsx` → hoja **TARGET VENDEDORES**.
- Tres tipos por vendedor: Julio 2026, Julio-Diciembre 2026, 2027.
- Todo en USD. Tipo de cambio leído del mismo Excel (1500 ARS/USD).

### 8) Export a Excel (3 formatos)
Botón verde "Exportar a Excel" abre un diálogo: **¿Cómo querés descargar?**

| Formato | Cuándo | Hojas |
|---|---|---|
| **Excel ejecutivo** | Mandar al gerente / equipo | Consolidado + una hoja por vendedor + **Visitas** |
| **Power BI** | Importar en Power BI Desktop | Fact_Pedidos + Dim_Vendedor + Dim_Producto + Dim_Cliente + Dim_Calendario + Dim_Campania + **Fact_Visitas** + Parametros |
| **Python / IA / ML** | Análisis con pandas | `master_ml` (tabla larga con 24 columnas) + productos_catalogo + universo_clientes + targets_long + campanias + **visitas** + parametros |

### 9) Zona de peligro (solo admin)
- Dentro del modal **Usuarios**, sección con borde rojo: **"Borrar todo el historial"**.
- Click → prompt pidiendo contraseña (`1234`) → confirmación final → borra `pedidos`, `visits`, `campaigns` y limpia `userData`. Los roles de usuarios se conservan.
- La app se recarga sola al terminar.

---

## Stack técnico

- **HTML5 + Vanilla JS** (sin framework, sin bundler).
- **Leaflet 1.9.4** (CDN) para mapa y polígonos.
- **CartoDB Positron** como tile layer.
- **SheetJS xlsx 0.18.5** (CDN) para generar Excel en el navegador.
- **Firebase 10.7.1 compat SDK** (CDN): Auth (Google) + Firestore + persistencia offline IndexedDB.
- **Python 3 + openpyxl** para el build offline (lee Excels y genera HTML autosuficiente).
- **localStorage** como cache local + buffer offline.

## Modelo de datos en Firestore

```
roles/{uid}              { email, displayName, role, vendor, createdAt, assignedBy }
userData/{uid}           { email, displayName, contacted: [keys], orders: {key: lines}, lastSeen }
pedidos/{auto_id}        { ownerUid, ownerEmail, stage, key, tipo, province, locName, clientName,
                           month, monthIdx, year, confirmedAt, finalizedAt, lines: [...] }
campaigns/{auto_id}      { name, filterType, filterValues, targetType, targetAmount,
                           startDate, endDate, createdBy, createdAt }
visits/{auto_id}         { ownerUid, ownerEmail, vendor, provincia, localidad, tienda, tipo, local,
                           tamano, fidelidad, relevancia, pop, necesidadPuntual, espacio: [base64],
                           oportunidad, masVendido, masPreguntan, ayudaTienda, frenteLocal: base64,
                           tipoVenta, ponderacionMostrado, ponderacionEcommerce, competencia,
                           categoriaCliente, fecha, mes, anio, createdAt }
```

**Reglas de seguridad** (publicadas en Firestore Console):
- `roles`: solo el propio uid lee el suyo, admin lee/escribe todo.
- `userData`: cada uid escribe el propio; admin/viewer pueden leer todos.
- `pedidos`: admin/viewer/vendor leen (para sugerencias cruzadas); admin escribe todo, vendor solo los propios.
- `campaigns`: todos los autenticados leen; solo admin crea/modifica/borra.
- `visits`: todos los autenticados leen; admin y vendor crean las suyas; admin puede modificar todas.

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

GitHub Pages republica en 30-90 segundos. Los datos en Firestore (contactados, pedidos, visitas, campañas, roles) **no se tocan** — solo se actualiza el HTML.

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
- **Cartelito negro abajo-izquierda**: feedback de sincronización ("Conectado a la nube", "Pedido pendiente guardado", "Campaña creada", "Visita registrada", "Pedido confirmado definitivamente", etc.).
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
- El **Excel exportado** es tu backup manual paralelo (3 formatos, descargable cualquier momento).
- **Zona de peligro** del admin: borrado total con doble seguridad (clave `1234` + confirmación final). Roles preservados.

## Roadmap (próximos pasos sugeridos)

- **Pre-invitar por email** (admin carga un email + rol antes de que la persona se loguee).
- **Export automático diario** de Firestore → Google Cloud Storage.
- **PWA** (manifest + service worker) para instalar como app en celu y operar 100% offline.
- **Notificaciones push** cuando el gerente crea una campaña o cuando se acerca el deadline de un pedido pendiente.
- **Dominio personalizado** (Vercel, Cloudflare Pages o dominio propio en lugar de marianoerbino.github.io).
- **Edición de visitas** (hoy son solo lectura una vez enviadas).
- **Integración con SAP / ERP** para que pedidos confirmados se carguen automáticamente.

---

## Créditos y contacto

App desarrollada para Shimano Argentina (transición Baraldo → venta directa). Mantenida por Mariano Erbino (data scientist Shimano). Para reportar bugs o sugerir mejoras: erbinomariano@gmail.com.

> Última actualización del README: 5 de junio de 2026.
