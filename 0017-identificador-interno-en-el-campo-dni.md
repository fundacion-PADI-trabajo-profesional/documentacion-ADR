---
title: "ADR-0017: Identificador interno del colegio en el campo DNI del alumno"
nav_order: 17
---

# ADR-0017: Identificador interno del colegio en el campo DNI del alumno

## Contexto

Los colegios no siempre disponen del DNI de cada alumno, o les resulta difícil pedirlo, y pidieron
que el dato sea opcional en la carga individual y en la carga masiva.

El DNI, sin embargo, no es un dato descriptivo más: es la **clave de identidad del alumno** en tres
flujos del sistema.

- **Alta individual**: se usa para detectar duplicados y para reactivar alumnos dados de baja.
  `personas.dni` tiene índice único.
- **Carga masiva**: cada fila del Excel se clasifica como nuevo / promovido / repitente / retroceso /
  reactivado comparando su DNI con lo que ya existe. Es lo que hace posible el pase de año.
- **Creación de evaluaciones**: `POST /evaluaciones` localiza al alumno por DNI, y el formulario se
  niega a evaluar a un alumno sin DNI.

Hacer el campo realmente opcional obliga a modificar los tres flujos en backend y frontend (2 a 3
días) y no resuelve cómo reconocer al mismo alumno de un año al otro. Agregar una clave alternativa
por nombre, apellido y fecha de nacimiento suma 1 a 2 días más y trae falsos positivos propios.

La columna `personas.dni` es `text` sin restricción de formato ni de largo, y ni el backend ni el
frontend validan que el valor sea numérico.

## Decisión

Cuando un alumno no tiene DNI disponible, el colegio carga en el campo DNI un **identificador
interno** armado con las letras del jardín, la fecha de nacimiento como día, mes y año de dos cifras
cada uno, y la inicial del nombre seguida de la inicial del apellido; todo junto, sin espacios ni
guiones y en mayúsculas. Ejemplo: `SM040621RM` (jardín SM, nacido el 4 de junio de 2021, Ramiro
Martínez). El colegio puede armarlo sin llevar ningún registro.

Reglas del procedimiento, acordadas con la Fundación y fuera de la app:

1. Las letras del jardín las asigna la Fundación PADI, para que dos colegios no usen las mismas.
2. El identificador acompaña al alumno todos los años, aunque cambie de colegio dentro de PADI.
3. Cuando se consigue el DNI real, se reemplaza desde la edición del alumno antes de la siguiente
   importación masiva, para que el Excel ya lleve el DNI.

Lo que cambia en la app (solo frontend):

- La distinción es **puramente sintáctica**: un valor con letras es un identificador interno; un
  valor de solo dígitos es un DNI real (`src/utils/dni.ts`). **No se interpreta el prefijo**: la
  asociación alumno → colegio ya existe en `estudiantes.escuela_id` y no se duplica en el código.
- El frontend normaliza lo que se escribe (mayúsculas, solo letras y dígitos) tanto en el
  formulario individual como al leer el Excel de carga masiva, para que `sm-040621rm` y
  `SM040621RM` sean el mismo alumno.
- El alta individual y la plantilla de carga masiva muestran cómo armar el identificador: la
  plantilla titula la columna "DNI / ID interno", lleva una nota en el encabezado y una hoja
  "Instrucciones". La convención está en un solo lugar (`ID_INTERNO` en `src/utils/dni.ts`).
- La carga masiva rechaza identificadores repetidos dentro del mismo archivo antes de enviar; el
  alta individual ya rechazaba un identificador existente por el índice único. De paso, un DNI real escrito como `45.123.456` queda como `45123456`.
- Las pantallas y el exporte a Excel rotulan el valor como "ID interno" cuando tiene letras, para
  que nadie lo copie a un documento oficial creyendo que es un DNI.
- El backend no cambia.

## Alternativas Consideradas

1. **DNI opcional (nullable) de verdad** — la columna ya lo admite, pero exige cambiar la validación
   del alta, la búsqueda por DNI en alta individual y masiva, y que las evaluaciones localicen al
   alumno por id en lugar de DNI. Además, sin DNI cada importación crea alumnos nuevos: el pase de
   año deja de funcionar para esos chicos.
2. **Clave alternativa por nombre + apellido + fecha de nacimiento + escuela** — reconoce al alumno
   sin DNI, pero un acento distinto o un error de tipeo genera un duplicado, y no hay forma de
   fusionar alumnos con evaluaciones cargadas.
3. **Identificador generado por la app** (aleatorio, ej. `SD-7K3M9Q2A`) — evita que el colegio
   administre números, pero hoy la plantilla de carga masiva se descarga vacía, así que el colegio
   no tendría cómo recuperar los códigos para el Excel del año siguiente. Queda como evolución
   posible junto con una planilla precargada.
4. **DNI numérico inventado** — descartado: puede coincidir con el DNI real de otro chico y la carga
   masiva lo tomaría como la misma persona, cambiándole la sala y la escuela sin avisar.

## Consecuencias

### Positivas

- Cambio de menos de un día, solo en frontend. Los tres flujos que dependen del DNI siguen
  funcionando sin modificaciones.
- El pase de año por carga masiva sigue reconociendo al alumno mientras el Excel lleve el mismo
  identificador.
- No se incorporan datos sensibles nuevos; de hecho se cargan menos.

### Negativas / Compromisos

- La columna `dni` contiene valores que no son documentos. Cualquier consumidor futuro de la base
  (reportes, integraciones) debe conocer esta convención; este ADR es la referencia.
- La unicidad de las letras por colegio depende de un procedimiento manual de la Fundación, no de
  la app. Si dos colegios usan el mismo prefijo, la carga masiva "traslada" a un alumno en lugar
  de crear otro. La previsualización lo muestra como promovido o repitente en vez de nuevo, y esa
  es la señal para detectarlo.
- Dos alumnos del mismo jardín con igual fecha de nacimiento e iniciales generan el mismo
  identificador. El alta individual lo rechaza como duplicado y la carga masiva lo detecta dentro
  del archivo; contra un alumno ya cargado, la importación lo tomaría como la misma persona. Es tan
  improbable que no se explica a los colegios; si ocurre, se cambia una letra del código.
- Evoluciones posibles si el procedimiento manual resulta frágil: planilla de carga masiva
  precargada con los alumnos actuales y sus identificadores, y una columna `codigo` en `escuelas`
  para que la app valide el prefijo.
