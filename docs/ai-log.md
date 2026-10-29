# 📝 Bitácora de IA

Registro de las interacciones con el agente de IA: herramienta y modelos utilizados, prompts clave, errores o alucinaciones y cómo se corrigieron.

## Entrada inicial

- **Fecha:** 22/10/2026
- **Tarea:** Scaffolding del entorno agéntico del proyecto de automatización de n8n (Zapatería Paso Firme).
- **Rama:** `setup-agent-workflow`
- **Agente:** OpenCode
- **Modelo:** Flint 40B Free (proveedor opencode, capa gratuita)
- **Contexto:** Preparar la estructura del repositorio para el desarrollo agéntico de los flujos: `AGENTS.md`, `specs/`, `workflows/`, `scripts/` y esta bitácora.

## Prompts clave

- Creación del `AGENTS.md` adaptado al proyecto: integración n8n ↔ app de la tienda, requisitos técnicos, workflow tipo del proyecto del gimnasio.

## Errores / alucinaciones

- Ninguno en esta fase inicial de scaffolding.

## Decisiones

- El agente no accede a la base de datos de la tienda directamente; todo pasa por la API REST.
- Los importes (`abonado`, `pendiente`, `precio`) los calcula la app y el agente los toma tal cual de las respuestas de la API.

## Entrada — HU-01: Confirmación al apartar

- **Fecha:** 22/10/2026
- **Tarea:** Redacción de la especificación técnica del flujo de confirmación al crear un apartado.
- **Rama:** `1-hu-01-confirmación-al-apartar`
- **Agente:** ChatGPT
- **Modelo:** GPT-6
- **Contexto:** Revisión de los esquemas, rutas, controlador y modelo de apartados para documentar el evento `apartado.creado` y los datos disponibles para la confirmación.

## Prompts clave

- Revisión de `backend/services/n8n.py`, `backend/schemas/apartado.py`, `backend/routes/apartados.py`, `backend/controllers/apartados.py` y `backend/models/apartado.py`.
- Elaboración de `specs/1-confirmacion-al-apartar.md` a partir de la implementación existente.

## Errores / alucinaciones

- No se han identificado errores confirmados durante la revisión. El payload real de n8n y los detalles del envío siguen pendientes de validación.

## Decisiones

- La aplicación es la fuente de verdad para los datos y los importes del apartado.
- El evento `apartado.creado` se programa desde el controlador mediante una tarea en segundo plano.
- Quedan pendientes la selección del servicio de correo, el canal de aviso a la tienda y la estrategia de reintentos y prevención de duplicados.

## Entrada — HU-01: corrección de los puntos 1, 2 y 3 de la revisión

- **Fecha:** 27/10/2026
- **Tarea:** Corregir los tres primeros puntos de la revisión del flujo de confirmación al apartar: validación de datos obligatorios con avisos diferenciados, formato del correo y reintentos con Error Workflow.
- **Rama:** `1-hu-01-confirmación-al-apartar`
- **Agente:** OpenCode
- **Modelo:** DeepSeek V4.1 Flash Free (proveedor opencode, capa gratuita)
- **Contexto:** El flujo `HU-01. Confirmación al apartar` ya existía en la instancia local de n8n, pero enviaba el correo con `\n` dentro de HTML (se perdían los saltos de línea), añadía el pie de n8n, mostraba importes y fechas en formato inglés y no distinguía entre "sin email" y "evento incompleto". Tampoco tenía reintentos ni Error Workflow.

## Prompts clave

- «Haga los puntos 1, 2 y 3 por favor» (sobre la revisión anterior del flujo).
- Revisión de `specs/1-confirmacion-al-apartar.md`, `backend/schemas/apartado.py` y `backend/services/n8n.py` para confirmar el payload real antes de tocar el flujo.

## Cambios realizados

- **Nodo Code `Preparar mensaje`:** valida los datos obligatorios (`id`, `cliente`, `producto`, `precio`, `abonado`, `pendiente`, `fecha_limite`), calcula la lista de campos que faltan y da formato español a importes (`45,00 €`) y fechas (`10/01/2027`).
- **IF `¿Datos completos?`:** si faltan datos, no se envía nada al cliente y se avisa a la tienda con los campos que faltan.
- **IF `Validar Email`:** si el cliente no tiene correo, se avisa a la tienda y no se intenta el envío.
- **Nodo `Aviso Tienda - Datos Incompletos`:** nuevo aviso a la tienda para eventos incompletos.
- **Correos en texto plano** (`emailFormat: text`) y sin pie de n8n (`appendAttribution: false`).
- **Reintentos:** 3 intentos con 5 s de espera en los tres nodos de envío.
- **Error Workflow:** flujo nuevo `HU-05. Avisar de errores`, que avisa a la tienda del flujo, el paso, el error y el enlace a la ejecución.
- **Exportación:** `workflows/HU-01-confirmacion-al-apartar.json` y `workflows/HU-05-avisar-de-errores.json`.

