# ADR-0002: Firebase Cloud Functions como Plataforma de Despliegue del Backend

## Contexto

El sistema PADI necesitaba una infraestructura para ejecutar el backend de la API REST. Los criterios de evaluación incluyeron:

- **Costo:** Al ser un proyecto de tesis sin financiamiento externo, la plataforma debía ser gratuita o de costo mínimo durante la fase de desarrollo y piloto.
- **Gestión de infraestructura:** El equipo de desarrollo es reducido y no cuenta con recursos para administrar servidores.
- **Escalabilidad automática:** El tráfico puede variar significativamente dependiendo del período del ciclo escolar.
- **Integración con el ecosistema:** La arquitectura adoptó Firebase Hosting para el frontend y Google Cloud Secret Manager para la gestión de secretos, por lo que Firebase Cloud Functions se integra de forma nativa con el resto del stack.
- **Soporte de Node.js y TypeScript:** Requisito tecnológico fundamental del proyecto.

Se evaluaron distintas opciones de despliegue antes de tomar una decisión.

## Decisión

Se adoptó **Firebase Cloud Functions (Gen 2) con Express.js** como plataforma de backend serverless.

La función principal (`api`) es de tipo `onRequest`, lo que la expone como un endpoint HTTPS que recibe todas las peticiones y las delega al servidor Express. La configuración establece:

- **Runtime:** Node.js 22
- **Región:** `us-central1`
- **Gestión de secretos:** integrada con Google Cloud Secret Manager

Express actúa como un framework HTTP convencional dentro del entorno serverless, lo que permite reutilizar middlewares estándar (CORS, Helmet, rate limiting) sin modificaciones.

## Alternativas Consideradas

- **Servidor tradicional en VPS (EC2, Compute Engine):** Ofrece control total sobre el entorno de ejecución y sin limitaciones de tiempo de ejecución. Fue descartado por la necesidad de gestionar actualizaciones del sistema operativo, configurar balanceo de carga y pagar por tiempo de cómputo independientemente del uso.

- **Google Cloud Run (contenedor Docker):** Plataforma serverless basada en contenedores, más flexible que Cloud Functions. Fue evaluada seriamente: el `Dockerfile` existente en el repositorio es evidencia de esta consideración. Fue descartada por la mayor complejidad en el pipeline de CI/CD (construcción y publicación de imágenes Docker) y por la curva de aprendizaje adicional que representaba para el alcance del proyecto.

- **Plataformas PaaS (Heroku, Railway, Render):** Simplifican el despliegue sin gestión de infraestructura. Fueron descartadas por su nivel gratuito limitado, posible latencia adicional por regiones geográficas distantes, y menor integración con el resto del stack de Google Cloud.

- **Vercel o Netlify Functions:** Orientadas principalmente a frontends con funciones de borde. Fueron descartadas por ofrecer menor control sobre el entorno de ejecución del servidor y peor integración con PostgreSQL.

## Consecuencias

### Positivas
- **Sin gestión de infraestructura:** Google administra el aprovisionamiento, el escalado y las actualizaciones del entorno de ejecución.
- **Escalado automático:** Las instancias se crean y destruyen automáticamente según la demanda, sin configuración adicional.
- **Modelo de costo eficiente:** El modelo de facturación por invocación es adecuado para el volumen de uso esperado en un piloto escolar.
- **Integración nativa con Google Cloud:** Acceso directo a Secret Manager, Cloud Logging e Identity and Access Management (IAM) sin configuración adicional.
- **Despliegue integrado en CI/CD:** El comando `firebase deploy --only functions` es atómico y se integra directamente en el workflow de GitHub Actions.

### Negativas / Compromisos
- **Cold start:** Las funciones que llevan tiempo inactivas experimentan una demora inicial en la primera invocación mientras el runtime se inicializa. Esto afecta la experiencia del usuario si el sistema no recibe tráfico frecuente.
- **Sin estado persistente entre invocaciones:** Cada ejecución es independiente; no es posible mantener estado en memoria entre peticiones. Esto motivó el patrón de inicialización lazy descrito en ADR-0009.
- **Tiempo de ejecución limitado:** Las funciones tienen un tiempo máximo de ejecución configurable. Tareas de procesamiento muy prolongadas requieren soluciones alternativas.
- **Acoplamiento al proveedor (vendor lock-in):** El uso de `firebase-functions` y `firebase-admin` introduce dependencias específicas de Google Cloud que dificultarían una migración futura a otra plataforma.
