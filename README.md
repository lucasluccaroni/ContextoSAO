# Contexto del Proyecto — SAO Bar

## Descripción del negocio
Bar que abre los viernes. Un solo operador maneja todo desde la caja.
La máquina de caja corre macOS. Reemplaza un Excel artesanal.
La jornada comienza el viernes a la noche y termina el sábado a la madrugada
(siempre cruza la medianoche), por lo que no puede modelarse por fecha calendario.

> Lucas (desarrollador) trabaja en Windows. La dueña (cliente) usa macOS.
> Las fuentes deben cargarse via Google Fonts o embedidas en el proyecto
> React — nunca depender de instalación local en ningún OS.

## Stack tecnológico
- **Frontend:** React + Vite
- **Base de datos:** Firebase Firestore
- **Autenticación:** Firebase Authentication
- **Estilos:** TailwindCSS
- **Plataforma:** Web, corre en el navegador (macOS, sin restricciones)
- **Sin backend propio:** React se conecta directo a Firebase via SDK

## Módulos del sistema
1. **ABM de Productos**
2. **Comandas** (el más complejo)
3. **Caja del Día** (pantalla separada, solo lectura, tiempo real)
4. **Cierre de Caja** (incluye carga de gastos como paso previo)
5. **Historial y reportes** (solo rol admin, solo lectura — ver alcance en Decisiones arquitectónicas)

---

## Sistema de autenticación y roles

### Herramienta
Firebase Authentication para identidad (email + password) + Firestore para roles. Sin Custom Claims — requieren Firebase Admin SDK que solo corre en servidor. Sin pantalla de registro: los usuarios los crea el desarrollador directamente desde la consola de Firebase.

### Roles y permisos

| Pantalla | `admin` (dueña) | `empleado` |
|---|---|---|
| ABM de Productos | ✅ | ✅ |
| Comandas | ✅ | ✅ |
| Caja del Día | ✅ | ❌ |
| Cierre de Caja | ✅ | ❌ |
| Historial | ✅ | ❌ |

### Flujo técnico del login
1. Firebase Auth valida email + password → devuelve `user` con `uid`
2. La app hace `getDoc(doc(db, "usuarios", uid))` → obtiene el rol
3. El rol se guarda en `AuthContext` (contexto global de React)
4. React Router protege rutas con un componente `<ProtectedRoute>`

Rutas protegidas:
- `/productos` y `/comandas` — requieren login (cualquier rol)
- `/caja`, `/cierre` e `/historial` — requieren rol `admin`
- `/login` — pública, redirige si ya hay sesión activa

### Seguridad en Firestore (Security Rules)
Las rutas protegidas en React son la primera capa (UI). Las Firestore Security Rules son la segunda: corren del lado de Firebase y bloquean lecturas/escrituras no autorizadas aunque el usuario acceda directamente a la base de datos desde la consola del navegador. Las colecciones `cierresDeCaja` y datos de caja se restringen a rol `admin`.

### Topnav con roles
- **Admin:** muestra Productos / Comandas / Caja / Cierre / Historial
- **Empleado:** muestra solo Productos / Comandas
- Lado derecho del topnav: nombre del usuario logueado + botón "Cerrar sesión" (conviven con la fecha existente)

---

## Estado del diseño en Figma

- **Archivo:** `YyrxEN0RhwdqKY05xdeZ5U` ("SAO Bar — Wireframes")
- **Resolución:** 1440×900
- **Fuente definitiva (pendiente de aplicar tras confirmación del cliente):** Creepster (display) + Livvic (UI) — ver sección UI Kit

> ⚠️ Los frames actuales tienen inconsistencias entre sí (tamaños de
> tipografía, posiciones, hex codes, alineación de texto en botones).
> Son válidos como referencia visual pero NO como fuente de verdad de
> estilos. Todos serán re-scripteados cuando el cliente confirme el estilo
> definitivo. Ver sección "Plan de rediseño y consistencia".

### Frames completados

