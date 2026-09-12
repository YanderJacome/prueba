# Especificación — Versión 1: Catálogos del módulo Investigación

> **Versión 1** ([mapa](../0_mapa_versiones.md)) · La primera rebanada
> vertical del módulo Investigación: **las seis tablas sin clave foránea**,
> sus siete endpoints cada una y las tres capas completas. Ante conflicto con
> este documento, manda la [constitución](../../1_constitution.md).

## 0. Qué entrega esta versión, en una línea

**El CRUD de los seis catálogos del módulo, de punta a punta: su API y sus
pantallas.**

Tres procesos que se levantan con un comando:
FRONT Blazor (:8071) ──HTTP──> API C# (:8070) ──SQL──> SQL Server (:11470)


El front **no tiene línea hacia la base de datos**, y no la va a tener nunca.

## 1. Propósito de la v1

Construir la API y las pantallas de los **seis catálogos sin dependencias
foráneas salientes** del módulo Investigación:

- `area_conocimiento` (218 filas)
- `objetivo_desarrollo_sostenible` (17 filas)
- `area_aplicacion` (21 filas)
- `termino_clave` (0 filas en el catálogo de referencia)
- `universidad` (6 filas)
- `linea_investigacion` (0 filas en el catálogo de referencia)

La v1 no busca cubrir el módulo completo: busca **dejar el patrón montado y
verificado** para todas las tablas sin FK salientes. Las tablas con FK
(`docente`, `grupo_investigacion`, `semillero`, y las relaciones muchos-a-muchos)
son la v2 y v3.

> **¿Por qué `linea_investigacion` está en la v1 si es referenciada por otras
> tablas?** La regla de la metodología dice "tablas sin FK", es decir, tablas
> que **no declaran llaves foráneas en su propia estructura**. `linea_investigacion`
> no tiene FK salientes, solo entrantes (otras tablas la referencian). Así como
> `area_conocimiento` es referenciada por `ac_linea` y está en la v1, lo mismo
> aplica a `linea_investigacion`.

## 2. Alcance

**Incluye**

- El CRUD completo de **cada una de las 6 tablas sin FK salientes**: listar
  (con límite), obtener por ID, crear, reemplazar, actualizar parcialmente y
  eliminar.
- **Borrado lógico**: `DELETE` marca `activo = 0` y los listados filtran los
  inactivos (Artículo 6 de la constitución).
- Un endpoint de diagnóstico y la documentación interactiva en `/swagger`.
- La prueba de capas: el servicio corriendo con un repositorio falso, sin
  base de datos.
- **6 pantallas en Blazor Server**, una por cada tabla, con su propia ruta
  (ej. `/areas-de-conocimiento`, `/ods`, `/areas-aplicacion`,
  `/terminos-clave`, `/universidades`, `/lineas-investigacion`).

**NO incluye** — y no se anticipa nada de esto (Artículo 1)

- Ninguna tabla con FK salientes: `docente`, `grupo_investigacion`,
  `semillero`, `participa_semillero`, `participa_grupo`, `semillero_linea`,
  `grupo_linea`, `ac_linea`, `ods_linea`, `aa_linea`.
- Autenticación, JWT, roles ni usuarios: eso es la v3.
- Reactivar un registro inactivo (`activo = 1`). Nadie lo pidió.
- Búsqueda por texto, ordenamiento ni paginación con desplazamiento: el
  único filtro de la v1 es `?limite`.

## 3. Requisitos funcionales

### RF1 — Listar (GET + query string)

`GET /api/{tabla}` → 200 con el sobre `{tabla, limite, total, datos:[…]}`.

- Devuelve **solo las activas**.
- Parámetro opcional `limite` (entero > 0; por defecto 1000).
- Sin filas activas → **204** sin cuerpo.

**Tablas aplicables:** `area_conocimiento`, `objetivo_desarrollo_sostenible`,
`area_aplicacion`, `termino_clave`, `universidad`, `linea_investigacion`.

### RF2 — Obtener por ID (GET + parámetro de ruta)

`GET /api/{tabla}/{id}` → 200 con el registro.

- El `id` puede ser numérico (INT) o texto (`termino` en `termino_clave`).
- Para `linea_investigacion`, el `id` es numérico (IDENTITY, la base lo genera).
- Inexistente **o inactivo** → 404.

### RF3 — Crear (POST + cuerpo completo)

`POST /api/{tabla}` con todos los campos obligatorios (según la tabla).

- Nace con `activo = 1`.
- ID ya existente → **500** (la base defiende la PK).
- **Para `linea_investigacion`:** el `id` es IDENTITY, **no se envía en el POST**.
  El cuerpo solo lleva `nombre` y `descripcion`.

