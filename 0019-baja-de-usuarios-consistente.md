---
title: "ADR-0019: Baja de usuarios consistente entre Supabase Auth y la base de la aplicación"
nav_order: 19
---

# ADR-0019: Baja de usuarios consistente entre Supabase Auth y la base de la aplicación

## Contexto

La identidad de un usuario vive en dos lugares: la cuenta en Supabase Auth (quién puede entrar) y el
perfil en la tabla `usuarios` (quién es en la aplicación: nombre, rol). De ese perfil cuelgan
`personas` → `profesores` → asignaciones a escuelas y aulas → evaluaciones. El `id` es el mismo en
Auth, `usuarios` y `profesores` por convención del sistema.

Desde mayo de 2026 la baja de un usuario ("Eliminar" en el panel de Equipo PADI) era una revocación:
borraba la cuenta de Auth, intentaba borrar la fila de `usuarios` y, si eso fallaba, solo lo
registraba en el log y respondía `200 "Usuario eliminado"`. `personas` y `profesores` no se tocaban
para no perder las evaluaciones históricas del docente.

En noviembre de 2026 la Fundación eliminó una docente y no pudo volver a invitarla con el mismo
correo. El diagnóstico mostró tres problemas encadenados:

1. Un perfil de `usuarios` sin cuenta de Auth queda **"zombi"**: nadie puede entrar con él, pero el
   correo sigue ocupado (`UNIQUE`) y bloquea el re-alta.
2. "Reenviar invitación" sobre ese perfil (panel o autoservicio) llamaba a `inviteUserByEmail`, que
   **creaba una cuenta de Auth con otro `id`**. La docente activaba la cuenta y el login respondía
   "Perfil de usuario no encontrado", porque el perfil tenía el `id` viejo.
3. El panel decía "eliminado" aunque el perfil siguiera presente, así que nadie sabía que la baja
   había quedado a medias. Además el profesor seguía figurando como asignado en Aulas y Escuelas.

## Decisión

La baja define un **estado final** y no reporta éxito hasta llegar a él:

- `deleteUser`: revoca el acceso en Auth (si la cuenta ya no existe, continúa: la baja es
  repetible) y, en **una sola transacción**, cierra las asignaciones activas del docente
  (`fecha_fin`), desvincula la persona (`personas.usuario_id = null`), quita el registro de
  encargado y borra el perfil. Si la transacción falla, el endpoint responde error, no `200`.
  `personas`, `profesores` y evaluaciones se conservan como histórico.
- `createUser`: si el alta choca con un perfil que tiene el mismo correo y ese perfil no tiene
  cuenta de Auth (resto de una baja incompleta anterior), lo limpia con la misma rutina y reintenta
  el alta. Si el perfil sí tiene cuenta, falla como antes.
- `reenviarInvitacion` (panel y autoservicio): si el perfil no tiene cuenta de Auth, **no invita**
  (devuelve `sin_cuenta`). El panel muestra "Eliminalo y volvé a crearlo"; el autoservicio responde
  el mismo mensaje neutro de siempre.

La regla general: cuando una operación toca dos almacenes, se define el estado final, la operación
es idempotente (se puede repetir hasta llegar) y el éxito se reporta solo al llegar.

## Alternativas Consideradas

- **Baja lógica real** (`usuarios.eliminado_at` + índice único parcial sobre `email`): más limpia a
  largo plazo, pero exige migración, tocar login y listados, y una pantalla de "reactivar".
  Descartada por ahora; esta decisión no la impide.
- **Reparar automáticamente al reenviar** (crear la cuenta de Auth con el mismo `id`): la API de
  administración de Supabase no permite fijar el `id` al crear. Descartada.
- **Mantener la revocación y solo mejorar el mensaje**: no resuelve el correo bloqueado ni la cuenta
  con `id` distinto. Descartada.

## Consecuencias

### Positivas

- Eliminar y volver a crear un usuario con el mismo correo funciona, y limpia solo los restos de
  bajas anteriores.
- Ya no se pueden crear cuentas de Auth sin perfil desde "Reenviar invitación".
- El panel informa cuando la baja no se completó; la operación se puede repetir sin daño.
- Un docente dado de baja deja de figurar como activo en Aulas y Escuelas; su histórico queda.

### Negativas

- Un docente dado de baja y vuelto a crear recibe un `profesor` nuevo: sus asignaciones no se
  heredan (se cargan de nuevo) y sus evaluaciones anteriores quedan bajo el profesor viejo, que ya
  no aparece en listados. Es el mismo criterio de "baja" que para un cambio de rol.
- La reparación de perfiles zombi existentes antes de este cambio se hizo a mano en la base.
