# Flujo 4 — Aviso de ventas grandes (`venta.registrada`)

**Issue:** #9
**Nivel:** Esencial
**Flujo:** 4 — Venta grande
**Estado:** Especificación pendiente de revisión humana

## 1. Objetivo

Que Marisa se entere por correo cuando se registre una venta directa por encima de
un importe que ella misma decide, sin tener que estar en la tienda. El importe
límite **no está fijo en el flujo**: se guarda en los datos propios del flujo
(static data), así que cambiarlo no obliga a editar ni a republicar nada.

## 2. Entrada

- **Disparador:** el enrutador (`00. Enrutador de la tienda`) llama a este flujo
  con `Execute Workflow` cuando recibe `evento = "venta.registrada"`.
- **Datos que llegan:** `{ "evento": "venta.registrada", "datos": { ... } }`,
  donde `datos` es un `VentaOut` de la app:

  | Campo | Tipo | Ejemplo |
  |---|---|---|
  | `id` | número | `28` |
  | `concepto` | texto | `"Zapato marrón talla 42"` |
  | `importe` | número | `150.0` |
  | `forma_pago` | texto | `"efectivo"` |
  | `fecha` | fecha ISO | `"2026-10-29"` |

- **Respuesta:** ninguna (el enrutador ya respondió 200 a la app).

## 3. Diseño del flujo

Nombre: `HU-09. Aviso de ventas grandes`
Archivo: `workflows/HU-09-aviso-de-ventas-grandes.json`

| # | Nodo | Tipo | Qué hace |
|---|---|---|---|
| 0 | `Cuando lo llama el enrutador` | Execute Workflow Trigger | Acepta todos los datos del padre |
| 1 | `Leer venta y umbral` | Code | Normaliza `datos`, lee el umbral de static data (por defecto 150 €) y calcula si la venta es "grande" |
| 2 | `¿Es venta grande?` | IF | `es_grande === true` |
| 3 | `Aviso a Marisa` | Email | Envía el resumen a `marisa@pasofirme.es` |
| 4 | `Ignorar venta` | NoOp | Salida "no" del IF: no hace nada |

Conexiones:

```text
Cuando lo llama el enrutador → Leer venta y umbral → ¿Es venta grande?
    sí  → Aviso a Marisa → (fin)
    no  → Ignorar venta
```

### 3.1. Contrato con el enrutador

- Recibe el **mismo objeto** que envió la app: `{ "evento", "datos" }`, en `$json`.
- No tiene webhook ni `Switch`: el reparto es responsabilidad del enrutador.
- No tiene `settings.errorWorkflow`: lo asume el enrutador (regla de una sola capa,
  ver spec del enrutador §7).

### 3.2. Código del nodo `Leer venta y umbral`

```js
// La app es la fuente de verdad: aqui solo se valida y se da formato.
const body = $json.body || $json;
const datos = body.datos || {};

// Umbral configurable en una variable de n8n (Settings -> Variables),
// clave UMBRAL_VENTA_GRANDE. Asi Marisa puede cambiarlo desde la UI sin
// tocar el flujo. Si la variable no existe o no es valida, se usa el
// valor por defecto.
const UMBRAL_POR_DEFECTO = 150;
let umbral = UMBRAL_POR_DEFECTO;
if (typeof $vars !== 'undefined' && $vars.UMBRAL_VENTA_GRANDE !== undefined && $vars.UMBRAL_VENTA_GRANDE !== null) {
  const v = Number($vars.UMBRAL_VENTA_GRANDE);
  if (Number.isFinite(v)) umbral = v;
}

const importe = Number(datos.importe);
const esGrande = Number.isFinite(importe) && importe >= umbral;

const concepto = datos.concepto || '(sin concepto)';
const formaPago = datos.forma_pago || '(sin forma de pago)';
const fecha = datos.fecha || '(sin fecha)';
const id = datos.id === undefined || datos.id === null ? 'desconocido' : datos.id;

const eur = (n) => {
  const num = Number(n);
  if (!Number.isFinite(num)) return '(sin importe)';
  const [ent, dec] = num.toFixed(2).split('.');
  return ent.replace(/\B(?=(\d{3})+(?!\d))/g, '.') + ',' + dec + ' €';
};

return [{ json: { id, concepto, importe, formaPago, fecha, umbral, esGrande, importeFormateado: eur(importe), umbralFormateado: eur(umbral) } }];
```

### 3.3. Correo a Marisa

