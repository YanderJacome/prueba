# Quickstart — Versión 1: arranque y smoke test

## 1. Arranque

Un solo comando, desde la raíz del proyecto:

```powershell
docker compose up -d --build
```
La primera vez tarda unos minutos: descarga la imagen de SQL Server,  
espera a que el motor responda, crea la base con sus 19 tablas y sus semillas,  
y compila la API y el front. Al terminar:
 | Qué | Dónde |
|---|---|
| API — diagnóstico | http://localhost:8070/ |
| Documentación interactiva | http://localhost:8070/swagger |
| Front — pantalla de inicio | http://localhost:8071/ |
| Listado de áreas | http://localhost:8070/api/area_conocimiento |
| Listado de líneas de investigación | http://localhost:8070/api/linea_investigacion |
| SQL Server (SSMS o SQLTools, opcional) | `localhost,11470` · usuario `sa` |

> **¿La contraseña?** Está en el `docker-compose.yml`, a la vista: esta es una plantilla didáctica y esa es la excepción declarada en el Artículo 7 de la constitución. Para correr el sistema no hace falta —el compose se la entrega a los contenedores—; solo se necesita para conectarse por fuera con SSMS o SQLTools.
>
> En su proyecto de aula eso no se copia: ahí va en un `.env` fuera de git, con un `.env.example` adentro.

Si cambia la contraseña, no basta con editar el compose:

```powershell
docker compose down -v        # -v borra el volumen: la base olvida el sa viejo
docker compose up -d --build
```
Sin el `-v`, el usuario `sa` sigue existiendo dentro del volumen con la clave anterior y el login falla — con un error que no menciona los volúmenes por ninguna parte.

## 2. Smoke test de la API

Los comandos van numerados igual que los criterios de aceptación de `2_spec.md`. Si los 11 pasan, la versión está terminada.

```powershell
# 1. Un solo comando: la API responde y dice qué versión es
curl http://localhost:8070/
#    → {"mensaje":"API Investigación — catálogos del módulo","version":"v1","contratos":"/swagger"}
```

### 2.1 Listar cada tabla (criterio 2)

```powershell
# Áreas de conocimiento: 218
curl http://localhost:8070/api/area_conocimiento
#    → {"tabla":"area_conocimiento","limite":1000,"total":218,"datos":[...]}

# ODS: 17
curl http://localhost:8070/api/objetivo_desarrollo_sostenible
#    → {"tabla":"objetivo_desarrollo_sostenible","limite":1000,"total":17,"datos":[...]}

# Áreas de aplicación: 21
curl http://localhost:8070/api/area_aplicacion
#    → {"tabla":"area_aplicacion","limite":1000,"total":21,"datos":[...]}

# Términos clave: 0 (vacía)
curl http://localhost:8070/api/termino_clave
#    → {"tabla":"termino_clave","limite":1000,"total":0,"datos":[]}

# Universidades: 6
curl http://localhost:8070/api/universidad
#    → {"tabla":"universidad","limite":1000,"total":6,"datos":[...]}

# Líneas de investigación: 0 (vacía)
curl http://localhost:8070/api/linea_investigacion
#    → {"tabla":"linea_investigacion","limite":1000,"total":0,"datos":[]}

# Con límite: exactamente 3
curl "http://localhost:8070/api/area_conocimiento?limite=3"
#    → total: 3
```

### 2.2 Obtener por ID (criterio 3)

```powershell
# Área que existe
curl http://localhost:8070/api/area_conocimiento/1
#    → {"id":1,"granArea":"Ciencias Naturales","area":"Matemáticas","disciplina":"Matemáticas puras"}

# Área que no existe
curl -i http://localhost:8070/api/area_conocimiento/999
#    → 404

# Término clave que existe (PK de texto)
curl http://localhost:8070/api/termino_clave/IA
#    → {"termino":"IA","terminoIngles":"Inteligencia Artificial"}

# Término que no existe
curl -i http://localhost:8070/api/termino_clave/XYZ
#    → 404

# Línea de investigación que no existe (vacía)
curl -i http://localhost:8070/api/linea_investigacion/1
#    → 404
```

### 2.3 Ciclo de los cinco verbos en cada tabla (criterio 4)

**Para `area_conocimiento` (ID numérico, se envía en POST):**

```powershell
# POST: crear
curl -X POST http://localhost:8070/api/area_conocimiento `
  -H "Content-Type: application/json" `
  -d '{"id":999,"granArea":"Ciencias Naturales","area":"Matemáticas","disciplina":"Teoría de números"}'
#    → 200

# PUT: reemplazo completo
curl -X PUT http://localhost:8070/api/area_conocimiento/999 `
  -H "Content-Type: application/json" `
  -d '{"granArea":"Humanidades","area":"Filosofía","disciplina":"Ética"}'
#    → 200 filasAfectadas: 1

# PATCH: actualización parcial
curl -X PATCH http://localhost:8070/api/area_conocimiento/999 `
  -H "Content-Type: application/json" -d '{"disciplina":"Ética aplicada"}'
