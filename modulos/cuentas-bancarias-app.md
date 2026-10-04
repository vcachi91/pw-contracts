# Cuentas bancarias desde la app: switch por empresa y por empleado

Quién puede **agregar** su cuenta bancaria desde la app, y qué pasa con la
cuenta que agrega. Acordado con el dueño el 03/10/2026 (`sprints/proximo-paquete.md` §1).

**Repos:** pw-appbackend (la regla y el endpoint), pw-adminbackend y
pw-adminfrontend (los dos switches), pw-mobileapp (el botón).

## 1. Reglas

| # | Regla |
|---|---|
| 1 | **Empresa:** encendido o apagado. Arranca **apagado** en todas, que es como funciona hoy. |
| 2 | **Empleado:** *según la empresa* (por defecto), *permitir* o *bloquear*. La excepción del empleado manda sobre la empresa. |
| 3 | Apagado = no puede **agregar** cuentas desde la app. Las que ya tiene se conservan y sigue retirando a ellas. |
| 4 | **Borrar** una cuenta desde la app no existe, esté como esté el switch. |
| 5 | Una cuenta agregada desde la app nace **sin verificar**. No aparece para retirar hasta que operaciones la verifica en el panel. |
| 6 | Si la cuenta **ya es de otra persona**, se rechaza con un mensaje de contáctenos. Dos usuarios no comparten cuenta. |
| 7 | Un despedido no agrega cuentas, diga lo que diga el switch. |

## 2. Datos (base `app`)

| Tabla | Columna | Valores |
|---|---|---|
| `companies` | `bank_accounts_enabled` | `0` (por defecto) / `1` |
| `companies` | `bank_accounts_toggled_at`, `bank_accounts_toggled_by` | cuándo y qué admin (`admin_users.id`) |
| `users` | `bank_accounts_mode` | `inherit` (por defecto) / `allow` / `block` |
| `users` | `bank_accounts_toggled_at`, `bank_accounts_toggled_by` | cuándo y qué admin |

**¿Puede agregar?** (`pw-appbackend/app/Support/CuentasBancarias.php`, único lugar):

```
despedido                → no
users.bank_accounts_mode = block → no
users.bank_accounts_mode = allow → sí
inherit                  → companies.bank_accounts_enabled de su empresa
```

Si las columnas todavía no existen (backend desplegado antes que el SQL), la
respuesta es **no**: se comporta como hoy.

## 3. API de la app (pw-appbackend)

**`can_add`** — la app lo lee para mostrar u ocultar "Agregar cuenta". Las apps
viejas lo ignoran.

- `GET /api/v1/accounts/saved-accounts-by-user` → `can_add` (bool) arriba, tanto
  en el 200 como en el 404 de "no tiene cuentas verificadas". Agrega también
  `pending` (cuántas cuentas suyas esperan verificación), para que la app pueda
  decir "tu cuenta está en revisión" en vez de mostrar una lista vacía.
- `GET /api/v1/home` → `contents.bank_accounts.can_add`.

**`POST /api/v1/accounts/save-bank-account`**

| Caso | Respuesta |
|---|---|
| No puede agregar | 403 `error_code: bank_accounts_disabled` · "Por ahora no puedes agregar cuentas desde la app. Escríbenos y te ayudamos: https://wa.me/50769958313" |
| La cuenta (banco + número) ya es de otra persona | 409 `error_code: account_in_use` · "Esta cuenta ya está registrada a nombre de otra persona. Si es tuya, escríbenos: https://wa.me/50769958313" |
| Ya la tiene él mismo | 400, mensaje en español |
| Nueva | 200 (como hoy), la cuenta queda con `verify_enterprise = 0` |
| Existía, pero ya no la usa nadie | Se le vincula y **vuelve a `verify_enterprise = 0`** |

Antes de este cambio, si la cuenta existía el backend se la vinculaba a quien
la escribiera **heredando la verificación**: un error de tipeo podía mandar un
adelanto a la cuenta de otra persona. Además el endpoint seguía abierto aunque
la app hubiera quitado el botón en marzo de 2026.

## 4. Solo se retira a cuentas verificadas

- **App:** `RequestController::isMyAccountId` exige `verify_enterprise = 1`. La
  app ya solo listaba las verificadas; esto cierra el camino de llamar a la API
  a mano con el id de una cuenta sin verificar.
- **Bot de WhatsApp:** **PENDIENTE DE DECISIÓN DEL DUEÑO.** Hoy el bot ofrece
  **todas** las cuentas del usuario: sin verificar, inactivas y hasta borradas.
  En los últimos 60 días salieron 11 adelantos de 5 personas por el bot a
  cuentas sin verificar. Mientras eso siga así, encender el switch permitiría
  agregar una cuenta por la app y retirar a ella por WhatsApp sin que nadie la
  verifique. El dueño pidió no tocar el flujo de WhatsApp, así que **el switch
  no se enciende en ninguna empresa hasta que decida**.

## 5. Panel admin

- **Empresa** (ficha de la empresa): tarjeta "Cuentas bancarias desde la app"
  con el switch. `PUT /api/v1/admin/companies/{id}/bank-accounts {enabled}`.
  Permiso `companies.toggle_bank_accounts`: super_admin y admin_ops.
- **Empleado** (edición de la ficha): selector *Según la empresa / Permitir /
  Bloquear*, que viaja en el `bank_accounts_mode` del guardado normal
  (`employees.update`). Muestra qué resulta hoy para esa persona.
- Los dos cambios quedan en la auditoría.

## 6. App (pw-mobileapp) — sale con la próxima versión

- "Agregar cuenta" se muestra solo si `can_add` es verdadero.
- La pantalla de agregar no corre desde marzo de 2026: probarla de punta a
  punta en un APK de release, con traba contra doble toque.
- Después de agregar: aviso de que la cuenta queda en revisión.
- Errores `bank_accounts_disabled` y `account_in_use`: mostrar el mensaje del
  backend tal cual, con el enlace de WhatsApp.

## 7. Orden de despliegue

1. SQL + pw-appbackend + pw-adminbackend + pw-adminfrontend. Todo apagado:
   nada cambia para nadie, y el endpoint de agregar queda cerrado.
2. Versión de la app.
3. Forzar actualización.
4. Encender por empresa o por persona — **después** de resolver §4 (bot).
