# ADR-0009: Inicialización Lazy de Clientes Externos en Entorno Serverless

## Contexto

En el entorno de Firebase Cloud Functions, cuando una función se despliega o cuando su instancia se destruye por inactividad, el módulo JavaScript se carga nuevamente desde cero en el próximo request (fenómeno conocido como *cold start*). Durante esta inicialización, si el código de nivel superior intenta instanciar un cliente como `PrismaClient` o el cliente de Supabase, pueden ocurrir los siguientes problemas:

1. **Variables de entorno no disponibles:** En el entorno de Firebase Cloud Functions Gen 2, los secretos de Google Cloud Secret Manager son inyectados en las variables de entorno de forma asíncrona después de que el módulo se carga. Si `PrismaClient` se instancia al cargar el módulo, la variable `DATABASE_URL` aún no está disponible, causando un error de inicialización.

2. **Múltiples instancias de conexión:** Si el cliente se instancia en el nivel del módulo, cada importación del módulo podría generar múltiples instancias, agotando el pool de conexiones de PostgreSQL.

3. **Error de conexión en frío:** La conexión a PostgreSQL puede no establecerse correctamente durante la fase de inicialización del módulo, antes de que el entorno esté completamente configurado.

Estos problemas se manifestaron durante el desarrollo local con el emulador de Firebase y debieron resolverse antes del despliegue en producción.

## Decisión

Se implementó el patrón de **inicialización lazy con singleton** para los clientes externos: `PrismaClient` y el cliente de Supabase.

Cada cliente está encapsulado en una función factory (`getPrisma()`, `getSupabase()`) que sigue este comportamiento:

```typescript
// Ejemplo conceptual del patrón (src/config/prismaClient.ts)
let prismaInstance: PrismaClient | null = null;

export function getPrisma(): PrismaClient | null {
  if (!process.env.DATABASE_URL) return null;  // Degradación segura
  if (!prismaInstance) {
    prismaInstance = new PrismaClient();        // Inicialización en primer uso
  }
  return prismaInstance;
}
```

Las características del patrón implementado son:

- **Lazy initialization:** El cliente se crea únicamente cuando es requerido por primera vez, no al cargar el módulo.
- **Singleton:** Una vez creado, se reutiliza la misma instancia en todas las solicitudes subsiguientes que procese la misma instancia de la función.
- **Degradación segura (graceful degradation):** Si las variables de entorno necesarias no están disponibles, la función retorna `null` en lugar de lanzar una excepción, permitiendo que el sistema reporte un error controlado.

## Alternativas Consideradas

- **Inicialización eager (al cargar el módulo):** Instanciar los clientes directamente en el nivel del módulo (`const prisma = new PrismaClient()`). Fue descartada porque causa errores de conexión cuando las variables de entorno de producción no están disponibles en el momento de carga del módulo.

- **PgBouncer o middleware de connection pooling:** Colocar un proxy de pool de conexiones entre Firebase Functions y PostgreSQL para gestionar las conexiones de forma eficiente. Fue descartada por la complejidad de setup adicional y porque el patrón singleton es suficiente para el volumen de uso esperado del sistema.

- **Inicialización dentro de cada handler de solicitud:** Crear una nueva instancia de `PrismaClient` por cada solicitud HTTP. Fue descartada porque crea y destruye conexiones con cada request, lo que es ineficiente y puede agotar el límite de conexiones de PostgreSQL bajo carga.

- **Variables globales de Node.js:** Usar `global.prismaInstance` para compartir la instancia entre módulos en el mismo proceso. Fue descartada por no ser un patrón idiomático en TypeScript y por potenciales problemas con el sistema de tipos.

## Consecuencias

### Positivas
- **Resolución de cold start:** Los secretos de Google Cloud se inyectan correctamente antes de que el cliente se instancie, dado que la instanciación ocurre al procesar el primer request, no al cargar el módulo.
- **Reutilización de conexiones:** La misma instancia del cliente (y por extensión, la misma conexión a la base de datos) se reutiliza para todas las solicitudes que procesa una misma instancia de la función, reduciendo la sobrecarga de establecer conexiones.
- **Degradación controlada:** Si un secreto no está configurado en un entorno dado (por ejemplo, en tests), el sistema retorna `null` en lugar de lanzar una excepción no controlada.
- **Testabilidad:** El patrón de función factory facilita el mockeo en tests, ya que se puede inyectar una instancia de Prisma falsa sin modificar el módulo.

### Negativas / Compromisos
- **Complejidad adicional:** En lugar de importar directamente el cliente, todos los servicios deben llamar a `getPrisma()` y gestionar el caso en que retorne `null`, lo que agrega verificaciones de nulidad en la capa de repositorios.
- **Primera solicitud más lenta:** La primera request después de un cold start sigue siendo más lenta, ya que el cliente debe inicializarse y establecer la conexión. El patrón lazy no elimina el cold start; solo lo desplaza al momento del primer request en lugar de al momento de carga del módulo.
- **Conexión persistente no siempre garantizada:** En entornos de alta concurrencia, múltiples instancias de la función pueden crearse en paralelo, cada una con su propia instancia del cliente. Esto no es un bug pero debe considerarse al dimensionar el límite de conexiones en PostgreSQL.
