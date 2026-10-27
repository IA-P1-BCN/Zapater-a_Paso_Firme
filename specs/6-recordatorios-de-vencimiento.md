# HU-06 — Recordatorios de vencimiento

**Issue:** #6
**Nivel:** 🟡 Medio
**Flujo:** 6 — Recordatorios y vencimientos
**Archivo del flujo:** pendiente de exportar a `workflows/`
**Estado:** Implementado y probado (parte de recordatorio)

## 1. Objetivo

Como **cliente**, quiero que me avisen antes de que se acabe el plazo de mi apartado, para no perder mis zapatos ni lo que ya he pagado.

El workflow avisa por correo a los clientes cuyo plazo vence en **7 días** y en **1 día**, con lo que les falta por pagar y la fecha límite.

> **Alcance:** esta HU cubre solo los recordatorios. La otra mitad del flujo 6 —vencer por API los apartados pasados de plazo y generar el vale— es la **HU-07** y está pendiente.

## 2. Disparador

**Schedule Trigger:** cada mañana a las **9:00**.

El flujo **consulta la API directamente**; no depende de los avisos del webhook (ver `docs/guia-de-la-app.md`).

## 3. Entradas

`GET http://tienda:8000/api/apartados?estado=activo` con cabecera `X-API-Key` (credencial Header Auth).

Devuelve solo los apartados vivos. Campos relevantes:

| Campo | Descripción |
|---|---|
| `id` | Identificador del apartado |
| `cliente` | Nombre del cliente |
| `email` | Correo del cliente |
| `producto` | Calzado reservado |
| `fecha_limite` | Último día para pagar (`AAAA-MM-DD`) |
| `estado` | `activo` (el endpoint ya filtra) |
| `pendiente` | Importe que falta por pagar (lo calcula la app) |

## 4. Reglas de negocio

1. La app es la fuente de verdad: el mensaje usa `pendiente` y `fecha_limite` tal cual; **no recalcula importes**.
2. Solo se avisa de apartados con `estado = activo` y con `email`.
3. Dos disparadores posibles:
   - `fecha_limite` = **mañana** → aviso "1 día".
   - `fecha_limite` = **hoy + 7** → aviso "7 días".
4. Son días distintos (van separados 6 días), así que **cada cliente recibe cada aviso como mucho un día concreto**.
5. **Cada aviso, una sola vez:** n8n guarda en la memoria del flujo (`$getWorkflowStaticData('global')`) la clave `id-días-fecha_limite` de lo ya enviado y lo salta en ejecuciones posteriores.
6. **Primero se envía, después se marca:** la marca se pone en un nodo posterior al envío, para que si el correo falla se reintente en la próxima ejecución.
7. Si un día no hay recordatorios, **no se envía nada y el flujo no falla**.
8. Los apartados **ya pasados de plazo** no coinciden con ninguna de las dos fechas, por eso no reciben recordatorio (los trata la HU-07).
9. La API usa `X-API-Key`; la URL base dentro de Docker es `http://tienda:8000`.

## 5. Proceso

1. **Cada mañana a las 9:00** lanza el flujo.
2. **Apartados activos** pide `GET /api/apartados?estado=activo`.
3. **Recordatorios de hoy** (Code):
   - Calcula `hoy + 1` y `hoy + 7`.
   - Filtra los apartados con correo cuya `fecha_limite` coincida con alguna de las dos.
   - Descarta los que ya están en la memoria de avisos enviados.
   - Monta `asunto` y `texto` por apartado, e incluye la clave de deduplicación.
4. **Aviso al cliente** (Send Email): un correo por item con `={{ $json.email }}`, `={{ $json.asunto }}` y `={{ $json.texto }}`.
5. **Marcar como enviado** (Code): añade las claves a la memoria del flujo.

```
9:00 → Apartados activos → Recordatorios de hoy → Aviso al cliente → Marcar como enviado
```

## 6. Salidas

Un correo por cliente. No hay resumen a Marisa.

