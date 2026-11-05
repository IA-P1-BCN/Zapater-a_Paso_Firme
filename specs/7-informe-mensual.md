# HU-08 — Informe mensual para Marisa

**Issue:** #8
**Nivel:** 🟡 Medio
**Flujo:** 7 — Informe mensual
**Archivo del flujo:** pendiente de exportar a `workflows/`
**Estado:** Implementado y probado con septiembre de 2026 (pendiente de activar)

## 1. Objetivo

Como **Marisa**, quiero un informe a principios de mes con lo recaudado y lo que me deben, para saber cómo va el negocio sin hacer cuentas.

El workflow envía por correo, el día 1 de cada mes, un informe del mes anterior con lo recaudado (ventas directas y abonos por separado), el total pendiente de cobro y los apartados nuevos, liquidados y vencidos del mes.

> **Alcance:** esta HU cubre solo el informe. No modifica datos en la app: solo lee.

## 2. Disparador

**Schedule Trigger:** el día **1 de cada mes a las 8:00** (zona horaria `Europe/Madrid`).

El informe es siempre del **mes anterior completo**. El flujo calcula el periodo a partir de la fecha de ejecución (en un nodo Code llamado *Mes anterior*): el primer y el último día del mes anterior y su nombre en español.

## 3. Entradas

Todas las llamadas son `GET` a `http://tienda:8000` con cabecera `X-API-Key` (credencial Header Auth). Si no se envía `pagina`, la API devuelve todos los registros.

| Nodo | Ruta | Parámetros | Campos que se usan |
|---|---|---|---|
| Ventas del mes | `/api/ventas` | `desde`, `hasta` | `importe` |
| Abonos del mes | `/api/abonos` | `desde`, `hasta` | `importe` |
| Apartados | `/api/apartados` | ninguno (todos los estados) | `id`, `fecha`, `estado`, `pendiente` |
| Todos los abonos | `/api/abonos` | ninguno (todos) | `apartado_id`, `fecha` |
| Vales | `/api/vales` | ninguno | `apartado_id`, `fecha` |

`desde` y `hasta` van en formato `AAAA-MM-DD` y la API incluye los dos días extremos.

## 4. Reglas de negocio

1. El informe cubre el **mes anterior completo**, del día 1 al último día.
2. **Recaudado** = ventas directas + abonos de apartados. Se muestran **por separado** y también el total.
3. **Los importes los da la app.** Cada importe (`importe`, `pendiente`) se toma tal cual; n8n solo los suma (en céntimos, para evitar errores de redondeo) y los muestra con 2 decimales. No se recalcula ningún importe.
4. **Total pendiente de cobro** = suma de `pendiente` de los apartados con `estado = activo` en el momento de la ejecución. Es una foto del día del informe, no un dato del mes.
5. **Apartados nuevos del mes:** los que tienen `fecha` (creación) dentro del mes.
6. **Apartados liquidados del mes:** los que están en estado `liquidado` y cuyo **último abono** cae dentro del mes. La lista de apartados no trae fecha de liquidación, así que se usa la fecha del último abono.
7. **Apartados vencidos del mes:** los vales cuya `fecha` cae dentro del mes. Es el día en que la app marcó el apartado como vencido y generó el vale, **no** la fecha límite del apartado.
8. Si en el mes no hay ventas ni abonos, el informe se envía igualmente con importes a 0.00.
9. Si la app no devuelve apartados, el flujo falla y **no se envía** un informe incompleto.
10. Se envía **un solo correo** en texto plano, sin la marca de n8n.

## 5. Proceso

1. **Cada día 1 a las 8:00** lanza el flujo.
2. **Mes anterior** (Code) calcula `desde`, `hasta` y la etiqueta del mes.
3. **Ventas del mes** pide `GET /api/ventas?desde&hasta`.
4. **Abonos del mes** pide `GET /api/abonos?desde&hasta`.
5. **Apartados** pide `GET /api/apartados` sin filtro de estado.
6. **Todos los abonos** pide `GET /api/abonos` sin fechas (para hallar el último abono de cada apartado).
7. **Vales** pide `GET /api/vales`.
8. **Montar informe** (Code) calcula las cifras y redacta `asunto` y `texto`.
9. **Informe a Marisa** (Send Email) envía el correo.

```
Día 1, 8:00 → Mes anterior → Ventas del mes → Abonos del mes → Apartados → Todos los abonos → Vales → Montar informe → Informe a Marisa
```

