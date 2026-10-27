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