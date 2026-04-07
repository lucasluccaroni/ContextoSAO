# Contexto del Proyecto — SAO Bar

## Descripción del negocio
Bar que abre los viernes. Un solo operador maneja todo desde la caja.
La máquina de caja corre macOS. Reemplaza un Excel artesanal.

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
3. **Caja del Día** (pantalla separada, no integrada en Comandas)
4. **Cierre de Caja** (incluye carga de gastos como paso previo)

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

### Flujo técnico del login
1. Firebase Auth valida email + password → devuelve `user` con `uid`
2. La app hace `getDoc(doc(db, "usuarios", uid))` → obtiene el rol
3. El rol se guarda en `AuthContext` (contexto global de React)
4. React Router protege rutas con un componente `<ProtectedRoute>`

Rutas protegidas:
- `/productos` y `/comandas` — requieren login (cualquier rol)
- `/caja` y `/cierre` — requieren rol `admin`
- `/login` — pública, redirige si ya hay sesión activa

### Seguridad en Firestore (Security Rules)
Las rutas protegidas en React son la primera capa (UI). Las Firestore Security Rules son la segunda: corren del lado de Firebase y bloquean lecturas/escrituras no autorizadas aunque el usuario acceda directamente a la base de datos desde la consola del navegador. Las colecciones `cierresDeCaja` y datos de caja se restringen a rol `admin`.

### Topnav con roles
- **Admin:** muestra Productos / Comandas / Caja / Cierre
- **Empleado:** muestra solo Productos / Comandas
- Lado derecho del topnav: nombre del usuario logueado + botón "Cerrar sesión" (conviven con la fecha existente)

---

## Estado del diseño en Figma

- **Archivo:** `YyrxEN0RhwdqKY05xdeZ5U` ("SAO Bar — Wireframes")
- **Resolución:** 1440×900, dark mode
- **Fuente usada en scripts:** Inter (Regular, Medium, Bold)

### Frames completados

| Frame | Posición |
|---|---|
| ABM de Productos | x=0, y=0 |
| Comandas (original) | x=0, y=0 (aprox) |
| Panel derecho con historial (variante actualizada) | x=160, y=1000 |
| Modal — Comanda enviada | x=1600, y=0 |
| Modal — Error stock insuficiente | x=1600, y=560 |
| Modal — Error de conexión | x=2360, y=560 |
| Ticket — Cocina | x=3400, y=0 |
| Login | x=4200, y=0 |

> El frame original de Comandas queda como referencia hasta que se confirme
> que la variante con historial está lista. Borrar el viejo manualmente.

### Convención visual del topnav

Todos los frames comparten el mismo topnav horizontal de 56px de alto:
- Izquierda: logo placeholder + "SAO Bar"
- Centro: links de navegación — Productos / Comandas / Caja / Cierre (condicional por rol)
- Derecha: fecha + nombre de usuario + botón cerrar sesión
- Link activo: texto en color primario + línea de subrayado azul debajo
- Link inactivo: texto secundario, sin subrayado

> ⚠️ El frame Comandas fue creado inicialmente con un sidebar vertical
> izquierdo (inconsistente). Se corrigió manualmente eliminando el sidebar,
> corriendo el contenido 64px a la izquierda, y agregando el topnav
> horizontal. Todos los frames nuevos deben usar este patrón desde el inicio.

### Login — convenciones visuales
- Sin topnav — única pantalla fuera del shell principal
- Tarjeta centrada (360px de ancho) sobre fondo `#111111`
- Logo placeholder: rectángulo punteado 48×48px, igual al del topnav
- Sin link de registro ni "¿Olvidaste tu contraseña?" — sistema cerrado
- Pie de pantalla: "¿Problemas para acceder? Contactá al administrador."
- Estado de error: borde rojo en ambos inputs + mensaje inline debajo del campo contraseña. Borde rojo en ambos (no solo uno) porque Firebase no especifica cuál de los dos falló.

### Logo placeholder en topbar

El topbar incluye un rectángulo punteado de 28×28px como placeholder del
logo del cliente. Incluye una anotación "← logo .png" visible en el diseño.
Al implementar, se reemplaza con un `<img src={logo}>` en el componente
del topbar. El cliente aún no entregó el PNG.

### Layout del ABM de Productos

Patrón master-detail horizontal:
- **Panel izquierdo (tabla):** ancho dinámico, ocupa el espacio restante
- **Divisor:** 1px vertical, color `#2e2e2e`
- **Panel derecho (formulario):** ancho fijo 300px, fondo ligeramente más
  oscuro que el panel principal
- **Modal "Nuevo producto":** overlay centrado sobre el frame completo,
  independiente del panel lateral. Se abre al hacer click en "+ Nuevo
  producto". Contiene los mismos campos que el formulario lateral más
  el comportamiento de stock inicial descrito abajo.

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

### Detalle visual del campo stock actual

En el formulario (panel lateral y modal), el campo "Stock actual" tiene
tratamiento visual diferenciado para dejar claro que es solo lectura:
- Fondo más oscuro que los inputs editables (`#141414`)
- Texto en color gris tenue + estilo itálica
- Hint debajo: `"Solo lectura — se descuenta con cada comanda"`

En el **modal de alta**, el valor del campo muestra el texto:
`"= stock inicial al crear"` — comunica explícitamente la lógica de
negocio: al crear un producto, `stock = stockInicial` de forma automática.

---

## Modelo de datos Firestore

### `productos/{productoId}`
```js
{
  nombre: string,
  categoria: string,       // "cerveza" | "trago" | "sinAlcohol" | "comida"
  precio: number,
  stockInicial: number,    // se carga cada viernes antes de abrir
  stock: number,           // se decrementa con cada comanda (read-only en UI)
  unidad: string,          // "unidad" | "porcion" | "litro" | "ml"
  activo: boolean
}
```

