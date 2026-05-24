# ADR-0012: Gestión de Secretos con Google Cloud Secret Manager

## Contexto

El backend requiere acceder a credenciales sensibles para funcionar:

- `DATABASE_URL`: cadena de conexión a PostgreSQL en Supabase, que incluye usuario, contraseña y host.
- `SUPABASE_URL`: URL de la instancia de Supabase.
- `SUPABASE_KEY`: clave de servicio (`service_role`) de Supabase con permisos administrativos.

Estas credenciales no pueden incluirse en el código fuente ni en los artefactos de despliegue (imágenes Docker, archivos de configuración en el repositorio), ya que el repositorio puede ser accedido por múltiples personas y los artefactos pueden quedar registrados en sistemas de logging o control de versiones.

La política de seguridad del proyecto establece que las credenciales de producción no deben estar disponibles en el entorno de desarrollo local ni en el repositorio de código.

## Decisión

Se adoptó **Google Cloud Secret Manager** como solución de gestión de secretos para el entorno de producción.

Los secretos de producción se declaran en el array `secrets` de la configuración de la Cloud Function en `src/index.ts`:

```typescript
export const api = onRequest(
  {
    secrets: ["DATABASE_URL", "SUPABASE_URL", "SUPABASE_KEY"],
    // ... otras configuraciones
  },
  server
);
```

Firebase Cloud Functions Gen 2 inyecta automáticamente estos secretos como variables de entorno (`process.env.DATABASE_URL`, etc.) al momento de ejecutar la función, pero solo en el contexto de ejecución de producción. Los secretos nunca se escriben en disco ni en logs.

Para el entorno de **desarrollo local**, se utiliza un archivo `.env.local` que **no se incluye en el control de versiones** (está en `.gitignore`). Este archivo es creado manualmente por cada desarrollador con las credenciales del entorno de desarrollo.

Los módulos de configuración (`src/config/env.ts`) abstraen la lectura de variables de entorno, proveyendo una interfaz tipada para acceder a los secretos sin referencias directas a `process.env` dispersas por el código.

## Alternativas Consideradas

- **Variables de entorno en el CI/CD (GitHub Actions Secrets):** Almacenar las credenciales como secretos de GitHub y exponerlas como variables de entorno durante el despliegue. Fue evaluada seriamente y es una alternativa válida. Se descartó en favor de Google Cloud Secret Manager porque este último permite rotación de secretos sin modificar el pipeline de CI/CD, ofrece auditoría de accesos integrada con Google Cloud IAM, y es la solución nativa del ecosistema de Firebase/Google Cloud.

- **Credenciales en Firebase Remote Config:** Almacenar configuración en Firebase Remote Config, que permite actualizar valores sin redesplegar. Fue descartada porque Remote Config está diseñado para configuración de aplicaciones cliente (frontend), no para secretos del servidor.

- **Archivo `.env` cifrado en el repositorio:** Cifrar el archivo de variables de entorno con herramientas como `git-crypt` o `sops` e incluirlo en el repositorio. Fue descartada porque agrega complejidad en la gestión de claves de cifrado y no sigue el principio de que los secretos nunca deben residir en el repositorio, aunque estén cifrados.

## Consecuencias

### Positivas
- **Secretos nunca en el código fuente:** El repositorio no contiene ninguna credencial de producción. Un desarrollador que clone el repositorio no puede acceder al entorno de producción sin permisos explícitos en Google Cloud IAM.
- **Rotación de credenciales sin redespliegue de código:** Si una credencial se ve comprometida, puede ser rotada en Google Cloud Secret Manager sin modificar el código ni el pipeline de CI/CD.
- **Auditoría de accesos:** Google Cloud Secret Manager registra cada acceso a un secreto, permitiendo identificar accesos no autorizados o inesperados.
- **Control de acceso granular:** Los permisos de acceso a cada secreto se gestionan mediante Google Cloud IAM, permitiendo que solo la cuenta de servicio de Firebase Functions tenga acceso a los secretos de producción.
- **Integración transparente con Firebase Functions:** La declaración de secretos en el array `secrets` de la configuración de la función es todo lo que se requiere; Firebase e GCP gestionan el resto.

### Negativas / Compromisos
- **Complejidad de configuración inicial:** Configurar Google Cloud Secret Manager requiere crear los secretos en la consola de GCP o mediante `gcloud CLI`, asignar permisos IAM a la cuenta de servicio de Firebase, y declarar los secretos en el código. Este setup es más complejo que simplemente definir variables de entorno en un archivo.
- **Dependencia de la consola de GCP:** Para actualizar un secreto en producción es necesario acceder a la consola de Google Cloud o usar la CLI de `gcloud`, lo que requiere permisos apropiados y no es inmediato.
- **Latencia adicional en arranque:** En cada cold start, Firebase Functions recupera los secretos desde Secret Manager antes de iniciar la función. Esto agrega una pequeña latencia al proceso de inicialización.