| Frame | Posición |
|---|---|
| ABM de Productos | x=0, y=0 |
| Comandas (original, referencia) | x=0, y=0 (aprox) |
| Comandas con historial y beeper | x=160, y=1000 |
| Modal — Comanda enviada | x=1600, y=0 |
| Modal — Error stock insuficiente | x=1600, y=560 |
| Modal — Error de conexión | x=2360, y=560 |
| Ticket — Cocina con beeper | x=3400, y=0 |
| Login | x=4200, y=0 |
| Caja del Día | x=2000, y=0 (aprox) |
| Cierre — 01 Inicio | x=5156, y=0 |
| Cierre — 02 Modal PIN (normal) | x=5156, y=1043 |
| Cierre — 03 Modal PIN (error) | x=6669, y=1043 |
| Cierre — 04 Paso 1 Gastos (vacío) | x=7400, y=0 |
| Cierre — 05 Paso 1 Gastos (con datos) | x=7400, y=1043 |
| Cierre — 06 Paso 2 Resumen y cierre | x=7400, y=2086 |
| Cierre — 07 Cierre exitoso | x=7400, y=3400 |
| Comandas — con Beeper (rediseño) | x=10000, y=0 |
| UI Kit v2 | x=0, y=2000 |

> El frame original de Comandas queda como referencia hasta que se confirme
> que la variante con historial está lista. Borrar el viejo manualmente.

### Pantallas pendientes de diseño en Figma
1. Historial y reportes (lista de cierres + detalle de cierre)
2. Pantalla de bienvenida / caja cerrada (JornadaGuard)

### Login — convenciones visuales
- Sin topnav — única pantalla fuera del shell principal
- Tarjeta centrada (360px de ancho) sobre fondo `#111111`
- Logo placeholder: rectángulo punteado 48×48px, igual al del topnav
- Sin link de registro ni "¿Olvidaste tu contraseña?" — sistema cerrado
- Pie de pantalla: "¿Problemas para acceder? Contactá al administrador."
- Estado de error: borde rojo en ambos inputs + mensaje inline debajo del campo contraseña. Borde rojo en ambos (no solo uno) porque Firebase no especifica cuál de los dos falló.

### Layout del ABM de Productos
Patrón master-detail horizontal:
- **Panel izquierdo (tabla):** ancho dinámico, ocupa el espacio restante
- **Modal "Nuevo producto":** overlay centrado sobre el frame completo, independiente del panel lateral. Se abre al hacer click en "+ Nuevo producto". Contiene los mismos campos que el formulario lateral más el comportamiento de stock inicial descrito abajo.

> El prototipo ON_CLICK botón → overlay está pendiente de conectar
> manualmente en Figma (Prototype tab → arrastrar trigger al frame modal,
> configurar como Open overlay con animación Dissolve).

### Comportamiento visual de productos inactivos
En la vista **"Todos"** de la tabla, los productos inactivos se muestran
con **opacidad 40%** — visibles pero claramente diferenciados de los activos.
No se ocultan.

La pestaña de filtro **"Inactivos (N)"** muestra el conteo dinámico entre
paréntesis y tiene estilo visual apagado (color y borde más tenues que las
demás pestañas) para distinguirla semánticamente.

### Detalle visual de los campos de stock en el formulario
El formulario de producto (panel lateral y modal de alta/edición) maneja tres campos de stock con comportamientos distintos:

- **Stock ideal** — editable. Hint: `"Cantidad objetivo para reponer cada semana"`
- **Stock inicial** — editable solo en el formulario de edición (la dueña lo toca cuando repone mercadería). Al guardar, `stock = stockInicial` automáticamente. Hint: `"Actualizalo cuando repongas mercadería"`
- **Stock actual** — siempre solo lectura. Fondo más oscuro (`#141414`). Hint: `"Solo lectura — se descuenta con cada comanda"`

En el **modal de alta**, `stockInicial` es editable (la dueña carga el lote inicial). `stockActual` muestra el texto `"= stock inicial al crear"` — comunica que al crear, `stock = stockInicial` automáticamente. `stockIdeal` también es editable desde el alta.

Al guardar cambios sobre un producto existente: si se modificó `stockInicial`, la app también escribe `stock = stockInicial`. Esto representa una reposición de mercadería declarada por la dueña. `stock` nunca se edita directamente desde la UI.

