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
- **Rama:** 0-enrutador-implementacion
- **Spec:** specs/0-enrutador-de-la-tienda.md (mergeada en dev con #32 y corregida con #33)

### Que se ha hecho

- Flujo 00. Enrutador de la tienda creado: webhook POST /tienda con respuesta
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

---

## Entrada — HU-07: Apartados vencidos con su vale

- **Fecha:** 29/10/2026
- **Tarea:** Vencer por API los apartados pasados de plazo, avisar al cliente con el código y el importe de su vale, y enviar a Marisa la lista de zapatos que vuelven a la venta.
- **Agente:** OpenCode
- **Modelo:** MiMo-V2.6-Flash Free
- **Contexto:** La app **no vence nada sola**: un apartado pasado de plazo sigue `activo` hasta que alguien llama a `POST /api/apartados/{id}/vencer`. El flujo 6 ya existía con la parte de recordatorios (HU-06), pero sin la mitad de vencimientos.

### Prompts clave

- «Como Marisa, quiero que los apartados que se pasan de plazo se cierren solos y el cliente reciba un vale… ahora tengo este, dime cómo hacerlo».
- «Pero dime paso a paso cómo se hace en n8n».
- «En el vencer apartado me sale: `Invalid URL: {{ ... }}. URL must start with "http"`».

### Decisiones

- **Se amplía el flujo 6, no se crea uno nuevo.** El README asigna la HU-07 al **flujo 6** («Recordatorios y vencimientos»).
- La rama nueva sale de `Apartados activos` con **una segunda conexión**, para no tocar la rama de recordatorios que ya funcionaba.
- **Dos nodos Code independientes** (`Avisar a los clientes` y `Lista para Marisa`), cada uno agregando los N resultados del POST. Se descartó intentar sacar dos salidas de un solo Code.
- **La fuente del vale es la respuesta del `POST`**, no el webhook: devuelve `{apartado, vale}` y evita una consulta extra.
- **Se rechazó usar el webhook `apartado.vencido` para el correo del cliente**: el cron ya tiene el dato, y además la lista de Marisa no se puede agregar entre ejecuciones distintas del webhook.
- **Se rechazó el nodo Merge** otra vez (misma razón que en HU-10: espera a todas las entradas).
- **No se usa memoria de workflow**: el vencimiento es idempotente por sí solo, porque el apartado deja de estar `activo`.

### Errores / alucinaciones

| # | Error | Causa real | Corrección |
|---|---|---|---|
| 1 | `Invalid URL: {{ 'http://…' }}. URL must start with "http"` | El campo URL estaba en modo **Fixed**: n8n recibía el `{{ … }}` literal | Pasar el campo a **Expression** y dejarlo en **una sola línea** |
| 2 | El correo salía con `To: =laura12@example.com` y asunto `=Tu apartado #12…` | El campo tenía `=={{ … }}`: en modo Expression n8n **ya pone** un `=` por dentro, y se escribió otro a mano | Borrar el `=` que se escribe a mano |
| 3 | *(No es error)* Recordatorio duplicado a la misma clienta | `$getWorkflowStaticData` **no se guarda en ejecuciones manuales** (documentado por n8n) | Publicar el flujo; queda documentado en la spec |
| 4 | Marisa recibió la lista a las 11:45 aunque los 6 ya se habían vencido a las 11:42 | Se pulsó **Execute step** en un nodo intermedio: n8n reutiliza los datos de la ejecución anterior | No es un fallo del flujo |

### Pruebas

| Caso | Resultado |
|---|---|
| 6 apartados fuera de plazo (ids 11–16) | ✅ 6 vales: `VALE-00011` … `VALE-00016` |
| Importes de los vales | ✅ 40 + 45 + 30 + 45 + 60 + 5 = **225,00 €**, cuadra con lo abonado |
| Correos a clientes | ✅ 6, con código e importe |
| Correo a Marisa | ✅ 1, con la lista y el total |
| Estado de la app | ✅ activo 23 → 17 · vencido 1 → 7 |
| Reejecución | ✅ `Fuera de plazo` 0 items, ningún correo |
| Día sin vencidos | ✅ la rama no se ejecuta |

### Aprendizaje para el resto del proyecto

> **Cualquier campo con `{{ … }}` debe estar en Expression, y en las URLs además en una sola línea.** Es el mismo fallo que dejó el flujo de abonos (HU-03) sin enviar ni una petición, y el que dejó HU-10 con cero POSTs.

### Pendiente

- Añadir el **Error Workflow** (`HU-05`) al flujo (requisito 8).
- `Retry On Fail` en `Vencidos del día`.
- Publicar el flujo.
- Exportar `workflows/06-vencimiento.json` con `active: false`.

---

## 📌 Entradas restauradas (29/10/2026)

Estas seis entradas se perdieron al sobrescribirse este fichero y se han restaurado el 29/10/2026.

**Herramienta:** OpenCode · modelo `MiMo-V2.6-Flash Free` (capa gratuita).

> **Rama:** la copia local de trabajo no está inicializada como repositorio Git, así que no se pudo determinar la rama de cada HU. Completar con la rama real del repositorio.

---

## Entrada — HU-10: Migración del Excel de Marisa

- **Fecha:** 27–29/10/2026
- **Tarea:** Migrar a la app los apartados y abonos que Marisa llevaba a mano en un Excel, usando la API.
- **Agente:** OpenCode
- **Modelo:** MiMo-V2.6-Flash Free
- **Contexto:** El histórico venía en CSV exportados del Excel (`data/excel_actual_ventas.csv` y el histórico). La app **no tenía importador ni endpoint de borrado**, así que la migración se hizo contra la API y debía ser **idempotente**: poderse ejecutar varias veces sin duplicar nada.

### Arquitectura (`workflows/HU-10.json`, 10 nodos)

`Ejecutar migración` (manual) → `Apartados en la app` + `Abonos en la app` (GET de lo ya cargado) → `Read/Write Files from Disk` → `Extract from File` → `Limpiar y clasificar` (Code) → `¿Es dudoso?` (IF) → rama verdadera → `Informe` → `Informe a Marisa`

### Prompts clave

- «Quiero que me arregles lo del CSV» (delegación explícita de la corrección tras el fallo de la primera versión).
- «Lo que en el n8n no puedo publicarlo porque no hay trigger».

### Decisiones importantes

- **El nodo Merge se rechazó a propósito.** La documentación de n8n dice: *«The node waits for the execution of all connected inputs»*. Con una rama que a veces no se dispara, el flujo **se quedaría esperando para siempre**.
- Los dudosos viajan como **campo** (un array `dudosos` dentro de cada item emitido), no como items sueltos. Así **solo se dispara una rama** del IF ⇒ **exactamente un correo**, nunca dos.
- Por eso la condición del IF cambió de `tipo === 'dudoso'` a `tipo === 'nada'`.
- Deduplicación de abonos con **dos señales**: la principal, `apartado.abonado >= precio - resta` (columna `resta` del CSV); la de respaldo, mismo `apartado_id` + `fecha` + `importe`.
- Los importes se toman **tal cual de la API**, nunca se recalculan en n8n.

### Errores / alucinaciones

| # | Error | Causa real | Corrección |
|---|---|---|---|
| 1 | Se leían **24.480 filas** en vez de 62 | `Execute Once` **desactivado** en `Read/Write File`: 31 repeticiones × 2.945 filas | Activar `Execute Once` |
| 2 | **Cero peticiones POST salieron de n8n**, sin ningún error visible | La URL del POST estaba en **modo texto plano y en varias líneas**; n8n no la interpretaba como expresión | URL en modo **Expression (fx)** y en **una sola línea** |
| 3 | Los fallos pasaban desapercibidos | `onError: continueRegularOutput` seguía la ejecución aunque el POST fallara | Quitar `onError` y dejar que el error se propague |
| 4 | El informe contaba lo que iba a crear, no lo creado | El nodo Code leía el plan en vez de `$('Crear en la app')` | Leer los items reales del nodo de escritura |
| 5 | «Access to the file is not allowed» | n8n 2.x limita el acceso a ficheros a `~/.n8n-files` por defecto | En `docker-compose.yml`: `./data:/home/node/data:ro` **y** `N8N_RESTRICT_FILE_ACCESS_TO=/home/node/data` |
| 6 | Al diagnosticar, el proceso se quedaba sin memoria | `execution_data.data` es un **grafo de índices**: cadenas como `"5"` que apuntan a `blob[5]`; resolverlo a ciegas agota la memoria | Leer `database.sqlite` con `node:sqlite` y seguir los índices de forma selectiva |

### Otras decisiones

- Sin endpoint de borrado en la API: los abonos duplicados se eliminaron directamente en SQLite, **con copia de seguridad previa** (`data/tienda-backup-antes-dedup.db`). Borrados los ids 96 y 98 (Carmen) y 97 (Ana).
- **La base de datos de n8n nunca se tocó a mano**: n8n mantiene los flujos en memoria y sobrescribiría los cambios. Todo cambio de código se entrega al alumno para que lo pegue en la UI.

### Resultados verificados

| Comprobación | Resultado |
|---|---|
| Apartados creados | **7** (ids 25–31) con sus fechas originales |
| Totales frente al Excel de Marisa | **Cuadran al céntimo**: 30 / 35 / 5 / 50 / 5 / 10 / 35 |
| Estado final | 31 apartados, **95 abonos** |
| Reejecución | **Idempotente**: crea 0 apartados y 0 abonos |
| Correo final | `Apartados: 0 (planeados 0)`, `Abonos: 0`, `YA ESTABAN CARGADOS: 10`, `PARA QUE DECIDAS (2)` |

### Para que decida Marisa (2 dudosos)

- **Línea 4** — Lucía Gómez: abono inicial de **4 €**, por debajo del mínimo de 5 €. La API lo rechazaba con un 422.
- **Línea 10** — Lucía Gómez: abono huérfano de **41 €** marcado «LIQUIDADO», sin apartado al que atribuir.

### Efectos secundarios a tener en cuenta

- Los apartados migrados **no tienen `email`** ⇒ los recordatorios de HU-06 **los omiten**.
- El campo `notas` del Excel **se pierde**: la API no tiene campo equivalente.
- `forma_pago` queda por defecto en `efectivo`.

### Sobre «publicar» el flujo

El botón **Publish** de n8n 2.x solo aplica a los triggers con algo que poner en producción (webhook → *production URL*; horarios → empiezan a correr). Un **Manual Trigger** no tiene nada que publicar: se ejecuta a mano con «Execute Workflow». Por eso la migración **no se publica, y no debe publicarse**.

---

## Entrada — HU-04 · Cierre diario (flujo 5)

- **Fecha:** 27/10/2026
- **Tarea:** Pedir a la API las ventas y abonos del día, sumarlos por forma de pago y enviar el resumen a Marisa.
- **Agente:** OpenCode
- **Modelo:** MiMo-V2.6-Flash Free
- **Contexto:** Especificación en `specs/5-cierre-diario.md`. El flujo (`workflows/02-05 Cierre diario.json`) tiene 5 nodos: `Schedule Trigger` 21:00 → `Ventas de hoy` → `Abonos de hoy` → `Code` → `Enviar cierre a Marisa`.

### Prompts clave

- Redacción de `specs/5-cierre-diario.md` contrastando el flujo ya montado con la API real de la tienda.

### Decisiones

- Los importes y las formas de pago los aporta la API; el nodo `Code` **solo agrega y formatea** (`30,00 €`), nunca recalcula.
- El `Code` concentra el cálculo para que los nodos de correo solo elijan destinatario y asunto.

### Errores / alucinaciones

- No se registran errores de diseño en esta fase. Los problemas aparecieron **después**, en la revisión del repositorio (ver entrada final).

### Hallazgos posteriores (29/10)

- El nombre interno del flujo era **«02 Cierre diario»**, cuando por la numeración oficial del README debe ser el **flujo 5**.
- Existían **dos exportaciones del mismo flujo**, ambas con `active: true`: importadas las dos, Marisa recibiría **dos cierres cada noche**.

---

## Entrada — HU-03 · Abonos a distancia (flujo 3)

- **Fecha:** 27/10/2026
- **Tarea:** Registrar abonos hechos por Bizum o transferencia sin que Marisa tenga que apuntarlos a mano.
- **Agente:** OpenCode
- **Modelo:** MiMo-V2.6-Flash Free
- **Contexto:** Especificación en `specs/3-abonos-a-distancia.md` y flujo en `workflows/HU-03 - Abonos por Bizum o Transferencia.json`.

### Prompts clave

- Elaboración de `specs/3-abonos-a-distancia.md` contrastando lo que pide el README («correo del banco + IA para extraer ordenante, importe y concepto») con lo que el equipo acabó implementando.

### Decisiones

- El equipo optó por un **Form Trigger** (formulario manual) en lugar de leer el correo del banco con IMAP.
- La app sigue siendo la única fuente de verdad de los importes: el formulario aporta el dato, pero `abonado` y `pendiente` los calcula la API.

### Errores / alucinaciones

- **Bug abierto (verificado el 29/10):** la URL del alta de abono quedó como **texto literal**:
  ```
  http://tienda:8000/api/apartados/{{ $json.apartado_id }}/abonos
  ```
  Al estar en modo **Plain Text**, n8n nunca sustituyó la variable ⇒ **422 en todos los intentos**.
- **Corrección pendiente:** pasar la URL a modo **Expression (fx)** y dejarla en **una sola línea**.

### Pendiente de documentar

- El README (flujo 3) pide leer los avisos del banco por correo. El formulario es **decisión del equipo**, no un error, pero hay que dejarla **escrita en la spec** con su justificación.

---

## Entrada — HU-06 · Recordatorios y vencimientos (flujo 6)

- **Fecha:** 27/10/2026
- **Tarea:** Avisar a los clientes cuyo plazo vence y devolver a la venta los apartados vencidos.
- **Agente:** OpenCode
- **Modelo:** MiMo-V2.6-Flash Free
- **Contexto:** Especificación en `specs/6-recordatorios-de-vencimiento.md`. El flujo `workflows/06-vencimiento.json` tiene 5 nodos: `Schedule Trigger` → `Recordatorios de hoy` (Code) → `Apartados activos` (GET) → `Aviso al cliente` → `Code`.

### Prompts clave

- Redacción de la especificación y verificación del aviso con el flujo `04-aviso-vencimiento-manana`.

### Errores / alucinaciones

- No se registran errores de diseño: el aviso se verificó funcionando.

### Efecto de HU-10

- Los **apartados migrados del Excel no tienen `email`**, así que **los recordatorios los omiten**. No es un fallo del flujo, pero hay que saberlo: son 7 apartados que Marisa debe avisar a mano si hace falta.

---

## Entrada — HU-08 · Informe mensual (flujo 7)

- **Fecha:** 27/10/2026
- **Tarea:** Recaudación del mes, pendiente de cobro y apartados nuevos / liquidados / vencidos, enviados por correo a Marisa.
- **Agente:** OpenCode
- **Modelo:** MiMo-V2.6-Flash Free
- **Contexto:** El flujo `workflows/08-informe-mensual.json` hace cuatro peticiones (`/api/ventas`, `/api/abonos`, `/api/apartados`, `/api/abonos`) contra `http://tienda:8000`.

### Prompts clave

- Encargo de elaborar la especificación y exportar el flujo del informe mensual. *(La redacción literal del prompt no quedó registrada; completar con el mensaje real usado por el equipo.)*

### Errores / alucinaciones

- **La especificación está vacía:** `specs/8-informe-mensual.md` tiene **0 bytes**. Se creó el 27/10 y nunca se rellenó.
- **El flujo está mal numerado:** se llama «08-informe-mensual», pero el informe mensual es el **flujo 7** del README. El **8** es el Error Workflow. Importados tal cual, habría **dos flujos "08"**.

---

## Entrada — Revisión del repositorio y limpieza previa a la entrega

- **Fecha:** 28–29/10/2026
- **Tarea:** Revisar el estado de `workflows/` y `specs/` a partir de una lista de problemas detectada por un compañero del equipo.
- **Agente:** OpenCode
- **Modelo:** MiMo-V2.6-Flash Free
- **Contexto:** El compañero pasó una lista priorizada de incidencias. Se contrastó **cada punto** con la instancia real de n8n y con los ficheros del repositorio **antes de tocar nada**.

### Qué se ACEPTÓ de la revisión

| Punto del compañero | Confirmado |
|---|---|
| Cierre diario duplicado, ambos con `active: true` | ✅ Sí: importados los dos, Marisa recibiría dos cierres cada noche |
| Llamada a `http://host.docker.internal:8000/` | ✅ Sí: desde dentro del contenedor de n8n no resuelve; el resto del proyecto usa `http://tienda:8000/` |
| `marisa@lacuchara.example` | ✅ Sí: residuo de la guía de la pastelería de la que partió el proyecto |
| `active: true` en los JSON exportados | ✅ Sí: con `--activeState=fromJson` se activarían sin haberlos probado |
| `specs/8-informe-mensual.md` en blanco | ✅ Sí: 0 bytes |
| Faltan HU-02 (recibo de abono) y la venta grande | ✅ Sí: no existen |

### Qué se RECHAZÓ de la revisión

| Afirmación del compañero | Realidad verificada |
|---|---|
| «`Lilo0zkcgABZtBGx` no existe ni en el repo ni en la instancia» | **Sí existe.** Se llama `session2-00-error` y está **activo**. El problema real es otro: `HU-05-avisar-de-errores.json` **nunca se importó**, así que conviven un flujo de errores vivo con nombre de prueba y otro en el repo que nadie cargó |
| «`05 Cierre diario (1).json` es la versión buena; `02-05` es la vieja» | **Son el mismo flujo.** 5 nodos idénticos, mismo código de 2.159 caracteres, mismas URLs `http://tienda:8000`. Difieren en **10 bytes**, exactamente la longitud del nombre. El correcto es `02-05 Cierre diario.json`: es el que coincide con el flujo vivo de la instancia (`02/05 Cierre diario`) |
| «`5-cierre-diario.md` es en realidad la HU-04» | **Es correcto y no hay que tocarlo.** El README asigna **flujo 5 = cierre diario = HU-04**. El error estaba en los JSON, que se llamaban «02» |

### Hallazgos nuevos (no estaban en la revisión)

- **Tres direcciones de correo distintas conviviendo:** `marisa@zapateria.example` (los flujos que ya funcionan en la instancia), `marisa@pasofirme.es` (HU-01 y HU-05) y `marisa@lacuchara.example` (el duplicado). Además el remitente es `zapateria@example.com` en los flujos vivos y `sistema@pasofirme.es` en HU-05.
- **`08-informe-mensual.json` está mal numerado** (el informe es el flujo 7 y el 8 es el Error Workflow): crea **colisión** si se importa el HU-05 como flujo 8.
- **El Error Workflow usa dos identificadores distintos** para lo mismo: `Lilo0zkcgABZtBGx` en los cierres e `hu05avisoserror` en HU-01.
- **La numeración está revuelta:** unos ficheros usan número de flujo y otros número de HU.

### Cambios aplicados

- Borrado `05 Cierre diario (1).json` (el duplicado).
- `http://host.docker.internal:8000/` → `http://tienda:8000/`.
- `marisa@lacuchara.example` → `marisa@zapateria.example`.
- `HU-01` con `errorWorkflow: hu05avisoserror`.

### Pendientes tras la revisión

- `active: true` → `false` en `02-05`, `06`, `08`, `HU-01` y `HU-05`.
- Error Workflow en `06-vencimiento`, `08-informe-mensual`, `HU-03` y `HU-10`.
- Renombrar `08-informe-mensual.json` → `07-informe-mensual.json`.
- Unificar la dirección de correo (**decisión del equipo**, no es una corrección).
- Rellenar `specs/8-informe-mensual.md`.
- Documentar en la spec de HU-03 por qué se usó un formulario.
- Importar `HU-05` y eliminar `session2-00-error`.

## Entrada — Arreglo de la demo y auditoría previa (29/10/2026)

- **Fecha:** 29/10/2026
- **Herramienta:** OpenCode
- **Rama:** `dev`
- **Contexto:** Víspera de la presentación. La demo «crear un apartado y que llegue el correo» no funcionaba.

### Qué fallaba

- El flujo que escuchaba en la instancia era `01 Eventos de la zapateria`, con el path `zapateria` (la app llama a `/webhook/tienda` → **404**) y un Switch que comparaba con `encargo.creado` / `pago.registrado`, los nombres de la pastelería. **Nunca tuvo una sola ejecución.**
- El HU-01 del repo se había convertido en un subflujo (`executeWorkflowTrigger`) que esperaba un enrutador que no estaba importado.
- La credencial SMTP del repo (`SwfZwWbtMvYyq5bi`) no existía en la instancia; la real es `eHXPDCqGJYZp1rLm`.
- El correo del cierre salía con `\n` literales y con la firma de n8n.

### Qué se hizo

- Flujo nuevo **`01 · Confirmación y recibos`** (12 nodos): webhook `/tienda` → Switch por `body.evento`.
  - `apartado.creado` → validaciones (datos completos, email) → confirmación al cliente.
  - `abono.registrado` → recibo con lo pagado y lo pendiente; si `estado = liquidado`, «ya puedes recoger tus zapatos».
  - Reutiliza el Code de validación y formato del HU-01 del repo.
- **HU-05** importado y conectado como Error Workflow en los seis flujos que lo necesitan.
- Nombres genéricos corregidos: `HTTP Request` → `Registrar abono`, `If` → `¿Es un correo de pago?`, `Send an Email` → `Avisar a Marisa (pago dudoso)`, `Code in JavaScript` → `Resumir el día`, `Schedule Trigger` → `Cada mañana a las 9:00` / `Primer día del mes`.
- Archivados: `01 Eventos de la zapateria`, `04 Aviso de vencimiento mañana` (duplicaba el 06), `session2-00-error` y `HU-03 · Abonos por Bizum`.
- Reintento (3 intentos, 3 s) en las lecturas HTTP.
- Correo del cierre: `text` → `={{ $json.texto }}` y `appendAttribution: false`.
- Exportados los siete flujos a `workflows/` con `active: false` y nombres por número de flujo.

### Pruebas (instancia real)

| Caso | Resultado |
|---|---|
| Crear apartado | correo «Resumen de tu apartado» con los importes de la app |
| Abono parcial de 15 € | correo «Recibo de tu abono»: 15 € pagados, 25 € pendientes |
| Abono final de 25 € | correo «¡Ya puedes recoger tus zapatos!» |
| Evento con datos incompletos | aviso a Marisa; no se envía nada al cliente |
| Abono sin email | falla el nodo de correo → HU-05 avisa con flujo, nodo y enlace |
| Cierre diario | correo con los 4 abonos del día y total 60 € |
| `/webhook/tienda` | 200 · `/webhook/zapateria` y rutas viejas: 404 |

### Errores del agente

- `import:workflow` **sobrescribe** si el JSON conserva el `id`: una copia de prueba del cierre sustituyó al original. Se restauró desde el JSON exportado y se repitió la prueba con una copia **sin** `id`.
- Archivar en SQL **no despublica**: el webhook de la copia seguía respondiendo hasta ejecutar `unpublish:workflow`.
- La primera auditoría de expresiones daba falsos positivos: marcaba como rotas las que sí tenían el prefijo `=` correcto.

### Decisión pendiente

- El repo contiene `00-enrutador-de-la-tienda.json` y `HU-09-aviso-de-ventas-grandes.json` (rama del enrutador) que **no están desplegados**. Se ha consolidado enrutador + confirmación + recibos en el flujo 01. Hay que decidir si se mantiene el enrutador o se retira del repo.

