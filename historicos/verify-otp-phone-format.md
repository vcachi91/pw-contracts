# Bug `POST /auth/verify-otp`: el teléfono no se normaliza → "OTP does not matched" / "User not found"

> **Es un bug de backend.** La app no puede arreglarlo: manda un solo valor de teléfono, pero el
> backend lo necesita en dos formatos distintos al mismo tiempo.

## Síntoma
El usuario ingresa el OTP correcto en el registro y siempre sale "código inválido o expirado".

## Evidencia (logs reales de la app)
La app loguea el request y la respuesta de `verify-otp`:

```
# Enviando el teléfono en 8 dígitos:
verify-otp → request={phone: 67854781, otp: 354932}
verify-otp ← error="OTP does not matched"        # encuentra al user, pero el OTP no valida

# Enviando el teléfono en E.164:
verify-otp → request={phone: +50767854781, otp: 354932}
verify-otp ← error="User not found"              # ni siquiera encuentra al user
```

## Causa raíz: el teléfono está guardado en formatos inconsistentes
Confirmado mirando la BD:
- **`users.phone` = `50767854781`** (con prefijo 507)
- **tabla del OTP `.phone` = `67854781`** (8 dígitos, sin 507)
- **Twilio Verify**: la verificación se creó al enviar el OTP con el número en E.164 **`+50767854781`**
  (formato con el que `register`/`resend-otp` mandan a WhatsApp).

`verify-otp` usa **el mismo string de teléfono** para (a) buscar al user, (b) buscar/validar el OTP y
(c) el `verificationCheck` de Twilio — pero cada uno está en un formato distinto. Por eso **ningún
valor único** que mande la app cuadra todo:

| App envía | User lookup (`users.phone=50767854781`) | OTP / Twilio | Resultado |
|---|---|---|---|
| `67854781` | match (bidireccional) | Twilio espera `+50767854781` | `OTP does not matched` |
| `+50767854781` | NO match (BD no tiene el `+`) | — | `User not found` |

## Fix pedido (backend)
Normalizar el teléfono **dentro de `verify-otp`** antes de cada uso, igual que ya se hace en
`register`:
1. **Unificar el formato de guardado**: que `users.phone` y la tabla del OTP usen el MISMO formato
   (recomendado: solo 8 dígitos, o solo E.164 — pero uno solo, consistente). Hoy difieren
   (`50767854781` vs `67854781`).
2. En `verify-otp`, derivar el formato que necesita cada paso a partir del teléfono recibido:
   - Lookup de user / OTP en BD → el formato en que están guardados.
   - `verificationCheck` de Twilio → **E.164 `+507XXXXXXXX`**.
3. Idealmente hacer el lookup **bidireccional** (con/sin 507) como ya hiciste en `register`, para
   tolerar data legacy.

## Contrato con la app (NO cambia)
La app manda el teléfono en **8 dígitos** en `register`, `resend-otp` y `verify-otp` (es el contrato:
"la app manda local, el backend normaliza"). El backend es quien debe poner el `+507` para Twilio.
No pidan que la app mande `+507` en verify-otp: rompe el lookup del user (`User not found`).

## Estado app
- `auth_controller.checkPhoneVerification()`: envía `phone` en 8 dígitos + `otp` como **string**
  (se corrigió un `int.parse` que perdía ceros a la izquierda). Logging temporal de request/response
  activo para diagnosticar (`🔐 verify-otp → / ←`).