## Errores / alucinaciones

- El agente dio por buena la condición del IF `Validar Email` con el operador `notEmpty` sobre `{{ $json.email }}`. En esta versión de n8n (2.44.6) esa condición devolvía `false` aunque el correo llegara informado, así que todas las confirmaciones acababan en el aviso "sin email". Se detectó revisando el `execution_data` de una ejecución real (el IF recibía `email: "ana101@example.com"` y aun así salía por la rama falsa).
- **Corrección:** el nodo Code calcula `tiene_email` (1/0) y el IF compara un número con `1`, igual que la condición de datos completos, que sí funcionaba.
- Al importar por CLI los flujos quedan desactivados; hay que publicarlos (`publish:workflow`) y reiniciar n8n para que el webhook vuelva a registrarse.

## Decisiones

- La validación y el formato se concentran en un único nodo Code para no repetir la lógica en los tres correos (DRY).
- Los textos de los avisos se generan también en el Code: los nodos de correo solo eligen destinatario y asunto.
- Se mantiene `appendAttribution: false` para no añadir publicidad de n8n a los correos del cliente.
- El Error Workflow es un flujo aparte y reutilizable por el resto de flujos del proyecto.

## Pruebas

| Caso | Resultado |
|---|---|
| Apartado completo con email (101) | Correo al cliente con importes y fecha en formato español, sin pie de n8n |
| Apartado sin email (102) | Aviso a la tienda con los datos del apartado |
| Apartado con datos incompletos (103) | Aviso a la tienda con la lista de campos que faltan |
| Evento `apartado.creado` sin `datos` | Aviso a la tienda, apartado "desconocido" |
| Evento distinto (`apartado.cancelado`) | El flujo no hace nada |
| Email con formato inválido (106) | 3 intentos, ejecución en error y aviso del Error Workflow a la tienda |

## Entrada — HU-05: Avisos de errores (flujo 8)

- **Fecha:** 27/10/2026
- **Tarea:** Formalizar el flujo de avisos de errores: especificación, limpieza del mensaje, pruebas, exportación y documentación.
- **Rama:** `5-hu-05-avisos-de-errores`
- **Agente:** OpenCode
- **Modelo:** DeepSeek V4.1 Flash Free (proveedor opencode, capa gratuita)
- **Contexto:** El flujo existía como soporte de HU-01, pero solo enviaba un correo con los datos en crudo del Error Trigger: sin limpiar, sin redactar credenciales y con la hora vacía. La HU-05 pide que el aviso lleve flujo, paso y error, y que nunca incluya contraseñas ni claves.

## Prompts clave

- «Bueno, tendría que realizar ahora el siguiente flujo: HU-05 · Avisos de errores (flujo 8) … desarrollando las especificaciones primero, para luego hacer el flujo y todo lo que se incluya».

## Cambios realizados

- **Spec nueva:** `specs/8-avisos-de-errores.md` (ontología, entradas del Error Trigger, reglas, criterios de aceptación y pruebas).
- **Nodo Code `Preparar aviso`:** aplica valores por defecto si faltan datos, limpia saltos de línea y espacios, **redacta credenciales** (`password`, `api_key`, `apikey`, `token`, `secret`, `authorization`, `Bearer ...`), quita el sufijo `[line N]` que añade n8n y recorta el mensaje a 300 caracteres.
- **Nodo `Avisar a la tienda`:** sigue enviando texto plano sin pie de n8n, con 2 intentos y 3 s de espera.
- **Exportación:** `workflows/HU-05-avisar-de-errores.json` sustituido por la versión definitiva (3 nodos).
- **Limpieza:** borrado de la instancia el flujo temporal `ZZ prueba error (borrar)` y sus 3 ejecuciones.

## Errores / alucinaciones

- Se dio por hecho que el Error Trigger traía `execution.startedAt`. **No lo trae**: la línea «Cuándo» salía vacía. Se corrigió usando la hora en que se genera el aviso.
- n8n añade `[line 1]` al mensaje de los errores del nodo Code (además de `error.lineNumber`). Se elimina al limpiar el texto para que el aviso quede legible.
- El mensaje del error puede contener credenciales en claro (el del flujo de prueba llevaba `password=secreto123` y `api_key=ABC-987`); por eso la redacción se hace en el Code y **no** se envía la traza (`stack`).
- Las pruebas se lanzaron con `curl` desde PowerShell: con JSON en línea el webhook respondía 404/422. Hubo que enviar el cuerpo desde un fichero (`--data-binary @fichero.json`).

## Decisiones

- El texto del aviso se genera en un único nodo Code: el nodo de correo solo envía `asunto` y `cuerpo`.
- No se envía la traza completa ni parámetros de nodos: solo flujo, paso, mensaje, fecha y enlace.
- HU-05 **no** se apunta a sí mismo como Error Workflow, para no crear avisos en cadena.
- Destinatario único: `marisa@pasofirme.es`.

