# ADR-0008: TanStack React Query para la Gestión del Estado del Servidor

## Contexto

El frontend del sistema PADI realiza frecuentes solicitudes al backend para obtener y mutar datos: listados de evaluaciones, estudiantes, docentes, estadísticas, entre otros. Sin una solución de gestión de estado del servidor, cada componente necesitaría implementar por su cuenta:

- Las llamadas `fetch` o `axios` con su ciclo de vida (loading, error, success).
- La lógica de caché para evitar solicitudes redundantes al mismo endpoint.
- La invalidación y refresco de datos luego de mutaciones (por ejemplo, al crear una evaluación).
- El manejo del estado de loading y error para mostrar indicadores visuales al usuario.

La alternativa de usar únicamente `useState` + `useEffect` para cada solicitud genera código repetitivo, difícil de mantener y propenso a condiciones de carrera (por ejemplo, respuestas de solicitudes desactualizadas que llegan después de solicitudes más recientes).

## Decisión

Se adoptó **TanStack React Query v5** (`@tanstack/react-query`) como librería de gestión de estado del servidor.

La estrategia de uso en el proyecto es:

- **`useQuery`:** Para todas las operaciones de lectura (GET). Provee automáticamente estados de `isLoading`, `isError`, `data` y `isFetching`, además de caché configurable por query key.
- **`useMutation`:** Para todas las operaciones de escritura (POST, PATCH, DELETE). Permite definir callbacks `onSuccess` e `onError` para invalidar cachés o mostrar notificaciones.
- **Query Keys:** Identificadores únicos por consulta que permiten invalidar y refrescar datos de forma selectiva luego de mutaciones.

La gestión de estado de la sesión de usuario (datos del usuario autenticado) se mantiene fuera de React Query, en `localStorage` y en el contexto de React, dado que es un estado de la aplicación y no un estado del servidor.

## Alternativas Consideradas

- **Redux Toolkit + RTK Query:** Solución completa de gestión de estado con soporte integrado para data fetching. Fue descartada porque Redux agrega complejidad de boilerplate (reducers, actions, slices) que excede los requisitos del proyecto. RTK Query es similar en funcionalidad a React Query pero con mayor acoplamiento al ecosistema Redux.

- **SWR (stale-while-revalidate):** Librería de Vercel con filosofía similar a React Query. Fue descartada porque React Query ofrece una API más rica para mutaciones, invalidación de caché condicional y gestión de dependencias entre queries, que son necesarias en los flujos complejos de evaluación.

- **Zustand + fetch manual:** Librería de gestión de estado global minimalista combinada con llamadas `fetch` manuales. Fue descartada porque requiere implementar manualmente toda la lógica de caché, loading states y revalidación que React Query provee de forma integrada.

- **React Context + useEffect + fetch:** Solución nativa de React sin dependencias adicionales. Fue descartada por las razones enunciadas en el contexto: propensión a condiciones de carrera, código repetitivo y ausencia de caché.

## Consecuencias

### Positivas
- **Reducción de boilerplate:** Los estados de loading, error y data se obtienen directamente del hook `useQuery` sin implementación manual de lógica de ciclo de vida.
- **Caché automático:** Los datos consultados se almacenan en caché por un tiempo configurable (stale time), evitando solicitudes redundantes cuando el usuario navega entre secciones de la aplicación.
- **Sincronización de estado post-mutación:** Luego de crear, actualizar o eliminar un recurso, la invalidación de la query correspondiente garantiza que los listados se actualicen automáticamente con los datos más recientes del servidor.
- **DevTools integradas:** TanStack Query ofrece un panel de DevTools que muestra el estado de todas las queries activas, su caché y su estado de fetching, facilitando la depuración.
- **Reintento automático:** Por defecto, React Query reintenta automáticamente las solicitudes fallidas un número configurable de veces antes de reportar el error.

### Negativas / Compromisos
- **Curva de aprendizaje del concepto de query keys:** El sistema de query keys puede resultar confuso inicialmente, especialmente en casos donde la clave debe incluir parámetros dinámicos (por ejemplo, el ID de una evaluación) para que la invalidación sea precisa.
- **Dependencia adicional:** Agrega una dependencia de terceros al proyecto. Aunque TanStack Query es ampliamente adoptado y mantenido activamente, representa un riesgo de obsolescencia a largo plazo.
- **Complejidad en el manejo de paginación:** Aunque React Query provee `useInfiniteQuery` para paginación infinita, la integración con componentes de tabla paginada (como los de MUI) requiere adaptadores adicionales.
