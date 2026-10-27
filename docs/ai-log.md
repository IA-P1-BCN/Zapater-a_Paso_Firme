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