### RF4 — Reemplazar (PUT + cuerpo completo)

`PUT /api/{tabla}/{id}` con todos los campos del cuerpo (el ID va en la URL).

- **Todos los campos del cuerpo son obligatorios**: es un reemplazo.
- Falta uno → 422.
- Devuelve `filasAfectadas`; inexistente → 404.
- **Para `linea_investigacion`:** el cuerpo solo lleva `nombre` y `descripcion`.

### RF5 — Actualizar parcialmente (PATCH + cuerpo parcial)

`PATCH /api/{tabla}/{id}` con los campos que se quieran cambiar.

- Solo se modifican los enviados.
- Cuerpo vacío → 400.
- Devuelve `filasAfectadas`; inexistente → 404.

### RF6 — Eliminar (DELETE, borrado lógico)

`DELETE /api/{tabla}/{id}` marca `activo = 0`.

- Devuelve `filasAfectadas`.
- Inexistente **o ya inactiva** → 404.

### RF7 — Diagnóstico

`GET /` → JSON con mensaje, versión (`"v1"`) y la ruta de los contratos.

### RF8 — Las PANTALLAS de los seis catálogos

En `http://localhost:8071/` hay un menú con enlaces a cada catálogo:

| Pantalla | Ruta | Tabla |
|----------|------|-------|
| Áreas de conocimiento | `/areas-de-conocimiento` | `area_conocimiento` |
| ODS | `/ods` | `objetivo_desarrollo_sostenible` |
| Áreas de aplicación | `/areas-aplicacion` | `area_aplicacion` |
| Términos clave | `/terminos-clave` | `termino_clave` |
| Universidades | `/universidades` | `universidad` |
| Líneas de investigación | `/lineas-investigacion` | `linea_investigacion` |

**Cada pantalla** ofrece:

- Una tabla con todas las columnas.
- Un formulario para agregar.
- El mismo formulario para editar, con **dos botones**:
  - «Guardar la ficha completa» (PUT)
  - «Guardar solo lo que cambié» (PATCH)
- Un botón «Retirar» que pide confirmación.

**Tres reglas de estas pantallas** (se comprueban en todas):

1. **No le hablan al usuario en jerga.** Ni «PUT», ni «422», ni el nombre de
   la tabla. Los botones se llaman como el usuario piensa.
2. **Un error no pierde lo escrito.** Si la API rechaza el guardado, el
   formulario vuelve con lo que la persona había digitado.
3. **Vacío no es error.** Un catálogo sin fichas muestra un recuadro que lo
   dice, no un aviso rojo.

## 4. Requisitos no funcionales

- **Un solo comando**: `docker compose up -d --build` (Artículo 4).
- **Tres capas con interfaces** y solo el ensamblador conociendo clases
  concretas (Artículo 3).
- **SQL a mano y siempre parametrizado** (`@parametro`), sin ORM de
  entidades (Artículo 2). *(Esta línea se mantiene como está, aunque en tu
  plan usas EF Core; el profesor la incluye en la spec funcional. La
  desviación se documenta en el plan.)*
- **Todo en español** (Artículo 8).
- **Cada listado** de las 6 tablas responde en **menos de 1 segundo** en un
  equipo de escritorio corriente.
- La API publica su documentación interactiva en `/swagger`.
- El front es un **tercer proceso** y no comparte código con la API.

## 5. Criterios de aceptación

1. **Un solo comando.** `docker compose up -d --build` deja corriendo SQL
   Server —con la base creada, sus 19 tablas y las semillas de los 6
   catálogos—, la API y el front. `GET http://localhost:8070/` responde el
   diagnóstico con `"version":"v1"`.

2. **Listar cada tabla.**
   - `GET /api/area_conocimiento` devuelve `total:218`
   - `GET /api/objetivo_desarrollo_sostenible` devuelve `total:17`
   - `GET /api/area_aplicacion` devuelve `total:21`
   - `GET /api/termino_clave` devuelve `total:0` (vacía)
   - `GET /api/universidad` devuelve `total:6`
   - `GET /api/linea_investigacion` devuelve `total:0` (vacía)
   - Con `?limite=3` cada una devuelve **exactamente 3** (o todas si hay menos).

3. **Obtener por ID.** Para cada tabla, un ID existente devuelve el registro
   correcto; un ID inexistente → **404**.

