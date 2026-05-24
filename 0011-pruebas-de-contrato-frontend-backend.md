# ADR-0011: Pruebas de Contrato entre Frontend y Backend

## Contexto

El frontend y el backend del sistema PADI se desarrollan como proyectos independientes, cada uno con su propio repositorio y ciclo de despliegue. Esta independencia genera el riesgo de **desincronización de contratos de API**: situaciones en que el backend retorna un campo con un nombre o tipo que el frontend no espera, o viceversa.

Un ejemplo concreto de este problema es la diferencia de convenciones de nomenclatura: el backend en TypeScript con Prisma tiende a usar `snake_case` (por la convención de PostgreSQL y Prisma), mientras que el frontend en React/TypeScript prefiere `camelCase`. Una inconsistencia en estos nombres (por ejemplo, el backend retorna `nombre_completo` y el frontend espera `nombreCompleto`) produce bugs silenciosos: la UI renderiza campos vacíos sin lanzar excepciones, lo que hace que el error sea difícil de detectar en pruebas manuales.

Las pruebas de integración end-to-end con servicios reales serían la solución más robusta, pero requieren levantar el backend, la base de datos y el frontend en un entorno controlado, lo que añade complejidad al pipeline de CI/CD.

## Decisión

Se adopta una estrategia dual que combina **documentación formal con OpenAPI/Swagger** y **pruebas de contrato estáticas**:

**1. Documentación OpenAPI con Swagger**

El backend expone una especificación OpenAPI generada con `swagger-jsdoc` y una interfaz interactiva con `swagger-ui-express`. Esta especificación define formalmente los contratos de la API: rutas, métodos, parámetros, esquemas de request y response. Sirve como fuente de verdad para el frontend y como documentación viva del sistema.

**2. Pruebas de contrato estáticas**

El suite de tests del backend (`test/contracts/frontend-backend-alignment.test.ts`) lee los archivos fuente del frontend utilizando el sistema de archivos y verifica que:

- Los nombres de campos retornados por el backend corresponden a los nombres esperados por el código del frontend (por ejemplo, en los módulos de `src/api/*.ts` del frontend).
- Las interfaces TypeScript del frontend están alineadas con los modelos del backend.
- No existen referencias a campos que han sido renombrados o eliminados en el backend.

Este tipo de test se ejecuta como parte del pipeline de CI/CD: el workflow de GitHub Actions clona tanto el repositorio del backend como el del frontend, y los tests de contrato verifican la compatibilidad antes de autorizar el despliegue.

Ambas herramientas son complementarias: Swagger define *qué* debe cumplirse como contrato, y los tests de contrato verifican *que el código real lo cumple*.

## Alternativas Consideradas

- **Biblioteca de tipos compartida (shared package):** Crear un paquete npm interno con las interfaces TypeScript compartidas entre frontend y backend, publicado en un registro privado o como un workspace de monorepo. Fue descartada porque añade complejidad de infraestructura (gestión del paquete, versionado, publicación) que excede el alcance del proyecto. El monorepo con workspaces fue considerado pero no implementado en esta iteración.

- **Pruebas end-to-end con Playwright o Cypress:** Pruebas que ejecutan flujos completos del usuario en un navegador real, verificando que los datos aparecen correctamente en la UI. Fueron descartadas como sustituto de las pruebas de contrato porque requieren levantar todos los servicios (incluyendo la base de datos), tienen mayor tiempo de ejecución, y son más frágiles ante cambios de UI no relacionados con los contratos de API.

## Consecuencias

### Positivas
- **Detección temprana de inconsistencias:** Los errores de contrato se detectan en el pipeline de CI/CD antes de llegar a producción, sin necesidad de pruebas manuales en un entorno integrado.
- **Sin servicios en ejecución:** Las pruebas de contrato son puramente estáticas (leen archivos de texto), lo que las hace rápidas y eliminan la necesidad de configurar una base de datos de testing para este propósito.
- **Documentación formal de la API:** La especificación OpenAPI generada con Swagger provee una interfaz interactiva para explorar y probar los endpoints, facilitando el desarrollo del frontend y la incorporación de nuevos integrantes al proyecto.
- **Integración en CI/CD:** El workflow de GitHub Actions clona ambos repositorios y ejecuta los tests de contrato como un paso previo al despliegue, garantizando que el backend nunca se despliegue con un contrato roto.

### Negativas / Compromisos
- **Fragilidad ante refactorizaciones de formato de código:** Los tests que leen archivos fuente del frontend son sensibles al formato del código (espacios, saltos de línea, nombres de variables). Un cambio de estilo en el frontend (por ejemplo, introducir un formateador diferente) puede romper los tests de contrato sin que el contrato real haya cambiado.
- **Cobertura parcial:** Los tests de contrato estáticos verifican nombres de campos pero no pueden verificar fácilmente tipos de datos, estructuras anidadas complejas o comportamientos condicionales sin mayor sofisticación.
- **Acoplamiento de repositorios en CI/CD:** El pipeline de CI/CD del backend requiere acceso de lectura al repositorio del frontend para ejecutar los tests de contrato, introduciendo un acoplamiento entre los dos ciclos de vida de despliegue.
- **Mantenimiento de la especificación OpenAPI:** Las anotaciones Swagger en el código deben mantenerse actualizadas junto con los cambios en los endpoints, lo que agrega una responsabilidad de mantenimiento adicional.
- **Mantenimiento de los tests de contrato:** A medida que el sistema evoluciona, los tests de contrato deben actualizarse junto con los cambios en la API, lo que agrega una responsabilidad de mantenimiento adicional.
