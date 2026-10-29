# HU-07 — Apartados vencidos con su vale (flujo 6, parte B)

> **Alcance:** esta HU es la **segunda mitad del flujo 6**. La primera (recordatorios de 7 días y 1 día) está en [`6-recordatorios-de-vencimiento.md`](6-recordatorios-de-vencimiento.md).
>
> **Flujo en n8n:** `06-vencimiento` · **Exportado:** `workflows/06-vencimiento.json`

---

## 1. Objetivo

> Como **Marisa**, quiero que los apartados que se pasan de plazo se cierren solos y el cliente reciba un vale, para recuperar los zapatos y no tener que gestionarlo uno a uno.

La app **no vence nada sola**: un apartado pasado de plazo sigue `activo` hasta que alguien llama a `POST /api/apartados/{id}/vencer`. Esa llamada la hace n8n.

---

## 2. Disparador

| | |
|---|---|
| **Nodo** | `Schedule Trigger` (el mismo del flujo 6) |
| **Cuándo** | Cada mañana a las **9:00** |
| **Condición de trabajo** | Si no hay apartados fuera de plazo, la rama **no se ejecuta** y no se envía ningún correo |

---

## 3. Entradas

### 3.1. Apartados activos

`GET http://tienda:8000/api/apartados?estado=activo` con la cabecera `X-API-Key`.

Cada apartado llega como **un item**. Campos que usa esta HU:

| Campo | Uso |
|---|---|
| `id` | Para construir la URL de vencer y el código del vale |
| `cliente`, `email` | Destinatario del aviso |
| `producto`, `talla` | Lista de Marisa (qué zapatos vuelven a la venta) |
| `fecha_limite` | Decidir si está fuera de plazo |
| `abonado` | **No se usa para calcular**: la app lo convierte en el importe del vale |

### 3.2. Qué se considera «fuera de plazo»

```
fecha_limite < hoy
```

Comparación **de texto** sobre fechas `AAAA-MM-DD` (funciona porque el formato es ordenable).

> No hace falta mirar `estado`: el `GET` ya trae solo `activo`.

---

## 4. Reglas de negocio

1. **Solo se vence lo que ya pasó su fecha límite.** Si se intenta antes, la app responde **422**.
2. **El importe del vale lo calcula la app**: es exactamente lo que el cliente había abonado. **n8n no recalcula nada.**
3. **El código del vale lo genera la app**: `VALE-` + el id del apartado a 5 dígitos (ej. `VALE-00016`).
4. **El vale no se devuelve en efectivo**: sirve para una compra posterior (regla del negocio, README §Vencimiento).
5. **`vencido` es un estado final**: un apartado vencido ya no admite abonos.

---

## 5. Proceso

```
Schedule Trigger (9:00)
  └─ Apartados activos  (GET /api/apartados?estado=activo)
       ├─ Recordatorios de hoy → Aviso al cliente → Marcar como enviado   ← HU-06
       │
       └─ Fuera de plazo  (Code)                        ← 0..N items
            └─ Vencer apartado  (POST /api/apartados/{id}/vencer)
                 ├─ Avisar a los clientes  (Code) → Aviso del vale   (Email)   → N correos
                 └─ Lista para Marisa      (Code) → Vencidos del día (Email)   → 1 correo
```

### 5.1. Nodos de esta HU

| # | Nodo | Tipo | Qué hace |
|---|---|---|---|
| 1 | `Fuera de plazo` | Code | Filtra los que tienen `fecha_limite < hoy` |
| 2 | `Vencer apartado` | HTTP Request | `POST /api/apartados/{id}/vencer` — **una vez por apartado** |
| 3 | `Avisar a los clientes` | Code | Recoge los N resultados y prepara un correo por cliente con correo |
| 4 | `Aviso del vale` | Send Email | Envía el vale a cada cliente |
| 5 | `Lista para Marisa` | Code | Agrupa los N resultados en **un solo** texto |
| 6 | `Vencidos del día` | Send Email | Envía **un solo** correo con la lista a Marisa |

### 5.2. Qué devuelve `POST /vencer`

```json
{
  "apartado": { "id": 16, "cliente": "Paula Ramos", "estado": "vencido", "...": "..." },
  "vale": { "id": 2, "codigo": "VALE-00016", "apartado_id": 16,
            "fecha": "2026-10-29", "importe": 5.0, "usado": 0 }
}
```

Ese objeto es **todo** lo que necesitan los dos correos. No hace falta ninguna consulta extra.

---

## 6. Configuración que no se puede olvidar

| Nodo | Ajuste | Por qué |
|---|---|---|
| `Vencer apartado` | URL en **Expression** y **en una sola línea** | En texto fijo n8n envía `{{ ... }}` literal → `Invalid URL` |
| `Vencer apartado` | `Execute Once` **APAGADO** | Si está encendido, solo vence **el primero** |
| `Vencer apartado` | **Sin** `On Error → Continue` | Ocultaría los fallos |
| Los 3 nodos Code | **Run Once for All Items** | Es lo que permite agregar la lista |
| Los 3 nodos Email | `Append n8n Attribution` **desactivado** | Que no salga publicidad de n8n |
| Los nodos Email | **Retry On Fail**: 3 intentos, 5 s | — |