#    → 200 filasAfectadas: 1

# GET: confirmar
curl http://localhost:8070/api/area_conocimiento/999
#    → Humanidades / Filosofía / Ética aplicada

# DELETE: borrado lógico
curl -X DELETE http://localhost:8070/api/area_conocimiento/999
#    → 200

# DELETE repetido → 404
curl -i -X DELETE http://localhost:8070/api/area_conocimiento/999
#    → 404
```

**Para `linea_investigacion` (IDENTITY, no se envía id en POST):**

```powershell
# POST: crear (sin id, la base lo genera)
curl -X POST http://localhost:8070/api/linea_investigacion `
  -H "Content-Type: application/json" `
  -d '{"nombre":"Línea de prueba","descripcion":"Descripción de prueba"}'
#    → 200, devuelve el id generado (ej. 1)

# PUT: reemplazo completo
curl -X PUT http://localhost:8070/api/linea_investigacion/1 `
  -H "Content-Type: application/json" `
  -d '{"nombre":"Línea actualizada","descripcion":"Nueva descripción"}'
#    → 200 filasAfectadas: 1

# PATCH: actualización parcial
curl -X PATCH http://localhost:8070/api/linea_investigacion/1 `
  -H "Content-Type: application/json" -d '{"descripcion":"Descripción actualizada"}'
#    → 200 filasAfectadas: 1

# GET: confirmar
curl http://localhost:8070/api/linea_investigacion/1
#    → {"id":1,"nombre":"Línea actualizada","descripcion":"Descripción actualizada"}

# DELETE: borrado lógico
curl -X DELETE http://localhost:8070/api/linea_investigacion/1
#    → 200
```

**La pareja que enseña la diferencia: MISMO cuerpo, dos verbos:**

```powershell
# Primero creamos un área de prueba
curl -X POST http://localhost:8070/api/area_conocimiento `
  -H "Content-Type: application/json" `
  -d '{"id":888,"granArea":"Prueba","area":"Prueba","disciplina":"Prueba"}'

# PUT con un solo campo → 422 (reemplazo exige todo)
curl -i -X PUT http://localhost:8070/api/area_conocimiento/888 `
  -H "Content-Type: application/json" `
  -d '{"granArea":"Humanidades","disciplina":"Ética"}'
#    → 422: falta 'area'

# PATCH con el MISMO cuerpo → 200 (actualización parcial)
curl -i -X PATCH http://localhost:8070/api/area_conocimiento/888 `
  -H "Content-Type: application/json" `
  -d '{"granArea":"Humanidades","disciplina":"Ética"}'
#    → 200
```

### 2.4 El borrado es LÓGICO, y se comprueba (criterio 5)

```powershell
# Ver total antes (218 + 1 de prueba = 219)
curl http://localhost:8070/api/area_conocimiento
#    → total: 219

# Eliminar
curl -X DELETE http://localhost:8070/api/area_conocimiento/888
#    → 200 filasAfectadas: 1

# Ver total después (vuelve a 218)
curl http://localhost:8070/api/area_conocimiento
#    → total: 218

# Pero la fila SIGUE en la base (activo=0). Comprobarlo:
docker compose exec sqlserver bash -c '/opt/mssql-tools18/bin/sqlcmd `
  -S localhost -U sa -P "$MSSQL_SA_PASSWORD" -C -d investigacion_local `
  -Q "SELECT id, activo FROM area_conocimiento WHERE id = 888"'
#    → 888 | 0
```

### 2.5 La validación es la frontera (criterio 6)

```powershell
# POST sin campo obligatorio → 422
curl -i -X POST http://localhost:8070/api/area_conocimiento `
  -H "Content-Type: application/json" `
  -d '{"id":999,"area":"Matemáticas","disciplina":"X"}'
#    → 422 con errores: falta granArea

# POST con ID duplicado → 500
curl -i -X POST http://localhost:8070/api/area_conocimiento `
  -H "Content-Type: application/json" `
  -d '{"id":1,"granArea":"X","area":"Y","disciplina":"Z"}'
#    → 500

# POST para linea_investigacion enviando id (no debe enviarse) → 422
curl -i -X POST http://localhost:8070/api/linea_investigacion `
  -H "Content-Type: application/json" `
  -d '{"id":999,"nombre":"Línea","descripcion":"X"}'
#    → 422: error indicando que id no debe enviarse
```

