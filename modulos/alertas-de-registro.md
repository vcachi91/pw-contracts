# Alertas de registro fallido

Cuando un registro falla, en cualquier canal, queda en el log y le llega un
correo al dueño para atenderlo de inmediato. Pedido del dueño el 04/10/2026.

**Repos:** pw-appbackend y pw-adminbackend. La clase es la misma en los dos:
`App\Support\AlertaRegistro`.

## 1. Qué hace

`AlertaRegistro::fallo($canal, $paso, $error, $datos)`:

1. escribe una línea `[REGISTRO-FALLO] canal=… paso=…` en `storage/logs/laravel.log`;
2. manda un correo a **vcachi91@gmail.com** con canal, paso, hora, usuario,
   teléfono, empresa y el error.

El destinatario se puede cambiar con `REGISTRO_ALERTA_EMAIL` en el `.env` de
cada backend.

## 2. Reglas

| # | Regla |
|---|---|
| 1 | **Nunca rompe el registro.** Todo va en try/catch: si el correo no sale, el log igual quedó. |
| 2 | **No inunda.** Un correo por fallo distinto cada 10 minutos, y un tope de 30 por hora por backend. El log sí guarda cada caso. |
| 3 | **Nada sensible.** No viajan contraseñas, códigos, firmas ni fotos. En un error de validación van los campos que fallaron, nunca los valores. |
| 4 | **Solo avisa.** No cambia lo que ve el empleado ni lo que responde la API. |

## 3. Dónde avisa

| Canal | Qué fallo | Dónde está el aviso |
|---|---|---|
| `app` | Completar el registro (incluye fotos que no se pueden leer) | `AuthService::completeRegistration` |
| `app` | Error no controlado al pedir el código, verificarlo, buscar el pre-registro o guardar la firma | `bootstrap/app.php` de pw-appbackend |
| `whatsapp` | Completar el registro desde el bot (pasa por el mismo service) | `AuthService::completeRegistration` |
| `whatsapp` | Cerrar el registro del bot antes de llegar al service (bajar las fotos, crear el usuario) | `WhatsAppChatbotController::finalizeBotRegistration` |
| `kiosco` | Crear el empleado desde un pre-registro | `PreRegistrationController` (pw-adminbackend) |
| `kiosco` / `panel` | Error no controlado al guardar un pre-registro o al dar de alta un empleado | `app/Exceptions/Handler.php` de pw-adminbackend |

## 4. Qué NO avisa

- Que alguien abandone el registro a la mitad: eso no es un fallo. Se ve en la
  bandeja de registros incompletos.
- Respuestas inválidas en el bot (tres errores seguidos mandan al agente).
- Validaciones normales del panel o del kiosco (un 422 con su mensaje).
- Un código de verificación mal escrito.

## 5. Cómo buscar en el servidor

```sh
grep 'REGISTRO-FALLO' storage/logs/laravel.log | tail -20
```

## 6. Estado (04/10/2026)

Desplegado en los dos backends. Prueba: un aviso desde cada servidor, enviado
solo al correo del dueño; el mismo aviso repetido no generó un segundo correo,
y los campos `password` y `signature_img` de la prueba no salieron ni en el log
ni en el correo.