### `comandas/{comandaId}`
```js
{
  numero: number,          // secuencial legible (#42, #43...)
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
      stockInicial: number,
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

### Colecciones de solo lectura (se editan por consola Firebase, sin UI)
- `categorias`: `{ id, nombre, label }` — categorías de productos
- `categorias_gastos`: `{ id, nombre, label }` — categorías de gastos
- `unidades`: `{ id, nombre, label }` — unidades de medida de productos

---

## Decisiones arquitectónicas tomadas

### Autenticación y roles

**Firebase Auth + Firestore para roles** (sin Custom Claims). Los Custom Claims requieren Firebase Admin SDK, que solo corre en servidor o Cloud Functions — incompatible con el stack sin backend. El rol se lee desde `usuarios/{uid}` inmediatamente después del login y se propaga vía `AuthContext`.

**`<ProtectedRoute>`** en React Router intercepta rutas restringidas y redirige si el rol no es suficiente. Las Firestore Security Rules son la segunda capa independiente de la UI.

### Módulo de Comandas — flujo "Enviar comanda"

**Operación atómica:** `runTransaction()` de Firestore. Nunca `batch write`.
Dentro de la transacción en un solo commit:
1. `get` contador (`contadores/comandas`)
2. `get` de cada producto involucrado
3. Validar `stock >= cantidad` por cada ítem
4. `update` stock de cada producto
5. `set` nueva comanda con `numero = ultimo + 1`
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
- Contenido: `SAO BAR` — fecha — hora — `#42` grande — ítems (nombre + cantidad)
- Implementación: `window.print()` con `@media print`. Sin ESC/POS.
- ⚠️ Cliente posiblemente tiene ticketera térmica. Modelo desconocido,
  en averiguación. Diseño actual en 80mm estándar, ajustar si hace falta.

### Historial de comandas en panel derecho

- Vive debajo del botón "+ Nueva comanda"
- Separado por línea divisoria + label "HISTORIAL"
- Campos por fila: `número | total | medio de pago` — solo lectura, sin acciones
- Las más recientes arriba
- **Query:** `where fecha >= inicioDia, orderBy fecha desc, limit(10)`
- Listener en tiempo real con `onSnapshot`

### Lógica de negocio — Gastos

Los gastos se cargan el mismo viernes, al momento del cierre de caja.
La fecha del gasto es `serverTimestamp()` — automática, no editable.
El cierre es por jornada: los gastos cargados en esa sesión son exactamente
los que entran en ese cierre, sin lógica de rangos de fechas.

### Módulo Cierre de Caja — flujo de dos pasos

**Paso 1 — Carga de gastos**
- Lista editable en memoria (`useState` local, nada se escribe aún)
- Campos por ítem: categoría (select desde `categorias_gastos`) + monto
- Botón "Continuar" → persiste todos los gastos con `batch write` y avanza

**Paso 2 — Resumen y cierre**
- Ventas, gastos, ganancia neta, stock restante
- Botón "Cerrar caja" → escribe un documento en `cierresDeCaja` pre-calculado

**Por qué `batch write` en gastos y no `runTransaction`:** no hay validaciones
cruzadas. `runTransaction` se reserva para Comandas donde hay stock que validar.

---

## Comportamientos de UI pendientes
*(sin wireframe — se resuelven al codificar)*

- Botones `+/−` de cantidad: `useState` local
- Botón "Nueva comanda": limpia el panel si la comanda está a medio hacer
- Efectivo / Mercado Pago: selección exclusiva
- Cantidades: no negativas, no superan el stock disponible

---

## Pantallas pendientes de diseño en Figma

1. Caja del Día
2. Cierre de Caja (flujo de dos pasos: carga de gastos → resumen y cierre)

---

## Patrones y convenciones

- **Figma Scripter:** scripts generados en el chat, ejecutados manualmente
  en el plugin. No usar MCP de Figma para crear elementos (consume cuota Starter).
  Desactivar el conector MCP de Figma entre sesiones.
- **Flujo de diseño:** mockup interactivo en el chat → revisión → ajuste
  → script → ejecución manual. Nunca script sin aprobación del mockup.
- **Variantes en Figma:** cuando se actualiza un frame, se crea uno nuevo
  al lado. El viejo queda como referencia hasta confirmación manual.
- **Nuevo chat por pantalla o módulo** para controlar tokens.
- **Sin backend:** toda la lógica vive en el cliente React + Firebase SDK.
- **`window.print()`** para impresión, no ESC/POS.

---

## Hoja de ruta de desarrollo
*(una vez terminada la fase de diseño en Figma)*

1. Setup del proyecto: Vite + React + Firebase + TailwindCSS + GitHub
2. Login + AuthContext + ProtectedRoute + Firestore Security Rules
3. ABM de Productos
4. Comandas
5. Caja del Día
6. Cierre de Caja (paso 1: carga de gastos → paso 2: resumen y cierre)
7. Historial y reportes *(versión futura)*

---

## Volumen de negocio real (referencia del Excel)

- Recaudación por noche: entre $700.000 y $2.600.000 ARS
- El viernes 21/02 fue el pico registrado: ~$2.617.000 total

---

## Notas de sesión

- El conector de GitHub en claude.ai web no está disponible como conector
  nativo todavía. El flujo de contexto es: pegar este README al inicio
  de cada chat nuevo.
- Cuando arranque el desarrollo, este archivo vive en el repo de código
  como `PROYECTO.md` y Claude Code lo lee directamente.
