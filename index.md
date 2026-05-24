# Registro de Decisiones de Arquitectura — Sistema PADI

Este directorio contiene los Registros de Decisiones de Arquitectura (ADRs) del sistema de evaluación pedagógica desarrollado para Fundación PADI como trabajo profesional.

Cada ADR documenta una decisión de diseño significativa: el contexto que la motivó, la decisión tomada, las alternativas descartadas y las consecuencias observadas o esperadas.

---

## Índice de ADRs

| Número | Título | 
|---|---|
| [ADR-0001](0001-arquitectura-en-capas-backend.md) | Arquitectura en Capas para el Backend | 
| [ADR-0002](0002-firebase-cloud-functions-backend.md) | Firebase Cloud Functions como Plataforma de Despliegue del Backend | 
| [ADR-0003](0003-postgresql-como-base-de-datos.md) | PostgreSQL como Sistema de Gestión de Base de Datos | 
| [ADR-0004](0004-supabase-como-proveedor-de-autenticacion.md) | Supabase como Proveedor de Autenticación | 
| [ADR-0005](0005-prisma-como-orm.md) | Prisma como ORM para el Acceso a Datos | 
| [ADR-0006](0006-control-de-acceso-basado-en-roles.md) | Control de Acceso Basado en Roles (RBAC) | 
| [ADR-0007](0007-react-typescript-frontend.md) | React con TypeScript como Framework de Frontend | 
| [ADR-0008](0008-tanstack-query-gestion-estado-servidor.md) | TanStack React Query para la Gestión del Estado del Servidor | 
| [ADR-0009](0009-inicializacion-lazy-clientes-externos.md) | Inicialización Lazy de Clientes Externos en Entorno Serverless | 
| [ADR-0010](0010-formato-de-respuesta-unificado-api.md) | Formato de Respuesta Unificado en la API REST | 
| [ADR-0011](0011-pruebas-de-contrato-frontend-backend.md) | Pruebas de Contrato entre Frontend y Backend | 
| [ADR-0012](0012-gestion-de-secretos-google-cloud.md) | Gestión de Secretos con Google Cloud Secret Manager | 
| [ADR-0013](0013-endurecimiento-capa-http.md) | Endurecimiento de la Capa HTTP (Helmet, CORS y Rate Limiting) | 

---

## Formato de los ADRs

Todos los ADRs incluyen las siguientes secciones:

- **Contexto:** Descripción del problema o necesidad que motivó la decisión, incluyendo las restricciones y requisitos relevantes.
- **Decisión:** La decisión tomada, descrita de forma clara y precisa.
- **Alternativas consideradas:** Las opciones evaluadas y descartadas, con justificación.
- **Consecuencias:** Resultados positivos y negativos/compromisos (trade-offs) de la decisión adoptada.
