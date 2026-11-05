# Flujo 0 — Enrutador de la tienda (`/webhook/tienda`)

**Issue:** — (infraestructura; relacionado con los flujos 1, 2, 4 y 8)
**Nivel:** Esencial (soporte)
**Flujo:** 0 — Enrutador de eventos
**Estado:** Especificación pendiente de revisión humana

## 1. Objetivo

Que la app de la tienda tenga **una sola puerta de entrada** a n8n y que cada evento
llegue al flujo que le corresponde, sin que dos flujos se peleen por la misma ruta.

La app ya envía todos los eventos a la misma URL (`N8N_WEBHOOK_URL`) con la forma:

```json
{ "evento": "apartado.creado", "datos": { "...": "..." } }
```

Hoy esa URL la ocupa **HU-01**, que además filtra por su cuenta el evento que le
interesa. Funciona mientras solo exista un flujo de eventos, pero no escala.

## 2. El problema que resuelve

n8n **no permite dos webhooks con la misma ruta**. Cuando se importa el segundo,
la ruta se registra una sola vez y el segundo flujo **nunca recibe nada**: no da
error, simplemente se queda mudo (comprobado en n8n 2.44.6, `webhook.service.js`,
`createWebhookIfNotExists`).

Consecuencias hoy:

- La HU-02 (recibo de abono) no puede escuchar `abono.registrado` sin quitarle la
  ruta a HU-01.
- La venta grande (flujo 4) no puede escuchar `venta.registrada`.
- Si alguien crea otro webhook `/tienda`, uno de los dos deja de funcionar **sin
  avisar**.

La solución es un flujo pequeño que recibe todo y reparte.

## 3. Entrada

- **Nodo:** Webhook `POST /webhook/tienda`.
- **Cuerpo esperado:** `{ "evento": "<nombre>", "datos": { ... } }`.
- **Respuesta:** `200` inmediata, con el cuerpo por defecto de n8n
  (`{"message":"Workflow was started"}`).

### 3.1. Por qué se responde de inmediato

La app envía el evento en segundo plano con **timeout de 5 s** y no mira la
respuesta (`backend/services/n8n.py`). Un flujo como HU-01 puede tardar más de 5 s
si el correo falla y reintenta (3 intentos × 5 s), así que:

- Si el enrutador **espera** a terminar para responder, la app corta a los 5 s y
  registra un aviso de error aunque el correo acabe saliendo bien.
- Respondiendo al momento, la app sigue a lo suyo y n8n termina el trabajo.

**Contrapartida:** un `200` significa "n8n ha recibido el evento", **no** "el
evento se ha procesado". Los fallos se ven en la ejecución y en el aviso de HU-05.

## 4. Diseño del flujo

Nombre: `00. Enrutador de la tienda`
Archivo: `workflows/00-enrutador-de-la-tienda.json`

| # | Nodo | Tipo | Qué hace |
|---|---|---|---|
| 0 | `Entrada de la tienda` | Webhook | `POST /tienda`, respuesta inmediata |
| 1 | `Leer evento` | Code | Normaliza `evento` y `datos`. **Nunca lanza error**: si falta algo, usa valores vacíos |
| 2 | `¿Qué evento es?` | Switch | Una salida por evento + salida de reserva |
| 3 | `Ejecutar HU-01` | Execute Workflow | Llama a `HU-01. Confirmación al apartar` |
| 4 | `Ignorar evento` | NoOp | Salida de reserva: no hace nada |

Conexiones de la versión 1 (solo lo que ya existe):

```text
Entrada de la tienda → Leer evento → ¿Qué evento es?
    apartado.creado  → Ejecutar HU-01 → (fin)
    (reserva)        → Ignorar evento
```

Los demás eventos se añaden cuando exista su flujo. **No se crean flujos vacíos**
solo para tener la rama: un `Execute Workflow` que apunta a un flujo inexistente
falla.

### 4.1. Contrato entre enrutador y subflujo

El enrutador entrega al subflujo **el mismo objeto** que llegó, sin envolver:

```json
{ "evento": "apartado.creado", "datos": { ... } }
```

