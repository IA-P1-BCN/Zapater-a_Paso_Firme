# HU-03 — Abonos por Bizum o transferencia sin apuntar nada a mano

**Issue:** #3
**Nivel:** Esencial
**Flujo:** 3 — Abonos a distancia
**Archivo del flujo:** `workflows/HU-03 - Abonos por Bizum o Transferencia.json`
**Estado:** Implementado, pendiente de enviar el recibo al cliente (HU-02)

## 1. Objetivo

Como Marisa, quiero que los pagos que me llegan por Bizum o transferencia se registren solos en el apartado correcto, para dejar de revisar los avisos del banco y apuntarlos a mano.

## 2. Disparador

**Form Trigger** titulado «Correo del banco» con un campo `correo` (textarea). Permite pegar a mano el texto de un aviso del banco para probar sin depender del correo real.

> En producción este disparador se sustituiría por un IMAP/Gmail Trigger que lea la bandeja de la tienda. El resto del flujo no cambia.

## 3. Entradas

### 3.1. Texto del correo del banco

Se pega en el formulario. Ejemplos reales en `data/correos/` (`01_bizum_carmen.eml` … `09_bizum_silvia.eml`).

### 3.2. Apartados activos de la app

`GET http://tienda:8000/api/apartados?estado=activo` con cabecera `X-API-Key` (credencial Header Auth). Devuelve un array de apartados con `id`, `cliente`, `email`, `producto`, `fecha_limite`, `estado`, `pendiente`.

### 3.3. Extracción con IA

Nodo **Information Extractor** con modelo **Groq** (`openai/gpt-oss-120b`, temperatura 0). Devuelve:

| Campo | Descripción |
|---|---|
| `es_pago` | `true` si es un Bizum/transferencia; `false` si es publicidad u otra cosa |
| `ordenante` | Nombre de quien paga (en mayúsculas, tal cual) |
| `importe` | Cantidad en euros como **número** (`"10,00 EUR"` → `10.0`) |
| `forma_pago` | `bizum` o `transferencia` |
| `concepto` | Texto del concepto tal cual |

## 4. Reglas de negocio

1. La app es la fuente de verdad: n8n usa `pendiente`, `estado` y `fecha_limite` tal cual; **no recalcula importes**.
2. Solo se registran abonos en apartados `estado = activo`.
3. La app solo acepta `efectivo`, `tarjeta` y `bizum`; el flujo **convierte `transferencia` → `bizum`** antes de registrar (la app rechazaría `transferencia` con 422).
4. Si el importe supera lo pendiente, **no se registra** y se avisa a Marisa.
5. Si el apartado está fuera de plazo (`fecha_limite` ya pasada), **no se registra** y se avisa a Marisa.
6. Si el pago no coincide con un único apartado (ninguno o varios), **no se registra** y se avisa a Marisa.
7. Un correo que no es un pago (publicidad) se ignora sin más.
8. La API usa `X-API-Key`; la URL base dentro de Docker es `http://tienda:8000`.

## 5. Proceso

1. **On form submission** recibe el texto del correo.
2. **Extraer datos** (IA + Groq) decide `es_pago` y extrae ordenante, importe, forma_pago y concepto.
3. **If**: si `es_pago` es falso → **ignorar correo** (NoOp) y termina.
4. Si es un pago → **Encargos pendientes** pide los apartados activos a la app.
5. **Buscar el encargo** (Code):
   - Desenvuelve la lista de apartados y filtra los `activos`.
   - Si el concepto trae un número (`apartado 2`), busca por `id`.
   - Si no, empareja por nombre: primer token igual y el segundo apellido empieza igual (funciona con «JORGE M.»).
   - Convierte `transferencia` → `bizum`.
   - Si hay 0 o varios candidatos → `decision: revisar`.
   - Si el importe supera lo pendiente o está fuera de plazo → `decision: revisar` con motivo.
   - Si todo cuadra → `decision: registrar` con `apartado_id`.
6. **¿Se puede registrar?** (If sobre `decision`):
   - `registrar` → **HTTP Request** `POST /api/apartados/{id}/abonos` con `{importe, forma_pago}`.
   - `revisar` → **Send an Email** avisando a Marisa con el motivo.

## 6. Salidas

### 6.1. Abono registrado (rama `registrar`)

`POST /api/apartados/{id}/abonos` devuelve el apartado actualizado (`pendiente` ya rebajado, `estado`).

> **Pendiente:** falta conectar un Send Email que envíe al cliente su recibo (HU-02) con el `pendiente` que devuelve la app.

### 6.2. Aviso a Marisa (rama `revisar`)

Correo a `marisa@zapateria.example` con asunto «Pago para revisar: {importe} € de {ordenante}» y el cuerpo: motivo, ordenante, importe, forma y concepto.

## 7. Gestión de errores

- Correo que no es pago → se ignora (NoOp).
- Varios apartados posibles / desconocido → aviso a Marisa, no se registra.
- Importe > pendiente → aviso a Marisa con el motivo.
- Fuera de plazo → aviso a Marisa con el motivo.
- Datos insuficientes de la IA → el Code los trata como «no encontrado» y avisa a Marisa.
- Error de la API o de envío → lo gestiona n8n (pestaña Executions).

## 8. Criterios de aceptación

- [ ] Los correos de `data/correos/` se resuelven bien (ver tabla de la sección 9).
- [ ] Los importes y el pendiente son los que devuelve la app, sin recalcular.
- [ ] Un correo que no es pago (publicidad) no registra nada ni avisa.
- [ ] Si el pago es correcto, se registra el abono en el apartado activo.
- [ ] Si el pago es dudoso o no corresponde, no se registra y Marisa recibe el motivo.
- [ ] No hay claves escritas en los nodos.

## 9. Casos de prueba (`data/correos/`)

| Correo | Ordenante | Resultado esperado |
|---|---|---|
| `01_bizum_carmen` | CARMEN RUIZ GARCIA | `registrar` |
| `02_bizum_jorge` | JORGE M. (concepto "apartado 2") | `registrar` (por número) |
| `03_transferencia_lucia` | LUCIA GOMEZ PEREZ | `registrar` (liquidado si paga justo lo que falta) |
| `04_bizum_laura` | LAURA ORTEGA (paga de más) | `revisar` (importe > pendiente) |
| `05_bizum_romero` | ROMERO D (sin concepto) | `revisar` (ambiguo) |
| `06_bizum_paula` | PAULA RAMOS LEON (fuera de plazo) | `revisar` (fuera de plazo) |
| `07_transferencia_desconocido` | Desconocido | `revisar` (no encontrado) |
| `08_publicidad_banco` | — | ignorar (no es pago) |
| `09_bizum_silvia` | SILVIA | `registrar` |

## 10. Pendiente / decisiones abiertas

- **Falta el recibo al cliente (HU-02):** tras el POST de registro, conectar un Send Email al cliente con el `pendiente` que devuelve la app.
- **Corrección menor en el aviso a Marisa:** el texto usa `{{ $json.forma }}`, pero el Code devuelve `forma_pago`. Debería ser `{{ $json.forma_pago }}` o el campo «Forma» saldrá vacío.
- **Sustituir el Form Trigger por IMAP/Gmail** cuando se pase a producción.
- **Correo de Marisa:** ahora es `marisa@zapateria.example`; confirmar el real.
