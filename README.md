# Shimano - App Vendedores (Market Scan Argentina)

Aplicación web interactiva para que los vendedores de Shimano Argentina gestionen sus zonas, clientes, prospectos y pedidos sobre un mapa nacional. Toda la lógica corre **client-side** (un único `index.html` autosuficiente con datos embebidos), por lo que se puede publicar sin servidor — por ejemplo en **GitHub Pages** — y abrir desde celular o computadora.

---

## ¿Para qué sirve?

Cada vendedor puede:

- Ver en un mapa de Argentina **solo su zona** pintada con su color, ocultando las demás.
- Filtrar por **Zona / Provincia / Localidad** (tres selectores encadenados).
- Recorrer **localidades y clientes/prospectos** de su zona.
- **Marcar clientes contactados** (la localidad y su polígono en el mapa cambian de color a verde a medida que avanza el cubrimiento).
- Hacer **doble-click sobre un cliente** para ver historial de venta MercadoLibre y recomendaciones cruzadas por provincia.
- **Crear, editar, confirmar o cancelar pedidos por cliente** con filtros por Categoría / Familia / Subfamilia / Búsqueda libre.
- Ver una sección de **Sugeridos** dentro del pedido — productos que otras casas de pesca de la misma provincia ya pidieron (basado en los pedidos confirmados del propio sistema), indicando **qué casas lo pidieron**.
- **Exportar todo a Excel** (resumen por vendedor + detalle + pedidos pendientes + pedidos confirmados).

Toda la información sensible se guarda **en el dispositivo de cada usuario** (`localStorage`); no hay backend ni cuenta.

---

## Estructura

```
APP VENDEDORES/
├── index.html       # App lista para abrir (~2.3 MB, incluye datos embebidos)
├── build_app.py     # Script Python que regenera index.html desde los masters
└── README.md        # Este archivo
```

---

## Cómo usar (guía del vendedor)

1. Abrir el link de GitHub Pages (o doble-click en `index.html`).
2. **Selector "Zona"** arriba: elegir su zona (Z1–Z7). El mapa hace zoom a su área y oculta las demás.
3. **Selectores "Provincia" / "Localidad"** se llenan automáticamente con las opciones disponibles dentro de la zona elegida.
4. **Sidebar derecha** (3 pestañas):
   - **Localidades**: tarjetas por localidad, click → fly-to en el mapa.
   - **Clientes**: lista de clientes + prospectos. **Checkbox** = contactado (se pinta verde y la burbuja/polígono del mapa también).
     - Click en una tarjeta → fly-to.
     - **Doble-click** → ventana con **historial MercadoLibre** y **recomendaciones por provincia**.
   - **Pedidos** (3 sub-vistas):
     - **Crear**: lista todos los clientes contactados. **Doble-click** → pregunta `¿Querés crear un pedido para X?` → abre la ventana de pedido.
     - **Pendientes**: pedidos en curso (con productos cargados, sin confirmar).
     - **Confirmados**: pedidos cerrados con su mes/año (modo solo lectura).
5. **Ventana de pedido**:
   - Izquierda: picker de productos con filtros Categoría / Familia / Subfamilia + búsqueda por nombre/código (665 SKUs del master).
   - Derecha: caja amarilla "**Sugeridos para este cliente**" → productos que **otras casas de pesca de la misma provincia** ya confirmaron, mostrando **qué casas lo pidieron** y unidades; click los agrega al pedido.
   - Líneas del pedido con cantidad editable.
   - Footer: **Cancelar pedido** (rojo) elimina todo, **Confirmar pedido** (verde oscuro) pide mes/año y mueve a Confirmados.
6. **Botón "Exportar a Excel"** arriba a la derecha: descarga `.xlsx` con varias hojas: Resumen, una por vendedor, Todos, Pedidos Pendientes, Pedidos Confirmados.
7. Click en una **burbuja del mapa** filtra automáticamente al nivel correspondiente (vendedor → provincia → localidad).
8. En **mobile** la barra y la sidebar se acomodan en una columna; las burbujas se ocultan para no tapar el mapa.

---

## Datos persistentes (localStorage del navegador)

| Clave | Contenido |
|---|---|
| `shimano_zonas_contacted_v1` | Set de clientes/prospectos marcados como contactados |
| `shimano_zonas_orders_v1` | Pedidos pendientes por cliente (objeto con líneas) |
| `shimano_zonas_confirmed_v1` | Pedidos confirmados por cliente × mes/año |

Cada vendedor tiene su propio estado en su navegador. Para sincronizar entre dispositivos hace falta exportar a Excel y compartir.

---

## Fuentes de datos

El `build_app.py` lee y embebe en el HTML los siguientes archivos:

- `MASTERFILES/PROSPECTOS/MAPAS/Mapa_Argentina_Shimano.html` → polígonos provinciales (FeatureCollection 24 provincias).
- `MASTERFILES/PROSPECTOS/MAPAS/Mapa_<Provincia>_Shimano.html` (25 archivos) → localidades con lat/lon y clientes/prospectos + polígonos de departamentos/partidos.
- `MASTERFILES/ZONAS/TARGETS VENDEDORES-ZONAS.xlsx` → mapeo cliente → vendedor → zona.
- `FORECAST/DATOS_CRUDOS/Masterfile Shimano Venta Ult 365 Días.xlsx` → ventas MercadoLibre últimos 365 días (~12.700 órdenes, 175 sellers).
- `MASTERFILES/PRODUCTO/MASTERFILE PRODUCTOS PESCA.xlsx` → 665 SKUs con Categoría / Familia / Subfamilia.

