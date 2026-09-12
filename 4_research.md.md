# Decisiones — Versión 1: Catálogos del módulo Investigación (EF Core)

> Cada decisión con sus alternativas y su razón. Esto es memoria del
> proyecto: sirve para **no volver a discutir** lo ya discutido, y para que
> quien llegue después —persona o IA— entienda por qué el sistema es así.
>
> Numeración `D-v1-N`, sin repetir entre versiones.

---

## D-v1-1 — EF Core, no Dapper

**Contexto.** Hay que llevar filas de SQL Server a objetos de C#.

**Alternativas.** (a) **Entity Framework Core**: ORM completo, genera SQL y mapea automáticamente. (b) **Dapper**: micro-ORM, SQL a mano. (c) **ADO.NET puro**.

**Decisión: (a) Entity Framework Core.** El profesor autorizó expresamente su uso, y el equipo optó por él para simplificar el desarrollo del acceso a datos, trabajar con LINQ y reducir el código boilerplate de los repositorios. El SQL que genera EF Core es parametrizado y eficiente para las consultas de catálogos de esta versión.

**Consecuencias.** El SQL deja de estar "a la vista" en el repositorio (se genera automáticamente). Esto es una desviación del Artículo 2 de la constitución, justificada por la autorización del profesor. La capa de repositorio sigue existiendo, pero implementada con `DbContext`.

**Estado:** vigente para toda la v1.

---

## D-v1-2 — Las tres capas desde el día 1, no un MVP en un archivo

**Contexto.** Seis tablas y 42 endpoints caben en seis controladores de 80 líneas cada uno.

**Alternativas.** (a) Todo en los controladores y refactorizar en la v2. (b) Capas con interfaces desde el principio.

**Decisión: (b).** "Refactorizar después" es una promesa que nadie cumple con fecha de entrega encima, y la v2 llega con diez tablas con FK: el momento de separar sería el peor posible. Además, la prueba sin base de datos —criterio 7— es **imposible** sin la interfaz del repositorio.

**Consecuencias.** 18 archivos de peticiones, 6 servicios, 6 repositorios y 6 controladores donde cabría uno. A cambio, la v2 agrega tablas sin tocar la arquitectura.

**Estado:** vigente.

---

## D-v1-3 — Una petición por verbo

**Contexto.** `POST`, `PUT` y `PATCH` reciben cuerpos parecidos pero con reglas distintas: el `PATCH` admite campos ausentes y el `PUT` no.

**Alternativas.** (a) Una sola clase por tabla con todos los campos opcionales y validar a mano según el verbo. (b) Tres clases por tabla: `Crear`, `Reemplazo`, `Actualizar`.

**Decisión: (b).** Con (a) la regla queda escondida en `if`s dentro del servicio; con (b) la declara el tipo, y el 422 lo produce el framework antes de que el negocio se entere. Es lo que hace demostrable la pareja `PUT` 422 / `PATCH` 200 del criterio 4.

**Consecuencias.** 18 clases (6 tablas × 3) pequeñas y muy parecidas. Se acepta.

**Estado:** vigente.

---

## D-v1-4 — El borrado lógico se resuelve en el `UPDATE`, no consultando antes

**Contexto.** `DELETE` debe responder 404 si el registro no existe **o ya está inactivo** (C4, C5).

**Alternativas.** (a) Consultar primero y luego actualizar. (b) Un solo `UPDATE … WHERE id = @id AND activo = 1` y mirar las filas afectadas.

**Decisión: (b).** Una sola ida a la base, sin ventana entre la consulta y la escritura. Cero filas significa exactamente "no existe o ya estaba inactiva", que es la respuesta que pide la spec. En EF Core se hace con `FirstOrDefault` + `SaveChangesAsync`, y se comprueba si la entidad es `null`.

**Consecuencias.** El mensaje del 404 no distingue entre "nunca existió" y "ya estaba borrada" — y **está bien**: para la API son el mismo caso.

