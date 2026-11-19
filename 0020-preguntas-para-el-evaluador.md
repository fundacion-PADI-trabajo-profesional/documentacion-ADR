---
title: "ADR-0020: Preguntas para el evaluador como datos de la evaluación"
nav_order: 20
---

# ADR-0020: Preguntas para el evaluador como datos de la evaluación

## Contexto

La Fundación pidió (13/11/2026, precisado el 18/11/2026) registrar en cada evaluación tres datos de
contexto que responde quien evalúa, no el chico:

- cuántas sesiones de intervención tiene completadas al momento de la reevaluación (sólo en la
  evaluación de cierre): menos de 5, entre 5 y 9, 10 o más;
- si el chico está en tratamiento y si tiene CUD (Certificado Único de Discapacidad);
- observaciones libres.

Además pidió ver, en los listados de alumnos y junto al semáforo de evaluaciones, si el chico tiene
CUD o está en tratamiento.

En el mismo pedido se quitó "Evaluaciones" como sección del inicio: tener dos llegadas a la
evaluación confundía a las usuarias, y el camino natural es Estudiantes → alumno → Evaluar.

Dos tensiones a resolver: tratamiento y CUD son atributos del chico pero se preguntan en una
evaluación; y son datos de salud de menores.

## Decisión

**Los datos viven en la evaluación.** Cuatro columnas nullable en `evaluaciones_estudiante`:
`sesiones_completadas` (`menos_5` | `entre_5_y_9` | `10_o_mas`), `en_tratamiento` (boolean),
`tiene_cud` (boolean) y `observaciones` (hasta 500 caracteres). `null` significa "sin responder".
Cada evaluación es una foto con fecha de cómo estaba el chico ese día.

**No son un área.** No suman puntaje, no cambian el estado de la evaluación ni el semáforo, y nada
es obligatorio. En la ficha aparecen como una quinta tarjeta ("Preguntas para el evaluador") con
estado propio: completa cuando se respondió tratamiento y, en un cierre, también las sesiones.

**Tratamiento y CUD son independientes** (sí/no + casilla), no una opción única: un chico puede
tener CUD y además estar en tratamiento.

**Una sola regla de "cuál vale hoy"** para los listados: por cada dato, el de la evaluación más
reciente que lo tenga respondido (`resumirContextoEvaluador` en
`backend/functions/src/utils/datos-evaluador.ts`). El backend la aplica y devuelve por alumno
`contexto_evaluador: { en_tratamiento, tiene_cud }` en `GET /estudiantes` y en las aulas del
docente; el frontend sólo dibuja insignias de una letra (C, T) para los "sí".

**Endpoint propio:** `PUT /evaluaciones/:id/datos-evaluador` reemplaza los cuatro campos. Valida
con `DATOS_EVALUADOR_INVALIDOS` (400) y guarda `sesiones_completadas` en `null` si la evaluación no
es de cierre. Permisos iguales al guardado de respuestas: la docente sobre sus evaluaciones o las de
sus aulas activas; director y encargado dentro de su escuela o zona (ADR 0015); Equipo PADI todas.
Los datos vuelven en `GET /evaluaciones/:id`.

**Visibilidad:** ve el dato quien ya puede ver las evaluaciones de ese chico. No se agrega al
reporte PDF de escuela ni al export a Excel.

**"Evaluaciones" deja de ser una sección:** se quitan las tarjetas del inicio de los cuatro roles.
La ruta `/evaluaciones` se mantiene porque es a donde lleva "Evaluar"; al volver desde ahí se
regresa a la pantalla de origen.

## Alternativas consideradas

- **Guardar tratamiento y CUD en el estudiante.** Más simple de leer, pero pisa la historia (no se
  sabría que en la evaluación inicial no estaba en tratamiento) y obliga a otra pantalla de carga.
- **Modelarlas como un área más con preguntas.** Reutiliza el wizard, pero el modelo de áreas asume
  respuestas 0/1, puntaje y regla de aprobación; habría que excluir el área de cada cálculo,
  estadística y reporte.
- **Tabla aparte `evaluaciones_datos_evaluador`.** Sólo se justifica si los datos crecen o si hay
  varias cargas por evaluación; hoy son cuatro campos uno a uno con la evaluación.
- **Una opción única de cuatro valores** (no necesita / aún no empezó / en tratamiento / tiene CUD).
  Mezcla dos preguntas distintas y pierde un dato cuando se dan los dos.
- **Número libre de sesiones.** La Fundación pidió rangos; el número exacto queda para el futuro
  registro de la intervención.

## Consecuencias

- La migración es aditiva (`backend/scripts/sql/2026-11-18-preguntas-evaluador.sql`) y **debe
  correrse antes de desplegar el backend**: el detalle y los listados seleccionan las columnas.
- El test de contrato del backend lee el frontend: el frontend se mergea primero.
- El listado general de evaluaciones ya no tiene entrada desde el inicio; dirección y Equipo PADI
  llegan a una evaluación desde el alumno. La pantalla sigue existiendo por URL y desde "Docentes".
- Si a una evaluación de cierre con sesiones cargadas se le cambia el tipo a inicial, el valor queda
  guardado pero no se muestra ni cuenta; se limpia la próxima vez que se guarde la sección.
- El registro de la intervención (sesiones como contador, seguimiento durante el año) queda fuera:
  este modelo no lo bloquea, porque las sesiones son un dato de la evaluación de cierre.
- Datos de salud de menores: cualquier nueva salida (reportes, exports, estadísticas) requiere una
  decisión explícita.