## Pruebas

| Caso | Resultado |
|---|---|
| Flujo de prueba que falla con credenciales en el mensaje | Aviso con `password=*** y api_key=***`, sin `[line 1]` |
| Error Trigger sin datos (`{}`) | Aviso con valores por defecto, sin fallar |
| Mensaje de 500 caracteres | Recortado a 300 con `...` |
| Fallo real de HU-01 (correo inválido) | Aviso con flujo, paso, mensaje y enlace a la ejecución |
| Ejecución correcta de HU-01 | Sin aviso |

## Flujo 0 - Enrutador de la tienda

- **Fecha:** 2026-10-29
- **Herramienta:** OpenCode (Beam Free), con subagentes explore y general.
- **Rama:**  -enrutador-implementacion
- **Spec:** specs/0-enrutador-de-la-tienda.md (mergeada en dev con #32 y corregida con #33)

### Que se ha hecho

- Flujo  0. Enrutador de la tienda creado: webhook POST /tienda con respuesta
  inmediata, Code Leer evento (nunca falla) y Switch por evento, que llama a
  HU-01 con Execute Workflow y deja los demas eventos en Ignorar evento.
- HU-01 migrada a subflujo: webhook y Switch eliminados, Code Preparar mensaje
  usa $json.body || , y se le quita settings.errorWorkflow.
- settings.errorWorkflow = hu05avisoserror solo en el enrutador.
- HU-05 sin cambios: en n8n 2.44.6 el Error Trigger ya recibe el mensaje real en
  error.message (no en error.description).

### Pruebas (instancia real)

| Caso | Resultado |
|---|---|
| 1. partado.creado completo | 200 inmediato; HU-01 envia el correo al cliente |
| 2. bono.registrado | 200; se ignora (sin ejecucion hija) |
| 3. Cuerpo vacio {} | 200; se ignora |
| 4. Evento desconocido | 200; se ignora |
| 5. partado.creado con datos: {} | HU-01 envia el aviso de datos incompletos |
| 6. Ruta unica /tienda | Una sola fila en webhook_entity |
| 7. Fallo del subflujo (email invalido) | Un solo aviso de HU-05, con el mensaje real |
| 8. Apartado creado desde la app (id 28) | Correo al cliente, igual que antes |

### Problema encontrado y resuelto

Al publicar el enrutador, la activacion fallaba con
"SQLITE_CONSTRAINT: UNIQUE constraint failed" porque un outbox anterior de HU-01
intentaba registrar el mismo webhook /tienda. Se reseteo el estado de
publicacion del enrutador (workflow_publication_trigger_status,
workflow_publication_outbox, workflow_publication_retry_state) y se publico
de nuevo; a partir de ahi el webhook quedo registrado correctamente.
## Flujo 4 - Aviso de ventas grandes (HU-09)

- **Fecha:** 2026-10-29
- **Herramienta:** OpenCode (Beam Free).
- **Rama:** 9-aviso-de-ventas-grandes
- **Spec:** specs/4-aviso-de-ventas-grandes.md

### Que se ha hecho

- Flujo HU-09. Aviso de ventas grandes creado como subflujo (Execute Workflow Trigger).
- Nodo Code Leer venta y umbral lee el umbral de la variable de n8n
  UMBRAL_VENTA_GRANDE (por defecto 150 EUR) y calcula si la venta es grande.
- Nodo IF ¿Es venta grande? (>=) → Email a Marisa / NoOp Ignorar venta.
- Correo a Marisa con concepto, importe, forma de pago, fecha e id.
- Enrutador actualizado: rama enta.registrada → Ejecutar HU-09.
- Variable UMBRAL_VENTA_GRANDE = 150 (tipo number) creada en n8n.

### Pruebas (instancia real)

| Caso | Resultado |
|---|---|
| 1. Venta grande (250 EUR) | 200 inmediato; HU-09 envia correo a Marisa |
| 2. Venta pequena (40 EUR) | 200; HU-09 termina en Ignorar venta, sin correo |
| 3. Importe justo en el umbral (150 EUR) | Correo a Marisa (>=) |
| 4. Apartado creado (regresion) | Correo al cliente, igual que antes |
| 5. Evento desconocido | 200; se ignora |

### Problema encontrado y resuelto

El enrutador no se actualizaba al reimportar porque import:workflow crea un
workflow nuevo si el JSON no trae el id. Se anadio el id al JSON exportado
(1YZEliGwwre5n7R5) para que la importacion actualice en lugar de duplicar.
Ademas, la activacion del enrutador fallaba por un outbox anterior de HU-01
que intentaba registrar el mismo /tienda; se reseteo el estado de publicacion
y se publico de nuevo.