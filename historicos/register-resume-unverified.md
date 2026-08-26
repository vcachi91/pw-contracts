# Registro: retomar si el teléfono existe pero NO está verificado — Brief para Backend

> Problema: si un usuario hace `POST /auth/register` (se crea el `user` + se envía OTP) pero
> **no completa el OTP**, su teléfono queda guardado. Al reintentar el registro con el mismo
> número, hoy el backend lo rechaza como **duplicado** y el usuario queda en un callejón sin
> salida ("limbo"). Queremos que **retome** la verificación en vez de bloquearse.

## Cambio pedido: hacer `POST /auth/register` idempotente para no verificados

| Caso | Comportamiento esperado |
|---|---|
| Teléfono **no existe** | Crear `user` + enviar OTP → **200** (igual que hoy) |
| Existe pero **NO verificado** (registro incompleto, sin OTP confirmado) | **Reenviar OTP** y responder **200 con el MISMO shape** que un registro nuevo (la app no nota diferencia y va directo a la pantalla de OTP) |
| Existe y **YA verificado / cuenta completa** | Responder **error** con un `message` reconocible (ver abajo) |

### Detalle del caso "no verificado" (el importante)
- Reutilizar el mismo `user` (no crear uno nuevo, no romper unicidad de `phone`).
- Regenerar/reenviar el código OTP por WhatsApp (mismo canal que el registro normal).
- Responder 200 con el mismo cuerpo que un alta nueva, para que el cliente haga `Get.to(OtpPage)`
  sin cambios. **Idempotente:** reintentar N veces solo reenvía el OTP.

### Detalle del caso "ya verificado"
- NO reenviar OTP (la cuenta ya existe y está completa).
- Devolver un error (status a tu criterio, p.ej. **409**) cuyo **`message` contenga la frase de
  invitación a iniciar sesión**, porque el cliente enruta por el texto del mensaje (no ve el
  status code). Mensaje canónico sugerido:

  > `"Este número ya tiene una cuenta activa. Inicia sesión."`

  El cliente detecta (case-insensitive) que el mensaje contiene **"inicia"/"iniciar"** + **"ses"**
  y, en ese caso, **redirige a la pantalla de Login** mostrando ese mismo mensaje.
  Para cualquier otro error de registro, el cliente solo muestra el mensaje (snackbar) como hoy.

## Por qué el cliente no puede distinguir solo
En el registro todavía no hay token, así que la app no puede consultar el estado del usuario.
Necesita que el propio `/auth/register` decida: **200 = retomar**, **error con "inicia sesión" =
ir a login**. El `507` (E.164) no tiene nada que ver: el duplicado es por el mismo número, no por
el prefijo (la app envía 8 dígitos; el backend normaliza a `+507XXXXXXXX`).

## Lado app (ya implementado en este repo)
- `auth_controller.signUp()`: en éxito → `OtpPage` (cubre el "retomar" automáticamente).
  En error, si el `message` parece "ya tiene cuenta → inicia sesión", redirige a `LoginScreen`;
  si no, muestra el snackbar como antes (sin regresión mientras el backend no implemente esto).
- Relacionado: se corrigió que el OTP de registro se enviaba con `int.parse` (perdía ceros a la
  izquierda y rompía Twilio Verify). Con eso, los huérfanos casi no se generan; este endpoint
  cubre los que ya existen y cualquier caso futuro.

## Limpieza recomendada (opcional)
- Job que purgue/expire `users` sin verificar tras X horas, para no acumular registros huérfanos.