### Layout de Caja del Día
Pantalla de solo lectura y monitoreo en tiempo real. No tiene acciones
que modifiquen datos — toda acción definitiva vive en Cierre de Caja.

Estructura:
- **Tres metric cards** en la parte superior: Efectivo (verde) / Mercado Pago (azul) / Total recaudado (blanco). Cada card muestra también el conteo de comandas.
- **Lista de últimas 10 comandas** de la jornada: número | descripción resumida | pill de medio de pago | total. Solo lectura, sin acciones.
- **Link "Ir al cierre de caja →"** al pie, discreto — solo navegación, no es el botón de cierre.

Query: `where jornadaId == jornadaActiva, orderBy fecha desc, limit(10)`.
Listener en tiempo real con `onSnapshot`.

---

## UI Kit

Página "UI Kit" creada dentro del archivo Figma `YyrxEN0RhwdqKY05xdeZ5U`.
Generada via Figma Scripter. **Versión actual: v2 — basada en el wireframe
de Comandas (con Beeper) como frame de referencia.**

### Tipografías definidas

| Rol | Fuente | Origen | Uso |
|---|---|---|---|
| Display | Creepster | Google Fonts | Logo "SAO Bar" en topnav, títulos decorativos |
| UI | Livvic | Google Fonts | Todo el resto: botones, labels, datos, secciones |

Ambas se cargan via `@import` de Google Fonts. No dependen de instalación local — funcionan en cualquier OS y navegador.

### Paleta de colores
Ver sección `## Convenciones visuales` para la paleta definitiva.

### Componentes documentados en el UI Kit
- Swatches de colores de marca, funcionales y badges de categoría
- Escala tipográfica (Creepster display + Livvic UI en todos sus tamaños)
- Topnav completo con todos sus elementos
- Barra de título de pantalla (56px, fondo `#080A0D`)
- Botones: todos los estados con nota de uso
- Badges de categoría (los 4 con hex exactos)
- Estados de stock: disponible / stock bajo / sin stock
- Cards de producto (3 estados)
- Inputs: normal, placeholder, solo lectura
- Selector de medio de pago
- Métricas de caja

---

## Plan de rediseño y consistencia

### Problema identificado
Los scripts generados hasta ahora tienen valores hardcodeados en cada
conversación por separado. Esto produce drift entre frames: tipografías
que varían 1-2px, hex codes distintos para el mismo color, posiciones
inconsistentes, alineación de texto variable en botones.

Los frames actuales son válidos como referencia visual para mostrarle
al cliente, pero no tienen consistencia suficiente para ser fuente de
verdad de estilos.

### Solución: objeto de tokens + helpers reutilizables

Cuando el cliente confirme el estilo definitivo, antes de tocar cualquier
frame se define un **objeto `T` de tokens** que es la única fuente de
verdad para todos los scripts:

```javascript
const T = {
  // Colores
  bg:        "#080A0D",
  surface:   "#1A1A1A",
  surface2:  "#2E2E2E",
  border:    "#2E2E2E",
  textPrim:  "#F2F2F2",
  textMuted: "#9D9D9D",
  naranja:   "#F26A1B",
  cyan:      "#30CFF2",
  verde:     "#1D9E75",
  azul:      "#378ADD",
  rojo:      "#E2484A",
  amber:     "#BA7517",

  // Tipografía
  fontDisplay: "Creepster",
  fontUI:      "Livvic",
  sizeTitle:   22,
  sizeSubtitle:18,
  sizeBody:    14,
  sizeSmall:   13,
  sizeLabel:   12,
  sizeMeta:    11,

  // Layout
  topnavH:   56,
  titleBarH: 56,
  padPage:   24,
  padCard:   16,
  cardR:     8,
  btnR:      8,
  btnH:      44,
  btnHSm:    40,
  gap:       10,
};
```

Junto con helpers reutilizables: `addBtn(variant)`, `addTopnav(activeLink, userName)`,
`addTitleBar(label)`, `addCard(x, y, w, h)`, `addBadge(label, categoria)`,
`addDot(color)` — que garantizan que cada componente sea idéntico en todos
los frames.

