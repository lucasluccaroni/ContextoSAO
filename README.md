# ContextoSAO
Contexto del proyecto SAO



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
4. **Cierre de Caja + Carga de Gastos**

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
  ]
}
```

### `gastos/{gastoId}`
```js
{
  fecha: Timestamp,
  descripcion: string,
  monto: number,
  categoria: string,
  medioPago: string,
  comprobante: string
}
```

### `contadores/comandas`
```js
{ ultimo: number }         // se incrementa dentro de runTransaction()
```

### Colecciones de solo lectura (se editan por consola Firebase, sin UI)
- `categorias`: `{ id, nombre, label }`
- `unidades`: `{ id, nombre, label }`

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
2. Cierre de Caja
3. Carga de Gastos

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
5. Cierre de Caja + Carga de Gastos
6. Historial y reportes *(versión futura)*

---

## Notas de sesión

- El conector de GitHub en claude.ai web no está disponible como conector
  nativo todavía. El flujo de contexto es: pegar este README al inicio
  de cada chat nuevo.
- Cuando arranque el desarrollo, este archivo vive en el repo de código
  como `PROYECTO.md` y Claude Code lo lee directamente.
