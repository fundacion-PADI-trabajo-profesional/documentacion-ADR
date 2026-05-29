# ADR-0014: Manejo Diferenciado de Errores HTTP (403 vs 500)

## Contexto

El controlador de estadísticas (`estadisticas.controller.ts`) expone 18 endpoints que pueden fallar por dos razones estructuralmente distintas:

1. **Errores de autorización:** El usuario autenticado intenta acceder a un recurso que está fuera de su alcance (por ejemplo, un director consultando estadísticas de una aula de otra escuela). Estos errores son esperados, predecibles y deben comunicarse al cliente con un código HTTP 403 (Forbidden).

2. **Errores inesperados:** Fallas en la base de datos, referencias nulas, excepciones de red u otros estados que no deberían ocurrir en condiciones normales. Estos deben devolver HTTP 500 (Internal Server Error) y no exponer detalles internos del sistema.

En la implementación original, todos los `catch` del controlador tenían la firma `catch (error: any)` y devolvían indiscriminadamente `res.status(403)`, sin importar la naturaleza del error:

```typescript
// Antes: todos los errores devuelven 403 indiscriminadamente
} catch (error: any) {
  return res.status(403).json(commonResponse(false, error.message, null));
}
```

Esto generaba dos problemas concretos:
- **Falsos negativos de seguridad:** Un error de base de datos devolvía 403, lo que podía confundir al cliente y ocultar fallos de infraestructura.
- **Fuga de información:** `error.message` de un error inesperado podía exponer detalles internos (stack traces, nombres de tablas, mensajes de Prisma) al cliente.

Adicionalmente, TypeScript señalaba `error: any` como una práctica insegura, ya que inhabilita el análisis de tipos sobre el objeto de error.

## Decisión

Se introdujo la clase `AuthorizationError` como tipo explícito para errores de autorización, y se actualizaron todos los bloques `catch` del controlador de estadísticas para distinguir entre ambos casos.

### 1. Clase `AuthorizationError`

Se creó el archivo `functions/src/utils/errors.ts`:

```typescript
/** Error lanzado cuando el usuario no tiene permisos para realizar la operación. */
export class AuthorizationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "AuthorizationError";
  }
}
```

### 2. Uso en la capa de servicios

Todos los `throw new Error(...)` que representan denegación de acceso en `estadisticas.service.ts` fueron reemplazados por `throw new AuthorizationError(...)`:

```typescript
// Ejemplos de sitios actualizados:
if (!pertenece) throw new AuthorizationError("No tenés acceso a esta aula");
if (!docente) throw new AuthorizationError("No se encontró el perfil de docente");
if (!perteneceAula) throw new AuthorizationError("Estudiante no pertenece a tus aulas");
```

La función `validateRol` del servicio también lanza `AuthorizationError("Acceso denegado")`.

### 3. Bloques `catch` tipados en el controlador

Los 18 bloques `catch` del controlador fueron actualizados de `error: any` a `error: unknown`, con discriminación de tipo:

```typescript
// Después: discriminación explícita entre 403 y 500
} catch (error: unknown) {
  if (error instanceof AuthorizationError) {
    return res.status(403).json(commonResponse(false, error.message, null));
  }
  return res.status(500).json(commonResponse(false, "Error interno del servidor", null));
}
```

El mensaje de error 500 es siempre `"Error interno del servidor"`, sin exponer detalles del error original al cliente.

## Alternativas Consideradas

- **Mantener `error: any` con condición sobre el mensaje:** Verificar `error.message.includes("permisos")` para distinguir casos. Descartado porque es frágil (depende del texto del mensaje) y no garantiza exhaustividad ante nuevos tipos de error.

- **Campo discriminante en el error (`error.type = "authorization"`):** Usar un objeto de error plano con una propiedad tipo en lugar de una clase. Descartado por ser menos idiomático en TypeScript; `instanceof` es el mecanismo estándar para discriminar tipos de error.

- **Middleware de manejo de errores centralizado (`app.use(errorHandler)`):** Registrar un middleware de error global en Express que interprete todos los errores del sistema. Fue evaluado pero descartado para este caso: el controlador de estadísticas es el único módulo con esta distinción en la versión actual, y agregar una arquitectura de error global aumentaría la complejidad sin un beneficio proporcional.

- **Propagar errores 403 como respuestas normales en vez de excepciones:** Hacer que el servicio devuelva `null` o un objeto `{ allowed: false }` en lugar de lanzar. Descartado porque requeriría modificar las firmas de retorno de todas las funciones del servicio y agregar verificaciones adicionales en cada llamada del controlador.

## Consecuencias

### Positivas
- **Corrección semántica de los códigos HTTP:** Los errores de autorización devuelven 403; los errores inesperados devuelven 500. Los clientes y herramientas de monitoreo pueden distinguir entre ambos casos.
- **Sin fuga de información en errores 500:** El mensaje de error enviado al cliente en caso de error inesperado es siempre genérico, sin exponer detalles internos de la implementación.
- **Eliminación de `error: any`:** Los 18 bloques `catch` del controlador ahora usan `error: unknown`, conforme a las guías de TypeScript estricto.
- **Extensibilidad:** Para agregar nuevos tipos de error con códigos HTTP específicos (por ejemplo, `NotFoundError` → 404), basta con crear una nueva clase en `utils/errors.ts` y agregar una rama `instanceof` en el catch.

### Negativas / Compromisos
- **Cobertura limitada al controlador de estadísticas:** Los demás controladores del sistema no fueron actualizados en esta iteración y mantienen sus bloques `catch` con `error: any`. Esto es deuda técnica identificada.
- **Propagación requiere disciplina:** Para que la discriminación funcione correctamente, todos los lanzadores de errores de autorización en el servicio deben usar `AuthorizationError`. Un `throw new Error(...)` no capturado por `instanceof` caerá en la rama 500.