Los nodos de petición tienen activados **Always Output Data** (un mes sin movimientos no corta el flujo) y **Execute Once** (los nodos siguientes no se repiten por cada registro).

## 6. Salidas

Un correo a Marisa.

**Asunto:** `Informe de {mes de año}: lo recaudado y lo que te deben`

**Cuerpo:**

```
Hola Marisa,

Este es el resumen de {mes de año}:

RECAUDADO
Ventas directas: {importe} EUR
Abonos de apartados: {importe} EUR
Total recaudado: {importe} EUR

PENDIENTE DE COBRO
Total que te deben: {importe} EUR

APARTADOS DEL MES
Nuevos: {n}
Liquidados: {n}
Vencidos: {n}

Zapatería Paso Firme
```

## 7. Gestión de errores

- **Mes sin ventas o sin abonos:** el nodo devuelve una fila vacía (Always Output Data), el código la ignora y el informe sale con 0.00.
- **La app no devuelve apartados:** el nodo *Montar informe* lanza un error y no se envía nada.
- **Error de la API o del envío:** lo gestiona n8n (pestaña *Executions*).
- **Nombres de nodo:** *Montar informe* busca los nodos por su nombre (`Mes anterior`, `Ventas del mes`, `Abonos del mes`, `Apartados`, `Todos los abonos`, `Vales`). Si se renombra alguno, el flujo falla.
- **Modo del campo:** `Subject` y `Text` del nodo de correo deben ir en modo **Expression**.

## 8. Criterios de aceptación

- [ ] El día 1 llega un informe con lo recaudado el mes anterior.
- [ ] Ventas directas y abonos aparecen por separado.
- [ ] Incluye el total pendiente de cobro.
- [ ] Incluye los apartados nuevos, liquidados y vencidos del mes.
- [ ] Los importes coinciden con los de la app.
- [ ] Si el mes no tiene movimientos, el informe se envía con ceros y el flujo no falla.
- [ ] No hay claves escritas en los nodos.

## 9. Casos de prueba

Datos de ejemplo comprobados contra la API el **2026-10-27**.

### 9.1. Septiembre de 2026 (ejecución manual del flujo)

| Dato | Esperado |
|---|---|
| Ventas directas | 3772.75 EUR (109 ventas, ids 86 a 194) |
| Abonos de apartados | 289.00 EUR (21 abonos) |
| Total recaudado | 4061.75 EUR |
| Total pendiente de cobro | 557.15 EUR (apartados activos) |
| Apartados nuevos | 6 (ids 5, 6, 7, 8, 18 y 19) |
| Apartados liquidados | 2 |
| Apartados vencidos | 0 |

Los liquidados se comprueban en *Todos los abonos*: son los apartados liquidados cuyo último abono cae en septiembre.

### 9.2. Octubre de 2026 (para ver un vencido)

En el nodo *Mes anterior*, cambia temporalmente `$now.minus({ months: 1 })` por `$now` y ejecuta. Esperado: **7 apartados nuevos** (ids 1, 2, 3, 4, 17, 23 y 24) y **1 vencido** (el apartado 22, con vale del 21/10). Después **restaura** `$now.minus({ months: 1 })`.

### 9.3. Mes sin movimientos

Cambia temporalmente el mes en *Mes anterior* a uno sin ventas ni abonos. Esperado: informe con **0.00** en ventas y abonos, sin error. Después restaura el nodo.

### 9.4. Importes iguales a los de la app

Compara ventas, abonos y pendiente del informe con los que muestra la app o con la hoja correspondiente de `/api/excel`.

## 10. Pendiente / decisiones abiertas

- **Exportar el flujo** a `workflows/` y **activarlo**; cambiar el destinatario al correo real de Marisa.
- **Pendiente de cobro y plazos pasados:** cuenta todos los apartados `activo`, así que incluye los que ya pasaron de plazo pero aún no se han marcado como vencidos (hoy los ids 13, 14, 15 y 16). Al ejecutarse la HU-07 pasarán a vencidos y dejarán de contar como pendiente.
- **Fecha de liquidación:** se aproxima con la fecha del último abono. Si la app llegara a guardar la fecha real de liquidación, habría que usarla.
- **Fecha de vencimiento:** se usa la fecha del vale. Si se tarda en ejecutar la HU-07, un apartado vencido puede aparecer en el informe de un mes distinto al de su fecha límite.
- **Dependencia de la API:** el informe da por hecho que, sin `pagina`, las rutas devuelven todos los registros.