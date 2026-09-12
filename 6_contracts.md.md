# Contratos HTTP — Versión 1: los 42 endpoints exactos

> Base: `http://localhost:8070` · Documentación interactiva en
> `/swagger`. Lo que este documento dice se cumple **al pie de la letra**
> (Artículo 9): un cliente puede exigirlo sin leer el código.

## 0. Convenciones globales

**Sobre de lectura** (listados):

```json
{ "tabla": "area_conocimiento", "limite": 1000, "total": 218, "datos": [ … ] }
```
**Sobre de error**:
```
{ "estado": 422, "mensaje": "Datos inválidos.", "detalle": "…",
  "errores": ["El campo granArea es obligatorio."] }
```
`errores[]` aparece **solo** en el 422.

**Los nombres de los campos JSON van en camelCase** (`granArea`, `nombre`,  
`terminoIngles`, `descripcion`, `filasAfectadas`), que es lo que [ASP.NET](https://asp.net/) Core hace **por defecto**: no hay que configurar nada, y por lo tanto no hay nada que se pueda configurar mal.

> **Ojo con la diferencia entre la ruta y el cuerpo.** La ruta es  
> `/api/area_conocimiento` —con guion bajo, porque nombra la tabla  
> (Artículo 10)— y el cuerpo usa `granArea`. No es una inconsistencia: la  
> ruta identifica **el recurso**, y el JSON sigue la convención de quien lo  
> consume. El front de la v4 va a leer `granArea` sin traducir nada.
> 
> Y algo que se ve mejor así: **el JSON no es una ventana a la tabla.** Si  
> mañana la columna se renombra, el contrato no tiene por qué cambiar —  
> justamente porque no son lo mismo.


**Catálogo de códigos** (Artículo 10):
| Situación | Código |
| :--- | :--- |
| Lectura correcta · escritura correcta | 200 |
| Lectura sin filas activas | 204 (sin cuerpo) |
| Regla de negocio rota (limite ≤ 0, PATCH sin campos) | 400 |
| Cuerpo inválido: falta un campo, tipo equivocado, texto muy largo | 422 |
| El ID no existe, o está inactivo | 404 |
| La base rechaza (llave duplicada) o falla | 500 (motor en detalle) |
## 1. `GET /` — Diagnóstico

````
GET /
→ 200 { "mensaje": "API Investigación — catálogos del módulo",
 "version": "v1",
 "contratos": "/swagger" }
````
**Sin desenlaces de error, y a propósito:** no recibe parámetros ni cuerpo,  
y no consulta la base. Si este endpoint no responde 200, el problema no es  
de contrato — es que la API no está arriba.
## 2. `GET /api/{tabla}[?limite=N]` — Listar
**Tablas aplicables:**  `area_conocimiento`, `objetivo_desarrollo_sostenible`,  
`area_aplicacion`, `termino_clave`, `universidad`, `linea_investigacion`.
````
GET /api/area_conocimiento
→ 200 { "tabla":"area_conocimiento", "limite":1000, "total":218,
        "datos":[
          {"id":1,"granArea":"Ciencias Naturales",
           "area":"Matemáticas","disciplina":"Matemáticas puras"},
          …
        ] }

GET /api/objetivo_desarrollo_sostenible
→ 200 { "tabla":"objetivo_desarrollo_sostenible", "limite":1000, "total":17,
        "datos":[
          {"id":1,"nombre":"Fin de la pobreza","categoria":"Social"},
          …
        ] }

GET /api/area_aplicacion
→ 200 { "tabla":"area_aplicacion", "limite":1000, "total":21,
        "datos":[
          {"id":1,"nombre":"Agricultura"},
          …
        ] }

GET /api/termino_clave
→ 200 { "tabla":"termino_clave", "limite":1000, "total":0,
        "datos":[] }

GET /api/universidad
→ 200 { "tabla":"universidad", "limite":1000, "total":6,
        "datos":[
          {"id":1,"nombre":"Universidad Nacional","tipo":"Pública","ciudad":"Bogotá"},
          …
        ] }

GET /api/linea_investigacion
→ 200 { "tabla":"linea_investigacion", "limite":1000, "total":0,
        "datos":[] }

GET /api/area_conocimiento?limite=3
→ 200 { …, "limite":3, "total":3, "datos":[ 3 elementos ] }

→ 204 (sin cuerpo) si no hay filas activas
→ 400 si limite <= 0
````
Devuelve **solo** las filas con `activo = 1`. El campo `activo`  **no viaja  
en la respuesta**: es un detalle interno, no parte del catálogo.

## 3. `GET /api/{tabla}/{id}` — Obtener una
**Tablas aplicables:** todas.
````
GET /api/area_conocimiento/1
→ 200 { "id":1, "granArea":"Ciencias Naturales",
        "area":"Matemáticas", "disciplina":"Matemáticas puras" }

GET /api/objetivo_desarrollo_sostenible/1
→ 200 { "id":1, "nombre":"Fin de la pobreza", "categoria":"Social" }

GET /api/area_aplicacion/1
→ 200 { "id":1, "nombre":"Agricultura" }

GET /api/termino_clave/IA
→ 200 { "termino":"IA", "terminoIngles":"Inteligencia Artificial" }

GET /api/universidad/1
→ 200 { "id":1, "nombre":"Universidad Nacional", "tipo":"Pública", "ciudad":"Bogotá" }

GET /api/linea_investigacion/1
→ 200 { "id":1, "nombre":"Línea de ejemplo", "descripcion":"Descripción de ejemplo" }

GET /api/area_conocimiento/999          ← no existe
→ 404 { "estado":404, "mensaje":"Área de conocimiento no encontrada.",
        "detalle":"No existe un área con el ID 999." }

GET /api/termino_clave/XYZ              ← no existe
→ 404 { "estado":404, "mensaje":"Término clave no encontrado.",
        "detalle":"No existe un término con el valor XYZ." }

GET /api/linea_investigacion/999        ← no existe
→ 404 { "estado":404, "mensaje":"Línea de investigación no encontrada.",
        "detalle":"No existe una línea con el ID 999." }
````
Una fila **inactiva** responde igual: 404 (C5)
## 4. `POST /api/{tabla}` — Crear

**Tablas aplicables:** todas.
### 4.1 `POST /api/area_conocimiento`

Cuerpo (petición `AreaConocimientoCrear` — los cuatro obligatorios):
````
POST /api/area_conocimiento
body {"id":999,"granArea":"Ciencias Naturales",
      "area":"Matemáticas","disciplina":"Teoría de números"}
→ 200 { "estado":200, "mensaje":"Área de conocimiento creada exitosamente." }

body {"id":999,"area":"Matemáticas"}        ← falta granArea y disciplina
→ 422 { "estado":422, "mensaje":"Datos inválidos.",
        "errores":["El campo granArea es obligatorio.",
                   "El campo disciplina es obligatorio."] }

body {"id":1, …}                            ← ID duplicado (PK)
→ 500 con el error del motor en detalle
````
### 4.2 `POST /api/objetivo_desarrollo_sostenible`
````
POST /api/objetivo_desarrollo_sostenible
body {"id":18,"nombre":"Nuevo ODS","categoria":"Social"}
→ 200 { "estado":200, "mensaje":"ODS creado exitosamente." }

body {"nombre":"Nuevo ODS"}                 ← falta categoria
→ 422 { "errores":["El campo categoria es obligatorio."] }
````
### 4.3 `POST /api/area_aplicacion`
````
POST /api/area_aplicacion
body {"id":22,"nombre":"Nueva área"}
→ 200 { "estado":200, "mensaje":"Área de aplicación creada exitosamente." }

body {"nombre":"Nueva área"}                ← falta id
→ 422 { "errores":["El campo id es obligatorio."] }
````
### 4.4 `POST /api/termino_clave`

````
POST /api/termino_clave
body {"termino":"NUEVO","terminoIngles":"NEW"}
→ 200 { "estado":200, "mensaje":"Término clave creado exitosamente." }
body {"termino":"NUEVO"}                    ← terminoIngles es opcional, no da error
→ 200 { "estado":200, "mensaje":"Término clave creado exitosamente." }
body {}                                     ← falta termino
→ 422 { "errores":["El campo termino es obligatorio."] }
````
### 4.5 `POST /api/universidad`

````

POST /api/universidad
body {"id":7,"nombre":"Nueva Universidad","tipo":"Privada","ciudad":"Medellín"}
→ 200 { "estado":200, "mensaje":"Universidad creada exitosamente." }
body {"nombre":"Nueva Universidad"}         ← falta tipo y ciudad
→ 422 { "errores":["El campo tipo es obligatorio.",
 "El campo ciudad es obligatorio."] }
````
### 4.6 `POST /api/linea_investigacion` — (IDENTITY, no se envía `id`)

````

POST /api/linea_investigacion
body {"nombre":"Nueva línea","descripcion":"Descripción de la línea"}
→ 200 { "estado":200, "mensaje":"Línea de investigación creada exitosamente." }
body {"nombre":"Nueva línea"}               ← falta descripcion
→ 422 { "errores":["El campo descripcion es obligatorio."] }
body {"id":999,"nombre":"Nueva línea","descripcion":"X"}   ← id no debe enviarse
→ 422 { "errores":["El campo id no debe enviarse (es autogenerado)."] }
````
**Regla común para todas:** El registro nace con `activo = 1`. **El cuerpo no acepta `activo`**: si llega, se ignora (§4 del modelo de datos).

**Para `linea_investigacion`:** el `id` es **IDENTITY**, por lo que no se envía en el `POST`. La base lo genera automáticamente.
 
 ## 5. `PUT /api/{tabla}/{id}` — Reemplazo COMPLETO

**Tablas aplicables:** todas.

### 5.1 `PUT /api/area_conocimiento/{id}`

````

PUT /api/area_conocimiento/999
body {"granArea":"Humanidades","area":"Filosofía","disciplina":"Ética"}
→ 200 { "estado":200, "mensaje":"Área de conocimiento reemplazada.",
 "filasAfectadas":1 }
body {"granArea":"Humanidades","disciplina":"Ética"}   ← falta area
→ 422 { …, "errores":["El campo area es obligatorio."] }
PUT /api/area_conocimiento/9999                        ← no existe
→ 404
````

### 5.2 `PUT /api/objetivo_desarrollo_sostenible/{id}`

````

PUT /api/objetivo_desarrollo_sostenible/18
body {"nombre":"ODS Actualizado","categoria":"Ambiental"}
→ 200 { "estado":200, "mensaje":"ODS reemplazado.",
 "filasAfectadas":1 }
body {"nombre":"ODS Actualizado"}          ← falta categoria
→ 422 { "errores":["El campo categoria es obligatorio."] }
````
### 5.3 `PUT /api/area_aplicacion/{id}`

````

PUT /api/area_aplicacion/22
body {"nombre":"Área actualizada"}
→ 200 { "estado":200, "mensaje":"Área de aplicación reemplazada.",
 "filasAfectadas":1 }
body {}                                     ← falta nombre
→ 422 { "errores":["El campo nombre es obligatorio."] }
````
### 5.4 `PUT /api/termino_clave/{termino}`

````

PUT /api/termino_clave/NUEVO
body {"terminoIngles":"NEW_UPDATED"}
→ 200 { "estado":200, "mensaje":"Término clave reemplazado.",
 "filasAfectadas":1 }
body {}                                     ← no se puede reemplazar sin campos
→ 422 { "errores":["El campo terminoIngles es obligatorio."] }
````
> **Nota:** En `termino_clave`, el `termino` es la PK y va en la URL. No  
> se puede cambiar en el PUT. El cuerpo solo recibe `terminoIngles` (y es  
> obligatorio en el PUT, porque el reemplazo exige todos los campos).

### 5.5 `PUT /api/universidad/{id}`

````
PUT /api/universidad/7
body {"nombre":"Universidad Actualizada","tipo":"Pública","ciudad":"Bogotá"}
→ 200 { "estado":200, "mensaje":"Universidad reemplazada.",
 "filasAfectadas":1 }
body {"nombre":"Universidad Actualizada"}  ← falta tipo y ciudad
→ 422 { "errores":["El campo tipo es obligatorio.",
 "El campo ciudad es obligatorio."] }
````
### 5.6 `PUT /api/linea_investigacion/{id}`

````

PUT /api/linea_investigacion/1
body {"nombre":"Línea actualizada","descripcion":"Nueva descripción"}
→ 200 { "estado":200, "mensaje":"Línea de investigación reemplazada.",
 "filasAfectadas":1 }
body {"nombre":"Línea actualizada"}        ← falta descripcion
→ 422 { "errores":["El campo descripcion es obligatorio."] }
````
**Regla común:**  **Todos los campos del cuerpo son obligatorios** en el PUT.  
Reemplazar es poner todo de nuevo. El ID no va en el cuerpo — identifica la  
fila, no se cambia.

## 6. `PATCH /api/{tabla}/{id}` — Actualización PARCIAL

**Tablas aplicables:** todas.

### 6.1 `PATCH /api/area_conocimiento/{id}`

````

PATCH /api/area_conocimiento/999
body {"disciplina":"Ética aplicada"}          ← solo lo que cambia
→ 200 { "estado":200, "mensaje":"Área de conocimiento actualizada.",
 "filasAfectadas":1 }
body {"granArea":"Humanidades","disciplina":"Ética"}   ← el MISMO cuerpo que
 el PUT rechazó
→ 200                                                   ← aquí es válido
body {}                                       ← nada que actualizar
→ 400 { "estado":400, "mensaje":"Parámetros inválidos.",
 "detalle":"No se envió ningún campo para actualizar." }
PATCH /api/area_conocimiento/9999
→ 404
````
### 6.2 `PATCH /api/objetivo_desarrollo_sostenible/{id}`

````

PATCH /api/objetivo_desarrollo_sostenible/18
body {"categoria":"Ambiental"}
→ 200 { "estado":200, "mensaje":"ODS actualizado.",
 "filasAfectadas":1 }
````
### 6.3 `PATCH /api/area_aplicacion/{id}`


````
PATCH /api/area_aplicacion/22
body {"nombre":"Nuevo nombre"}
→ 200 { "estado":200, "mensaje":"Área de aplicación actualizada.",
 "filasAfectadas":1 }
````
### 6.4 `PATCH /api/termino_clave/{termino}`

````
PATCH /api/termino_clave/NUEVO
body {"terminoIngles":"NEW_UPDATED"}
→ 200 { "estado":200, "mensaje":"Término clave actualizado.",
 "filasAfectadas":1 }
body {}                                       ← nada que actualizar
→ 400
````
### 6.5 `PATCH /api/universidad/{id}`

````

PATCH /api/universidad/7
body {"ciudad":"Cali"}
→ 200 { "estado":200, "mensaje":"Universidad actualizada.",
 "filasAfectadas":1 }
````
### 6.6 `PATCH /api/linea_investigacion/{id}`

````

PATCH /api/linea_investigacion/1
body {"descripcion":"Descripción actualizada"}
→ 200 { "estado":200, "mensaje":"Línea de investigación actualizada.",
 "filasAfectadas":1 }
````
**Regla común:** Solo se modifican los campos enviados. El ID no se cambia.  
Cuerpo vacío → 400.

**Esta pareja es la lección del contrato:** el mismo cuerpo da 422 en `PUT`  
y 200 en `PATCH`. No es un capricho — reemplazar exige todo, actualizar  
solo lo enviado.

## 7. `DELETE /api/{tabla}/{id}` — Eliminar (LÓGICO)

**Tablas aplicables:** todas.

```

DELETE /api/area_conocimiento/999
→ 200 { "estado":200, "mensaje":"Área de conocimiento eliminada.",
 "filasAfectadas":1 }
DELETE /api/area_conocimiento/999           ← segunda vez: ya está inactiva
→ 404
DELETE /api/area_conocimiento/9999          ← nunca existió
→ 404
```
**La fila no se borra:** queda con `activo = 0` y desaparece de los  
listados. Comprobarlo es el criterio 5 de la spec: el `total` vuelve al  
valor inicial y la fila sigue en la base.
## 8. El contrato de la PANTALLA

Los siete apartados anteriores son el contrato de la API con cualquiera que la consuma. Este es el de la pantalla con quien la usa, y son dos contratos distintos: el front es un cliente de la API, no el cliente.

| Pantalla | Dirección | Qué ofrece |
|---|---|---|
| Inicio | `http://localhost:8071/` | La entrada, con el menú a los 6 catálogos |
| Áreas de conocimiento | `http://localhost:8071/areas-de-conocimiento` | Tabla, «Agregar», «Editar» y «Retirar» |
| ODS | `http://localhost:8071/ods` | Tabla, «Agregar», «Editar» y «Retirar» |
| Áreas de aplicación | `http://localhost:8071/areas-aplicacion` | Tabla, «Agregar», «Editar» y «Retirar» |
| Términos clave | `http://localhost:8071/terminos-clave` | Tabla, «Agregar», «Editar» y «Retirar» |
| Universidades | `http://localhost:8071/universidades` | Tabla, «Agregar», «Editar» y «Retirar» |
| Líneas de investigación | `http://localhost:8071/lineas-investigacion` | Tabla, «Agregar», «Editar» y «Retirar» |

Cada pantalla tiene dirección propia, no una con el nombre de la tabla como parámetro (Artículo 10.1 · sección 6.1 de la metodología). Se puede guardar como marcador, poner en el menú y mandar por correo.

### Qué pantalla llama a qué endpoint

| Lo que hace el usuario | Lo que manda el front |
|---|---|
| Abrir la pantalla | `GET /api/{tabla}?limite=1000` |
| «Agregar» y guardar | `POST /api/{tabla}` |
| «Editar» | `GET /api/{tabla}/{id}` (ya viene en el listado) |
| «Guardar la ficha completa» | `PUT /api/{tabla}/{id}` |
| «Guardar solo lo que cambié» | `PATCH /api/{tabla}/{id}` con solo lo diligenciado |
| «Retirar», tras confirmar | `DELETE /api/{tabla}/{id}` |

### Cómo traduce el front los errores de la API

El front no repite ninguna validación de la API: manda, y muestra lo que vuelva. Su servicio traduce el sobre a una lista de textos:

| Lo que responde la API | Lo que ve el usuario |
|---|---|
| **422** con `errores[]` | Un aviso rojo por cada error, con el texto que mandó la API |
| **400 / 404 / 500** con `{mensaje, detalle}` | Un aviso rojo con esos dos textos |
| **La API no responde** | «El servicio no está disponible. ¿Está arriba la API?» |

La última fila es la que demuestra la arquitectura. Con la API apagada la pantalla sigue en pie —cabecera, menú, pie— y muestra ese aviso sin un solo dato. Si el front pudiera llegar a SQL Server por su cuenta, seguiría mostrando el catálogo.

Lo comprueba `pruebas_humo/humo_front.py`, que apaga la API a propósito.