### Plan de ejecución

1. El cliente confirma el estilo definitivo (paleta, tipografía)
2. Se perfecciona y cierra el UI Kit en Figma con los valores finales
3. Se define el objeto `T` final y la librería de helpers
4. Se re-scriptean **todos los frames desde cero** usando solo `T` y helpers — sin valores hardcodeados
5. Los frames viejos se borran manualmente una vez confirmados los nuevos

> Hasta que el cliente confirme: no re-scriptear nada. Los frames actuales
> quedan como referencia visual para la presentación.

---

## Convenciones visuales

> Basadas en el wireframe "Comandas — con Beeper" (x=10000, y=0) —
> frame de referencia para todas las pantallas del sistema.

### Estructura global de pantallas

Todas las pantallas siguen esta estructura:
- **Topnav** — 56px de alto, fondo `#080A0D`, trazo `#F2F2F2`
- **Barra de título** — 56px de alto, fondo `#080A0D`, trazo `#F2F2F2`, título en Livvic Medium 22 `#F2F2F2`
- **Paneles de contenido** — fondo `#F26A1B`, trazo `#F2F2F2` en todos los contenedores grandes para marcar límites

### Topnav

- Logo: PNG del cliente (28×28px)
- "SAO Bar": Creepster Regular 14 `#30CFF2`
- Links inactivos: Livvic Medium 14 `#F2F2F2`
- Link activo: Livvic Medium 14 `#30CFF2` + subrayado `#30CFF2` 2px
- Fecha y usuario: Livvic Medium 12 `#F2F2F2`
- Botón "Salir": fondo `#2E2E2E`, Livvic Regular 12 `#9D9D9D`, hover `#E2484A`

### Tipografía

| Uso | Familia | Estilo | Tamaño |
|---|---|---|---|
| Logotipo "SAO Bar" | Creepster | Regular | 14px |
| Título de pantalla | Livvic | Medium | 22px |
| Subtítulo | Livvic | Medium | 18px |
| Navegación / botones primarios | Livvic | Medium | 14px |
| Cuerpo / botones secundarios | Livvic | Regular | 13px |
| Labels de sección | Livvic | Regular | 12px |
| Hints / anotaciones | Livvic | Regular | 11px |

### Colores — Marca

| Nombre | Hex |
|---|---|
| Negro | `#080A0D` |
| Naranja | `#F26A1B` |
| Naranja alt | `#F25922` |
| Cyan | `#30CFF2` |
| Blanco | `#F2F2F2` |

### Colores — Funcionales

| Nombre | Hex |
|---|---|
| Efectivo | `#1D9E75` |
| Mercado Pago | `#378ADD` |
| Error / Peligro | `#E2484A` |
| Advertencia | `#BA7517` |
| Gris UI | `#9D9D9D` |

### Colores — Badges de categoría

| Categoría | Fondo | Fuente |
|---|---|---|
| cerveza | `#2A1F0A` | `#E6A830` |
| trago | `#1A1F3A` | `#7B9CF6` |
| sin alcohol | `#0E2A1A` | `#4DB87A` |
| comida | `#2A1A1A` | `#D97C7C` |

### Botones

| Botón | Fondo | Fuente | Peso |
|---|---|---|---|
| Acción primaria (Enviar comanda) | `#F2F2F2` | `#080A0D` | Bold 14 |
| Guardar cambios | `#30CFF2` | `#080A0D` | Bold 14 |
| Efectivo | `#1D9E75` | `#F2F2F2` | Regular 13 |
| Mercado Pago | `#378ADD` | `#F2F2F2` | Regular 13 |
| Secundario (+ Nueva comanda) | `#080A0D` | `#F2F2F2` | Regular 13 |
| Cancelar | `#1A1A1A` | `#9D9D9D` | Regular 13 |
| Destructivo (Cerrar caja) | `#E2484A` | `#F2F2F2` | Bold 14 |
| Salir | `#2E2E2E` | `#9D9D9D` | Regular 12 |

### Cards de producto

