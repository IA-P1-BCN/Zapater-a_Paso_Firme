# 10 · Migración del Excel antiguo (HU-10)

| | |
|---|---|
| **Historia** | HU-10 · *Como **Marisa**, quiero pasar los datos de mi Excel a la app, para no perder el histórico ni mantener dos sistemas.* |
| **Nivel** | Medio |
| **Flujo** | `workflows/HU-10 - Migracion del Excel.json` |
| **Disparador** | Manual (se ejecuta a mano, nunca automático) |
| **Estado** | ✅ Verificado el 28/10/2026 |

---

## 1. Qué hace

Lee el Excel antiguo de Marisa (exportado a CSV), **limpia los datos**, crea los
apartados y abonos en la app por API **conservando la fecha original** y deja en un
correo los casos que no puede decidir solo.

**Garantiza que ejecutarlo 20 veces deja la app igual que ejecutarlo una vez.**

---

## 2. Datos de entrada

`data/excel_actual_apartados.csv` — 12 filas, columnas:

```
fecha,cliente,telefono,modelo,talla,precio,abono,resta,notas
```

Son datos sucios a propósito, y así los resuelve el flujo:

| Suciedad del Excel | Ejemplo real | Cómo lo arregla |
|---|---|---|
| Dos formatos de fecha | `02/09/2026` y `7/9/26` | `aFecha()` → ambas a `AAAA-MM-DD` |
| Nombres repetidos distinto | `Carmen Ruiz` / `carmen ruiz` | `clave()` → minúsculas + espacios |
| Importes con símbolo | `15€` | `aNumero()` → `15` |
| Miles y decimales a la española | `1.842,30` | `aNumero()` → `1842.3` |
| Abono inicial bajo el mínimo | Lucía: `4` € | Se descarta y sale en el informe |
| Abono cuyo apartado aún no existe | Línea 10 | Se descarta y sale en el informe |

---

## 3. Cadena de nodos

```
Ejecutar migración
  → Apartados en la app      GET /api/apartados
  → Abonos en la app         GET /api/abonos?desde=2026-01-01&hasta=2026-12-31
  → Read/Write Files from Disk   /home/node/data/excel_actual_apartados.csv
  → Extract from File        Extract From CSV
  → Limpiar y clasificar     Code
  → ¿Es dudoso?              IF
       true  (tipo = 'nada') ───────────────→ Informe
       false (a crear) → Crear en la app ───→ Informe
                                              → Informe a Marisa
```

---

## 4. Configuración que no se puede olvidar

| Nodo | Ajuste crítico | Por qué |
|---|---|---|
| `Read/Write Files from Disk` | **Execute Once: activado** | ⭐ Sin esto, el fichero se lee una vez por cada item que llegan los GET (31 × 2.945) y el CSV se multiplica a **24.480 filas** |
| `Apartados en la app` y `Abonos en la app` | **Always Output Data: activado** | Para que el código pueda leerlos aunque vengan vacíos |
| `Crear en la app` | **URL en modo Expression, en una sola línea** | Un salto de línea la rompía y la petición ni salía |
| `Crear en la app` | **On Error: Stop Workflow** | Para que un fallo se vea en lugar de tragarse |
| `¿Es dudoso?` | Condición `{{ $json.tipo }}` **equals** `nada` | El nodo IF solo dispara una rama ⇒ siempre hay **un único correo** |

---

## 5. Cómo decide el nodo «Limpiar y clasificar»

Cada fila del CSV acaba en una de tres categorías:

| `tipo` | Cuándo | Qué se hace |
|---|---|---|
| `apartado` | Es la primera vez que aparece ese cliente+modelo | `POST /api/apartados` |
| `abono` | Ya apareció antes, o `notas` dice `abono`/`LIQUIDADO` | `POST /api/apartados/{id}/abonos` |
| `dudoso` | No se puede decidir solo | **No se envía**: va al informe |

### Detección de duplicados (dos señales, cualquiera sirve)

1. **La más fiable — el total de la app.** El CSV trae la columna `resta`, así que el
   total que Marisa esperaba cobrar es `precio - resta`. Si el apartado en la app ya
   acumula ese importe (`abonado`), el abono ya estaba aplicado.
2. **Complementaria — abono igual.** Existe ya un abono con mismo `apartado_id`,
   misma `fecha` y mismo `importe`.

> Los apartados se detectan por `cliente` + `producto` + `fecha`.

### Los dudosos no bifurcan el flujo

