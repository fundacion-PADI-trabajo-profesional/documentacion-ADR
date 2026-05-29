# ADR-0015: Protección IDOR en Endpoints de Estadísticas por Aula

## Contexto

El sistema PADI expone tres endpoints de estadísticas que reciben un parámetro `aulaId` en la URL:

- `GET /estadisticas/aprobacion-preguntas/:aulaId`
- `GET /estadisticas/distribucion-puntajes/:aulaId`
- `GET /estadisticas/progresion-estudiante-docente` (recibe `aulaId` como query param)

La arquitectura de autorización del sistema (ADR-0006) aplica dos niveles de control de acceso: verificación de rol en el middleware y filtrado de datos en el servicio. Sin embargo, para estos tres endpoints existía una vulnerabilidad de tipo **IDOR (Insecure Direct Object Reference)**: un usuario con rol `director` o `encargado_zona` podía proporcionar el ID de cualquier aula del sistema y obtener sus estadísticas, independientemente de si esa aula pertenecía a su escuela o zona.

El flujo vulnerable era:

```
Director escuela A → GET /aprobacion-preguntas/:aulaId_escuela_B → 200 OK con datos
```

La validación existente en la capa de servicios verificaba que el rol tuviera acceso a la escuela provista por query param (`escuela_id`), pero no cruzaba el `aulaId` con la escuela del usuario. Un director malintencionado podía obtener estadísticas de aulas de otras escuelas simplemente conociendo su ID.

Este tipo de vulnerabilidad no es detectable por el middleware `requireRole` (que solo verifica la presencia del rol, no el alcance del recurso específico) ni por el filtrado genérico del servicio (que no tiene visibilidad sobre la relación entre el `aulaId` solicitado y el alcance del usuario).

## Decisión

Se implementaron dos componentes en el controlador de estadísticas (`estadisticas.controller.ts`) que validan el alcance del usuario sobre el `aulaId` solicitado **antes** de invocar al servicio:

### 1. Función utilitaria `getEscuelaDeAula`

Se agregó la función `getEscuelaDeAula(aulaId)` en `functions/src/utils/scope.ts`, que resuelve la escuela a la que pertenece un aula dada:

```typescript
export async function getEscuelaDeAula(aulaId: string): Promise<string | null> {
  return withRLSContext(async (tx) => {
    const aula = await tx.aulas.findUnique({
      where: { id: aulaId },
      select: { escuela_id: true },
    });
    return aula?.escuela_id ?? null;
  });
}
```

### 2. Middleware de validación de alcance `validateAulaScope`

Se agregó la función `validateAulaScope(req, aulaId)` en el controlador, que aplica la misma jerarquía organizacional que la función `resolveEscuelaId` ya existente:

```typescript
async function validateAulaScope(req: AuthenticatedRequest, aulaId: string): Promise<void> {
  const rol = req.user!.rol;
  if (rol === "equipo_padi" || rol === "docente") return;

  const aulaEscuelaId = await getEscuelaDeAula(aulaId);

  if (rol === "director") {
    if (aulaEscuelaId !== (req.user!.escuela_id ?? null)) {
      throw new AuthorizationError("No tenés permisos para ver estadísticas de esa aula");
    }
  } else if (rol === "encargado_zona") {
    const zonaId = await getEncargadoZonaId(req.user!.id);
    const pertenece = await escuelaPerteneceAZona(aulaEscuelaId!, zonaId);
    if (!pertenece) throw new AuthorizationError("No tenés permisos para ver estadísticas de esa aula");
  }
}
```

La lógica de validación por rol es:

| Rol | Validación |
|---|---|
| `equipo_padi` | Sin restricción — acceso total |
| `docente` | Sin restricción en el controlador — el servicio valida que el docente esté asignado al aula |
| `director` | El `escuela_id` del aula debe coincidir con `req.user.escuela_id` |
| `encargado_zona` | La escuela del aula debe pertenecer a la zona del encargado |

### 3. Llamadas a `validateAulaScope` en los endpoints afectados

```typescript
// getAprobacionPreguntas y getDistribucionPuntajes — aulaId siempre presente
await validateAulaScope(req, aulaId);

// getProgresionEstudianteDocente — aulaId es opcional
if (aulaId) await validateAulaScope(req, aulaId);
```

La validación ocurre antes de cualquier llamada al servicio. Si falla, se lanza `AuthorizationError`, que el bloque `catch` del controlador convierte en HTTP 403 (ver ADR-0014).

## Alternativas Consideradas

- **Validación en la capa de servicios:** Incorporar la verificación del `aulaId` dentro de las funciones del servicio, junto con la lógica de negocio existente. Descartado porque el patrón ya establecido en el sistema (ADR-0006) ubica las verificaciones de alcance de recursos en el controlador, antes de delegar al servicio. Mezclar autorización y lógica de negocio en el servicio dificulta el testing y la auditoría.

- **Inyectar el `escuela_id` del aula desde el frontend:** Enviar el `escuela_id` correspondiente al aula como parámetro adicional en la solicitud, y validarlo en el backend. Descartado porque el cliente no es una fuente confiable: un atacante podría manipular el parámetro. La validación siempre debe realizarse contra la base de datos.

- **Restricción a nivel de base de datos con RLS:** Configurar políticas de Row Level Security en PostgreSQL para que una consulta sobre un aula devuelva vacío si el usuario no tiene acceso. Evaluado pero descartado para esta iteración: la implementación de RLS para esta política requiere propagar el rol y los identificadores del usuario a la sesión de base de datos, lo que está fuera del alcance del sistema actual.

- **Validar el alcance del aula en `requireRole`:** Extender el middleware de roles para incluir validaciones de recursos específicos. Descartado porque `requireRole` opera antes de parsear los parámetros de la ruta y no tiene acceso a `aulaId`.

## Consecuencias

### Positivas
- **Eliminación de la vulnerabilidad IDOR:** Los tres endpoints afectados ya no permiten que un director o encargado de zona consulte estadísticas de aulas fuera de su alcance jerárquico.
- **Reutilización de patrones existentes:** `validateAulaScope` utiliza las mismas funciones utilitarias (`getEncargadoZonaId`, `escuelaPerteneceAZona`) que el patrón `resolveEscuelaId` ya probado en el sistema, minimizando la introducción de nueva lógica.
- **Fallo seguro (fail-safe):** Si `getEscuelaDeAula` devuelve `null` (aula inexistente), la comparación `aulaEscuelaId !== req.user.escuela_id` falla para el director, denegando el acceso. El sistema falla de forma segura ante IDs inválidos.
- **Cobertura de tests:** Los tests de integración existentes en `estadisticas.test.ts` verifican que los intentos de acceso fuera de alcance devuelvan 403.

### Negativas / Compromisos
- **Consulta adicional a la base de datos:** `validateAulaScope` ejecuta una consulta `SELECT` sobre la tabla `aulas` para resolver el `escuela_id`. Esto agrega latencia a los tres endpoints afectados. Para un sistema con el volumen de PADI, el impacto es despreciable.
- **La validación para `docente` es delegada al servicio:** El rol `docente` omite la validación en `validateAulaScope`; la verificación de que el docente está asignado al aula ocurre en la capa de servicio. Esto es coherente con el comportamiento histórico del sistema, pero implica que la lógica de autorización para `docente` está distribuida en dos capas.
- **Cobertura limitada a estadísticas:** La vulnerabilidad IDOR fue identificada y corregida únicamente en los endpoints de estadísticas. Otros módulos del sistema con parámetros de recurso (`aulaId`, `estudianteId`, etc.) no fueron auditados en esta iteración.