**Estado:** vigente.

---

## D-v1-5 — El `id` es texto en `termino_clave` y numérico en las demás

**Contexto.** La tabla `termino_clave` tiene PK de texto (`termino` VARCHAR(30)). Las otras 5 tablas tienen PK numérica (`id` INT). Entre ellas, `linea_investigacion` tiene `id` IDENTITY, lo que significa que la base lo genera automáticamente.

**Alternativas.** (a) Forzar que todos los IDs sean numéricos, cambiando `termino_clave`. (b) Aceptar que cada tabla tiene su propio tipo de ID.

**Decisión: (b).** La base de datos viene dada y no se modifica en la v1 (Artículo 5). El controlador maneja ambos casos: para las tablas numéricas, convierte el `string` de la ruta a `int`; para `termino_clave`, lo usa directamente. Para `linea_investigacion`, el `id` no se envía en el `POST` porque la base lo genera.

**Consecuencias.** El tipo del parámetro de ruta es `string` en todos los endpoints; la conversión ocurre dentro del controlador. El repositorio recibe el tipo correcto (`int` o `string`) según la tabla.

**Estado:** vigente.

---

## D-v1-6 — Un contenedor aparte para inicializar la base

**Contexto.** SQL Server **no ejecuta los scripts que se le monten**: alguien tiene que conectarse al motor y correrlos.

**Alternativas.** (a) Instrucciones manuales en el README ("conéctese y corra esto"). (b) Un contenedor `sqlserver-init` que lo haga solo.

**Decisión: (b).** El Artículo 4 exige un solo comando; (a) lo rompe en la primera línea. El inicializador espera a que el motor **responda consultas**, crea la base si no existe, corre el script y se muere.

**Consecuencias.** Un servicio más en el compose, que termina en segundos y es idempotente.

**Estado:** vigente.

---

## D-v1-7 — El catálogo se corrige antes de sembrarlo (adaptado a 6 tablas)

**Contexto.** El Excel de referencia tiene errores de digitación (`Cienias Naturales` en lugar de `Ciencias Naturales` en 48 filas de `area_conocimiento`). También puede haber inconsistencias en los otros catálogos (ODS, áreas de aplicación, universidades). `linea_investigacion` y `termino_clave` no tienen datos en el catálogo de referencia.

**Alternativas.** (a) Cargar los datos tal cual: son los datos dados. (b) Corregir los errores de digitación al generar las semillas y documentarlo.

**Decisión: (b).** Un error de digitación no es un dato: es ruido de la fuente. Cargarlo lo dejaría a la vista en cada listado, en cada informe y en el dashboard de la v4. La corrección queda anotada en la cabecera de `db/investigacion.sql`, de modo que cualquiera puede ver qué se cambió.

**Consecuencias.** El script deja de ser una copia literal del Excel, y por eso mismo la cabecera del script tiene que decirlo.

**Estado:** vigente.

---

## D-v1-8 — Los campos JSON en camelCase

**Contexto.** Las columnas de la base son `gran_area`, `area`, `disciplina` (y en las otras tablas: `nombre`, `categoria`, `termino_ingles`, `tipo`, `ciudad`, `descripcion`, etc.). ¿El JSON las repite tal cual, o usa la convención de C#?

**Alternativas.** (a) **snake_case** en el JSON, igual que la base: un `SELECT` y una respuesta se leen igual. (b) **camelCase**, que es lo que ASP.NET Core hace por defecto.

**Decisión: (b).** Es el comportamiento por defecto: cero configuración. Además, el JSON no es una ventana a la tabla; la API es una frontera. Si mañana una columna se renombra, el contrato no tiene por qué cambiar. El front de la v4 lo consume directo: `granArea` es lo que espera quien escribe JavaScript.

**Consecuencias.** El JSON deja de parecerse a la tabla, y hay que traducir mentalmente al leer el repositorio. A cambio, `Program.cs` no lleva ni una línea de configuración de serialización.

