# Shimano - App Vendedores

App web para que el equipo comercial de Shimano Argentina gestione zonas, clientes, prospectos, pedidos, campañas y dashboards de ventas. Funciona desde celular o computadora con un único link público (GitHub Pages), con persistencia en la nube (Firebase Firestore) y sincronización en tiempo real entre dispositivos.

**Link público:** https://marianoerbino.github.io/app-vendedores/

---

## ¿Para qué sirve?

- Cada vendedor abre el link, se loguea con su cuenta Google, y ve **solo la información de su zona**.
- Recorre **localidades, clientes y prospectos** sobre un mapa interactivo de Argentina y los marca como contactados.
- Crea **pedidos** filtrando por categoría/familia/subfamilia/SKU, los confirma con mes/año, y los promueve a **confirmados definitivos**.
- Durante el armado del pedido ve **sugeridos**: productos que otras casas de pesca de la misma provincia ya pidieron (red global de vendedores).
- Tiene un **Dashboard** con ventas del mes/año (unidades + ARS + USD), top productos, top clientes, progreso contra los targets en USD (Jul 2026, Jul-Dic 2026, 2027) y avance de campañas activas.
- El **gerente de ventas y los analistas** (admin/viewer) ven la información agregada de los 6 vendedores, pueden filtrar por vendedor en el dashboard, crear campañas comerciales y asignar/revocar roles desde un panel de usuarios.
- **Exporta a Excel** en 3 formatos según el uso (ejecutivo / Power BI / Python-ML).

---

## Estructura del repo

```
APP VENDEDORES/
├── index.html         # App lista para abrir (~2.4 MB, datos embebidos + Firebase compat)
├── Shimano-Logo.png   # Logo Shimano (header + splash)
├── build_app.py       # Script Python que regenera index.html desde los masters
└── README.md          # Este archivo
```

---

## Sistema de roles (9 usuarios)

| Rol | Quién | Qué ve | Qué puede modificar |
|---|---|---|---|
| **admin** | Mariano | Todo (todas las zonas, pedidos, contactados, campañas, usuarios) | Todo + crear campañas + asignar roles |
| **vendedor** (×6) | Gonzalo, Federico, Boiero, Gil, Ioannis, Santiago | Solo su zona (selector bloqueado), solo sus propios pedidos | Solo lo suyo |
| **viewer** (×2) | Diego, Santiago | Todo (igual que admin) | Nada — botones de escritura deshabilitados |

**Bootstrap del admin:** la primera vez que `erbinomariano@gmail.com` ingrese, la app le asigna `role=admin` automáticamente. El resto entra como `role=unassigned` y ve una pantalla "Acceso pendiente" hasta que el admin los habilite desde el panel "Usuarios".

**Para activar un usuario nuevo:**
1. Esa persona entra al link y hace login con Google → la app crea su `roles/{uid}` con `role=unassigned`.
2. Mariano (admin) abre **Usuarios** desde el header, elige el rol (`vendedor` + vendor asignado, `viewer` o `admin`) y clickea **Guardar**.
3. El usuario refresca y ya entra con su rol activo.

---

## Funcionalidades por sección

### 1) Mapa interactivo
- 24 polígonos provinciales + 527 polígonos de departamentos/partidos pintados según el vendedor asignado a cada zona.
- 277 localidades con burbujas (visibles en desktop, ocultas en mobile para liberar pantalla).
- Selectores encadenados **Zona → Provincia → Localidad**.
- Vendedor con su zona bloqueada (no puede ver las demás).
- Click en una burbuja drillea (vendor → provincia → localidad).

### 2) Sidebar con 3 tabs

**Localidades**: tarjetas por localidad, click hace fly-to en el mapa.

**Clientes**: lista de clientes + prospectos del filtro activo. **Checkbox** = contactado.
- Marcar contactado **pinta verde** la burbuja del mapa y el polígono del departamento cuando el 100% de los clientes están contactados.
- **Doble-click en un cliente** → modal con historial de venta MercadoLibre + recomendaciones por provincia.
- Botón "Limpiar" desmarca todos los contactados de la selección.

**Pedidos** (3 sub-vistas):
- **Crear**: lista todos los clientes contactados. Doble-click pregunta *"¿Querés crear un pedido para X?"* → abre la ventana de pedido.
- **Pendientes**: pedidos ya confirmados por el vendedor con mes/año, pero pendientes de confirmación definitiva. Editable + tiene la sección **Sugeridos**.
- **Confirmados**: pedidos finales en modo solo lectura.

