# ADR-0001: Arquitectura en Capas para el Backend

## Contexto

El sistema PADI requiere un backend que gestione lógica de negocio compleja: control de acceso por roles, evaluaciones pedagógicas multi-etapa, jerarquía organizacional (zonas, escuelas, aulas) y generación de reportes estadísticos. Desde el inicio del diseño fue evidente que mezclar la lógica de ruteo HTTP, las reglas de negocio y el acceso a la base de datos en los mismos módulos generaría código difícil de mantener, extender y probar.

Se necesitaba un patrón de organización que:
- Separara claramente las responsabilidades de cada componente.
- Permitiera probar la lógica de negocio de forma independiente a la infraestructura.
- Facilitara la incorporación de nuevos endpoints o módulos siguiendo una convención predecible.
- Fuera comprensible por cualquier colaborador sin necesidad de documentación adicional.

## Decisión

Se adoptó una **arquitectura en capas estricta de cuatro niveles** para todo el backend:

| Capa | Ubicación | Responsabilidad |
|---|---|---|
| **Rutas** | `src/routes/*.router.ts` | Definición de endpoints HTTP y asignación de middlewares |
| **Controladores** | `src/controllers/*.controller.ts` | Recepción y validación de requests; formateo de responses |
| **Servicios** | `src/services/*.service.ts` | Lógica de negocio, filtrado por rol, coordinación entre repositorios |
| **Repositorios** | `src/repositories/*.repository.ts` | Acceso a datos mediante Prisma ORM |

El flujo de una petición sigue siempre la dirección: `Middleware → Ruta → Controlador → Servicio → Repositorio → Base de datos`.

Cada capa únicamente se comunica con la capa inmediatamente inferior; no se admiten dependencias salteadas (por ejemplo, un controlador no accede directamente a un repositorio).

## Alternativas Consideradas

- **Arquitectura hexagonal (puertos y adaptadores):** Ofrece mayor desacoplamiento al definir interfaces abstractas entre capas. Fue descartada por su complejidad de setup y la curva de aprendizaje que impondría, considerando que el equipo de desarrollo es reducido y el tiempo de tesis es acotado.

- **Patrón MVC clásico (sin capa de repositorio):** Simplifica la estructura colapsando repositorio y servicio en el controlador. Fue descartado porque mezcla lógica de negocio con acceso a datos, dificultando las pruebas unitarias y el reemplazo del ORM.

- **Módulos sin estructura predefinida:** Organizar el código por funcionalidad sin una separación de capas formal. Descartado por la falta de convención, que dificulta el mantenimiento a medida que el sistema crece.

## Consecuencias

### Positivas
- **Testabilidad:** Los servicios pueden probarse aislando el repositorio con mocks de Prisma, sin levantar una base de datos real.
- **Mantenibilidad:** La adición de un nuevo módulo (por ejemplo, `materiales`) requiere solo crear los archivos correspondientes en cada capa, siguiendo la misma convención.
- **Legibilidad:** Un desarrollador nuevo puede localizar dónde se encuentra cualquier lógica siguiendo la convención de nombres y ubicaciones.
- **Reemplazabilidad:** Si en el futuro se necesita cambiar de Prisma a otro ORM, el cambio está contenido en la capa de repositorios sin afectar los servicios.

### Negativas / Compromisos
- **Boilerplate adicional:** Funcionalidades simples como un endpoint de lectura directa requieren de igual modo pasar por las cuatro capas, lo que genera más archivos y código respecto a soluciones más compactas.
- **Sobreingeniería aparente para casos simples:** Para operaciones CRUD triviales, la separación puede percibirse como excesiva. Sin embargo, la consistencia del patrón justifica la uniformidad.