Dentro del subflujo eso está en `$json` (no en `$json.body`). Es el único cambio
de forma que hay que tener en cuenta al migrar los flujos.

## 5. Eventos y destinatarios

| `evento` | Lo emite | Flujo que lo atiende | Estado |
|---|---|---|---|
| `apartado.creado` | `POST /api/apartados` | HU-01 | Existe |
| `abono.registrado` | `POST /api/apartados/{id}/abonos` | HU-02 (recibo de abono) | Pendiente |
| `venta.registrada` | `POST /api/ventas` | HU-09. Aviso de ventas grandes | Implementado |
| `apartado.vencido` | `POST /api/apartados/{id}/vencer` | Nadie | **Decidir** (ver §11) |

Un evento **desconocido o vacío no es un error**: se ignora y la ejecución queda
correcta. No se avisa a Marisa para no llenarle el correo con ruido.

## 6. Migración de HU-01 (pasa a ser subflujo)

HU-01 deja de tener webhook y pasa a ser llamada por el enrutador. Cambios:

1. **Nodo 0 `Evento de la tienda` (Webhook)** → **`Cuando lo llama el enrutador`**
   (Execute Workflow Trigger), con "aceptar todos los datos".
2. **Nodo 1 `Filtrar Evento` (Switch)** → **se elimina**. El reparto es ahora
   responsabilidad del enrutador; mantener dos filtros es duplicar la regla.
   La conexión pasa de `Cuándo lo llama el enrutador` directamente a
   `Preparar mensaje`.
3. **Nodo 2 `Preparar mensaje` (Code)**, línea 2:
   - Antes: `const body = $json.body || {};`
   - Después: `const body = $json.body || $json;`
   Así el flujo acepta tanto la forma antigua (webhook) como la nueva (subflujo),
   lo que permite probarlo suelto.
4. **`settings.errorWorkflow`** → se quita de HU-01 (ver §7). Lo asume el enrutador.
5. **Reexportar** `workflows/HU-01-confirmacion-al-apartar.json`.

Todo lo demás (validación de datos, formato, IFs, reintentos, textos) **no se
toca**.

### 6.1. HU-05 no necesita cambios

Una primera lectura del código de `Execute Workflow` sugería que el padre fallaba
con un mensaje genérico (`Error executing workflow with item at index 0`) y que el
mensaje real viajaba en `error.description`, por lo que HU-05 tendría que usar
`error.description || error.message`.

**Comprobado en una instancia desechable con n8n 2.44.6** (enrutador + HU-01
migrada, provocando un fallo real dentro de HU-01): el Error Trigger recibe ya el
mensaje real del subflujo.

```text
error.message     : Credential with ID "..." does not exist for type "smtp".
error.description : undefined
nodo que falló    : Ejecutar HU-01
```

Por tanto **HU-05 se queda como está**: no hay cambio de una línea que hacer. Se
vuelve a comprobar en la prueba 7; si en esa prueba el aviso saliera genérico, se
aplicaría entonces el cambio.

## 7. Error Workflow: una sola capa

Regla: **un fallo genera un aviso, no dos.**

En n8n 2.44.6, cuando un subflujo falla, el padre también falla. Si ambos tienen
`errorWorkflow`, un único fallo puede producir **dos correos** (uno del subflujo y
otro del enrutador). Para evitarlo:

- **El enrutador SÍ tiene** `settings.errorWorkflow = hu05avisoserror`.
- **Los subflujos NO lo tienen** (se les quita a HU-01 y no se pone a los futuros).

Ventajas: cubre también los fallos del propio enrutador (por ejemplo, que el
subflujo esté despublicado y no se pueda llamar) y siempre hay exactamente un
aviso. En el aviso, el "paso" es el nodo `Ejecutar HU-01` y el mensaje es el real
del subflujo (verificado en 2.44.6, ver §6.1).

En las pruebas se comprobará de forma explícita que llega **un solo** correo
(§10, caso 7). Si en la práctica llegaran dos, se cambiará la regla a la inversa
(solo los subflujos).

## 8. Datos y seguridad