| Estado | Fondo | Dot | Stock text |
|---|---|---|---|
| Disponible (> 5 u.) | `#1A1A1A` | `#1D9E75` | `#9D9D9D` |
| Stock bajo (≤ 5 u.) | `#1A1A1A` | `#BA7517` | `#BA7517` |
| Sin stock | `#1A1A1A` op. 80% | `#E2484A` | badge `#E2484A` |

### Inputs

| Estado | Fondo | Texto |
|---|---|---|
| Normal | `#1A1A1A` | `#F2F2F2` |
| Placeholder | `#1A1A1A` | `#9D9D9D` |
| Solo lectura | `#141414` | `#9D9D9D` |

---

## Modelo de datos Firestore

### `jornadas/{jornadaId}`
```js
{
  fecha: Timestamp,        // momento en que se abrió la caja (serverTimestamp)
  fechaLabel: string,      // ej: "vie 09/05" — calculado al crear, para mostrar en UI
  estado: string           // "abierta" | "cerrada"
}
```

> **Por qué existe esta colección:** la jornada siempre cruza la medianoche.
> Filtrar por fecha calendario haría que las comandas de las 00:30 quedaran
> en "otro día". La jornada explícita resuelve esto y permite escalar a
> múltiples noches de apertura sin conflicto entre jornadas.

### `productos/{productoId}`
```js
{
  nombre: string,
  categoria: string,       // "cerveza" | "trago" | "sinAlcohol" | "comida"
  precio: number,
  stockIdeal: number,      // meta de stock que quiere tener la dueña al abrir cada jornada — editable en ABM
  stockInicial: number,    // stock al abrir la jornada — el cierre lo actualiza automáticamente
                           // a stock; la dueña lo edita en ABM solo cuando repone mercadería
  stock: number,           // stock actual — se decrementa con cada comanda (read-only en UI);
                           // al guardar cambios en stockInicial desde el ABM, stock = stockInicial
  unidad: string,          // "unidad" | "porcion" | "litro" | "ml"
  activo: boolean
}
```

### `comandas/{comandaId}`
```js
{
  jornadaId: string,       // referencia a jornadas/{jornadaId}
  numero: number,          // secuencial legible (#42, #43...)
  nroBeeper: number | null, // número de beeper entregado al cliente — null si no se usa
  fecha: Timestamp,
  items: [
    {
      productoId: string,  // referencia (para trazabilidad)
      nombreSNAP: string,  // snapshot histórico
      precioUnitSNAP: number,
      cantidad: number,
      subtotal: number
    }
  ],
  total: number,
  medioPago: string,       // "efectivo" | "mercadopago"
  estado: string           // "enviada"
}
```

### `cierresDeCaja/{cierreId}`
```js
{
  jornadaId: string,       // referencia a jornadas/{jornadaId}
  fecha: Timestamp,
  totalEfectivo: number,
  totalMP: number,
  totalGeneral: number,
  totalGastos: number,
  gananciaNeta: number,
  ventasPorCategoria: {
    cerveza: number,
    trago: number,
    sinAlcohol: number,
    comida: number
  },
  stockFinal: [
    {
      productoId: string,
      nombreSNAP: string,
      stockIdealSNAP: number,  // snapshot del ideal al momento del cierre
                               // (faltante se calcula en UI: stockIdealSNAP - restante)
      stockInicial: number,    // stock con el que arrancó la jornada
      vendidas: number,
      restante: number
    }
  ],
  gastosDetalle: [
    {
      categoria: string,
      monto: number
    }
  ],
  gastosIds: [string]
}
```

### `gastos/{gastoId}`
```js
{
  jornadaId: string,       // referencia a jornadas/{jornadaId}
  fecha: Timestamp,        // serverTimestamp() — automático, no editable
  categoria: string,
  monto: number
}
```

### `usuarios/{uid}`
```js
{
  email: string,
  nombre: string,          // nombre para mostrar en el topnav
  rol: string              // "admin" | "empleado"
}
```

> El `uid` es el mismo que genera Firebase Auth al crear el usuario.
> Los usuarios se crean desde la consola de Firebase — no existe pantalla
> de registro en la app.

