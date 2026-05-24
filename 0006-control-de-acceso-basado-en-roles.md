# ADR-0006: Control de Acceso Basado en Roles (RBAC)

## Contexto

El sistema PADI es utilizado por cuatro tipos distintos de usuarios con alcances de acceso muy diferentes:

- **Docentes:** Solo pueden acceder a los datos de sus propios estudiantes y evaluaciones en las aulas donde están asignados.
- **Directivos:** Pueden acceder a todos los datos de su escuela (estudiantes, docentes, aulas y evaluaciones).
- **Encargados de zona:** Pueden acceder a los datos de todas las escuelas de su zona geográfica.
- **Equipo PADI:** Tienen acceso total al sistema, incluyendo la gestión de usuarios, zonas y configuración global.

Esta diferenciación de acceso no es únicamente de visibilidad de la interfaz, sino que debe ser aplicada a nivel de datos: una consulta al endpoint `GET /evaluaciones` debe retornar únicamente las evaluaciones que el usuario tiene permitido ver, no todas las evaluaciones del sistema.

Adicionalmente, ciertos endpoints solo deben ser accesibles para determinados roles (por ejemplo, la creación de usuarios solo puede ser realizada por el equipo PADI).

## Decisión

Se implementó un sistema de **Control de Acceso Basado en Roles (RBAC)** con dos niveles de aplicación:

### Nivel 1: Control de Acceso a Endpoints (Middleware)

El middleware `requireRole(...roles)` en `src/middlewares/auth.middleware.ts` verifica que el rol del usuario autenticado esté incluido en la lista de roles permitidos para ese endpoint. Si no lo está, retorna un error HTTP 403 (Forbidden).

```
GET /evaluaciones        → requireRole('docente', 'director', 'encargado_zona', 'equipo_padi')
POST /admin/usuarios     → requireRole('equipo_padi')
GET /estadisticas        → requireRole('director', 'encargado_zona', 'equipo_padi')
```

### Nivel 2: Filtrado de Datos por Rol (Servicios)

En la capa de servicios, cada función de listado aplica filtros según el rol del usuario que realiza la consulta:

- **docente:** filtra por las aulas y escuelas a las que está asignado.
- **director:** filtra por su `escuela_id`.
- **encargado_zona:** filtra por la zona a la que está asignado.
- **equipo_padi:** no aplica ningún filtro (acceso total).

El rol y los identificadores del usuario se obtienen del objeto `req.user`, que el middleware de autenticación (`requireAuth`) construye a partir del token JWT validado.

Los cuatro roles posibles están definidos como un enum en Prisma: `docente`, `director`, `encargado_zona`, `equipo_padi`.

## Alternativas Consideradas

- **Control de Acceso Basado en Atributos (ABAC):** Modelo más granular donde los permisos se definen en función de atributos del usuario, del recurso y del entorno. Fue descartado por su complejidad de implementación y configuración, que excede los requisitos del sistema actual. El modelo RBAC con filtrado por jerarquía organizacional cubre todos los casos de uso identificados.

- **Autorización únicamente en el frontend:** Renderizar u ocultar elementos de la interfaz según el rol, sin aplicar restricciones en el backend. Fue descartado porque cualquier usuario con herramientas de inspección de red (DevTools) podría realizar llamadas directas a la API y obtener datos que no le corresponden. La seguridad debe aplicarse siempre en el servidor.

- **Roles binarios (admin / no admin):** Simplificar a dos niveles de acceso. Fue descartado porque el modelo de negocio de PADI requiere explícitamente cuatro niveles de jerarquía con diferentes alcances geográficos y organizacionales.

- **Lista de control de acceso (ACL) por recurso:** Asignar permisos individuales a cada usuario para cada recurso específico. Fue descartado por la complejidad operativa de administrar permisos a nivel de recurso individual en un sistema con potencialmente miles de evaluaciones, estudiantes y docentes.

## Consecuencias

### Positivas
- **Seguridad aplicada en profundidad:** Las restricciones de acceso se aplican tanto en el middleware (a nivel de endpoint) como en los servicios (a nivel de datos), siguiendo el principio de defensa en profundidad.
- **Separación de responsabilidades:** La autenticación (¿quién eres?) se resuelve en el middleware `requireAuth`; la autorización (¿qué podés hacer?) se resuelve en `requireRole` y en la lógica del servicio.
- **Extensibilidad:** Para agregar un nuevo rol en el futuro, basta con añadirlo al enum `Rol` en Prisma y actualizar las reglas de filtrado en los servicios correspondientes.
- **Trazabilidad:** El objeto `req.user` está disponible en toda la cadena de procesamiento de la solicitud, permitiendo que los servicios apliquen filtros sin necesidad de consultas adicionales a la base de datos.

### Negativas / Compromisos
- **Lógica de filtrado distribuida en los servicios:** Cada función de listado debe implementar su propia lógica de filtrado por rol. Si los roles o sus reglas cambian, pueden ser necesarias modificaciones en múltiples servicios.
- **Acoplamiento entre rol y alcance organizacional:** El sistema asume que cada rol tiene un alcance organizacional fijo (docente → aula, director → escuela, encargado → zona). Si un usuario necesita acceso a múltiples unidades de diferentes niveles, el modelo actual no lo soporta sin modificaciones.
- **Sin gestión de permisos en tiempo de ejecución:** Los permisos están codificados en el middleware y los servicios; no existe una interfaz de administración de permisos. Cambios en los permisos requieren un nuevo despliegue.
