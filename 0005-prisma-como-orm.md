# ADR-0005: Prisma como ORM para el Acceso a Datos

## Contexto

El backend necesita una capa de acceso a datos que:

- Permita escribir consultas a PostgreSQL en TypeScript de forma segura en tipos (type-safe), evitando errores de tipado en tiempo de compilación.
- Genere automáticamente tipos TypeScript a partir del esquema de la base de datos, eliminando la necesidad de mantener interfaces manuales sincronizadas con el schema.
- Ofrezca soporte para migraciones de base de datos, para gestionar la evolución del esquema a lo largo del desarrollo.
- Sea compatible con el entorno serverless de Firebase Cloud Functions, donde los clientes de base de datos tienen restricciones especiales de inicialización.
- Tenga una curva de aprendizaje razonable y buena documentación, considerando el contexto de un proyecto de tesis.

El esquema de datos es complejo, con más de 15 entidades y múltiples relaciones muchos-a-muchos (ver ADR-0003).

## Decisión

Se adoptó **Prisma** (versión 6) como ORM para todo el acceso a datos del backend.

El flujo de trabajo con Prisma en el proyecto es:

1. El esquema de la base de datos se define de forma declarativa en `functions/prisma/schema.prisma`.
2. El comando `npx prisma generate` genera el cliente tipado (`@prisma/client`) a partir del esquema.
3. El comando `npx prisma db push` sincroniza el esquema con la base de datos en desarrollo.
4. El `PrismaClient` se instancia de forma lazy (ver ADR-0009) y se utiliza en la capa de repositorios.

Todos los accesos a la base de datos están encapsulados en la capa de repositorios (`src/repositories/`), que expone métodos tipados basados en el cliente generado por Prisma.

## Alternativas Consideradas

- **TypeORM:** ORM maduro con soporte para múltiples bases de datos y patrones Active Record y Data Mapper. Fue descartado porque su integración con TypeScript, aunque mejorada en versiones recientes, requiere más configuración de decoradores y su generación de tipos es menos ergonómica que la de Prisma.

- **Drizzle ORM:** ORM moderno y muy liviano, orientado a TypeScript. Fue considerado como alternativa por su compatibilidad con entornos serverless y su reducido tamaño. Fue descartado por ser una biblioteca más nueva con menor ecosistema y documentación al momento de la decisión, lo que incrementaba el riesgo para un proyecto de tesis con plazos definidos.

- **Knex.js (query builder):** Librería de construcción de consultas SQL sin capa de abstracción de ORM completa. Fue descartada porque requiere definir los tipos de retorno manualmente, eliminando el beneficio de la generación automática de tipos que ofrece Prisma.

- **Driver nativo `pg` (sin ORM):** Máxima flexibilidad y rendimiento al escribir SQL directamente. Fue descartado por la complejidad de mantener las definiciones de tipos sincronizadas con el schema y la mayor probabilidad de errores de tipado en queries complejos.

## Consecuencias

### Positivas
- **Tipado automático y completo:** El cliente generado por Prisma expone métodos con tipos exactos de cada modelo y sus relaciones, eliminando clases enteras de errores en tiempo de compilación.
- **Schema como fuente de verdad única:** El archivo `schema.prisma` define tanto el modelo de base de datos como los tipos TypeScript utilizados en el código, evitando duplicaciones y desfases.
- **API de consulta expresiva:** Prisma ofrece una API fluida para filtros, ordenamientos, paginación e inclusión de relaciones (`include`, `select`, `where`) que cubre la mayoría de los casos de uso sin escribir SQL manual.
- **Introspección de base de datos:** Si el esquema en producción difiere del archivo `schema.prisma`, Prisma puede detectarlo mediante `prisma db pull`, facilitando la sincronización.
- **Integración con testing:** El cliente de Prisma puede ser mockeado con `vi.mock` en Vitest, permitiendo probar los servicios sin una base de datos real.

### Negativas / Compromisos
- **Tamaño del cliente generado:** El `@prisma/client` generado incluye los binarios del query engine nativos para cada plataforma de destino. En un entorno serverless, esto incrementa el tamaño del paquete desplegado y el tiempo de cold start.
- **Inicialización en entorno serverless:** El `PrismaClient` no está diseñado por defecto para entornos serverless y puede generar problemas de conexión si se instancia múltiples veces. Este problema requirió el patrón de inicialización lazy documentado en ADR-0009.
- **Consultas SQL complejas requieren `$queryRaw`:** Prisma no cubre todos los casos de uso SQL (por ejemplo, consultas con CTEs complejos o funciones específicas de PostgreSQL). Para estos casos, se debe recurrir a consultas crudas con `prisma.$queryRaw`, que escapan el sistema de tipos.
- **Curva de aprendizaje de la sintaxis de schema:** El lenguaje de definición de esquemas de Prisma (`PSL`) es específico de la herramienta y debe aprenderse adicionalmente al SQL estándar.
