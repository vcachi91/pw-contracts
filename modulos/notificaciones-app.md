# Notificaciones de la app: la campanita

Todo lo que le pasa a la cuenta del empleado queda guardado en la campanita de
la app y, además, le llega como push. Pedido del dueño el 04/10/2026.

**Repos:** pw-adminbackend (escribe avisos y manda los push), pw-appbackend
(lista, cuenta y marca leídas) y pw-mobileapp (la campanita).

## 1. Reglas del dueño (04/10/2026)

| # | Regla |
|---|---|
| 1 | Cualquier actividad sobre la cuenta del empleado genera un aviso: cuenta bancaria verificada, solicitud aprobada, campañas push, etc. |
| 2 | El aviso llega como **push** y queda **guardado** en la campanita. |
| 3 | La campanita muestra un **número rojo** con los avisos sin leer. |
| 4 | Al abrir la campanita, el número **desaparece**. |
| 5 | **Solo avisos nuevos:** nada anterior al 04/10/2026 se manda ni cuenta como sin leer. |

## 2. Cómo funciona

La bandeja es la tabla `push_notifications` (base `app`). **Escribir una fila
es avisar:**

1. la fila aparece en la campanita, con `status = 'unread'`;
2. `notificaciones:enviar` (pw-adminbackend, cada minuto) la manda como push y
   le pone `push_sent_at`.

Quien quiere avisar no necesita saber nada de Firebase. En pw-adminbackend se
usa `App\Services\Notificaciones\Bandeja::avisar()`. pw-appbackend no tiene
Firebase: inserta la fila y el envío lo hace el otro backend.

| Columna | Uso |
|---|---|
| `title`, `body` | El texto. `varchar(255)` las dos. |
| `type` | **Severidad**, no categoría: `success`, `error`, `info`, `warning`. |
| `call_to_action_type` | La **categoría** del aviso (ver §3). La app la usa para saber qué abrir. |
| `call_to_action_id` | Id relacionado: la orden, la cuenta, la campaña. |
| `status` | `unread` / `read`. |
| `push_sent_at` | Cuándo salió el push. Quien ya lo mandó por su cuenta (campañas) o no quiere push lo llena al insertar. |

`notificaciones:enviar` solo mira los últimos 2 días, intenta cada aviso una
sola vez (con o sin token) y marca `push_sent_at` **antes** de mandar: es
preferible un push que no salió a uno que sale dos veces. El aviso queda en la
campanita aunque el push falle. Reemplaza a `referrals:push`.

## 3. Qué avisa hoy

| Categoría (`call_to_action_type`) | Cuándo | Push | Quién lo escribe |
|---|---|---|---|
| `bank_account_verified` | Operaciones verifica una cuenta (página Accounts o ficha del empleado) | Sí | pw-adminbackend |
| `bank_account_pending` | El empleado agrega una cuenta desde la app | No (lo acaba de hacer él) | pw-appbackend |
| `order_verified` | Una solicitud pasa a `verified` | Sí | pw-adminbackend |
| `order_rejected` | Un admin cancela una solicitud (`cancel_reason = admin_rejected`) | Sí | pw-adminbackend |
| `account_approved` | El empleado pasa a `active` | Sí | pw-adminbackend |
| `campaign` | Cada destinatario de una campaña push, haya recibido el push o no | El de la campaña | pw-adminbackend |
| `referral_activated`, `referral_reward` | Programa de referidos | Sí | pw-appbackend |

El WhatsApp de solicitud aprobada y de cuenta aprobada **sigue saliendo igual**:
esto se suma, no lo reemplaza.

**Todavía no avisan:** adelantos extraordinarios (aprobado, rechazado), cambio
de salario o de empresa, y la solicitud recién creada. Se agregan con una
llamada a `Bandeja::avisar()` en el lugar donde ocurre.

## 4. API de la app (pw-appbackend)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/api/v1/notifications` | La lista, paginada de a 10, lo más nuevo arriba. Cada fila trae `status`. Ya existía. |
| GET | `/api/v1/notifications/unread-count` | `{ data: { unread: n } }` |
| POST | `/api/v1/notifications/read` | Marca **todas** como leídas. |
| GET | `/api/v1/home` | Suma `contents.notifications.unread`, para que el número esté al abrir la app sin otro pedido. |

## 5. App (pw-mobileapp) — sale con la próxima versión

- **Número rojo** sobre la campanita de la barra inferior (`99+` como tope).
- Se actualiza al entrar al dashboard, cada vez que se recarga el Home y cuando
  llega un push con la app abierta.
- **Al abrir la campanita** se carga la lista y recién entonces se marcan como
  leídas. Las que estaban sin leer quedan resaltadas en esa vista, con fondo
  suave y un punto rojo, para que se note cuáles son las nuevas.
- El controller de la campanita se crea solo en el dashboard, con sesión
  iniciada: pedir el número sin token dispara el cierre de sesión forzado.
- La versión publicada (1.3.16) ya muestra la lista, así que ve los avisos
  nuevos; el número rojo y el "leído" llegan con la versión nueva.

## 6. Estado (04/10/2026)

Backends desplegados y probados. Los 18 avisos anteriores se marcaron como
enviados y leídos (regla 5). Prueba real con el usuario de prueba del dueño
(#7882) en el emulador: aviso creado 01:13:33, push enviado 01:14:01, número
rojo en 1, y al abrir la campanita pasó a `read` y el número desapareció.