4. **Ciclo de los cinco verbos en cada tabla.** `POST` crea un registro de
   prueba → `PUT` lo reemplaza completo → `PATCH` le cambia un campo →
   `GET` lo confirma → `DELETE` lo desactiva, y un **segundo** `DELETE`
   responde **404**. Además, un `PUT` sin un campo obligatorio responde
   **422** mientras el **mismo cuerpo** enviado por `PATCH` responde
   **200**.

5. **El borrado es lógico, y se verifica en cada tabla.** Después del
   `DELETE` el `total` vuelve al valor inicial, y la fila sigue en la base
   con `activo = 0`.

6. **La validación es la frontera.** Para cada tabla, un `POST` sin un campo
   obligatorio → **422** con `errores:[…]`; un `POST` con ID duplicado →
   **500**. Para `linea_investigacion`, el `id` no se envía (es IDENTITY);
   enviarlo es un error que el controlador debe ignorar o rechazar.

7. **Prueba de capas.** El proyecto `pruebas/` ejecuta el servicio con un
   **repositorio de mentiras** (lista en memoria) para **cada una de las 6
   tablas**. Todas las verificaciones pasan **con SQL Server apagado**.

8. **Las 6 pantallas muestran sus catálogos.** Cada ruta del front trae los
   datos que dio la API. Se comprueba pidiendo a la API las primeras filas y
   buscándolas en la pantalla.

9. **El ciclo completo se puede hacer desde cada pantalla**, sin tocar
   Swagger ni `curl`: agregar, editar con los dos botones y retirar, en cada
   una de las 6 tablas.

10. **Ninguna pantalla le habla al usuario en jerga.** No aparecen «PUT»,
    «422», ni el nombre de la tabla.

11. **SON DOS PROCESOS, y se demuestra apagando uno.** Con
    `docker compose stop api-investigacion`, la pantalla **sigue respondiendo**
    —con su menú, su marco y su pie—, muestra «El servicio no está disponible»
    y **no muestra ni un dato**.

## 6. Clarificaciones

| # | La pregunta | La respuesta, con su razón | Dónde quedó |
|---|---|---|---|
| C1 | `termino_clave` tiene PK de texto (`termino`). ¿La API lo maneja igual que las demás? | **Sí.** El tipo del ID en la ruta es `string`; para las tablas con ID numérico se convierte a `int` en el controlador. | `5_data_model` §2 |
| C2 | `termino_ingles` es opcional (NULL). ¿Cómo se maneja en el contrato? | **No es obligatorio.** En `POST` y `PUT` se valida solo `termino`; `termino_ingles` puede ser `null` o no enviarse. | `6_contracts` |
| C3 | `area_conocimiento.disciplina` tiene VARCHAR(60), pero el catálogo tiene valores de hasta 124 caracteres (error del script). | **Se amplía a VARCHAR(150)** en el script de semillas. | `db/investigacion.sql` · `5_data_model` |
| C4 | Ninguna tabla del módulo trae `activo`. ¿Se agrega? | **Sí, a todas las 16 tablas del módulo.** La constitución (Artículo 6) lo exige. | Artículo 6 · RF6 |
| C5 | Un registro inactivo, ¿se puede consultar por su ID? | **No: responde 404.** Coherente con el listado. | RF2 · RF6 |
| C6 | ¿Se puede reactivar? | **No en la v1.** Queda en el NO incluye. | §2 Alcance |
| C7 | `?limite=0` o negativo, ¿es 422 o 400? | **400.** Es una regla de negocio. | RF1 · Artículo 10 |
| C8 | Crear con ID duplicado, ¿409 o 500? | **500.** La base defiende la PK; la v1 no tiene lógica de negocio para 409. | RF3 · criterio 6 |
| C9 | Los scripts no tienen datos para `termino_clave` ni para `linea_investigacion`. ¿Se dejan vacías? | **Sí.** El catálogo de referencia no las llena; quedan vacías y los criterios lo reflejan. | §3 RF1 · `7_quickstart` |
| C10 | `linea_investigacion.id` es IDENTITY. ¿Cómo se maneja en el POST? | **No se envía.** La base lo genera automáticamente. El cuerpo del POST solo lleva `nombre` y `descripcion`. | `3_plan` §4 · `6_contracts` §4.6 |

## 7. Definición de TERMINADA

La v1 está terminada —y solo entonces se escribe la spec de la v2— cuando:

1. Los **11 criterios de aceptación** pasan, verificados con el smoke test
   de [7_quickstart.md](7_quickstart.md) **corrido por una persona**.
2. La lista de [9_checklist.md](9_checklist.md) está en verde y firmada.
3. No queda ningún `[NECESITA ACLARACIÓN: …]` en este documento.
4. Se hace commit y **tag `v1`** (Artículo 1).