> ⚠️ **Trampa del `=`:** en modo Expression se escribe `{{ $json.email }}` **sin `=` delante**. Si se escribe `={{ $json.email }}`, n8n guarda `=={{ ... }}` y el correo sale con un `=` literal: `To: =laura12@example.com`. Pasó en la primera prueba.

---

## 7. Salidas

### 7.1. Correo al cliente

| | |
|---|---|
| **Para** | `{{ $json.email }}` |
| **Asunto** | `Tu apartado #12 ha vencido · Vale VALE-00012` |
| **Cuerpo** | Saludo, aviso de que los zapatos vuelven a la venta, **código**, **importe** y fecha del vale, y el producto |

Los clientes **sin correo** no reciben nada; salen marcados en la lista de Marisa.

### 7.2. Correo a Marisa

| | |
|---|---|
| **Para** | `marisa@zapateria.example` |
| **Asunto** | `Vencidos hoy: 6 apartado(s) · 225,00 € en vales` |
| **Cuerpo** | Una línea por zapato: nº de apartado, cliente, producto, talla, vale e importe · total en vales |

---

## 8. Gestión de errores

| Situación | Qué pasa |
|---|---|
| **422** — el apartado no está fuera de plazo o ya está vencido | El flujo se detiene con error y lo recoge el **Error Workflow** |
| **401** — falta la credencial | Igual |
| **Correo falla** | 3 reintentos cada 5 s |
| **La app no responde** | Error del nodo HTTP → Error Workflow |

> El flujo es **naturalmente idempotente**: al vencer, el apartado deja de estar `activo` y el `GET` ya no lo devuelve. **No hace falta memoria de workflow.**

---

## 9. Criterios de aceptación

- [x] Los apartados fuera de plazo se vencen por API y la app genera el vale.
- [x] El cliente recibe un mensaje con el código y el importe de su vale.
- [x] Marisa recibe la lista de zapatos que vuelven a la venta.

---

## 10. Casos de prueba

### 10.1. Ejecución real del 29/10/2026

**Antes:** 23 activos · 7 liquidados · 1 vencido · 31 total
**Después:** **17 activos** · 7 liquidados · **7 vencidos** · 31 total

| Apartado | Cliente | Producto | Vale | Importe |
|---|---|---|---|---|
| 11 | David Romero | Deportiva blanca | `VALE-00011` | 40,00 € |
| 12 | Laura Ortega | Deportiva blanca | `VALE-00012` | 45,00 € |
| 13 | Pablo Rubio | Bota montaña | `VALE-00013` | 30,00 € |
| 14 | Marta Molina | Botín piel negro | `VALE-00014` | 45,00 € |
| 15 | Sergio Delgado | Bota montaña | `VALE-00015` | 60,00 € |
| 16 | Paula Ramos | Zapatilla running azul | `VALE-00016` | 5,00 € |
| | | | **Total** | **225,00 €** |

- **6 correos** a los clientes ✅
- **1 correo** a Marisa con la lista y el total ✅
- Los importes coinciden con lo abonado, **al céntimo**

### 10.2. Reejecución el mismo día (idempotencia)

`Fuera de plazo` devuelve **0 items** → no se vence nada, **no sale ningún correo** ✅

### 10.3. Día sin apartados fuera de plazo

Igual que 10.2: la rama no se ejecuta. El flujo termina en `success` sin enviar nada.

---

## 11. Límites conocidos

### 11.1. Los recordatorios se duplican al probar a mano

El nodo `Marcar como enviado` (HU-06) usa `$getWorkflowStaticData('global')`. La documentación de n8n dice:

> *«Static data isn't available when testing workflows. The workflow must be published and called by a trigger or webhook to save static data.»*

Consecuencia: **pulsando Execute a mano**, el recordatorio se reenvía en cada ejecución. **Publicado y con el horario, no se duplica.**

> Si se quisiera blindar también en pruebas, n8n recomienda **Data Tables** en vez de static data.

### 11.2. El webhook `apartado.vencido` llega a HU-01 y se descarta

Al vencer, la app **también** envía el evento `apartado.vencido` a `/webhook/tienda`. El `Switch` de HU-01 tiene *Fallback Output = None*, así que **lo ignora sin error**.

⚠️ **No añadir una rama `apartado.vencido` en HU-01 que envíe correos**: los clientes recibirían **dos** avisos por cada vale.

### 11.3. Sin endpoint de borrado

Si se crea un apartado de prueba, no se puede borrar por API.

---

## 12. Pendiente / decisiones abiertas

| # | Pendiente |
|---|---|
| 1 | Añadir **Error Workflow** (`HU-05`) en Settings del flujo *(requisito 8 del README)* |
| 2 | Añadir **Retry On Fail** al nodo `Vencidos del día` (hoy no lo tiene) |
| 3 | Confirmar que el flujo está **publicado** (si no, el horario de las 9:00 no corre) |
| 4 | Renombrar el flujo a **`06 · Recordatorios y vencimientos`** (hoy se llama `06-vencimiento`) |
| 5 | Exportar el JSON a `workflows/` con `active: false` |
