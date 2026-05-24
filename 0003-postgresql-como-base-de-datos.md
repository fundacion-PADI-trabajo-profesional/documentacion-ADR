# ADR-0003: PostgreSQL como Sistema de Gestión de Base de Datos

## Contexto

El sistema PADI gestiona datos con alta complejidad relacional:

- Una **jerarquía organizacional** de múltiples niveles: zonas → escuelas → aulas → estudiantes.
- **Evaluaciones** compuestas por áreas de conocimiento, preguntas individuales y respuestas por estudiante.
- **Asignaciones múltiples:** un docente puede estar asignado a múltiples escuelas y aulas; un estudiante puede pertenecer a múltiples aulas.
- **Reglas de aprobación** por área y sala que determinan el resultado de cada evaluación.
- **Usuarios con roles** que deben tener acceso filtrado según su posición en la jerarquía organizacional.

Estas características implican relaciones muchos-a-muchos, integridad referencial entre entidades, y consultas complejas que combinan múltiples tablas. Además, se adoptó Supabase como plataforma de infraestructura, dado que ofrece PostgreSQL gestionado y servicios de autenticación integrados como parte de su ecosistema (ver ADR-0004).

## Decisión

Se adoptó **PostgreSQL** como sistema de gestión de base de datos relacional, alojado en la infraestructura gestionada de **Supabase**.

El esquema de datos se define y gestiona mediante **Prisma** como ORM (ver ADR-0005). La base de datos es la única fuente de verdad para todos los datos operacionales del sistema. Supabase provee además la URL de conexión, el panel de administración y las funciones de autenticación.

## Alternativas Consideradas

- **Firebase Firestore (NoSQL, documento):** Alternativa natural dado que el proyecto adoptó Firebase como plataforma de backend (ver ADR-0002). Fue descartada porque su modelo de datos basado en colecciones/documentos no se adapta bien a relaciones muchos-a-muchos ni a consultas transversales complejas. Además, carece de soporte nativo para transacciones complejas y joins, lo que habría forzado a replicar lógica relacional en la capa de aplicación.

- **MongoDB Atlas:** Base de datos documental gestionada en la nube. Fue descartada por las mismas razones que Firestore: el modelo de datos del sistema es inherentemente relacional, y forzar ese modelo en documentos habría generado inconsistencias y complejidad innecesaria.

- **MySQL:** Base de datos relacional madura con amplio soporte. Fue descartada en favor de PostgreSQL por la mejor integración de este último con Supabase (que es exclusivamente PostgreSQL), el soporte nativo de tipos de datos avanzados (JSONB, arrays, enums), y la mayor expresividad de sus capacidades de consulta.

- **SQLite:** Adecuada para desarrollo local, pero no apropiada para producción en un entorno serverless donde múltiples instancias de función podrían necesitar acceso concurrente a la misma base de datos.

## Consecuencias

### Positivas
- **Integridad referencial garantizada:** Las relaciones entre entidades (estudiantes, aulas, evaluaciones) están aseguradas por claves foráneas y restricciones declaradas en el esquema, previniendo inconsistencias de datos.
- **Consultas complejas con SQL:** Las estadísticas y reportes del sistema requieren agregaciones y joins entre múltiples tablas, que PostgreSQL maneja de forma nativa y eficiente.
- **Compatibilidad con Prisma:** Prisma soporta PostgreSQL de forma completa, incluyendo la generación automática de migraciones y tipado TypeScript.
- **Supabase como infraestructura gestionada:** Elimina la necesidad de administrar el servidor de base de datos, los backups y las actualizaciones de versión.
- **Soporte de transacciones ACID:** Operaciones críticas como la creación de una evaluación (que involucra múltiples tablas) pueden realizarse en una transacción atómica.
- **Tipos de datos enriquecidos:** PostgreSQL soporta nativamente tipos como `ENUM`, facilitando la representación de entidades como `Rol` y `EstadoEvaluacion`.

### Negativas / Compromisos
- **Dependencia de Supabase como proveedor de base de datos:** Si Supabase cambia sus condiciones de servicio o presenta una interrupción, afecta directamente la disponibilidad del sistema.
- **Latencia de red:** La base de datos está alojada en la infraestructura de Supabase, separada físicamente del backend en Firebase Cloud Functions. Cada consulta implica una llamada de red adicional, a diferencia de una base de datos colocada en el mismo servidor.
- **Gestión de conexiones en entorno serverless:** PostgreSQL está diseñado para un número limitado de conexiones persistentes. En un entorno serverless donde cada instancia de función crea su propia conexión, es posible alcanzar el límite de conexiones bajo carga alta. Este problema es mitigado por el patrón singleton descrito en ADR-0009.
