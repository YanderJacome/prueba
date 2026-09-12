# Constitución del proyecto — Módulo Investigación (v1 con EF Core)

> **Documento permanente.** Estas reglas rigen TODAS las versiones del
> proyecto. Cada versión tiene además su propia especificación en
> [versiones/](versiones/0_mapa_versiones.md); ante conflicto, la
> constitución gana.
>
> Este proyecto es una adaptación del ejemplo de referencia del módulo
> Investigación, con decisiones propias documentadas en los spec kits.

---

## Artículo 1 — El proyecto se construye POR VERSIONES y la especificación manda

- El sistema crece por **versiones incrementales** (v1, v2, …), cada una
  con su spec kit propio. Una versión está TERMINADA solo cuando pasa sus
  criterios de aceptación; entonces se hace commit, **tag** (`v1`, `v2`…)
  y solo después se escribe la spec de la siguiente.
- **No se anticipa** (**YAGNI**): nada de JWT en la v1, ni dashboard en la v2,
  ni tablas de más antes de la versión que las pida. El código de cada versión
  solo puede nombrar lo que su spec nombra.
- **Cerrado es cerrado:** una versión con tag no se reabre; los ajustes van
  a la siguiente.

## Artículo 1.1 — Una versión incluye SU FRONT

**Cada versión entrega su parte de la API *y* su parte del front.** No hay una
versión «de back» y otra «de front».

> **La regla operativa: una versión NO está cerrada si la API responde y la
> pantalla no.** Media versión no es una versión.

El front es **Blazor Server**, en su propio proyecto y en su propio contenedor,
hablando con la API **solo por HTTP**. No comparte código con la API, aunque
ambos estén en C#. Lo único que comparten es el JSON.

## Artículo 2 — Stack: C# y ASP.NET Core, con EF Core (autorizado)

- Lenguaje **C#** sobre **ASP.NET Core** (.NET 10): controladores con
  atributos, inyección de dependencias del framework y `async/await` en
  todo el acceso a datos.
- **El profesor ha autorizado expresamente el uso de Entity Framework Core**
  en lugar de Dapper. Por tanto, el acceso a datos se hace con EF Core,
  usando LINQ y el `DbContext`. El SQL lo genera EF Core, pero se acepta
  como decisión técnica justificada.
- Paquetes permitidos:
  - `Microsoft.EntityFrameworkCore.SqlServer`
  - `Microsoft.EntityFrameworkCore.Tools` (opcional)
  - `Swashbuckle.AspNetCore`
- No se usan otros paquetes sin que una spec lo pida.

## Artículo 3 — Arquitectura en tres capas con interfaces, desde el día 1
```
HTTP → Controller (valida el body contra la PETICIÓN del verbo → 422)  
→ IServicio<Entidad> (interfaz — reglas de negocio)  
→ IRepositorio<Entidad> (interfaz — el servicio no sabe qué motor hay)  
→ Repositorio<Entidad>EF (EF Core, LINQ, DbContext)  
→ la base de datos
```
- El controlador no toca la base; el servicio no conoce HTTP ni el motor; el
  repositorio no conoce HTTP. Los contratos son `interface` de C#.
- **Solo el ensamblador** (el registro de dependencias en `Program.cs`)
  conoce clases concretas. Todo lo demás recibe interfaces por constructor.
- El negocio comunica problemas con **excepciones**
  (`ArgumentException` → 400 · `NoEncontradoExcepcion` → 404) y el
  controlador las traduce a HTTP.

## Artículo 4 — Un solo comando

`docker compose up -d --build` deja TODO el sistema de la versión
funcionando, desde la primera versión. Sin pasos manuales y sin instalar
nada local más allá de Docker. El código va montado como volumen y corre
con `dotnet watch`: guardar un `.cs` recompila y reinicia solo.

## Artículo 5 — La base de datos viene DADA

- La base `investigacion` se crea **completa, con sus 19 tablas**, desde
  la v1, a partir del script provisto en `db/`. Se copia, no se genera.
- Los datos de los catálogos salen del Excel de referencia del módulo.
- **En la v1, la API solo puede nombrar las 6 tablas sin FK salientes:**
  `area_conocimiento`, `objetivo_desarrollo_sostenible`, `area_aplicacion`,
  `termino_clave`, `universidad` y `linea_investigacion`.
- Las tablas con FK salientes (`docente`, `grupo_investigacion`, etc.) son
  territorio de versiones posteriores.

## Artículo 6 — Borrado LÓGICO, siempre

- `DELETE` **nunca** borra la fila: marca `activo = 0`.
- Todos los listados **filtran los inactivos** por defecto.
- Toda tabla del módulo tiene su columna `activo BIT NOT NULL DEFAULT 1`.

## Artículo 7 — Los secretos van en variables de entorno

- La contraseña de `sa` vive en el `docker-compose.yml` (excepción didáctica
  declarada) y en `appsettings.json` para desarrollo local. En un proyecto
  real iría en un `.env` fuera de git.
- Ningún secreto se escribe en el código fuente.

## Artículo 8 — Todo en español, y el código sustenta sus decisiones

- Nombres, rutas, mensajes, comentarios y documentación: **en español**.
- Los comentarios explican **por qué** está hecho así —la decisión y su
  consecuencia—, no qué hace cada palabra del lenguaje.

## Artículo 9 — Contratos exactos

Los endpoints, formatos y códigos de estado de cada versión están en su
`6_contracts.md` y se cumplen **al pie de la letra** — incluido el
contraste didáctico `PUT` (reemplazo completo → 422 si falta un campo) vs
`PATCH` (parcial → 200 con el mismo cuerpo).

## Artículo 10 — Convenciones fijas

| Cosa | Convención |
|---|---|
| Puertos del proyecto | API **8070** · front **8071** · SQL Server **11470** |
| Base de datos | `investigacion_local` |
| Rutas | `/` (diagnóstico) · `/swagger` (documentación interactiva) · `/api/{tabla}` |
| Nombres | PascalCase en español; interfaces con prefijo `I`; carpetas `Controllers/`, `Modelos/`, `Peticiones/`, `Servicios/`, `Repositorios/`, `Excepciones/`, `Data/`, `pruebas/` |
| Sobre de respuesta | Lecturas: `{tabla, limite, total, datos}` · Errores: `{estado, mensaje, detalle}` (+ `errores:[…]` en el 422) |
| Errores | Cuerpo inválido → **422** · `ArgumentException` → **400** · `NoEncontradoExcepcion` → **404** · `DbUpdateException` y demás → **500** · lectura sin filas → **204** |
| Credenciales | Usuario `sa`. Contraseña en `docker-compose.yml` y `appsettings.json` (excepción didáctica). |

## Artículo 10.1 — Una ruta y un servicio por recurso, nunca genéricos

La API expone `/api/area_conocimiento`, `/api/linea_investigacion`, etc.
**No existe ni existirá** un `/api/{tabla}` con el nombre de la tabla como
parámetro. Del lado del front hay un `ServicioAreaConocimiento`, no un
`ApiService.Listar("area_conocimiento")`. Cada recurso tiene su propio
controlador, servicio, repositorio y pantalla.

## Artículo 11 — Cómo se enmienda esta constitución

Una regla se cambia **solo** así: se propone en el `4_research.md` de la
versión que la necesita, con su razón y sus consecuencias; si se acepta,
esta constitución **sube de versión** y se anota la fecha de enmienda.

---

*Versión 1.0.0 · Ratificada el 2026-09-08 · Adaptada para el proyecto con EF Core y 6 tablas.*
