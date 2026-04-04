# Contexto del Proyecto — SAO Bar

## Descripción del negocio
Bar que abre los viernes. Un solo operador maneja todo desde la caja.
La máquina de caja corre macOS. Reemplaza un Excel artesanal.

## Stack tecnológico
- **Frontend:** React + Vite
- **Base de datos:** Firebase Firestore
- **Estilos:** TailwindCSS
- **Plataforma:** Web, corre en el navegador (macOS, sin restricciones)
- **Sin backend propio:** React se conecta directo a Firebase via SDK

## Módulos del sistema
1. **ABM de Productos**
2. **Comandas** (el más complejo)
3. **Caja del Día** (pantalla separada, no integrada en Comandas)
4. **Cierre de Caja** (incluye carga de gastos como paso previo)

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

> El frame original de Comandas queda como referencia hasta que se confirme
> que la variante con historial está lista. Borrar el viejo manualmente.

### Convención visual del topnav

Todos los frames comparten el mismo topnav horizontal de 56px de alto:
- Izquierda: logo placeholder + "SAO Bar"
- Centro: links de navegación — Productos / Comandas / Caja / Cierre
- Derecha: fecha (ej. "vie 29/03")
- Link activo: texto en color primario + línea de subrayado azul debajo
- Link inactivo: texto secundario, sin subrayado

> ⚠️ El frame Comandas fue creado inicialmente con un sidebar vertical
> izquierdo (inconsistente). Se corrigió manualmente eliminando el sidebar,
> corriendo el contenido 64px a la izquierda, y agregando el topnav
> horizontal. Todos los frames nuevos deben usar este patrón desde el inicio.

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
  totalGastos: number,          // suma de todos los ítems de gastosDetalle
  gananciaNeta: number,         // totalGeneral - totalGastos
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
  gastosDetalle: [              // snapshot de los gastos de la jornada
    {
      categoria: string,        // ej: "Bebidas", "DJ Tortu", "Banda en vivo"
      monto: number
    }
  ],
  gastosIds: [string]           // IDs de los documentos en colección gastos
}
```

### `gastos/{gastoId}`
```js
{
  fecha: Timestamp,        // serverTimestamp() — automático, no editable
  categoria: string,       // texto del select — gestionado en Firebase console
  monto: number
}
```

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
- ✅ Éxito → modal con `#42` grande, lista de ítems (cantidad + nombre),
  botón "Imprimir" (ancho completo), link "Cerrar" debajo
- ❌ Error → modal genérico "Error al enviar la comanda" + línea descriptiva
  que cambia según el tipo, solo botón "Cerrar"

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
- Un solo scroll continuo con el resto del panel
- **Query:** `where fecha >= inicioDia, orderBy fecha desc, limit(10)`
- Siempre se leen exactamente 10 documentos — escala a eventos grandes sin cambiar nada
- Listener en tiempo real con `onSnapshot` — cuando se envía una comanda
  exitosamente aparece sola al tope sin recargar

> **Razonamiento de ubicación:** no es un módulo separado porque los casos
> de uso reales (detectar error en una comanda, verificar un cobro) son poco
> frecuentes en un bar chico. Tenerlo visible en el panel de Comandas permite
> consultarlo sin cambiar de pantalla mientras se arma el siguiente pedido.
> El costo en Firestore es despreciable: ~50-80 lecturas por noche, muy por
> debajo del tier gratuito de 50.000 lecturas/día.

### Lógica de negocio — Gastos

Los gastos se cargan el mismo viernes, al momento del cierre de caja.
No importa cuándo ocurrieron físicamente durante la semana — el operador
los vuelca todos de una sola vez antes de cerrar.

Por esto:
- La fecha del gasto es `serverTimestamp()` — automática, no editable
- El cierre es **por jornada**: los gastos cargados en esa sesión son
  exactamente los que entran en ese cierre, sin lógica de rangos de fechas
- Las categorías de gastos son gestionables por el cliente desde la consola
  de Firebase (`categorias_gastos`) — si surge una categoría nueva como
  "Banda en vivo", el cliente la agrega él mismo sin intervención del
  desarrollador
- El desglose por categoría se guarda como snapshot en `gastosDetalle`
  dentro del cierre — permite la comparativa histórica semana a semana,
  que es uno de los datos más valiosos del Excel actual

### Módulo Cierre de Caja — flujo de dos pasos

El operador ejecuta un flujo lineal de una sola intención al terminar la noche:

**Paso 1 — Carga de gastos**
- Lista editable en memoria (`useState` local, nada se escribe aún)
- Campos por ítem: categoría (select poblado desde `categorias_gastos`) + monto
- Botón "Agregar" → agrega ítem a la lista local
- Botón eliminar por ítem → lo quita de la lista local
- Subtotal calculado en tiempo real
- Botón "Continuar" → persiste todos los gastos con `batch write`
  (un documento por ítem en colección `gastos`) y avanza al paso 2

**Paso 2 — Resumen y cierre**
- Ventas de la noche: `totalEfectivo` / `totalMP` / `totalGeneral`
- Gastos cargados en paso 1: subtotal por ítem + `totalGastos`
- Ganancia neta: `totalGeneral − totalGastos`
- Stock restante por producto
- Botón "Cerrar caja" → escribe un documento en `cierresDeCaja` con todo
  pre-calculado y cierra la jornada

**Por qué `batch write` en gastos y no `runTransaction`:** no hay
validaciones cruzadas ni condiciones. Se escriben N documentos
independientes de una vez. `runTransaction` se reserva para Comandas
donde hay stock que validar.

---

## Comportamientos de UI pendientes
*(sin wireframe — se resuelven al codificar)*

- Botones `+/−` de cantidad: `useState` local — confirmar al armar
  arquitectura de componentes (puede no necesitar `useContext`)
- Botón "Nueva comanda": limpia el panel si la comanda está a medio hacer
- Efectivo / Mercado Pago: selección exclusiva (uno o el otro)
- Cantidades: no negativas, no superan el stock disponible del producto

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
2. Módulo ABM de Productos
3. Módulo de Comandas
4. Caja del Día
5. Cierre de Caja (paso 1: carga de gastos → paso 2: resumen y cierre)
6. Historial y reportes *(versión futura)*

---

## Notas de sesión

- El conector de GitHub en claude.ai web no está disponible como conector
  nativo todavía. El flujo de contexto es: pegar este README al inicio
  de cada chat nuevo.
- Cuando arranque el desarrollo, este archivo vive en el repo de código
  como `PROYECTO.md` y Claude Code lo lee directamente.
