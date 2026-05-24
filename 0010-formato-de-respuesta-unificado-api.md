# ADR-0010: Formato de Respuesta Unificado en la API REST

## Contexto

El frontend debe consumir múltiples endpoints del backend para obtener datos, realizar mutaciones y manejar errores. Sin un contrato establecido para el formato de las respuestas HTTP, cada endpoint puede retornar datos en formatos distintos, lo que obliga al frontend a implementar lógica de parseo diferenciada por endpoint y dificulta el manejo centralizado de errores.

En particular, el uso de solo códigos de estado HTTP (200, 400, 404, 500) no es suficiente para comunicar la naturaleza de un error al frontend de forma descriptiva, ya que el cuerpo de la respuesta puede variar entre frameworks y librerías.

Se necesitaba una estructura de respuesta predecible que:
- Funcione de forma consistente para respuestas exitosas y de error.
- Incluya un mensaje descriptivo adecuado para mostrar al usuario.
- Proporcione información de error estructurada para facilitar el diagnóstico.
- Sea fácil de deserializar en TypeScript.

## Decisión

Se definió un **modelo de respuesta unificado** (`ResponseModel`) para todos los endpoints de la API:

```typescript
interface ResponseModel<T = unknown> {
  success: boolean;       // Indica si la operación fue exitosa
  message: string;        // Mensaje legible por humanos
  data?: T;               // Payload de datos (presente en respuestas exitosas)
  error?: {               // Información de error (presente en respuestas fallidas)
    code?: string;        // Código de error interno (opcional)
    description?: string; // Descripción técnica del error (opcional)
  };
}
```

Todos los controladores utilizan la función `commonResponse` para construir las respuestas, garantizando que ningún endpoint retorne datos en un formato diferente.

Las respuestas exitosas siguen el patrón:
```json
{
  "success": true,
  "message": "Evaluación creada exitosamente",
  "data": { "id": 42, "estado": "N", ... }
}
```

Las respuestas de error siguen el patrón:
```json
{
  "success": false,
  "message": "No autorizado para acceder a este recurso",
  "error": {
    "code": "FORBIDDEN",
    "description": "El rol 'docente' no tiene permiso para este endpoint"
  }
}
```

Los códigos de estado HTTP se utilizan de forma complementaria al campo `success` para el manejo estándar de errores (200, 201, 400, 401, 403, 404, 500).

## Alternativas Consideradas

- **Solo códigos de estado HTTP con cuerpo libre:** Retornar el recurso directamente en el cuerpo de respuestas exitosas (sin wrapper) y un objeto de error estándar en respuestas fallidas. Descartada porque el frontend necesitaría lógica diferenciada para extraer los datos dependiendo del tipo de respuesta, y los errores del propio framework Express tienen formatos distintos a los errores de negocio.

- **Estándar JSend:** Especificación de formato de respuesta JSON similar al adoptado (`success`, `fail`, `error`). Fue considerada pero descartada en favor de un esquema propio más simple y directamente tipado en TypeScript, que unifica los casos `fail` y `error` del estándar JSend en un único estado `success: false`.

- **GraphQL:** Protocolo que resuelve el problema de contratos de API de forma más estructural, incluyendo un esquema tipado para queries y mutaciones. Fue descartado por la complejidad de setup y el cambio de paradigma que implica, que no estaba justificado por el alcance del proyecto.

- **Problema/RFC 7807 (Problem Details for HTTP APIs):** Estándar de IETF para reportar errores en APIs REST. Fue evaluado pero descartado por su complejidad relativa y por ser más adecuado para errores que para el manejo de respuestas exitosas.

## Consecuencias

### Positivas
- **Consistencia garantizada:** El frontend puede asumir siempre la misma estructura de respuesta y tratar todos los casos con la misma lógica de deserialización.
- **Manejo centralizado de errores:** El cliente HTTP del frontend puede implementar un interceptor global que verifique `success` y muestre mensajes de error o redirija al login según corresponda.
- **Legibilidad de la API:** Los consumidores de la API (incluyendo desarrolladores que realicen pruebas con herramientas como Postman) reciben siempre mensajes descriptivos junto con los datos.
- **Tipado TypeScript:** La interfaz `ResponseModel<T>` permite al frontend tipar correctamente el campo `data` con el tipo esperado para cada endpoint específico.

### Negativas / Compromisos
- **Redundancia con códigos de estado HTTP:** El campo `success: true` en una respuesta HTTP 200 es semánticamente redundante; los códigos de estado ya comunican el éxito o fracaso de la operación. Esta redundancia es un compromiso aceptado a favor de la simplicidad del cliente.
- **No sigue estrictamente REST puro:** En una API REST estricta, el cuerpo de la respuesta de un recurso debería ser el recurso mismo, sin un wrapper. El modelo adoptado agrega una capa de indirección que puede confundir a consumidores que esperan una API REST convencional.
- **Overhead en respuestas simples:** Para endpoints que retornan listas de recursos, el wrapper agrega bytes adicionales por cada respuesta. En el contexto del sistema, este overhead es insignificante.
