# Migrar OTP de SMS a WhatsApp — Brief para Backend

> La app móvil (`pw-mobileapp`) **no envía** el OTP, solo lo pide y lo verifica.
> Hay que cambiar **únicamente el canal de entrega** del código (SMS → WhatsApp) en el
> backend. La lógica de generar/guardar/validar el código **NO cambia**, y el
> **contrato de la API debe quedar igual** para no romper la app.

## Proveedor (elegir uno)

- **Opción A — WhatsApp Cloud API (Meta) directo:** más barato, requiere montar la
  integración con Meta. Control total.
- **Opción B — BSP/Proveedor (Twilio, Infobip, MessageBird, 360dialog):** más rápido
  de integrar (SDK listo), un poco más caro por mensaje. Para salir rápido.

## Requisitos previos (Meta, aplica a ambas opciones)

1. **Meta Business verificado** + **WhatsApp Business Account (WABA)**.
2. **Número de teléfono dedicado** registrado en la WABA (no puede ser un número que ya
   esté en la app de WhatsApp normal).
3. Crear y **aprobar una plantilla categoría `AUTHENTICATION`** con:
   - Componente de **código de un solo uso** (la variable es el código).
   - Botón **"Copiar código"** (`copy_code`) → es lo que el usuario toca para copiar y
     pegar en la app.

## Implementación

En los endpoints que **hoy disparan el SMS**, reemplazar el envío por una llamada a
WhatsApp que mande la **plantilla AUTHENTICATION** con el código como parámetro:

| Endpoint | Acción |
|---|---|
| `POST /auth/register` | OTP de registro |
| `POST /auth/forget-password` | OTP de recuperar contraseña |
| `POST /auth/resend-otp` | Reenvío de OTP |

- El número debe ir en **formato E.164** con código de país: `+507XXXXXXXX`
  (no solo el número local).
- **No tocar** la validación: `POST /auth/verify-otp` y
  `POST /auth/verify-forget-password-code` siguen igual.

## ⚠️ No romper el contrato con la app

- Mantener los `response_code` / `message` actuales.
- `GET /settings/public` debe seguir devolviendo `resend_otp_alive` (segundos) — la app
  lo usa para el contador de reenvío.

## Recomendaciones

- **Expiración del código: subir a 5–10 min.** (El usuario reportó "timeout muy poco";
  eso se arregla aquí, no en la app.)
- **Fallback a SMS:** si la entrega por WhatsApp falla o el número no tiene WhatsApp,
  reenviar por SMS. Muy recomendado para no perder usuarios.
- **Validar que el número tenga WhatsApp** antes de asumir entrega.
- **Costos:** las conversaciones de autenticación de WhatsApp se cobran por mensaje
  (tarifa de Panamá).

## Nota sobre iOS

El autocompletado nativo de iOS solo funciona con SMS, **no con WhatsApp** — por eso se
usa el botón **"Copiar código"** en la plantilla. La app ya está preparada para pegar y
auto-verificar al completar los 6 dígitos.

## Estado del lado app (ya hecho)

- Textos cambiados a "...a tu WhatsApp" (`assets/locales/es-PA.json`, `en-US.json`).
- UX del campo OTP: `autoFocus`, auto-verificación al completar, gating del reenvío
  (`lib/ui/auth/otp_form.dart`, `forgot_password_otp_screen.dart`,
  `components/default_pin_text_field.dart`).
