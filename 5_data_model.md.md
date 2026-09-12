# Modelo de datos — Versión 1: la base dada y los seis catálogos

## 1. La base viene completa; la v1 nombra seis tablas

La base `investigacion_local` se crea con sus **19 tablas** desde la
primera versión (Artículo 5): 16 del módulo y 3 de gestión de usuarios.
Eso es infraestructura **dada**, no algo que la v1 construya.

Lo que la v1 tiene permitido **nombrar en el código** son **las seis
tablas sin clave foránea saliente**:

- `area_conocimiento` (218 filas)
- `objetivo_desarrollo_sostenible` (17 filas)
- `area_aplicacion` (21 filas)
- `termino_clave` (0 filas en el catálogo de referencia)
- `universidad` (6 filas)
- `linea_investigacion` (0 filas en el catálogo de referencia)

Cualquier `SELECT`, `INSERT` o `JOIN` que mencione otra tabla viola el
alcance (Artículo 1). Las tablas con FK salientes (`docente`, `grupo_investigacion`,
`semillero`, y las relaciones muchos-a-muchos) son territorio de la v2 y v3.

> **¿Por qué `linea_investigacion` está en la v1 si es referenciada por otras
> tablas?** La regla de la metodología dice "tablas sin FK", es decir, tablas
> que **no declaran llaves foráneas en su propia estructura**. `linea_investigacion`
> no tiene FK salientes, solo entrantes (otras tablas la referencian). Así como
> `area_conocimiento` es referenciada por `ac_linea` y está en la v1, lo mismo
> aplica a `linea_investigacion`.

## 2. Las seis tablas sin FK salientes

### 2.1 `area_conocimiento`

| Columna | Tipo | Regla |
|---|---|---|
| `id` | `INT` | **PK** — numérico, correlativo en el catálogo (1, 2, 3…) |
| `gran_area` | `VARCHAR(60)` | No nulo, no vacío. Seis valores posibles (§3.1) |
| `area` | `VARCHAR(60)` | No nulo, no vacío |
| `disciplina` | `VARCHAR(150)` | No nulo. Ampliada desde 60 por el valor de 124 caracteres del catálogo (C3) |
| `activo` | `BIT NOT NULL DEFAULT 1` | Borrado lógico (C4). La API la escribe **solo** vía `DELETE` |

**Semillas:** 218 filas, con códigos `1A01` … `6E03` y las seis grandes áreas.
**Jerarquía:** gran área → área → disciplina (codificada en el `id`, pero la v1 no valida ese formato).

**Tres filas reales, para los ejemplos de los contratos:**
1A01 Ciencias Naturales Matemáticas Matemáticas puras  
1A02 Ciencias Naturales Matemáticas Matemáticas aplicadas  
6E03 Humanidades Otras Humanidades Teología

### 2.2 `objetivo_desarrollo_sostenible`

| Columna | Tipo | Regla |
|---|---|---|
| `id` | `INT` | **PK** — numérico (1 a 17) |
| `nombre` | `VARCHAR(60)` | No nulo, no vacío |
| `categoria` | `VARCHAR(45)` | No nulo, no vacío |
| `activo` | `BIT NOT NULL DEFAULT 1` | Borrado lógico |

**Semillas:** 17 filas (los ODS oficiales). Ejemplo: `1` · «Fin de la pobreza» · «Social».

### 2.3 `area_aplicacion`

| Columna | Tipo | Regla |
|---|---|---|
| `id` | `INT` | **PK** — numérico |
| `nombre` | `VARCHAR(60)` | No nulo, no vacío |
| `activo` | `BIT NOT NULL DEFAULT 1` | Borrado lógico |

**Semillas:** 21 áreas de aplicación (ej. «Agricultura», «Salud», «Educación»).

### 2.4 `termino_clave`

| Columna | Tipo | Regla |
|---|---|---|
| `termino` | `VARCHAR(30)` | **PK** — texto, no numérico (C1) |
| `termino_ingles` | `VARCHAR(30)` | Opcional (puede ser NULL) (C2) |
| `activo` | `BIT NOT NULL DEFAULT 1` | Borrado lógico |

**Semillas:** El catálogo de referencia no tiene datos para esta tabla; queda vacía en la v1 (C9).

### 2.5 `universidad`

