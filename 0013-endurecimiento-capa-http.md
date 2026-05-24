# ADR-0013: Endurecimiento de la Capa HTTP (Helmet, CORS y Rate Limiting)

## Contexto

El backend del sistema PADI es un servidor HTTP expuesto públicamente a través de Firebase Cloud Functions. Al ser accesible desde Internet, está sujeto a un conjunto de amenazas que deben mitigarse en la capa de transporte antes de que cualquier solicitud llegue a la lógica de negocio:

- **Ataques de inyección de cabeceras HTTP maliciosas:** Navegadores web ejecutan scripts y cargan recursos de acuerdo a las cabeceras HTTP que reciben. Sin cabeceras de seguridad apropiadas, es posible explotar vulnerabilidades como clickjacking (carga en iframes), MIME type sniffing, y la ausencia de políticas de referrer o de contenido.

- **Cross-Origin Resource Sharing (CORS) sin restricciones:** Si el servidor acepta solicitudes de cualquier origen, un sitio malicioso podría realizar solicitudes autenticadas al backend utilizando las credenciales del usuario (cookies o tokens almacenados en el navegador), un ataque conocido como Cross-Site Request Forgery (CSRF).

- **Ataques de fuerza bruta sobre endpoints de autenticación:** Los endpoints `POST /auth/login` y `POST /auth/register` son vectores naturales de ataques de fuerza bruta, donde un atacante intenta múltiples combinaciones de credenciales de forma automatizada hasta encontrar una válida.

Estas tres amenazas operan en la capa HTTP antes de que el middleware de autenticación (`requireAuth`) entre en acción, por lo que deben ser mitigadas de forma independiente y previa.

## Decisión

Se implementaron tres mecanismos de seguridad en la capa HTTP del servidor Express, configurados en `functions/src/server.ts`, aplicados en el siguiente orden:

---

### 1. Cabeceras de Seguridad HTTP con Helmet

Se aplica el middleware **Helmet.js** de forma global con su configuración por defecto:

```typescript
app.use(helmet());
```

Helmet configura automáticamente las siguientes cabeceras HTTP en todas las respuestas del servidor:

| Cabecera | Función |
|---|---|
| `X-Content-Type-Options: nosniff` | Impide que el navegador interprete recursos con un MIME type distinto al declarado, previniendo ataques de MIME sniffing |
| `X-Frame-Options: SAMEORIGIN` | Impide que la aplicación sea embebida en un `<iframe>` de otro origen, previniendo clickjacking |
| `Strict-Transport-Security` | Instruye al navegador a usar exclusivamente HTTPS para todas las solicitudes futuras al dominio |
| `X-XSS-Protection: 0` | Desactiva el filtro XSS del navegador (recomendado en versiones modernas donde puede ser explotado) |
| `Referrer-Policy: no-referrer` | Controla qué información de referrer se envía en solicitudes entre orígenes |
| `Cross-Origin-Resource-Policy` | Restringe desde qué orígenes pueden cargarse los recursos |

---

### 2. CORS con Lista Blanca de Orígenes

Se configura **CORS** con una función de validación de origen que implementa una lista blanca explícita:

```typescript
const allowedOrigins = [
  "http://localhost:5173",        // Dev frontend (Vite)
  "http://localhost:4173",        // Dev frontend (Vite preview)
  "https://fundacionpadi-41cb2.web.app",
  "https://fundacionpadi-41cb2.firebaseapp.com",
  process.env.FRONTEND_URL,       // Producción (inyectado en despliegue)
];

const corsOptions = {
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error("Origen no permitido por la política CORS"));
    }
  },
  methods: ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
  allowedHeaders: ["Content-Type", "Authorization"],
  credentials: true,
};

app.use(cors(corsOptions));
```

Las solicitudes sin cabecera `Origin` (herramientas de línea de comandos como `curl`, clientes REST como Postman, y potencialmente aplicaciones móviles) son permitidas, dado que CORS es una política del navegador y no aplica a clientes no-browser.

---

### 3. Rate Limiting en Endpoints de Autenticación

Se aplica **express-rate-limit** exclusivamente sobre el router de autenticación (`/auth`), que contiene los endpoints de inicio de sesión y registro:

```typescript
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // Ventana de 15 minutos
  max: 20,                    // Máximo 20 solicitudes por ventana por IP
  keyGenerator: (req) => {
    // Firebase Cloud Functions recibe tráfico detrás de un proxy;
    // se usa x-forwarded-for para obtener la IP real del cliente
    return req.headers["x-forwarded-for"]?.toString() || req.ip || "unknown";
  },
});

authRouter.use(authLimiter);
```

El rate limiter no se aplica a los endpoints protegidos del resto de la API, ya que estos requieren un token JWT válido, lo que por sí mismo limita el acceso a usuarios autenticados.

---

## Alternativas Consideradas

- **Configurar Helmet con Content Security Policy (CSP) personalizada:** La CSP es la cabecera de seguridad más poderosa para prevenir XSS, ya que define explícitamente desde qué orígenes pueden cargarse scripts, estilos e imágenes. Fue descartada en su forma personalizada por la complejidad de configuración: una CSP mal definida puede romper el funcionamiento de la aplicación (especialmente con frameworks como React que generan scripts dinámicos). Se adoptó la configuración por defecto de Helmet, que omite la CSP para evitar rupturas accidentales.

- **Rate limiting global sobre todos los endpoints:** Aplicar el limitador de velocidad a toda la API, no solo a `/auth`. Fue descartado porque los endpoints protegidos ya requieren autenticación previa, lo que implica que un atacante necesita un token válido para consumirlos. Aplicar rate limiting global agrega latencia a todos los requests sin un beneficio de seguridad significativo en el contexto del sistema.

- **Rate limiting basado en usuario autenticado (en lugar de IP):** Limitar las solicitudes por `user_id` en lugar de por dirección IP. Fue evaluado pero descartado para los endpoints de autenticación (donde el usuario aún no está autenticado) y considerado innecesario para los endpoints protegidos.

- **WAF (Web Application Firewall) externo:** Soluciones como Cloudflare WAF o Google Cloud Armor proveen filtrado de tráfico malicioso a nivel de red, antes de que las solicitudes lleguen al servidor. Fueron descartadas por su costo y complejidad de configuración, que están fuera del alcance de un proyecto de tesis.

- **Validación y sanitización de inputs (Zod, Joi):** La incorporación de un esquema de validación formal para los cuerpos de las solicitudes (usando librerías como Zod o Joi) fue identificada como una mejora de seguridad pendiente. En la implementación actual, la validación se realiza mediante verificaciones de presencia manuales en los controladores. Esta brecha es reconocida como deuda técnica.

## Consecuencias

### Positivas
- **Defensa en profundidad:** Las tres medidas operan de forma independiente y en capas: Helmet protege contra ataques a nivel de cabeceras de navegador, CORS restringe el acceso desde orígenes no autorizados, y el rate limiting reduce la viabilidad de ataques automatizados.
- **Configuración declarativa y centralizada:** Toda la configuración de seguridad HTTP está concentrada en `server.ts`, facilitando su auditoría y modificación.
- **Sin modificaciones a la lógica de negocio:** Las medidas de seguridad son middlewares de Express transversales que no requieren modificar controladores, servicios ni repositorios. Pueden actualizarse de forma independiente.
- **Compatibilidad con el entorno serverless:** El uso de `x-forwarded-for` en el key generator del rate limiter garantiza que la identificación de clientes sea correcta detrás del proxy de Firebase Cloud Functions.

### Negativas / Compromisos
- **Helmet sin CSP personalizada:** La ausencia de una Content Security Policy explícita deja abierta la posibilidad de ataques XSS basados en scripts inyectados, aunque el uso de Prisma con consultas parametrizadas mitiga la principal fuente de datos no confiables.
- **Rate limiting basado en IP no es infalible:** Un atacante con acceso a múltiples IPs (por ejemplo, una botnet) puede eludir el rate limiting distribuido por IP. Adicionalmente, múltiples usuarios legítimos detrás de un NAT compartido podrían verse afectados colectivamente por el límite.
- **CORS no reemplaza la autenticación del servidor:** CORS es una política del navegador; un cliente no-browser puede ignorarla. Por este motivo, CORS complementa pero no reemplaza al middleware `requireAuth`, que valida el token JWT en cada solicitud independientemente del origen.
- **Ausencia de validación formal de inputs:** La falta de un esquema de validación declarativo (Zod, Joi) significa que inputs malformados pueden llegar hasta la capa de servicios, donde son rechazados con errores menos descriptivos. Esta es una brecha de seguridad identificada como trabajo futuro.
