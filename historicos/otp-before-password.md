# OTP antes de la clave: registrar sin contraseña y establecerla JUSTO DESPUÉS del OTP — Brief para Backend

> **⚠️ Cambio de diseño (2026-06-08):** se descartó la idea previa de "establecer la clave recién al
> ser aprobado". El flujo definitivo es más simple: **la contraseña se pone inmediatamente después de
> verificar el OTP, sin esperar la aprobación.** Si ya empezaste a implementar lo anterior (forget-
> password para usuario aprobado sin clave), **eso ya NO hace falta.**

> **Objetivo de producto:** que el usuario **no cree la contraseña al inicio del registro**. Llena su
> info → verifica OTP → **ahí mismo crea su contraseña** → queda "en revisión" (pending_approval).
> Menos fricción al arrancar, y la clave solo existe para quien ya verificó su número.

---

## Flujo nuevo (extremo a extremo)

1. **Registro** (`POST /auth/register`): el usuario manda su teléfono **sin `password`**. Se crea el
   `user` (sin contraseña) y se envía el OTP por WhatsApp.
2. **OTP** (`POST /auth/verify-otp`): igual que hoy → devuelve token Sanctum. (Sin cambios.)
3. **Crear contraseña + completar registro** (`POST /auth/complete-registration`, con ese token):
   la app ahora manda **`password`** junto con toda la info (nombre, cédula, salario, banco,
   imágenes, firma). El backend **persiste la contraseña** en el `user` y lo deja `pending_approval`.
4. **Aprobación:** un admin aprueba en el panel → `active`. El usuario **ya tiene su contraseña**, así
   que entra con login normal (teléfono + clave). No hay paso extra de "establecer clave".

---

## Cambios pedidos al backend

### 1) `POST /auth/register` — `password` opcional / nullable
- `password` y `password_confirmation` pasan a ser **opcionales**. Si no vienen, crear el `user`
  **sin** contraseña (campo `password` NULL en BD) y enviar el OTP igual que hoy.
- **Compatibilidad:** si una app vieja todavía manda `password`, aceptarla como hoy (no romper).
- Mismo shape de respuesta (la app va al OTP igual).

### 2) `POST /auth/complete-registration` — recibir y GUARDAR la contraseña
- Este endpoint ahora **recibe `password`** (lo manda la app después del OTP) y debe **persistirlo**
  (hasheado) en el `user` que se creó sin clave en el paso 1. Es un `set` de la primera contraseña.
- El resto de la info (validación de `company_id`, `cedula`, `bank_accounts[]`, imágenes, firma,
  `detail[gross_salary]`, etc.) **sin cambios** respecto al contrato actual.
- Al terminar, el usuario queda **`pending_approval`** (igual que hoy; ver
  `registration-status-contract`: el 200 debe traer `data.user.status == "pending_approval"`).
- Validación sugerida: `password` requerido aquí (mín. 6 chars, con confirmación si la querés exigir).
  La app valida mínimo 6 y que coincidan antes de enviar.

### 3) Login de un usuario `pending_approval` (ya con contraseña)
- Como ahora el usuario **tiene contraseña aunque esté en revisión**, puede intentar `POST /auth/login`
  antes de ser aprobado. Mantener el comportamiento actual: responder de forma que la app muestre el
  estado real (p. ej. mensaje/estado "en revisión"); **no** tratarlo como credenciales inválidas.
  (La app hoy muestra el `message` del backend en un diálogo.)

---

## Resumen de impacto (qué cambia y qué NO)
- **Cambia:** `password` opcional en `/auth/register`; `/auth/complete-registration` ahora **setea** la
  contraseña.
- **NO cambia:** `verify-otp`, el shape de respuestas, el canal WhatsApp/Twilio, el resto de
  `complete-registration`, ni el login normal.
- **Descartado de la versión anterior:** NO se necesita habilitar forget-password para "primera clave"
  de un usuario aprobado. La clave ya se setea en el registro.
- **Sin migración de datos.** Cuentas existentes ya tienen clave; siguen igual. Las nuevas nacen sin
  clave en register y la reciben en complete-registration.

## Lado app (ya implementado en este repo, working tree)
- Registro: el último paso ya no pide contraseña (solo firma). `signUp()` manda solo `{phone}`.
- Tras `verify-otp` OK → nueva pantalla **CreatePasswordScreen** (crear + confirmar clave) →
  `completeRegistration()` envía la info **con `password`** → "en revisión".
- **Orden de despliegue:** el backend (puntos 1 y 2) va **primero**. Si la app deja de mandar
  `password` en register antes de que sea opcional → 422. Coordinar el cutover.

Relacionado: `docs/register-resume-unverified-backend.md` (register idempotente) y
`docs/verify-otp-phone-format-backend.md` (normalización de teléfono).