### 3) Ventana de pedido
- **Izquierda**: picker de productos con filtros Categoría / Familia / Subfamilia + búsqueda libre (665 SKUs del master).
- **Derecha**: caja amarilla **"Sugeridos para este cliente"** con productos que otras casas de pesca de la misma provincia ya pidieron (en tiempo real, desde Firestore). Indica **qué casas pidieron cada producto**.
- Líneas del pedido con campos editables: **precio unitario ARS** + cantidad. Total en unidades + $ ARS visible.
- Footer:
  - **Cancelar pedido** (rojo) — elimina todo el pedido con confirmación.
  - **Confirmar pedido** (verde) — pide mes/año y mueve a Pendientes.
- En vista Pendientes el footer cambia a: **Volver a borrador** / **Eliminar** / **Confirmar definitivo** → mueve a Confirmados.
- En mobile el modal se vuelve vertical (picker arriba, pedido actual abajo).

### 4) Dashboard (botón celeste arriba a la derecha)
- **Filtro por vendedor** (admin/viewer): "Todos" o vendedor individual.
- **Mes en curso**: unidades + facturado ARS + USD.
- **Acumulado anual YTD**: ídem.
- **Top 5 productos del año**: código + descripción + $ + unidades.
- **Top 5 clientes del año**: nombre + provincia + unidades + $.
- **3 cards de Targets en USD** con barra de progreso:
  - Target Julio 2026
  - Target Jul-Dic 2026 (semestre)
  - Target Anual 2027
- **Campañas activas**: barra de progreso por cada campaña vigente, naranja → verde al 100%.

### 5) Campañas comerciales (admin)
- Botón **Campañas** abre un modal donde el admin crea campañas con:
  - Nombre (ej. "POWER PRO")
  - Filtro: Familia / Subfamilia / Categoría + valor concreto del catálogo
  - Objetivo: USD o unidades + monto
  - Rango: desde / hasta
- Apenas se publica, todos los vendedores vigentes ven la campaña en su Dashboard con su progreso individual.

### 6) Targets de venta
- Cargados al build desde `MASTERFILES/ZONAS/TARGETS VENDEDORES-ZONAS.xlsx` → hoja **TARGET VENDEDORES**.
- Tres tipos: Julio 2026 (mes), Julio-Diciembre 2026 (semestre), 2027 (anual).
- Todo en USD. Tipo de cambio leído del mismo Excel (1500 ARS/USD).
- Progreso = suma de `qty × precio_ARS` en pedidos confirmados ÷ tipo de cambio.

### 7) Export a Excel (3 formatos)
Botón verde "Exportar a Excel" abre un diálogo: **¿Cómo querés descargar?**

| Formato | Cuándo | Hojas |
|---|---|---|
| **Excel ejecutivo** | Mandar al gerente / equipo, lectura humana | Consolidado (1 fila por vendedor con KPI + targets) + una hoja por vendedor (Z1..Z7) con detalle |
| **Power BI** | Importar en Power BI Desktop como modelo estrella | Fact_Pedidos + Dim_Vendedor + Dim_Producto + Dim_Cliente + Dim_Calendario (serie continua) + Dim_Campania + Parametros |
| **Python / IA / ML** | Análisis con pandas / scikit-learn / forecast | `master_ml` (UNA tabla larga con 24 columnas: fecha, year/month/day/quarter/year_month, estado, vendedor, zona, provincia, localidad, departamento, cliente, tipo, contactado, código, producto, categoría, familia, subfamilia, cantidad, precio_unit_ars, subtotal_ars, subtotal_usd) + productos_catalogo + universo_clientes (con lat/lon) + targets_long + campanias + parametros |

---

## Stack técnico

- **HTML5 + Vanilla JS** (sin framework, sin bundler).
- **Leaflet 1.9.4** (CDN) para mapa y polígonos.
- **CartoDB Positron** como tile layer.
- **SheetJS xlsx 0.18.5** (CDN) para generar Excel en el navegador.
- **Firebase 10.7.1 compat SDK** (CDN): Auth (Google) + Firestore + persistencia offline IndexedDB.
- **Python 3 + openpyxl** para el build offline (lee Excels y genera HTML autosuficiente).
- **localStorage** como cache local + buffer offline (la app sigue funcionando sin internet y sincroniza cuando vuelve la conexión).

## Modelo de datos en Firestore

```
roles/{uid}              { email, displayName, role, vendor, createdAt, assignedBy }
userData/{uid}           { email, displayName, contacted: [keys], orders: {key: lines}, lastSeen }
pedidos/{auto_id}        { ownerUid, ownerEmail, stage, key, tipo, province, locName, clientName,
                           month, monthIdx, year, confirmedAt, finalizedAt, lines: [...] }
campaigns/{auto_id}      { name, filterType, filterValues, targetType, targetAmount,
                           startDate, endDate, createdBy, createdAt }
```