**De dónde salió esta decisión.** El `6_contracts.md` tenía las dos convenciones mezcladas —`gran_area` en los cuerpos y `filasAfectadas` en las respuestas—, y eso solo se descubrió al escribir la primera clase de petición. Se unificó en camelCase para todas las tablas.

**Estado:** vigente.

---

## D-v1-9 — El front es un TERCER PROCESO, y no comparte código con la API (adaptado a 6 pantallas)

**Lo que se decidió.** El front va en su propio contenedor, en su propio puerto, con su propio proyecto de .NET. Habla con la API **solo por HTTP**. Hay **6 pantallas** (una por tabla), cada una con su propio servicio (`ServicioAreaConocimiento`, `ServicioObjetivoDesarrollo`, `ServicioAreaAplicacion`, `ServicioTerminoClave`, `ServicioUniversidad`, `ServicioLineaInvestigacion`). **Ningún servicio es genérico** (no existe un `ApiService.Listar("area_conocimiento")`).

**Lo que se descartó:**
- Servir las páginas desde la misma API (Razor Pages en el proyecto de la API) — rompe la separación de capas y la prueba del criterio 11.
- Compartir las clases de modelo entre API y front (referencia de proyecto) — ataría los dos procesos.
- Un `ApiService` genérico con el nombre de la tabla como parámetro — el compilador deja de revisar y la documentación no puede decir qué recursos existen.

**Cómo se verifica que la decisión se cumple:**
1. `FrontInvestigacion.csproj` **no tiene ningún paquete** de acceso a datos.
2. El servicio `front-blazor` **no depende de `sqlserver`** en el compose.
3. La prueba del criterio 11: `docker compose stop api-investigacion` deja la pantalla en pie, con su aviso y **sin un solo dato**.

**Estado:** vigente.

---

## D-v1-10 — La pantalla no le habla al usuario en jerga (adaptado a 6 pantallas)

**Lo que se decidió.** En ninguna de las 6 pantallas aparece ningún verbo HTTP, ningún código de estado, ni el nombre de ninguna tabla. Los dos botones de guardar se llaman **«Guardar la ficha completa»** y **«Guardar solo lo que cambié»**.

**Lo que se descartó:** nombrarlos «PUT» y «PATCH».

**Por qué importa.** Quien usa esto administra catálogos de investigación (áreas de conocimiento, ODS, áreas de aplicación, términos clave, universidades, líneas de investigación). «PUT» no le dice nada, y peor: le sugiere que necesita saber algo que no necesita. La distinción que sí le sirve —«¿mando todo o solo lo que toqué?»— es exactamente la que los dos nombres explican.

**Se comprueba automáticamente** sobre el texto visible, no sobre el HTML: el guion quita las etiquetas y decodifica las entidades antes de buscar.

**Estado:** vigente.

---

## D-v1-11 — `linea_investigacion` tiene IDENTITY, y el `POST` no envía el `id`

**Contexto.** En el script de la base de datos, `linea_investigacion.id` está definido como `INT IDENTITY(1,1)`, lo que significa que la base genera automáticamente el valor al insertar.

**Alternativas.** (a) Enviar el `id` en el `POST` y permitir que la base lo ignore o lo sobreescriba. (b) **No enviar el `id` en el `POST`**: el cuerpo de la petición solo lleva `nombre` y `descripcion`.

**Decisión: (b).** Es la práctica correcta con columnas IDENTITY. El cliente no debe decidir el valor de un campo que la base asigna automáticamente. Esto también evita conflictos de PK duplicada y simplifica la validación.

**Consecuencias.** `LineaInvestigacionCrear` es una clase de petición que **no tiene** una propiedad `Id`. En el controlador, la entidad se construye sin `Id` y la base lo genera al hacer `SaveChangesAsync`. En la respuesta, el `id` se devuelve si el cliente lo necesita.

**Estado:** vigente.