### 2.6 Prueba de capas (criterio 7)

```powershell
docker compose exec api-investigacion dotnet run --project pruebas
#    → todas las verificaciones pasan, con repositorios FALSOS en memoria
```

---

## 3. El front: la otra mitad de la versión (criterios 8 a 11)

```powershell
docker compose up -d --build
```

Ese mismo comando levanta tres contenedores:

| Qué | Dónde |
|---|---|
| LA PANTALLA (lo que ve el usuario) | http://localhost:8071 |
| La API — documentación interactiva | http://localhost:8070/swagger |
| SQL Server (opcional, para DBeaver) | `localhost:11470` · usuario `sa` |

### 3.1 La prueba automática del front

```powershell
python pruebas_humo/humo_front.py
```

Comprueba que:
- Las 6 pantallas responden (criterio 8)
- Los datos que muestran son los que dio la API
- No aparece jerga técnica (PUT, 422, nombres de tabla) (criterio 10)
- Con la API apagada la pantalla sigue en pie con su aviso (criterio 11)

La prueba apaga y vuelve a encender la API sola.

**Lo que esa prueba NO puede hacer:** Blazor Server manda los clics por una conexión persistente, así que un guion no puede llenar el formulario. Eso queda para el recorrido a mano (criterio 9).

### 3.2 El recorrido a mano (criterio 9)

Repetir este ciclo para **CADA UNA** de las 6 pantallas:

1. Abra `http://localhost:8071`. La pantalla de inicio tiene el menú con los 6 enlaces a los catálogos.
2. Elija una pantalla (ej. `/areas-de-conocimiento`). La barra de direcciones tiene una ruta propia, no un molde con parámetros.
3. **Agregue** un registro:
   - Para `area_conocimiento`: código `ZZ01`, gran área «Prueba», área «Prueba», disciplina «Prueba». Aparece en la tabla.
   - Para `linea_investigacion`: nombre «Línea de prueba», descripción «Descripción». *(El id lo genera la base, no se ve en el formulario)*.
4. **Agréguelo otra vez**, con el mismo identificador. Sale un aviso rojo con el mensaje que mandó la API — y el formulario conserva lo que usted escribió.
5. **Edítelo** y use **«Guardar solo lo que cambié»** dejando dos campos vacíos: guarda, y los campos que dejó en blanco quedan como estaban.
6. Ahora **«Guardar la ficha completa»** con un campo vacío: se rechaza. *El mismo formulario, dos comportamientos.*
7. **Retírelo.** Pide confirmación y desaparece de la tabla. Pero compruebe:
   ```powershell
   curl "http://localhost:8070/api/area_conocimiento/ZZ01"
   ```
   Responde **404** — y sin embargo la fila sigue en la base: el borrado es lógico (`activo=0`). Compruébelo con DBeaver si quiere.
8. **Apague la API** y recargue la pantalla:
   ```powershell
   docker compose stop api-investigacion
   ```
   La pantalla sigue cargando, con su menú y su pie, y dice que el servicio no está disponible. Eso demuestra que son dos procesos. Vuelva a levantarla con `docker compose start api-investigacion`.

---

## 4. Regresión

Esta es la primera versión: no hay nada anterior que probar. Desde la v2, esta sección conserva los smokes de todas las versiones cerradas y todos deben seguir pasando antes de cerrar la nueva.

---

## 5. Si algo falla

| Síntoma | Causa probable |
|---|---|
| `Login failed for user 'sa'` | Se cambió la contraseña sin `docker compose down -v` (§1) |
| La API responde 500 en todo, con *"No address associated with hostname"* | La API arrancó antes que la base. El compose lo evita con `depends_on`; si pasa, `docker compose restart api-investigacion` |
| `total: 0` en `area_conocimiento` (debería 218) | El inicializador no corrió o falló. `docker compose logs sqlserver-init` |
| `total: 0` en `linea_investigacion` | Es correcto (vacía en el catálogo de referencia) |
| El contenedor de SQL Server se reinicia solo | Contraseña que no cumple la política (8+ caracteres, mayúscula, minúscula, dígito y símbolo) o poca memoria: pide ~2 GB |
| Un inactivo aparece en el listado | A alguna consulta le falta `WHERE activo = 1` (`3_plan` §4.2) |
| `bad interpreter: /bin/bash^M` | `db/init.sh` se guardó con finales de línea de Windows. Es lo que previene `*.sh text eol=lf` en `.gitattributes` |
| El front muestra jerga técnica | Revisar las etiquetas de los botones y los mensajes de error. Deben decir «Guardar la ficha completa» y «Guardar solo lo que cambié», no «PUT» ni «PATCH» |