**Reglas de seguridad publicadas** (ver Firebase Console):
- `roles`: solo el propio uid lee el suyo, admin lee/escribe todo.
- `userData`: cada uid escribe el propio; admin/viewer pueden leer todos.
- `pedidos`: admin/viewer/vendor pueden leer (para sugerencias cruzadas); admin escribe todo, vendor solo los propios.
- `campaigns`: todos los autenticados leen; solo admin crea/modifica/borra.

## Fuentes de datos del build

El script `build_app.py` lee y embebe en `index.html`:

- `MASTERFILES/PROSPECTOS/MAPAS/Mapa_Argentina_Shimano.html` → polígonos provinciales (FeatureCollection 24 provincias).
- `MASTERFILES/PROSPECTOS/MAPAS/Mapa_<Provincia>_Shimano.html` (25 archivos) → 277 localidades con lat/lon, clientes/prospectos + 527 polígonos de departamentos/partidos.
- `MASTERFILES/ZONAS/TARGETS VENDEDORES-ZONAS.xlsx` → 1) clientes y zonas para asignar locality→vendedor, 2) hoja `TARGET VENDEDORES` con targets USD y tipo de cambio.
- `FORECAST/DATOS_CRUDOS/Masterfile Shimano Venta Ult 365 Días.xlsx` → MercadoLibre últimos 365 días (~12.700 órdenes, 175 sellers) usado para el modal de doble-click "Historial + recomendaciones".
- `MASTERFILES/PRODUCTO/MASTERFILE PRODUCTOS PESCA.xlsx` → 665 SKUs con código, descripción, categoría, familia, subfamilia.

Salida: `index.html` autosuficiente (~2.4 MB).

---

## Cómo regenerar `index.html` cuando cambien los masters

```powershell
python "C:\Users\shimano.sandbox\Desktop\APP VENDEDORES\build_app.py"
Copy-Item "C:\Users\shimano.sandbox\Desktop\MASTERFILES\PROSPECTOS\MAPAS\Mapa_Argentina_Shimano_Zonas.html" "C:\Users\shimano.sandbox\Desktop\APP VENDEDORES\index.html"
cd "C:\Users\shimano.sandbox\Desktop\APP VENDEDORES"
git add index.html
git commit -m "data refresh"
git push
```

GitHub Pages republica en 30-90 segundos. Los datos en Firestore (contactados, pedidos, campañas, roles) **no se tocan** — solo se actualiza el HTML.

## Operaciones día a día

- **Editar reglas de Firestore**: Firebase Console → tu proyecto → Firestore → Rules → editar → Publish.
- **Ver / borrar pedidos directos**: Firebase Console → Firestore Database → Data → colección `pedidos`.
- **Cambiar el rol de un usuario**: Mariano abre Usuarios desde el header, modifica el dropdown, guarda.
- **Backup manual periódico**: corré "Exportar a Excel" cualquiera de los 3 formatos y guardalo con la fecha — así tenés snapshots históricos para tu análisis y como respaldo.

## Detalles de UI / experiencia

- **Splash al iniciar**: logo Shimano centrado sobre fondo blanco con fade-in + zoom suave de 1.2 s; se mantiene visible ~2 s y se quita del DOM a los 3.5 s.
- **Logo en el header**: PNG real de Shimano. Click pregunta "¿Recargar la aplicación?" y al confirmar refresca la página (útil para forzar resync).
- **Splash + header en mobile**: logo en tamaño natural sin estirarse, badge del rol y botones del header organizados en filas centradas.
- **Cartelito negro abajo-izquierda**: feedback de sincronización ("Conectado a la nube", "Pedido pendiente guardado", "Pedido confirmado definitivamente", "Campaña creada/eliminada").
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

## Roadmap (próximos pasos sugeridos)

- **Export automático diario** de Firestore → Google Cloud Storage (snapshots periódicos).
- **PWA** (`manifest.json` + service worker) para instalar el ícono en home del celu y operar 100% offline.
- **Notificaciones push** cuando el gerente crea una campaña o cuando se acerca el deadline de un pedido pendiente.
- **Multi-empresa / multi-país** si Shimano extiende este modelo a Brasil/Chile/Uruguay.
- **Integración con SAP / sistema interno** para que los pedidos confirmados se carguen automáticamente al ERP.

---

## Créditos y contacto

App desarrollada para Shimano Argentina (transición Baraldo → venta directa). Mantenida por Mariano Erbino (data scientist Shimano). Para reportar bugs o sugerir mejoras: erbinomariano@gmail.com.

> Última actualización del README: 5 de junio de 2026.