### `contadores/comandas`
```js
{ ultimo: number }         // se incrementa dentro de runTransaction()
```

### `configuracion/caja`
```js
{
  pinCierre: string        // SHA-256 del PIN de cierre — nunca el PIN en texto plano
}
```

> SHA-256 disponible nativamente vía `crypto.subtle.digest()` sin dependencias.
> Solo legible por rol `admin` (Firestore Security Rules).
> El PIN se hashea en el cliente antes de comparar. Nunca en texto plano.

### Colecciones de solo lectura (se editan por consola Firebase, sin UI)
- `categorias`: `{ id, nombre, label }` — categorías de productos
- `categorias_gastos`: `{ id, nombre, label }` — categorías de gastos
- `unidades`: `{ id, nombre, label }` — unidades de medida de productos

---

## Decisiones arquitectónicas tomadas

### Autenticación y roles
**Firebase Auth + Firestore para roles** (sin Custom Claims). Los Custom Claims requieren Firebase Admin SDK, que solo corre en servidor o Cloud Functions — incompatible con el stack sin backend. El rol se lee desde `usuarios/{uid}` inmediatamente después del login y se propaga vía `AuthContext`.

**`<ProtectedRoute>`** en React Router intercepta rutas restringidas y redirige si el rol no es suficiente. Las Firestore Security Rules son la segunda capa independiente de la UI.

### Modelo de jornada — por qué jornada explícita y no fecha calendario
El bar siempre cruza la medianoche (viernes noche → sábado madrugada).
Filtrar comandas por `fecha >= inicioDelDía` haría que las comandas de las
00:30 quedaran en "otro día". Se adoptó una colección `jornadas` con
`estado: "abierta" | "cerrada"`. Todas las queries filtran por `jornadaId`
en lugar de por fecha. Esto además permite escalar a múltiples noches de
apertura sin conflicto entre jornadas.

### Flujo de apertura y cierre de jornada
**Abrir caja:** al entrar al sistema sin jornada activa, se muestra una
pantalla de bienvenida con botón "Abrir Caja". Cualquier operador puede
hacerlo — no requiere rol `admin`. El click crea el documento en `jornadas`
con `estado: "abierta"` y el `jornadaId` queda disponible para toda la app
via `AuthContext` o contexto dedicado.

**Cerrar caja:** el botón vive al final del flujo de Cierre de Caja (Paso 2).
Requiere PIN de 4 dígitos. El cliente hashea el input con SHA-256 y compara
contra `configuracion/caja.pinCierre`. Si coincide: escribe el documento en
`cierresDeCaja` y actualiza `jornadas/{id}` a `estado: "cerrada"`.

### Pantalla de bienvenida / caja cerrada
Nuevo componente `JornadaGuard` que envuelve toda la navegación:
- Si existe jornada con `estado: "abierta"` → acceso normal a todos los módulos
- Si no hay jornada abierta → pantalla de bienvenida con botón "Abrir Caja"

Pendiente de diseño en Figma.

### Módulo de Comandas — flujo "Enviar comanda"
**Operación atómica:** `runTransaction()` de Firestore. Nunca `batch write`.
Dentro de la transacción en un solo commit:
1. `get` contador (`contadores/comandas`)
2. `get` de cada producto involucrado
3. Validar `stock >= cantidad` por cada ítem
4. `update` stock de cada producto
5. `set` nueva comanda con `numero = ultimo + 1` y `jornadaId` activo
6. `update` contador `ultimo`

Si cualquier paso falla → rollback automático, ninguna escritura se aplica.

**Número secuencial:** documento `contadores/comandas` con campo `ultimo`.
Se incrementa dentro de la misma transacción — garantiza unicidad aunque
haya múltiples operaciones simultáneas.

**Feedback al operador:**
- ✅ Éxito → modal con `#42` grande, lista de ítems, botón "Imprimir", link "Cerrar"
- ❌ Error → modal genérico con línea descriptiva según tipo, solo botón "Cerrar"

**Estado del panel post-envío:**
- Éxito + Cerrar → panel se limpia, listo para nueva comanda
- Error + Cerrar → panel queda intacto para que el operador corrija