| Columna | Tipo | Regla |
|---|---|---|
| `id` | `INT` | **PK** — numérico |
| `nombre` | `VARCHAR(60)` | No nulo, no vacío |
| `tipo` | `VARCHAR(45)` | No nulo, no vacío (ej. «Pública», «Privada») |
| `ciudad` | `VARCHAR(45)` | No nulo, no vacío |
| `activo` | `BIT NOT NULL DEFAULT 1` | Borrado lógico |

**Semillas:** 6 universidades de referencia (ej. «Universidad Nacional», «Universidad de Antioquia»).

### 2.6 `linea_investigacion`

| Columna | Tipo | Regla |
|---|---|---|
| `id` | `INT IDENTITY(1,1)` | **PK** — numérico, **autogenerado por la base** (IDENTITY) |
| `nombre` | `VARCHAR(45)` | No nulo, no vacío |
| `descripcion` | `VARCHAR(256)` | No nulo, no vacío |
| `activo` | `BIT NOT NULL DEFAULT 1` | Borrado lógico |

**Semillas:** El catálogo de referencia no tiene datos para esta tabla; queda vacía en la v1 (C9).

**Particularidad:** El `id` es **IDENTITY**, lo que significa que la base lo genera automáticamente al insertar. Por lo tanto:
- El `POST /api/linea_investigacion` **no recibe `id`** en el cuerpo; solo `nombre` y `descripcion`.
- El `PUT` y `PATCH` usan el `id` en la ruta para identificar la fila, pero **no lo cambian**.
- La respuesta del `POST` puede devolver el `id` generado si el cliente lo necesita (ej. en el `Location` header o en el cuerpo de la respuesta).

## 3. Las semillas: resumen de filas

| Tabla | Filas | Observación |
| :--- | :--- | :--- |
| `area_conocimiento` | 218 | Catálogo completo de áreas |
| `objetivo_desarrollo_sostenible` | 17 | ODS oficiales |
| `area_aplicacion` | 21 | Áreas de aplicación |
| `termino_clave` | 0 | Vacía en el catálogo de referencia |
| `universidad` | 6 | Universidades de referencia |
| `linea_investigacion` | 0 | Vacía en el catálogo de referencia |

## 4. Invariantes: quién escribe qué en cada tabla

| Dato | Dueño | La API… |
|---|---|---|
| `id` (numérico o texto) | Quien crea el registro | Lo escribe **solo** en el `POST` para tablas sin IDENTITY. Para `linea_investigacion`, **no se envía** (la base lo genera). Un `PUT` o un `PATCH` **nunca** cambian el ID: identifica la fila. |
| Campos de contenido (`gran_area`, `nombre`, `termino_ingles`, `descripcion`, etc.) | La API | Los escribe libremente en `POST`, `PUT` y `PATCH` (según las reglas de cada verbo). |
| `activo` | La API, pero **solo** por `DELETE` | **Tiene prohibido** recibirlo en el cuerpo de `POST`, `PUT` o `PATCH`. Si llega, se ignora: reactivar no está en el alcance (C6). |
| Las otras 13 tablas del módulo y las 3 de usuarios | Nadie, en la v1 | No las nombra. |

## 5. Reglas de esta versión

1. **Todas las consultas se hacen a través del repositorio.** Con EF Core, se usa LINQ y el `DbContext`; no se escribe SQL a mano (Artículo 2, con la autorización del profesor).
2. **Todo listado** debe filtrar solo los registros activos (`WHERE activo = 1` en SQL, o `.Where(x => x.Activo)` en LINQ).
3. **La v1 no crea, altera ni borra objetos de la base:** el esquema viene dado desde `db/investigacion.sql`.
4. **Las PK** son `INT` excepto en `termino_clave` (texto). `linea_investigacion` tiene `INT IDENTITY` (autogenerado). La API debe manejar cada tipo correctamente.
5. **`termino_ingles` es opcional:** puede ser `NULL` en la base.
6. **La columna `activo` se agregó a todas las tablas del módulo** (Artículo 6 de la constitución). El script `db/investigacion.sql` la incluye con `DEFAULT 1`.
7. **Para `linea_investigacion`:** en el `POST` no se envía `id`. El controlador construye la entidad sin `Id` y la base lo genera al guardar.