- **Para:** `marisa@pasofirme.es`
- **Asunto:** `Venta grande: <concepto> (<importe> €)`
- **Cuerpo (texto plano):**

  ```
  Venta grande registrada en la tienda.

  Concepto: <concepto>
  Importe: <importe> €
  Forma de pago: <forma_pago>
  Fecha: <fecha>
  Id de la venta: <id>

  Umbral configurado: <umbral> €

  — Sistema de apartados, Zapatería Paso Firme
  ```

## 4. Cómo cambiar el umbral

El umbral vive en una **variable de n8n** (Settings → Variables), no en el código.
La clave es `UMBRAL_VENTA_GRANDE` y el valor es un número (por defecto 150).
Para cambiarlo, Marisa (o quien tenga acceso a n8n) edita esa variable desde la
UI y guarda; el cambio aplica en la siguiente ejecución, sin tocar ni
republicar el flujo.

Si la variable no existe o no es un número válido, el flujo usa 150 € por defecto.

> Nota: las variables de n8n se resuelven con `$vars` en el motor v1 (el que
> usa este proyecto, `executionOrder: v1`). Si en el futuro se migra al motor v2,
> habrá que revisar este punto, porque v2 aún no soporta `$vars`.

## 5. Datos y seguridad

- El flujo **no lee ni escribe** en la app: solo recibe y avisa.
- No registra el cuerpo del evento en ningún sitio.
- El JSON exportado no contiene credenciales.

## 6. Criterios de aceptación

- [ ] Existe el flujo `HU-09. Aviso de ventas grandes` como subflujo.
- [ ] El enrutador tiene una rama `venta.registrada` → HU-09.
- [ ] Una venta por encima del umbral envía un correo a Marisa con concepto, importe, forma de pago, fecha e id.
- [ ] Una venta por debajo del umbral no envía correo y la ejecución queda correcta.
- [ ] El importe igual al umbral se considera "grande" (>=).
- [ ] Sin umbral configurado se usa 150 € por defecto.
- [ ] Cambiar el umbral no obliga a editar el flujo.
- [ ] Un fallo dentro de HU-09 produce **un único** aviso de HU-05.
- [ ] La app funciona igual: registrar una venta desde la tienda envía el aviso si es grande.

## 7. Pruebas previstas

| Caso | Cómo se provoca | Resultado esperado |
|---|---|---|
| 1. Venta grande | `POST` con `venta.registrada` e `importe: 250` (umbral 150) | `200`, ejecución del enrutador + HU-09, correo a Marisa |
| 2. Venta pequeña | `POST` con `importe: 40` | `200`, HU-09 termina en `Ignorar venta`, sin correo |
| 3. Importe justo en el umbral | `POST` con `importe: 150` | Correo a Marisa (>=) |
| 4. Sin umbral inicial | Borrar la variable y `POST` con `importe: 250` | Usa 150 € por defecto y envía correo |
| 5. Cambiar umbral | Editar la variable a `200` y luego `POST` con `importe: 150` | No envía correo |
| 6. Ruta única | Consultar `webhook_entity` | Una sola fila con `webhookPath = tienda` (sin cambios) |
| 7. Fallo del subflujo | `POST` con datos que provoquen error en el correo | **Un solo** aviso de HU-05 |
| 8. Prueba real | Registrar una venta grande desde la app | Correo a Marisa, igual que antes |

## 8. Fuera de alcance

- Editar el umbral desde la app (requeriría un endpoint nuevo); desde n8n ya es posible (Settings → Variables).
- Avisar al cliente (solo a Marisa).
- Acumular o resumir varias ventas grandes en un solo correo.

## 9. Plan de implementación

1. Crear la rama desde el tablero (issue #9).
2. Escribir y revisar esta spec.
3. Añadir la rama `venta.registrada` → HU-09 en el enrutador y reexportarlo.
4. Crear el flujo `HU-09. Aviso de ventas grandes`, publicarlo y exportarlo.
5. Ejecutar la tabla de pruebas de §7.
6. Documentar en `docs/ai-log.md` y abrir PR a `dev`.

## 10. Decisiones pendientes

- **Umbral por defecto:** 150 € (confirmado).
- **Umbral compartido:** un solo umbral por instancia (= por tienda). Confirmado: hoy solo hay una tienda, no hay multi-tenant.
- **Forma de cambiarlo:** variable de n8n (`UMBRAL_VENTA_GRANDE`, Settings → Variables), editable desde la UI sin tocar el flujo. Confirmado.