**Impresión (ticket de cocina):**
- Formato papel térmico 80mm (área imprimible ~280px en Figma)
- Contenido: `SAO BAR` — fecha — hora — `#42` grande — nro beeper — ítems (nombre + cantidad)
- Implementación: `window.print()` con `@media print`. Sin ESC/POS.
- ⚠️ Cliente posiblemente tiene ticketera térmica. Modelo desconocido, en averiguación. Diseño actual en 80mm estándar, ajustar si hace falta.

### Historial de comandas en panel derecho
- Vive debajo del botón "+ Nueva comanda"
- Separado por línea divisoria + label "HISTORIAL"
- Campos por fila: `número | total | medio de pago | 🔔 beeper` — solo lectura, sin acciones
- Las más recientes arriba
- **Query:** `where jornadaId == jornadaActiva, orderBy fecha desc, limit(10)`
- Listener en tiempo real con `onSnapshot`

### Lógica de negocio — Gastos
Los gastos se cargan el mismo viernes, al momento del cierre de caja.
La fecha del gasto es `serverTimestamp()` — automática, no editable.
Los gastos llevan `jornadaId` y se filtran por él al construir el resumen
del cierre. Las categorías se gestionan desde la consola de Firebase
(`categorias_gastos`) sin intervención del desarrollador.

### Módulo Cierre de Caja — flujo de dos pasos
**Paso 1 — Carga de gastos**
- Lista editable en memoria (`useState` local, nada se escribe aún)
- Campos por ítem: categoría (select desde `categorias_gastos`) + monto
- Sin campo de descripción libre — la categoría es la descripción
- Si no hay gastos, se puede continuar directamente (total gastos = $0)
- Botón "Continuar" no persiste nada — los gastos siguen en memoria

**Paso 2 — Resumen y cierre**
- Ventas por medio de pago, desglose de gastos, ganancia neta, stock final
- Stock final: solo productos con al menos 1 unidad vendida (`vendidas >= 1`)
- Scroll de página completa — sin scroll interno en cards
- Botón "← Volver" regresa al Paso 1 sin pérdida de datos (todo sigue en memoria)
- Botón "Confirmar cierre" → solicita PIN → si es correcto, escribe en un único `batch write`:
  1. Todos los documentos `gastos/{gastoId}` con su `jornadaId`
  2. El documento `cierresDeCaja/{cierreId}` con los totales pre-calculados y su `jornadaId`
  3. Actualiza `jornadas/{jornadaId}` a `estado: "cerrada"`
  4. Por cada producto: `update productos/{productoId}` → `stockInicial = stock` (deja el punto de partida listo para la próxima jornada)

**Por qué todo se escribe al confirmar y no al hacer "Continuar":** evita
documentos huérfanos en Firestore si se abandona el flujo entre pasos.

**Por qué `batch write` y no `runTransaction`:** no hay validaciones
cruzadas. `runTransaction` se reserva para Comandas donde hay stock que validar.

**PIN de seguridad:**
- Vive en `configuracion/caja` como campo `pinCierre` (hash SHA-256)
- Se pide siempre al presionar "Cerrar caja", sin excepción
- Estado de error: borde rojo en input + mensaje inline, input se limpia, botón vuelve a deshabilitarse
- La función de hashing usa `crypto.subtle` nativo del navegador:

```js
async function hashPin(pin) {
  const encoded = new TextEncoder().encode(pin);
  const buffer = await crypto.subtle.digest("SHA-256", encoded);
  const hashArray = Array.from(new Uint8Array(buffer));
  return hashArray.map(b => b.toString(16).padStart(2, "0")).join("");
}
```

**Pantalla de cierre exitoso:**
- Resumen compacto: ventas por medio de pago, desglose de gastos, ganancia neta
- Botón "Imprimir / Guardar PDF" → `window.print()` con `@media print`
- En macOS el diálogo del sistema ofrece "Guardar como PDF" nativo, sin librería adicional
- Botón "Volver al inicio" → regresa a la pantalla de bienvenida (jornada cerrada)

