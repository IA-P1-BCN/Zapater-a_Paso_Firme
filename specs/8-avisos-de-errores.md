# HU-05 — Avisos de errores (flujo 8)

**Issue:** #5  
**Nivel:** Esencial  
**Flujo:** 8 — Error Workflow  
**Estado:** Especificación pendiente de revisión humana

## 1. Objetivo

Como Marisa, quiero que me avisen si una automatización falla, para no descubrir semanas después que algo dejó de funcionar.

El workflow no arregla el fallo: detecta que una ejecución ha terminado en error y manda un aviso claro a la tienda con el flujo, el paso y el error.

## 2. Disparador

No tiene webhook ni horario propio. Se dispara con el nodo **Error Trigger**, que n8n ejecuta cuando cualquier flujo configurado falla.

Cada flujo declara este workflow en su `settings.errorWorkflow`. En este repositorio eso significa que, al añadir o modificar un flujo, hay que dejarlo apuntando a `HU-05. Avisar de errores`.

No ocupa ninguna ruta HTTP: no compite con el webhook `/webhook/tienda` de los flujos de eventos.

## 3. Entradas

El Error Trigger entrega un objeto con la ejecución fallida:

| Campo | Uso |
|---|---|
| `workflow.id` | identificador del flujo que ha fallado |
| `workflow.name` | nombre legible del flujo ("HU-01. Confirmación al apartar") |
| `execution.id` | identificador de la ejecución |
| `execution.url` | enlace para revisar la ejecución en n8n |
| `execution.mode` | cómo se lanzó (webhook, manual, trigger programado) |
| `execution.lastNodeExecuted` | último paso ejecutado: el "paso" del aviso |
| `execution.error.message` | mensaje del error |
| `execution.error.stack` | traza técnica (no se envía) |
| `execution.error.node` | nodo que lanzó el error, **si n8n lo informa** (no siempre viene) |

El Error Trigger **no incluye la hora de inicio** de la ejecución, así que el aviso se fecha con el momento en que se genera (unos segundos después del fallo).

**No se usan:** `workflow.nodes`, `workflow.connections`, los parámetros de los nodos ni las credenciales. No hacen falta para el aviso y podrían arrastrar datos sensibles.

### 3.1. Entidades de la ontología

- **Flujo:** automatización con nombre e identificador. Un flujo puede fallar muchas veces.
- **Ejecución:** cada vez que un flujo se ejecuta. Tiene estado (`success`, `error`, `running`), inicio, fin y modo de lanzamiento.
- **Error:** causa del fallo de una ejecución. Tiene mensaje, nodo y traza.
- **Aviso:** correo que la tienda recibe por una ejecución fallida.

Relaciones y reglas:

- Una ejecución pertenece a un solo flujo y puede tener como máximo un error.
- Un error genera como máximo un aviso.
- Un aviso se refiere a una única ejecución y siempre identifica su flujo.

## 4. Reglas de negocio

1. Se avisa de **cualquier** ejecución que termine en error, venga de webhook, de un trigger programado o de una ejecución manual de prueba.
2. El aviso identifica siempre el **flujo**, el **paso** y el **error**.
3. El aviso **nunca** incluye contraseñas, claves de API, tokens ni cabeceras de autorización.
4. El aviso se escribe para Marisa: frases cortas, sin jerga y con el enlace a la ejecución para que la revise alguien técnico.
5. Este workflow **no se apunta a sí mismo** como Error Workflow: si el aviso falla, no se genera otro aviso (evita un bucle infinito).
6. El aviso no reintenta el flujo fallido. Los reintentos de la tarea son responsabilidad de cada nodo (`retryOnFail`).
7. Las credenciales SMTP viven en n8n; el JSON exportado no contiene ninguna.

## 5. Proceso del workflow

1. Recibir el evento de error con los datos de la ejecución.
2. Comprobar que hay datos suficientes. Si falta el nombre del flujo, la ejecución o el mensaje, se usan valores por defecto ("flujo desconocido", "sin mensaje").
3. Limpiar el texto: quitar saltos y espacios sobrantes, eliminar el sufijo `[line N]` que añade n8n, recortar el mensaje y **redactar** cualquier valor que parezca una credencial. La traza (`stack`) no se envía.
4. Montar el asunto y el cuerpo del aviso.
5. Enviar el correo a la tienda (texto plano, sin pie de n8n), con 2 intentos separados por 3 segundos.
6. Si el envío falla, la ejecución de HU-05 queda en error. No se relanza nada más.

## 6. Salidas

### 6.1. Correo a la tienda

- **De:** `sistema@pasofirme.es`
- **Para:** `marisa@pasofirme.es`
- **Asunto:** `Ha fallado el flujo «<nombre del flujo>»`
- **Formato:** texto plano

Cuerpo del mensaje:

```text
Hola Marisa:

Una automatización ha fallado.

Flujo:  <nombre del flujo>
Paso:   <último nodo ejecutado>
Error:  <mensaje, en una línea y sin credenciales>
Cuándo: <fecha y hora del aviso>
Enlace: <url de la ejecución>

Revisa la ejecución en n8n para ver el detalle.
```

### 6.2. Datos que nunca salen en el aviso

- Contraseñas y claves (`password`, `api_key`, `apikey`, `token`, `secret`, `authorization`).
- Cabeceras HTTP completas.
- Parámetros de nodos y credenciales.
- La traza completa del error (solo un resumen corto, si aporta).

## 7. Gestión de errores

- **Faltan datos del error:** se envía el aviso con valores por defecto en lugar de fallar.
- **Fallo del envío de correo:** 2 intentos con 3 segundos de espera; si falla, la ejecución de HU-05 queda en error y se ve en n8n.
- **Fallo del propio HU-05:** no hay aviso de aviso (no está configurado como su propio Error Workflow).
- **Error en un subflujo:** el error sube al flujo padre; el aviso lo genera el flujo que tiene configurado HU-05.

## 8. Criterios de aceptación

- [x] Un fallo en un flujo que tenga configurado HU-05 genera un aviso a la tienda. *(Comprobado con HU-01 y con un flujo de prueba.)*
- [x] El aviso incluye el nombre del flujo, el paso y el mensaje de error.
- [x] El aviso incluye el enlace a la ejecución fallida.
- [x] El aviso no incluye contraseñas, claves ni tokens.
- [ ] Todos los flujos exportados en `workflows/` tienen `settings.errorWorkflow` apuntando a HU-05. *(Hoy solo HU-01; el resto se revisa en el issue #31.)*
- [x] Un fallo de HU-05 no genera avisos en cadena. *(No se apunta a sí mismo.)*
- [x] El envío tiene reintentos configurados (2 intentos, 3 s). *(El fallo del propio envío no se ha forzado para no tocar la credencial SMTP; queda como prueba pendiente.)*
- [x] El workflow no accede a la base de datos ni a la API de la tienda.

## 9. Pruebas previstas

| Caso | Cómo se provoca | Resultado esperado |
|---|---|---|
| Fallo de envío de correo | Ejecutar HU-01 con un email inválido (`correo-invalido`) | 3 intentos de HU-01, ejecución en error y aviso de HU-05 |
| Fallo de una petición HTTP | Flujo de cierre con la API caída o URL incorrecta | Aviso con el nodo HTTP y el mensaje de conexión |
| Datos incompletos en el error | Simular el Error Trigger con `{}` | Aviso con valores por defecto, sin fallar |
| Credenciales en el mensaje | Error cuyo mensaje contenga `password=secreto123` | El aviso muestra `password=***`, nunca el valor |
| Fallo de HU-05 | Desasignar la credencial SMTP y forzar un error | No se generan avisos en cadena; la ejecución de HU-05 queda en error |
| Flujo correcto | Ejecución de HU-01 sin errores | No se envía ningún aviso |

### 9.1. Pruebas realizadas (27/10/2026)

| Caso | Resultado |
|---|---|
| Flujo de prueba que falla con `password=secreto123` y `api_key=ABC-987` en el mensaje | Aviso recibido en Mailpit: `password=*** y api_key=***`, con flujo, paso, fecha y enlace. Sin `[line 1]` al final |
| Error Trigger con `{}` (sin datos) | Aviso con "flujo desconocido", "paso desconocido" y "sin mensaje de error". No falla |
| Mensaje de 500 caracteres | Se recorta a 300 y termina en `...` |
| Fallo real de HU-01 (correo inválido, ejecución 33) | Aviso con el flujo, el nodo `Email Confirmacion Cliente` y `No recipients defined` |
| Ejecución correcta de HU-01 | No se envía ningún aviso |

Dos hallazgos del Error Trigger que la spec ya recoge: **no trae la hora de inicio** (se usa la hora del aviso) y **añade `[line N]`** al mensaje (se elimina al limpiar).

## 10. Fuera de alcance

- Reintentar el flujo que ha fallado.
- Abrir incidencias en GitHub o enviar avisos por Telegram.
- Agrupar varios fallos en un resumen periódico.

## 11. Decisiones pendientes de confirmar

- **Destinatarios:** hoy solo Marisa. ¿Se añade a la dependienta?
- **Repetición:** si un flujo programado falla todas las noches, Marisa recibirá un aviso por noche. ¿Se limita a uno al día por flujo o se deja así para no perder ningún fallo?
- **Nombre del archivo:** esta spec se llama `8-avisos-de-errores.md` porque el flujo es el 8 en el README. La spec del informe mensual está guardada como `8-informe-mensual.md`, así que conviene renombrarla a `7-informe-mensual.md` para no confundir los números.