**Asunto:**

- 1 día: `Mañana vence el plazo de tu apartado #id`
- 7 días: `En 7 días vence el plazo de tu apartado #id`

**Cuerpo:**

```
Hola {cliente},

Te avisamos de que MAÑANA se cumple el plazo de tu apartado.   ← o "en 7 DÍAS"

Producto: {producto}
Fecha límite: {fecha_limite}
Te queda por pagar: {pendiente} EUR

Si ya has pagado, disculpa las molestias.

Zapatería Paso Firme
```

El mensaje incluye siempre **lo que falta por pagar** y **la fecha límite**.

## 7. Gestión de errores

- **Día sin recordatorios:** el Code devuelve 0 items, no se envía nada y el flujo termina sin error.
- **Apartado sin correo:** se descarta del filtrado; no rompe el flujo.
- **Correo que falla:** como la marca se pone después de enviar, se reintentará en la siguiente ejecución.
- **Modo del campo:** los campos del nodo de correo deben ir en modo **Expression**; si van en *Plain Text* salen literales o vacíos.
- **Error de la API o de envío:** lo gestiona n8n (pestaña *Executions*); el aviso genérico es el flujo 8 (HU-05).

## 8. Criterios de aceptación

- [ ] Cada mañana se avisa a los clientes cuyo plazo vence en 7 días y en 1 día.
- [ ] Cada cliente recibe cada aviso una sola vez.
- [ ] El mensaje incluye lo que falta por pagar y la fecha límite.
- [ ] Los importes los da la app, no se recalculan en n8n.
- [ ] Si no hay recordatorios, no se envía nada y el flujo no falla.
- [ ] No hay claves escritas en los nodos.

## 9. Casos de prueba

Datos de ejemplo comprobados contra la API el **2026-10-27**.

### 9.1. Aviso de 1 día (caso real, sin tocar nada)

| Cliente | id | Pendiente | Vence |
|---|---|---|---|
| David Romero | 11 | 15,00 EUR | 2026-10-28 |
| Laura Ortega | 12 | 10,00 EUR | 2026-10-28 |

Esperado: **2 correos**, asunto *"Mañana vence el plazo de tu apartado #11 / #12"*.

### 9.2. Aviso de 7 días (hoy no hay casos)

Ningún apartado vence el 2026-11-03. Para probarlo, cambia temporalmente la línea del `en7` del Code a:

```javascript
const en7 = '2026-11-05';
```

Esperado: **1 correo** a **Elena Castro** (id 10, 70,00 EUR). Después **restaura** `$now.plus({ days: 7 }).toFormat('yyyy-MM-dd');`.

### 9.3. Día sin recordatorios

Cambia `en1` a una fecha sin nadie. Esperado: **ningún correo** y ejecución correcta.

### 9.4. Cada aviso una sola vez

1. Ejecutar **dos veces seguidas**.
2. Si la segunda repetía los correos, es porque n8n **no persiste la memoria en modo prueba**: activa el flujo con el disparador temporalmente a *every 5 minutes* y comprueba que en **producción** no repite.
3. Restaura el disparador a las 9:00.

## 10. Pendiente / decisiones abiertas

- **Exportar el flujo** a `workflows/` y **activarlo** (pendiente).
- **HU-07** (mitad pendiente del flujo 6): `POST /api/apartados/{id}/vencer` para los pasados de plazo (23/09, 07/10, 15/10, 22/10), vale generado por la app y aviso a la tienda.
- **Memoria de avisos:** vive en n8n (`$getWorkflowStaticData`), no en la app. Si se reinstancia n8n se pierde y podrían reenviarse avisos. Con los datos actuales no es un problema; si hiciera falta durabilidad, habría que registrarlos en la app o en Google Sheets.
- **Mensajes redactados por IA:** el README lo contempla en **Nivel Avanzado** (tono cercano, pero los importes siempre calculados por el flujo, nunca por el modelo). No hecho.