Los dudosos **no viajan como items**, sino como un campo `dudosos` dentro del item.
Si no hay nada que crear, el nodo emite un único item `tipo: 'nada'`.
Así el flujo recorre **una sola ruta** y el informe se envía siempre,
sin necesidad de un nodo Merge (que se quedaría esperando un input que no llega).

---

## 6. Resultado verificado

### Datos creados en la app

| id | Cliente | Producto | Precio | Abonado | Pendiente |
|---|---|---|---|---|---|
| 25 | Carmen Ruiz | Botín piel marrón | 89,90 | 30,00 | 59,90 |
| 26 | Jorge Martín | Zapatilla running azul | 64,50 | 35,00 | 29,50 |
| 27 | Pedro Sanz | Bota agua niño | 29,95 | 5,00 | 24,95 |
| 28 | Rosa Díaz | Zapato vestir hombre | 95,00 | 50,00 | 45,00 |
| 29 | Miguel Ángel López | Deportiva blanca | 55,00 | 5,00 | 50,00 |
| 30 | Sofía Navarro | Bailarina roja | 39,90 | 10,00 | 29,90 |
| 31 | Ana Torres | Mocasín ante | 72,00 | 35,00 | 37,00 |

Cuadran **al céntimo** con `precio - resta` del Excel de Marisa.

### Informe de la última ejecución

```
MIGRACIÓN DEL EXCEL - 28/10/2026 14:43

REALMENTE CREADO EN LA APP:
  Apartados: 0  (planeados 0)
  Abonos: 0  (planeados 0)

YA ESTABAN CARGADOS (no se ha duplicado): 10

PARA QUE DECIDAS (2):
  Linea 4: Abono inicial de 4.00 EUR: por debajo del mínimo de 5 EUR (la app lo rechazaría con 422)
  Linea 10: Aún no existe el apartado al que pertenece este abono: se cargará en la próxima ejecución
```

`Apartados: 0` **después de haber migrado** = la migración no duplica.

---

## 7. Los 2 casos que decide Marisa

Ambos son de **Lucía Gómez** (Sandalia tacón negra, 37, 45 €):

| Línea | Motivo | Qué proponerle a Marisa |
|---|---|---|
| 4 | Abono inicial de **4 €**, mínimo de la app: 5 € | O cobrarle 1 € más y cargar 5 €, o cargar el apartado sin abono inicial |
| 10 | Abono de **41 €** «LIQUIDADO - recogió» que no puede engancharse a ningún apartado | Si se creó el apartado de la línea 4, este 41 € lo completa y queda liquidado |

> Un detalle: si Marisa acepta cargar el apartado de Lucía con 5 €, la migración
> **no hace falta volver a ejecutarla**, porque el apartado ya se crea igual en la
> siguiente pasada.

---

## 8. Límites conocidos

- **No lee `data/excel_actual_ventas.csv`** (ventas del día y total mensual a mano).
  Queda pendiente como ampliación.
- **`notas` no se guarda**: la API no tiene campo para ello. Se pierden comentarios
  como «dijo que paga el resto el viernes».
- **Los migrados no tienen `email`**, así que los recordatorios (HU-06) no les escriben.
- **`forma_pago` no está en el Excel**, así que todo entra como `efectivo` (el valor
  por defecto de la API).

---

## 9. Cómo probarlo

1. **Desactiva `01 Eventos de la zapateria`** — al crear apartados la app dispara
   webhooks y mandaría confirmaciones de más.
2. Ejecuta el flujo y comprueba en http://localhost:8000 que aparecen los apartados.
3. Ejecútalo **otra vez**: el informe debe decir `Apartados: 0`, `Abonos: 0` y
   `YA ESTABAN CARGADOS: 10`.
4. Reactiva `01 Eventos de la zapateria`.

### Si algo falla

| Síntoma | Causa probable |
|---|---|
| `No file(s) found` | Falta el volumen `./data:/home/node/data:ro` en `docker-compose.yml` |
| `Access to the file is not allowed` | Falta `N8N_RESTRICT_FILE_ACCESS_TO=/home/node/data` (n8n 2.x restringe por defecto a `~/.n8n-files`) |
| `Node X hasn't been executed` | Pulsaste el play de un nodo suelto; usa **Execute Workflow** |
| `There is no connection back to...` | Un nodo se referencia con `$('...')` pero no está conectado por detrás |
| 24.480 filas en vez de 12 | `Execute Once` desactivado en `Read/Write Files from Disk` |