Salida: un único `index.html` ~2.3 MB autosuficiente.

---

## Cómo regenerar `index.html` después de actualizar masters

Cuando cambien los archivos fuente (más clientes, nuevos productos, nuevas zonas), correr el build:

```powershell
python "C:\Users\shimano.sandbox\Desktop\APP VENDEDORES\build_app.py"
```

El script sobrescribe `Mapa_Argentina_Shimano_Zonas.html` en la carpeta MAPAS. Luego copiar ese archivo como `index.html` en esta carpeta (y commit a Git si está versionado):

```powershell
Copy-Item "C:\Users\shimano.sandbox\Desktop\MASTERFILES\PROSPECTOS\MAPAS\Mapa_Argentina_Shimano_Zonas.html" "C:\Users\shimano.sandbox\Desktop\APP VENDEDORES\index.html"
```

**Importante:** las rutas a los masters están hardcodeadas al inicio de `build_app.py` (`MAPS_DIR`, `ZONAS_XLSX`, `MELI_XLSX`, `PRODUCTS_XLSX`). Ajustarlas si se mueven los archivos.

---

## Publicar en GitHub Pages (para que los vendedores accedan por link)

1. Crear un repo en GitHub (público o privado con Pages habilitado — Pages gratuito requiere público).
2. Subir el contenido de esta carpeta:
   ```powershell
   cd "C:\Users\shimano.sandbox\Desktop\APP VENDEDORES"
   git init
   git add index.html README.md
   git commit -m "Initial release - app vendedores"
   git branch -M main
   git remote add origin https://github.com/<USUARIO>/<REPO>.git
   git push -u origin main
   ```
   *(No subir `build_app.py` si no querés exponer rutas internas, o ponerlo en un branch privado).*
3. En GitHub: **Settings → Pages → Source = main → / (root) → Save**.
4. En 1–2 minutos el link queda activo: `https://<USUARIO>.github.io/<REPO>/`.
5. Compartir ese link con los vendedores — funciona en celular y compu sin instalar nada.

**Actualizar la app publicada:** regenerar `index.html` (paso anterior), `git add index.html && git commit -m "data refresh" && git push`.

---

## Stack técnico

- **HTML5 + Vanilla JS** (sin framework, sin bundler).
- **Leaflet 1.9.4** (CDN unpkg) para el mapa y polígonos.
- **CartoDB Positron** como tile layer (OSM).
- **SheetJS xlsx 0.18.5** (CDN jsDelivr) para generar el Excel en el navegador.
- **Python 3 + openpyxl** para el build (script offline).
- **localStorage** para persistencia por vendedor.

---

## Paleta por zona

| Zona | Vendedor | Color |
|---|---|---|
| Z1 | Gonzalo De La Rosa | `#00A9E0` (celeste CIAM Shimano) |
| Z2 | Federico Castelanelli | `#003366` (navy) |
| Z4 | Martin Boiero | `#E83A2E` (rojo) |
| Z5 | Mauricio Gil | `#F97316` (naranja) |
| Z6 | Ioannis Palkoudakis | `#8E44AD` (violeta) |
| Z7 | Santiago Esteban | `#F39C12` (ámbar) |

Verde de contactado: `#10b981`.

---

## Funcionalidades implementadas

- [x] Mapa nacional con polígonos por provincia y departamento.
- [x] 277 localidades con lat/lon y listas de clientes / prospectos.
- [x] Asignación automática locality → vendedor por mayoría.
- [x] Filtro encadenado Zona / Provincia / Localidad.
- [x] Click sobre burbuja drillea (vendedor → provincia → localidad).
- [x] Cluster de burbujas según zoom (3 niveles).
- [x] Checkbox de contactado por cliente con persistencia.
- [x] Polígono y burbuja se pintan verde cuando todos los clientes de la localidad están contactados (parcial = punto verde en la burbuja).
- [x] Tab "Pedidos" con sub-vistas Crear / Pendientes / Confirmados.
- [x] Modal de pedido con picker filtrable (665 SKUs) y caja "Sugeridos".
- [x] Sugeridos basados en **pedidos confirmados de otras casas de la misma provincia**, indicando qué casas lo pidieron.
- [x] Confirmación de pedido con selector Mes / Año.
- [x] Cancelar pedido (rojo) con confirmación.
- [x] Export a Excel con 5+ hojas.
- [x] Adaptación mobile (sidebar full-width abajo, modales fullscreen, burbujas ocultas).
- [x] Doble-click sobre cliente → modal de historial MELI + recomendaciones por provincia.

---

## Roadmap / mejoras posibles

- Backend liviano (Cloudflare Workers + KV o Firebase Firestore) para sincronizar pedidos entre dispositivos del mismo vendedor.
- Login por vendedor (Auth0 / GitHub OAuth) para filtrar por defecto a su zona.
- Histórico real de compras por cliente (cuando esté disponible la data directa, no solo MELI).
- PWA (`manifest.json` + service worker) para instalación en celular y uso offline.
- Métricas: % cobertura por zona, ranking vendedor.

---

## Créditos

App armada para Shimano Argentina (transición Baraldo → venta directa). Datos: equipo comercial Shimano + Mercado Libre + relevamiento propio.
