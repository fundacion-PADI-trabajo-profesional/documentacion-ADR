# ADR-0004: Supabase como Proveedor de Autenticación

## Contexto

El sistema PADI requiere un mecanismo de autenticación que:

- Identifique de forma segura a los usuarios (docentes, directivos, encargados de zona, equipo PADI).
- Emita tokens que puedan ser validados por el backend en cada solicitud.
- Gestione el ciclo de vida de las sesiones: inicio de sesión, expiración de tokens, recuperación de contraseña y cierre de sesión.
- Permita al backend verificar la identidad del usuario y recuperar su perfil (rol, escuela, nombre) para aplicar las reglas de autorización correspondientes.

El sistema no almacena contraseñas directamente; la autenticación debe ser delegada a un proveedor seguro y establecido. Adicionalmente, la arquitectura adoptó Supabase como plataforma de alojamiento de la base de datos PostgreSQL (ver ADR-0003), lo que abre la posibilidad de aprovechar sus servicios de autenticación integrados sin incorporar un proveedor adicional.

## Decisión

Se adoptó **Supabase Auth** como proveedor de autenticación, basado en el estándar **JWT (JSON Web Token)**.

El flujo de autenticación funciona de la siguiente manera:

1. El cliente envía credenciales (`email`/`password`) al endpoint `POST /auth/login` del backend.
2. El backend delega la verificación a la API de Supabase Auth, que retorna un `access_token` (JWT de corta duración) y un `refresh_token`.
3. El backend retorna ambos tokens al cliente.
4. El cliente almacena los tokens en `localStorage` e incluye el `access_token` en el header `Authorization: Bearer <token>` de cada solicitud subsiguiente.
5. El middleware `requireAuth` del backend valida el token contra Supabase, recupera el ID del usuario y consulta la tabla `usuarioPerfil` para obtener su rol y datos de perfil.
6. Cuando el `access_token` expira, el cliente utiliza el `refresh_token` para obtener un nuevo par de tokens.

El cliente de Supabase en el backend se inicializa con `persistSession: false`, dado que el entorno serverless no mantiene estado entre invocaciones.

## Alternativas Consideradas

- **Firebase Authentication:** Proveedor nativo del ecosistema Firebase ya utilizado para el hosting y las funciones. Fue descartado porque el proyecto adoptó Supabase como base de datos, y Firebase Auth almacena usuarios en un sistema separado de Firebase, lo que habría requerido sincronizar manualmente los perfiles de usuario con PostgreSQL. Supabase Auth, en cambio, crea automáticamente registros en el esquema `auth` de la misma base de datos PostgreSQL, simplificando la gestión.

- **Auth0:** Proveedor de identidad de clase empresarial con soporte avanzado para múltiples proveedores (OAuth, SAML). Fue descartado por su modelo de precios, que resulta costoso para el volumen de usuarios del sistema, y por introducir una dependencia adicional de terceros.

- **Implementación propia con JWT:** Generar y validar JWT utilizando librerías como `jsonwebtoken` y almacenar contraseñas hasheadas con `bcrypt` directamente en la base de datos. Fue descartado porque reimplementar un sistema de autenticación seguro es complejo y propenso a vulnerabilidades. Los proveedores establecidos como Supabase siguen las mejores prácticas de seguridad de forma probada.

- **Passport.js con estrategia local:** Framework de autenticación para Node.js que delega la estrategia a plugins. Fue descartado por la misma razón que la implementación propia: requiere gestionar el almacenamiento de contraseñas y la emisión de tokens, responsabilidades que un proveedor dedicado maneja mejor.

## Consecuencias

### Positivas
- **Seguridad delegada a un proveedor establecido:** Supabase sigue las mejores prácticas de seguridad para el almacenamiento de contraseñas (bcrypt), la emisión de JWT y la protección contra ataques comunes.
- **Integración con la base de datos:** Los usuarios de Supabase Auth viven en el mismo cluster de PostgreSQL que los datos de la aplicación, simplificando la sincronización y las consultas relacionales.
- **Gestión del ciclo de sesión:** Supabase Auth maneja nativamente la expiración de tokens, el refresh y la recuperación de contraseña por email, sin implementación adicional en el backend.
- **SDK disponible en frontend y backend:** El paquete `@supabase/supabase-js` provee una interfaz consistente tanto para el cliente como para el servidor.
- **Row Level Security (RLS) disponible:** Aunque no utilizado en esta versión, Supabase permite configurar políticas de seguridad a nivel de fila directamente en PostgreSQL, que podrían aprovecharse en futuras iteraciones.

### Negativas / Compromisos
- **Doble fuente de verdad para identidad de usuario:** El sistema mantiene un registro en `auth.users` (gestionado por Supabase) y otro en `usuarioPerfil` (gestionado por la aplicación). Esto requiere sincronización manual al crear usuarios y elimina la posibilidad de listar usuarios directamente desde PostgreSQL sin consultar ambas fuentes.
- **Dependencia de disponibilidad de Supabase:** La validación de tokens en cada request requiere una llamada a la API de Supabase. Si el servicio de autenticación de Supabase presenta una interrupción, el sistema completo se vuelve inaccesible.
- **Acoplamiento al SDK de Supabase:** El middleware de autenticación utiliza el cliente de Supabase, lo que dificulta el reemplazo del proveedor de autenticación sin refactorizar la capa de middleware.