### Módulo Historial y reportes
**Alcance definido (pendiente de diseño y desarrollo):**
- Lista de cierres anteriores: una fila por viernes con fecha, total recaudado y ganancia neta
- Al clickear un cierre: detalle completo — ventas por medio de pago, desglose de gastos, stock final con vendidas, restante y faltante por producto (`faltante = stockIdealSNAP - restante`, calculado en UI)
- Sin queries adicionales — todo viene de `cierresDeCaja` que ya existe
- Acceso: solo rol `admin`, ruta `/historial`
- "Historial" se agrega al topnav del admin junto a los otros links

---

## Comportamientos de UI pendientes
*(sin wireframe — se resuelven al codificar)*

- Botones `+/−` de cantidad: `useState` local
- Botón "Nueva comanda": limpia el panel si la comanda está a medio hacer
- Efectivo / Mercado Pago: selección exclusiva
- Cantidades: no negativas, no superan el stock disponible
- Modal de PIN en Cierre de Caja: input tipo `password`, SHA-256 en cliente, comparación contra `configuracion/caja.pinCierre`

---

## Pantallas pendientes de diseño en Figma
1. Historial y reportes (lista de cierres + detalle de cierre)
2. Pantalla de bienvenida / caja cerrada (JornadaGuard)

> ⚠️ Estas pantallas se diseñarán directamente con el sistema de tokens
> y helpers — una vez que el cliente confirme el estilo definitivo.

---

## Patrones y convenciones
- **Figma Scripter:** scripts generados en el chat, ejecutados manualmente en el plugin. No usar MCP de Figma para crear elementos (consume cuota Starter). Desactivar el conector MCP de Figma entre sesiones.
- **Flujo de diseño:** mockup interactivo en el chat → revisión → ajuste → script → ejecución manual. Nunca script sin aprobación del mockup.
- **Variantes en Figma:** cuando se actualiza un frame, se crea uno nuevo al lado. El viejo queda como referencia hasta confirmación manual.
- **Nuevo chat por pantalla o módulo** para controlar tokens.
- **Sin backend:** toda la lógica vive en el cliente React + Firebase SDK.
- **`window.print()`** para impresión, no ESC/POS.
- **URL para leer el README:** usar siempre `https://github.com/lucasluccaroni/ContextoSAO/blob/main/README.md` — la URL raw tiene caché agresivo y puede devolver versiones viejas.

---

## Hoja de ruta de desarrollo
*(una vez terminada la fase de diseño en Figma)*

1. Setup del proyecto: Vite + React + Firebase + TailwindCSS + GitHub
2. Login + AuthContext + ProtectedRoute + Firestore Security Rules
3. ABM de Productos
4. Comandas
5. Caja del Día
6. Cierre de Caja (paso 1: carga de gastos → paso 2: resumen y cierre)
7. Historial y reportes (lista de cierres + detalle por noche)

---

## Estimación de consumo Firestore por noche

Basado en ~60 comandas por noche (promedio real del Excel):

| Operación | Lecturas | Escrituras |
|---|---|---|
| Comandas (runTransaction) | ~240 | ~300 |
| Historial onSnapshot | ~600 | 0 |
| Caja del Día onSnapshot | ~100 | 0 |
| ABM de Productos | ~50 | ~10 |
| Cierre de Caja | ~60 | ~7 |
| **Total estimado** | **~1.050** | **~317** |

Costo en plan Blaze: < USD 0,01 por noche. El tier gratuito (50.000 lecturas/día)
cubre el consumo con ~98% de margen. Se recomienda plan Blaze con tarjeta
igualmente — no por costo sino para evitar el límite de 1GB de almacenamiento
del plan Spark y habilitar todas las funcionalidades.

---

## Volumen de negocio real (referencia del Excel)
- Recaudación por noche: entre $700.000 y $2.600.000 ARS
- El viernes 21/02 fue el pico registrado: ~$2.617.000 total

---

## Notas de sesión
- El conector de GitHub en claude.ai web no está disponible como conector nativo todavía. El flujo de contexto es: pegar este README al inicio de cada chat nuevo.
- Cuando arranque el desarrollo, este archivo vive en el repo de código como `PROYECTO.md` y Claude Code lo lee directamente.