- El enrutador **no lee ni escribe** en la app: solo recibe y reparte.
- No registra ni copia el cuerpo del evento en ningún sitio.
- La ruta es interna (red de Docker). No se añade token en esta versión; queda
  como decisión pendiente (§11).
- El JSON exportado no contiene credenciales.

## 9. Criterios de aceptación

- [ ] Existe **un solo** webhook con la ruta `tienda` en la instancia.
- [ ] Un `POST` a `/webhook/tienda` responde `200` de inmediato.
- [ ] `apartado.creado` llega a HU-01 y el cliente recibe su confirmación.
- [ ] Un evento desconocido o vacío no genera error ni correo.
- [ ] HU-01 ya no tiene nodo Webhook ni `settings.errorWorkflow`.
- [ ] El enrutador tiene `settings.errorWorkflow = hu05avisoserror`.
- [ ] Un fallo dentro de HU-01 produce **un único** aviso, con el mensaje real.
- [ ] La app funciona igual: crear un apartado desde la tienda envía el correo.

## 10. Pruebas previstas

| Caso | Cómo se provoca | Resultado esperado |
|---|---|---|
| 1. Evento normal | `POST` con `apartado.creado` y datos completos | `200`, una ejecución del enrutador, una de HU-01 y correo al cliente |
| 2. Evento sin consumidor | `POST` con `abono.registrado` | `200`, ejecución del enrutador, ninguna ejecución hija |
| 3. Cuerpo vacío | `POST` con `{}` | `200`, sin error, ignorado |
| 4. Evento desconocido | `POST` con `evento: "apartado.cancelado"` | `200`, ignorado |
| 5. Datos incompletos | `POST` con `apartado.creado` y `datos: {}` | HU-01 envía su aviso de "datos incompletos" |
| 6. Ruta única | Consultar `webhook_entity` | Una sola fila con `webhookPath = tienda` |
| 7. Fallo del subflujo | `POST` con `apartado.creado` y email inválido | **Un solo** aviso de HU-05, con `No recipients defined` |
| 8. Prueba real | Crear un apartado desde la app | Correo al cliente, igual que antes |

### 10.1. Comprobado antes de implementar

Los casos 1, 2, 4 y parte del 7 se han probado en una instancia n8n **desechable**
(volumen propio, ya borrado), sin tocar la instancia real:

- `POST` con `apartado.creado` → `200` en ~200 ms; el enrutador ejecutó
  `Entrada → Leer evento → ¿Qué evento es? → Ejecutar HU-01` y HU-01 recibió los
  datos correctos (`¿Datos completos?` dio paso al correo).
- `POST` con un evento desconocido → `200`; el enrutador terminó en
  `Ignorar evento`, sin ejecución hija.
- El fallo del subflujo disparó **una sola** ejecución de HU-05, así que la regla
  de una sola capa de `errorWorkflow` se cumple.

El único fallo que apareció fue el esperado: esa instancia no tiene la credencial
SMTP (`SwfZwWbtMvYyq5bi`).

## 11. Decisiones pendientes

- **`apartado.vencido`**: la app lo emite al vencer un apartado y hoy no lo
  escucha nadie. ¿Se enruta a la HU-06 o se deja sin consumidor?
- **Evento desconocido**: se ignora (recomendado). ¿Se quiere un aviso interno?
- **Seguridad**: ¿se añade una cabecera secreta compartida entre la app y n8n?
- **Duplicados**: la app envía una vez y no reintenta. Si algún día reintenta,
  hará falta una clave de idempotencia (`evento` + `id`).

## 12. Fuera de alcance

- Implementar HU-02 o la venta grande.
- Reintentos o cola de eventos en la app si n8n está caído.
- Firmar las peticiones (HMAC).

## 13. Plan de implementación

1. Crear la rama desde el tablero (la del issue correspondiente).
2. Importar el enrutador y publicarlo.
3. Migrar HU-01 a subflujo, publicarla y reexportar.
4. Ejecutar la tabla de pruebas de §10.
5. Documentar en `docs/ai-log.md` y abrir PR a `dev`.

HU-05 no se toca (§6.1). El enlace al issue #31 en
`specs/8-avisos-de-errores.md` ya está hecho en la rama de esta especificación.
