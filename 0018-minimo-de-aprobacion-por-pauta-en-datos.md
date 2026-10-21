---
title: "ADR-0018: Mínimo de aprobación por pauta como dato de la pregunta"
nav_order: 18
---

# ADR-0018: Mínimo de aprobación por pauta como dato de la pregunta

## Contexto

Una pauta de la Prueba PADI es el grupo de preguntas que comparten `numero` dentro de un área y una
sala. Algunas pautas tienen varias partes (Intrusos: tres categorías; Copiado de figuras: cuatro
figuras; Sonido inicial: cinco estímulos) y aprueban cuando el chico acierta una cantidad mínima de
partes.

Hasta ahora ese mínimo estaba en el código, con una sola regla para todas: la mitad de las partes
redondeada hacia arriba (3 → 2, 4 → 2, 5 → 3). La regla vivía en tres lugares del backend (el
cálculo del área al guardar respuestas, la ficha de la evaluación y el reporte de escuela) y uno del
frontend (la pantalla de la pauta y la revisión).

El cuadernillo de la Fundación fija un mínimo propio por pauta, y en cinco pautas no coincide con la
mitad: Intrusos de sala 5 aprueba con 3 de 3, las pautas de cuatro partes con 3 de 4 y Absurdo
verbal con 4 de 5. La Fundación confirmó que el cuadernillo es la referencia y pidió recalcular las
evaluaciones ya cargadas.

## Decisión

El mínimo de aprobación de una pauta es un **dato de las preguntas**, no una constante del código:

- Nueva columna `preguntas.minimo_aprobacion` (smallint, nula). Todas las partes de una pauta llevan
  el mismo valor; si difirieran, se usa el más alto.
- Valor nulo significa "la regla de antes" (mitad redondeada hacia arriba). Así las pautas simples y
  cualquier pauta sin valor cargado siguen funcionando igual, y no hace falta migrar nada para
  desplegar la columna.
- Una sola función de dominio por repo decide si la pauta aprueba (`apruebaPauta` con `minimo` en
  `backend/functions/src/utils/pautas.ts`; `estadoPauta` en `frontend/src/utils/pautas.ts`) y
  todos los lugares que calculan aprobación la usan. Se eliminó la copia inline que tenía la ficha
  de la evaluación.
- El backend devuelve `minimo_aprobacion` con cada pregunta, y el frontend lo usa para calcular y
  para mostrar "Se aprueba con X de Y" al evaluador. El frontend no tiene una tabla propia de
  mínimos.
- Las evaluaciones ya cargadas se recalculan con un script SQL que replica la regla del área
  (`backend/scripts/sql/2026-10-16-recalculo.sql`), precedido de una consulta de solo lectura que
  muestra qué cambiaría.

## Alternativas Consideradas

1. **Tabla de mínimos en el código** (un mapa sala/área/número → mínimo) — más rápido de hacer,
   pero cada corrección del cuadernillo exige un deploy y la tabla queda desalineada con las
   preguntas de la base, que son lo que la Fundación edita.
2. **Guardar el mínimo en `reglas_aprobacion`** (la tabla que ya tiene el umbral del área) — mezcla
   dos niveles distintos (área y pauta) en una tabla con una fila por sala/área, y obligaría a
   agregar una clave por pauta que hoy no existe (las pautas no tienen entidad propia, son un
   `numero` compartido).
3. **Parsear `aprueba_con`** (texto libre como "3 de 5 correctas") — frágil: el campo es descriptivo,
   lo escribe la Fundación y no sigue un formato único.
4. **Recalcular con un endpoint administrativo** en lugar de SQL — reutilizaría el código real, pero
   recorrer todas las evaluaciones supera el tiempo de una Cloud Function y el equipo no tiene
   credenciales de producción fuera de la plataforma; el SQL con dry run es verificable antes de
   aplicarse.

## Consecuencias

### Positivas
- La Fundación puede corregir un mínimo editando la pregunta, sin deploy.
- Una regla, cuatro usos: backend y frontend coinciden por construcción porque leen el mismo dato.
- Despliegue sin migración de datos: con la columna vacía nada cambia.

### Negativas
- Los resultados ya mostrados a las escuelas pueden cambiar tras el recálculo (aceptado por la
  Fundación).
- El valor se repite en cada parte de la pauta; un valor inconsistente entre partes se resuelve
  tomando el más alto, pero no hay restricción en la base que lo impida.
- El script de recálculo duplica en SQL la lógica de `calculateAreaScore`; si la regla del área
  cambia, hay que actualizar los dos.
