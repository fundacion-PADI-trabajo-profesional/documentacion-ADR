---
title: "ADR-0021: Una evaluación por etapa: tipo según la época del año y continuar la existente"
nav_order: 21
---

# ADR-0021: Una evaluación por etapa: tipo según la época del año y continuar la existente

## Contexto

Desde el 02/11/2026 un alumno puede tener como mucho una evaluación de cada tipo (inicial / cierre)
por sala y año calendario; el alta de una segunda del mismo tipo se rechaza con `TIPO_DUPLICADO`.

En la campaña de cierre de 2026 aparecieron igual evaluaciones duplicadas. El relevamiento en
producción del 24/11/2026 encontró, entre las evaluaciones cargadas desde agosto:

- 58 "iniciales" que en realidad eran cierres;
- 45 alumnos con dos o más evaluaciones del mismo momento (48 sobrantes): la mayoría, una inicial
  vacía o a medias junto al cierre de la misma persona; el resto, dos personas evaluando al mismo
  chico (una toma tres áreas y la docente del aula responde Socioemocional).

La causa estaba en el tipo que proponía el formulario de alta: sin historial proponía "inicial"
(también en noviembre) y, si el alumno tenía cierre pero no inicial, proponía "inicial" con un
aviso. Como el segundo cierre lo rechazaba el servidor, la segunda carga terminaba guardada como
inicial. Muchas escuelas no cargaron iniciales en 2026, así que ese era el caso común.

No hacía falta cambiar permisos: la docente asignada al aula ya puede ver y responder una evaluación
creada por otra persona (controlador de evaluaciones, desde junio de 2026). Faltaba llevarla hasta
esa evaluación en vez de dejarla crear otra.

Los datos existentes se corrigieron el 24/11/2026 con un script (respaldo, cambio de tipo y fusión):
`backend/scripts/sql/2026-11-24-fusion-evaluaciones.sql`.

## Decisión

**El tipo que toca lo da la fecha de carga.** Hasta julio el formulario propone inicial; de agosto
en adelante, cierre, aunque el alumno no tenga inicial. La persona puede cambiarlo. La regla es una
sola función pura del frontend, `tipoPorDefecto(evaluaciones, sala, año, mes)` en
`frontend/src/utils/tipo-evaluacion.ts`, con la constante `MES_INICIO_CIERRE = 8`.

**Si la evaluación que toca ya existe, se continúa esa.** Cuando el alumno ya tiene la evaluación
del tipo de la época (misma sala y año), o ya tiene el cierre, el formulario muestra un aviso
informativo con "Continuar esa evaluación" (abre la existente) y oculta el alta. Sólo ofrece crear
si todavía queda algo por crear (el cierre, cuando lo que existe es la inicial).

**El formulario sólo deja elegir lo que el servidor aceptaría.** `tipoPorDefecto` devuelve
`puedeCrear` con las mismas reglas del servidor (una de cada tipo; ninguna inicial después del
cierre) y las opciones que no se pueden crear aparecen deshabilitadas. El objetivo es que la persona
no llegue a un mensaje de error por algo que no eligió; el rechazo del servidor queda como red de
seguridad para el caso de dos personas creando a la vez, y también ofrece continuar la existente.

**No se crea una inicial con fecha igual o posterior a la del cierre.** El servidor rechaza el alta
de una inicial si el alumno ya tiene el cierre de esa sala y ese año con fecha igual o anterior,
con el mismo `TIPO_DUPLICADO` y el cierre en `error.details.evaluacion_existente`
(`verificarInicialDespuesDeCierre` en `backend/functions/src/services/evaluaciones.service.ts`).
Una inicial con fecha anterior al cierre se acepta: es la carga atrasada de una inicial real.

**La evaluación lleva el aula del alumno aunque el formulario no la mande.** El permiso de la docente
del aula se chequea contra el aula de la evaluación, así que una evaluación sin aula no la puede
continuar nadie más. Si el alta llega sin `aula_id`, el servidor la deduce de las aulas activas del
alumno: la única que tenga o, si tiene varias, la única de la misma sala que la evaluación; si no
puede decidir, la deja sin aula (`elegirAulaDeEvaluacion` en
`backend/functions/src/utils/evaluaciones-aula.ts`). Con esa misma regla se completaron el
24/11/2026 51 de las 53 evaluaciones que estaban sin aula
(`backend/scripts/sql/2026-11-24-asignar-aula-evaluaciones*.sql`).

**Criterio de la corrección de datos** (para repetirla si hiciera falta): entre las evaluaciones del
mismo alumno y sala queda la que tiene más respuestas; recibe las respuestas que le falten; ante dos
respuestas distintas a la misma pregunta vale la más reciente; las sobrantes se respaldan y se
borran; se recalculan puntaje y estado de las áreas tocadas. No se reevalúa a ningún chico.

## Alternativas consideradas

- **Dejar el tipo por defecto según el historial.** Es lo que había. Falla cuando el historial está
  incompleto, que fue el caso general en 2026: sin inicial cargada, proponía inicial en noviembre.
- **Fijar el tipo por la fecha sin dejar elegir.** Más simple, pero impide cargar tarde una inicial
  real o un cierre adelantado. Se prefirió proponer y frenar sólo el caso que no tiene sentido
  (inicial después del cierre).
- **Restricción única en la base** (alumno, sala, año, tipo). Daría la garantía más fuerte, pero el
  año sale de `fecha_creacion` y los datos históricos todavía no se revisaron completos; una
  restricción que falle al crearla frena el deploy. Queda como paso posterior.
- **Una evaluación por alumno y año, sin tipos.** No sirve: el reporte compara inicial contra cierre.
- **Evaluación compartida con varios evaluadores registrados** (tabla de participantes). Resuelve
  quién respondió cada área, pero es un cambio de modelo; hoy alcanza con que la docente del aula
  trabaje sobre la evaluación existente. La evaluación sigue guardando un solo `profesor_id`.

## Consecuencias

- Una segunda persona que entra a evaluar a un chico llega a la evaluación ya empezada en vez de
  crear otra. El camino que producía duplicados (inicial después del cierre) queda cerrado en el
  servidor, no sólo en la pantalla.
- La regla de época es una convención del calendario escolar argentino. Si la Fundación adelanta o
  atrasa la campaña de cierre, se cambia `MES_INICIO_CIERRE`.
- Quien evalúa sin estar asignada al aula (o una evaluación sin aula cargada) sigue sin poder abrir
  la evaluación de otra persona: recibe 403. Si ese caso aparece, es una decisión de permisos aparte.
- La deducción del aula depende de que las asignaciones alumno–aula estén bien. El 24/11/2026 había
  alumnos con dos aulas activas (16 de una misma escuela, en su sala 3 y en un aula de sala 5 con 84
  alumnos y ningún docente). Las estadísticas no se duplican por eso (cuentan evaluaciones, y cada
  una tiene un aula sola), pero conviene revisarlo antes de los reportes:
  `backend/scripts/sql/2026-11-24-control-datos-reporte.sql`.
- `profesor_id` muestra a quien creó la evaluación; no queda registro de quién respondió cada área.
- Cambiar el tipo de una evaluación existente (`PATCH /evaluaciones/:id`) no aplica el freno nuevo:
  es la herramienta para corregir a mano una evaluación mal rotulada.
- La tabla de respaldo de la corrección (`fusion_20261124_*`) queda en producción hasta que se
  decida borrarla; contiene respuestas de evaluaciones de menores y tiene RLS activado sin políticas.
