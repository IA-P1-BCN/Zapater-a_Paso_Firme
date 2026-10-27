# HU-04 — Cierre de caja automático

**Issue:** #4
**Nivel:** Esencial
**Flujo:** 5 — Cierre diario
**Archivo del flujo:** `workflows/02 Cierre diario.json`
**Estado:** Implementado y probado

## 1. Objetivo

Como **Marisa**, quiero recibir cada noche el resumen de lo cobrado en el día, para cerrar la caja sin sumar con calculadora.

El workflow recoge las ventas directas y los abonos de apartados del día, los suma por forma de pago y se lo manda en un solo correo.

## 2. Disparador

**Schedule Trigger:** cada noche a las **21:00**.

El flujo **consulta la API directamente**; no depende de los avisos del webhook de la app. Así funciona aunque n8n haya estado apagado durante el día (ver `docs/guia-de-la-app.md`).

## 3. Entradas

Las dos peticiones usan cabecera `X-API-Key` (credencial Header Auth) y el par de fechas `desde` / `hasta` = hoy (`{{ $now.format('yyyy-MM-dd') }}`).

### 3.1. Ventas directas

`GET http://tienda:8000/api/ventas?desde=hoy&hasta=hoy`

| Campo | Descripción |
|---|---|
| `id` | Identificador de la venta |
| `fecha` | Día de la venta (`AAAA-MM-DD`) |
| `concepto` | Qué se vendió (modelo y talla) |
| `importe` | Cantidad cobrada (lo calcula la app) |
| `forma_pago` | `efectivo`, `tarjeta` o `bizum` |

### 3.2. Abonos de apartados

`GET http://tienda:8000/api/abonos?desde=hoy&hasta=hoy`

| Campo | Descripción |
|---|---|
| `id` | Identificador del abono |
| `apartado_id` | Apartado al que pertenece |
| `fecha` | Día del abono (`AAAA-MM-DD`) |
| `importe` | Cantidad cobrada (lo calcula la app) |
| `forma_pago` | `efectivo`, `tarjeta` o `bizum` |

> Los dos endpoints devuelven una **lista**. Ambos nodos HTTP llevan *Always Output Data* activado para que el flujo no se pare si alguno viene vacío.

## 4. Reglas de negocio

1. La app es la fuente de verdad: n8n usa `importe` y `forma_pago` tal cual; **no recalcula importes**.
2. Ventas y abonos se piden **por separado** y se mantienen separados en el resumen.
3. Cada lista se suma **por forma de pago** (`efectivo`, `tarjeta`, `bizum`).
4. El total del día = total de ventas + total de abonos.
5. Si un día no hay movimientos, el resumen **lo dice en vez de fallar**.
6. El cuerpo del correo se monta **dentro del nodo Code** (donde `\n` es un salto de línea real); el campo *Text* del correo solo contiene la referencia `={{ $json.texto }}`.
7. La API usa `X-API-Key`; la URL base dentro de Docker es `http://tienda:8000`.

## 5. Proceso

1. **Cada noche a las 21:00** lanza el flujo.
2. **Ventas de hoy** pide `GET /api/ventas` del día.
3. **Abonos de hoy** pide `GET /api/abonos` del día.
4. **Code in JavaScript** desenvuelve las dos listas, suma por forma de pago, calcula los totales y monta `asunto` y `texto`.
5. **Enviar cierre a Marisa** manda el correo con `={{ $json.asunto }}` y `={{ $json.texto }}`.

```
21:00 → Ventas de hoy → Abonos de hoy → Code → Enviar cierre a Marisa
```

## 6. Salidas

Un único correo a Marisa.

- **Asunto:** `Cierre de caja del {fecha}` (fecha en `dd/MM/yyyy`).
- **Cuerpo:**

```
Hola Marisa,

Resumen de hoy:

VENTAS DIRECTAS (n):
  efectivo: X,XX EUR
  tarjeta: X,XX EUR
  bizum: X,XX EUR
  Total ventas: X,XX EUR

ABONOS DE APARTADOS (n):
  efectivo: X,XX EUR
  tarjeta: X,XX EUR
  bizum: X,XX EUR
  Total abonos: X,XX EUR

TOTAL COBRADO HOY: X,XX EUR

Zapatería Paso Firme
```

Las líneas de forma de pago solo aparecen si tienen importe. Si no hay movimientos, se añade `(Hoy no se ha cobrado nada.)`.

## 7. Gestión de errores

- **Un endpoint devuelve lista vacía:** se muestra `(ninguna)` / `(ninguno)`; el flujo sigue.
- **Día sin movimientos:** el resumen lo dice y el flujo **no falla**.
- **Error de la API o de envío:** lo gestiona n8n (pestaña *Executions*); para avisos de fallo está el flujo 8 (HU-05).
- **Campo *Text* en modo Plain Text:** el correo sale con `\n` literales. Debe ir en modo **Expression**.

## 8. Criterios de aceptación

- [ ] Cada noche llega un resumen con ventas directas, abonos y el total, separados por forma de pago (efectivo, tarjeta y Bizum).
- [ ] Los importes coinciden con los de la app (no se recalculan).
- [ ] Si ese día no hubo movimientos, el resumen lo dice en vez de fallar.
- [ ] No hay claves escritas en los nodos (solo la referencia a la credencial).
- [ ] Los nombres de los nodos se entienden.

## 9. Casos de prueba

### 9.1. Día sin movimientos (2026-10-27)

0 ventas y 0 abonos. Resultado esperado:

```
VENTAS DIRECTAS (0):
  (ninguna)

ABONOS DE APARTADOS (0):
  (ninguno)

TOTAL COBRADO HOY: 0.00 EUR
(Hoy no se ha cobrado nada.)
```

### 9.2. Día con movimientos (2026-10-20)

Ventas (2):

| Concepto | Importe | Forma |
|---|---|---|
| Botín piel negro T33 | 92,00 | efectivo |
| Botín piel negro T39 | 92,00 | bizum |

Abonos (6): apartados 1, 2, 3, 4, 17 y 24 → 10,00 efectivo + 5,00 bizum + 15,00 tarjeta + 5,00 bizum + 44,90 efectivo + 10,00 efectivo.

Resultado esperado:

```
VENTAS DIRECTAS (2):
  efectivo: 92.00 EUR
  bizum: 92.00 EUR
  Total ventas: 184.00 EUR

ABONOS DE APARTADOS (6):
  efectivo: 64.90 EUR
  tarjeta: 15.00 EUR
  bizum: 10.00 EUR
  Total abonos: 89.90 EUR

TOTAL COBRADO HOY: 273.90 EUR
```

Para probarlo, cambia temporalmente `desde` y `hasta` de los dos nodos HTTP a `2026-10-20` y vuelve a `{{ $now.format('yyyy-MM-dd') }}` al terminar.

## 10. Pendiente / decisiones abiertas

- **"Guarda el cierre":** la tabla de automatizaciones del README menciona que el flujo *guarda el cierre*, pero la API no expone ningún endpoint de cierres. De momento solo se envía el correo. Si se quiere persistir, habría que añadir endpoint en la app o usar Google Sheets (opción que el README ya contempla para el histórico de cierres).
- **Guardado del JSON:** el flujo está en `workflows/02 Cierre diario.json`; conviene exportarlo desde n8n tras cada cambio para que el archivo del repo quede igual que el flujo publicado.
- **Correo de Marisa:** confirmar la dirección real antes de publicar.
