# Estado del empleado: suspensión temporal y comentario

Pedido del dueño, 19/10/2026. Nació de un caso real: un empleado dado de baja
el 14/10 se volvió a registrar desde la app el 17/10 y quedó otra vez en la
cola de aprobación, porque la API de la app tomaba `inactive` siempre como
"registro a medias".

Toca cuatro repos: `pw-adminbackend` (dueño del estado), `pw-adminfrontend`
(la ficha), `pw-appbackend` (lo que ve el empleado) y `pw-hrbackend` (no pisa
el estado). La app móvil no cambia: recibe los mismos códigos que ya conoce,
con otro texto.

## 1. Estados de `app.users.status`

| Valor | Qué significa | Entra a la app |
|---|---|---|
| `active` | Aprobado y trabajando | sí |
| `pending_approval` | Terminó el registro, espera al admin | no ("en revisión") |
| `pending_profile` | Empezó el registro, no lo terminó | no ("termina tu registro") |
| `inactive` **sin** fila en `company_users` | Empezó el registro, OTP sin verificar | no, puede retomar el registro |
| `inactive` **con** fila en `company_users` | **Dado de baja** por la empresa o por Payway | no ("tu cuenta fue desactivada"); **no puede volver a registrarse** |
| `suspended` | Suspendido **por un tiempo** desde el panel Admin | no ("suspendida hasta el dd/mm/aaaa"); no puede volver a registrarse |
| `dismissed` | Despedido: cuenta congelada | entra, ve historial, no tiene saldo |

La regla "inactive con empresa = baja" vive en `App\Support\EstadoCuenta`
(`pw-appbackend`) y la usan `register`, `login`, `login/check` y el middleware
`EnsureUserIsActive`.

## 2. Columnas nuevas en `app.users`

Migración `2026_10_19_100000_add_suspension_to_users` (en `pw-adminbackend`;
misma base que lee `pw-appbackend`).

| Columna | Tipo | Para qué |
|---|---|---|
| `suspended_until` | date, null | **Último día suspendido** (inclusive). Solo tiene valor con `status = suspended`. |
| `status_comment` | varchar(500), null | Comentario opcional del último cambio a `inactive`, `suspended` o `dismissed`. Al volver a `active` se limpia. |
| `status_changed_at` | timestamp, null | Cuándo fue el último cambio de estado desde el panel o por el comando. |
| `status_changed_by` | bigint, null | `admin_users.id` que lo hizo; `null` si fue el comando. |

El historial completo (todos los cambios, con comentario) queda en
`activity_log` porque `status`, `suspended_until` y `status_comment` están en
`auditFields` del modelo `User` del Admin API.

## 3. Admin API (`pw-adminbackend`)

- `PUT /admin/employee/update/{id}` y `PUT /admin/employee/status-change`
  aceptan `status=suspended`, `suspended_until` (fecha) y `status_comment`
  (hasta 500 caracteres). Suspender sin fecha → 422 `suspended_until_required`;
  fecha ya pasada → 422 `suspended_until_past`.
- `EmployeeController::aplicarCambioDeEstado()` es el único lugar que escribe
  las cuatro columnas. Una suspensión vigente se puede re-fechar o re-comentar
  mandando el mismo `status=suspended` con otra fecha o comentario.
- Espejo en `hr.employees.status`: 1 solo si `active`; `suspended`, `inactive`
  y `dismissed` → 0.
- Comando `employees:reactivar-suspendidos` (scheduler, cada 10 minutos):
  `status = suspended` y `suspended_until < hoy` → `active`, fecha en null,
  comentario "Reactivado automáticamente al terminar la suspensión (hasta …)",
  `status_changed_by = null`, y `hr.employees.status = 1`.
- La ficha (`GET /admin/employee/{id}`) y el listado devuelven las columnas
  nuevas tal cual (`toArray()`); el listado filtra con `status=suspended`.

## 4. Panel Admin (`pw-adminfrontend`)

En la ficha del empleado, los dos selectores de estado ofrecen **Suspendido
(por un tiempo)**. Al elegirlo aparece el bloque "Suspendido durante" con
botones 7 / 15 / 30 días, 1 / 3 / 6 meses y Personalizado (fecha libre, no
anterior a hoy); el campo guarda el **último día** suspendido y lo dice.
Al elegir Inactivo, Suspendido o Despedido aparece el campo **Comentario
(opcional)**. Todo viaja en el mismo `PUT` de "Guardar" que el resto de la
ficha. La cabecera muestra el estado, "hasta dd/mm/aaaa" si está suspendido y
el último comentario. El listado tiene el badge ámbar "Suspendido" y el filtro.
Etiquetas y colores en `lib/estado-empleado.ts`.

## 5. API de la app (`pw-appbackend`)

- `login/check` y `login`: `suspended` → código `en_aprobacion` (el mismo que
  la app ya maneja como "no puede entrar") con el texto "Tu cuenta está
  suspendida hasta el dd/mm/aaaa. Si tienes dudas, contacta a tu empresa."
  Dado de baja → mismo código con "Tu cuenta fue desactivada por tu empresa.
  Si crees que es un error, contáctala."
- `register`: un dado de baja o suspendido recibe 422 con ese mismo texto y
  **no** retoma el registro ni recibe OTP. Antes se le reenviaba el OTP y
  terminaba en `pending_approval`.
- `EnsureUserIsActive` (toda la API autenticada): `suspended` → 403 con el
  texto de suspensión.

## 6. Enterprise API (`pw-hrbackend`)

Al sincronizar un empleado desde Enterprise, el estado de la app solo se
escribe si **no** es `suspended` ni `dismissed`: una suspensión o un despido
puestos desde el panel Admin no se pisan desde RH. Enterprise no suspende; si
hace falta, es un pedido aparte.

## 7. Lo que no se hizo a propósito

- No hay aviso al empleado (push/WhatsApp) al suspenderlo ni al reactivarlo.
- Enterprise no ve el comentario ni la fecha de suspensión.
- `dismissed` no cambió: sigue siendo "cuenta congelada", entra y ve historial.
