# HU-01 — Confirmación al apartar

**Issue:** #1  
**Nivel:** Esencial  
**Flujo:** 1 — Confirmación al apartar  
**Estado:** Especificación técnica pendiente de validar los detalles de envío

## 1. Objetivo

Como cliente, quiero recibir una confirmación cuando aparto unos zapatos, para saber qué producto he apartado, cuánto cuesta, cuánto he abonado, cuánto queda pendiente y cuál es la fecha límite.

El workflow automatiza la confirmación de los apartados creados en la aplicación de Zapatería Paso Firme.

## 2. Disparador

La aplicación envía un evento a n8n mediante `POST /webhook/tienda`.

El evento que inicia este workflow es:

`apartado.creado`

La estructura general del webhook está definida en `backend/services/n8n.py`:

- `evento`: nombre del evento.
- `datos`: información asociada al evento.

En `backend/controllers/apartados.py`, la función `crear()` programa el envío a n8n con `n8n.avisar("apartado.creado", nuevo)`, donde `nuevo` es el apartado devuelto por el modelo.

El envío se programa como tarea en segundo plano mediante FastAPI.

## 3. Entradas

### 3.1. Datos recibidos

El modelo `backend/models/apartado.py` devuelve el apartado creado con sus datos y los importes calculados. Los campos definidos por `ApartadoOut`, en `backend/schemas/apartado.py`, son:

| Campo | Descripción |
|---|---|
| `id` | Identificador del apartado |
| `fecha` | Fecha del apartado |
| `cliente` | Nombre del cliente |
| `telefono` | Teléfono, si está disponible |
| `email` | Correo electrónico, si está disponible |
| `producto` | Modelo o descripción del calzado |
| `talla` | Talla, si está disponible |
| `precio` | Precio total |
| `fecha_limite` | Fecha límite del apartado |
| `estado` | Estado del apartado |
| `abonado` | Importe abonado |
| `pendiente` | Importe pendiente |

Estos nombres corresponden al esquema de salida de la API y a los datos devueltos por el modelo. Se debe verificar el payload real en una ejecución de prueba de n8n antes de implementar el workflow.

### 3.2. Entidades de la ontología

- **Cliente:** persona que realiza el apartado. Puede tener nombre, teléfono y correo electrónico.
- **Producto:** calzado identificado por su descripción y talla.
- **Apartado:** registro que relaciona al cliente con el producto e incluye precio, fecha, fecha límite, estado e importes.

Relaciones:

- Un cliente puede tener varios apartados.
- Cada apartado está asociado a un cliente.
- Cada apartado identifica el calzado reservado.

## 4. Reglas de negocio

1. La aplicación es la fuente de verdad para los datos del apartado.
2. n8n no debe acceder directamente a la base de datos.
3. n8n no debe recalcular el precio, el importe abonado, el importe pendiente ni la fecha límite.
4. El mensaje debe utilizar los valores proporcionados por la aplicación.
5. La ausencia de correo electrónico debe gestionarse sin provocar un fallo completo del proceso.
6. El apartado debe poder identificarse mediante su campo `id` cuando sea necesario revisar un error.
7. La API de la aplicación utiliza el encabezado `X-API-Key` para las peticiones protegidas. La base URL configurada para la comunicación desde n8n es `http://tienda:8000`.

## 5. Proceso del workflow

1. Recibir el webhook de la tienda.
2. Comprobar que el evento es `apartado.creado`.
3. Validar que están disponibles los datos obligatorios para la confirmación.
4. Comprobar si el cliente dispone de correo electrónico.
5. Si tiene correo, preparar y enviar la confirmación con los datos del apartado.
6. Si no tiene correo, no intentar enviar el mensaje y generar un aviso para la tienda.
7. Gestionar los errores de validación y de envío, conservando información suficiente para identificar el apartado afectado.

No se necesita una consulta adicional a la API si el payload recibido contiene todos los datos necesarios. Si faltan datos, se deberá decidir y documentar cómo recuperarlos mediante la API antes de implementar esa parte.

## 6. Salidas

### 6.1. Confirmación al cliente

El mensaje debe incluir:

- Modelo o descripción del calzado (`producto`).
- Talla (`talla`).
- Precio total (`precio`).
- Importe abonado (`abonado`).
- Importe pendiente (`pendiente`).
- Fecha límite (`fecha_limite`).

El envío debe completarse en menos de un minuto desde la creación del apartado, según el criterio de aceptación de la historia de usuario.

### 6.2. Aviso a la tienda

Cuando no haya correo electrónico disponible, el workflow debe avisar a la tienda de que no se ha podido enviar la confirmación.

**Pendiente de definir:** canal, destinatario y contenido exacto del aviso.

## 7. Gestión de errores

- **Correo ausente:** no intentar el envío y avisar a la tienda.
- **Datos obligatorios incompletos:** no enviar una confirmación incompleta; registrar el error e identificar el apartado cuando sea posible.
- **Error de envío:** registrar el fallo y no considerar la confirmación enviada si el servicio no lo ha confirmado.
- **Error inesperado:** conservar contexto suficiente para facilitar su revisión.

La estrategia de reintentos y la prevención de mensajes duplicados deben definirse antes de implementar el workflow.

El servicio actual de notificación de la aplicación a n8n utiliza un timeout de cinco segundos y registra una advertencia si no puede enviar el evento. No se ha verificado una estrategia de reintentos en ese servicio.

## 8. Criterios de aceptación

- [ ] La creación de un apartado activa el workflow mediante el evento `apartado.creado`.
- [ ] El cliente recibe la confirmación en menos de un minuto.
- [ ] El mensaje incluye el producto y la talla, cuando estos datos están disponibles.
- [ ] El mensaje incluye el precio total, el importe abonado y el importe pendiente.
- [ ] El mensaje incluye la fecha límite.
- [ ] Los importes y la fecha coinciden con los valores de la aplicación.
- [ ] Si el cliente no tiene correo electrónico, el workflow no falla únicamente por esa causa y avisa a la tienda.
- [ ] Los errores técnicos pueden identificarse y revisarse.
- [ ] El workflow no accede directamente a la base de datos ni sustituye las reglas de negocio de la aplicación.

## 9. Dependencias y decisiones pendientes

Antes de dar por finalizado el workflow, confirmar:

- El payload real recibido por n8n en una ejecución de prueba.
- El servicio de correo que se utilizará y la configuración segura de sus credenciales.
- El canal y destinatario de los avisos a la tienda.
- La estrategia de reintentos y prevención de duplicados.
- El tratamiento de los campos opcionales `email` y `talla`.
- Las pruebas de aceptación y sus resultados esperados.

## 10. Fuera de alcance

Este workflow no crea apartados, registra abonos ni gestiona apartados vencidos. Esas operaciones pertenecen a la aplicación y a sus respectivos flujos de automatización.