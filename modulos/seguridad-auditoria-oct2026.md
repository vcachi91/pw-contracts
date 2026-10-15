# Auditoría de seguridad de octubre 2026 — qué se cerró y qué falta

Claude Security recorrió los cinco repos el 14 y 15 de octubre de 2026 y dejó
113 hallazgos: 1 crítico, 14 altos, 53 medios y 45 bajos. Los graves se
verificaron contra el código desplegado antes de tocar nada; no eran teóricos.

Este archivo es el estado, no el informe. El informe vivo está en
`https://138.197.97.96:3004/<token>/`, con el token en
`/home/dev/payway-security-runner/web_token.txt`.

---

## Lo cerrado el 15/10/2026

| Qué | Dónde | Commit |
|---|---|---|
| Webhook sin validar la firma de Twilio y filtrando las credenciales | `pw-appbackend` | `1c00e08` |
| Contraseñas en texto plano en el log | `pw-appbackend` | `cbaffee` |
| `123456` seguía entrando por `login()` | `pw-appbackend` | `cbaffee` |
| Las cuentas nuevas nacían con `123456` | los tres backends | `081eca1`, `56b6fb7`, `36cf5ce` |
| `login/check` revelaba cuentas desde cualquier país | `pw-appbackend` | `2190242` |
| Firma legal sobrescribible | `pw-appbackend` | `1bd6798` |
| Inyección SQL en el filtro de audiencias | `pw-adminbackend` | `068b1a2` |
| 32 rutas de escritura con permiso de solo lectura | `pw-adminbackend` | `d1a34ec` |
| Desactivado volvía a entrar, y el scopeado se ampliaba solo | `pw-hrbackend` | `a4c9e5f` |
| Cabeceras fijas de `/v1/admin/*` | `pw-hrbackend` | `ffb29d2` |

Respaldos de cada pieza en `/root/backup-seguridad-*` de su servidor.

### El webhook, que era el crítico

`POST /api/v1/whatsapp/webhook` es público porque lo llama Twilio, pero no
comprobaba que **quien** llama sea Twilio. `MediaUrl0` llegaba en ese cuerpo sin
firmar y después se descargaba con `Http::withBasicAuth(SID, TOKEN)` sin mirar
el host: quien apuntara esa URL a un servidor suyo recibía las credenciales que
controlan el número de WhatsApp de Payway, y de paso podía hacer que el servidor
consultara direcciones internas.

Ahora `handle()` valida `X-Twilio-Signature` con `Twilio\Security\RequestValidator`
y las credenciales solo salen hacia `api.twilio.com` por https, sin seguir
redirecciones.

> **Cómo volver atrás sin desplegar código:** `WA_WEBHOOK_FIRMA_ESTRICTA=false`
> en el `.env` de `pw-backend`. Con `false` valida y anota en el log lo que
> habría rechazado, pero deja pasar. Se desplegó así a propósito; se activó el
> rechazo tras 8 peticiones reales validadas, y se confirmó con tráfico real
> después (200 en las legítimas, 403 solo en las forjadas de prueba).

### La contraseña automática

`123456` estaba escrita en seis sitios. El arreglo del 15/10 en `loginCheck`
cubría lo que **muestra** la app, pero `login()` seguía entregando token a quien
pegara directo a la API: 125 cuentas activas estaban así. Ahora:

- `login()` trata esa clave igual que no tener clave, con la constante
  `AuthService::CONTRASENA_AUTOMATICA`.
- Las cuentas nuevas nacen con `password = NULL`, así que caen en
  `sin_contrasena` y la app las manda a crear la suya.
- `RegisterRequest` y las tres reglas de `completeRegistration` la rechazan.

Las 69 cuentas no activas que la tenían quedaron en `NULL`.

### Dos cosas que la auditoría dijo mal

**La cadena de escalada del panel Enterprise no era explotable.** El middleware
leía `getallheaders()['admin_key']` y nginx descarta las cabeceras con guion
bajo (`underscores_in_headers off`, el valor por defecto), así que esas cuatro
rutas devolvían 401 siempre. Al reescribirlo se volvieron alcanzables con
`admin-key`; quedaron apagadas tras `HR_ADMIN_ENDPOINTS_ENABLED`.

**Eran 32 rutas de permisos, no 13.** El barrido completo del archivo de rutas
encontró más del mismo tipo. Nadie tiene asignados `admin_view` ni
`admin_finance`, así que el riesgo real era `admin_ops` escribiendo sin
permiso; se le agregaron `companies.create` y `companies.update`, que sí usa.

---

## Lo que falta

### Necesita decisión del dueño

- **Rotar el token de Twilio.** No hay forma de saber si ya se filtró. Agendado
  de madrugada, que es cuando el corte de unos minutos no se nota.
- **Anular la clave de DeepSeek** en su consola. Viaja dentro de todas las
  versiones publicadas de la app hasta la 1.3.17.
- **El `com_admin` de cada empresa nueva nace con `123456`** y la clave va por
  correo en texto plano (`welcome.blade.php` y `new-employee-created.blade.php`
  de `pw-adminbackend`). Arreglarlo cambia lo que recibe la empresa cliente al
  darse de alta, así que el texto nuevo se acuerda antes.
- **La página pública `/auth/register` del panel Admin.** Crea una fila en
  `admin_users` sin ninguna autenticación. Queda acotado porque nace inactiva y
  sin rol, y el login rechaza inactivos — ninguna cuenta se creó nunca por ahí.
  Lo que sí permite es apartar correos y teléfonos, que tienen índice único.

### Necesita diseño, no es un parche

- **El token del link de firma sigue siendo el teléfono en base64**, o sea
  adivinable. Lo que se cerró el 15/10 evita el daño real (no se puede pisar una
  firma existente, la imagen se valida, se quitó la ruta duplicada), pero el
  token sigue siendo forjable para alguien que todavía no firmó.

  Hacerlo bien pide una **tabla de tokens de un solo uso**: el link se genera en
  DOS repos —`pw-appbackend` (el bot) y `pw-adminbackend` (el panel)— que no
  comparten código ni `APP_KEY`, así que un HMAC obligaría a un secreto
  compartido en dos `.env`. Una tabla encaja con cómo se comunican estos
  backends y da vencimiento y un solo uso de verdad. Es cambio de esquema que
  toca dos backends: se acuerda acá antes de escribirlo.

### Menores

- Las transacciones gratis del mes se revisan al **armar el resumen** pero no al
  confirmar (`Gratuidad::decidir` solo se llama en `createRequestSummary`), así
  que armando varios resúmenes antes de confirmar se usan de más. Hoy no afecta
  a nadie: nadie tiene gratis asignadas.
- Los 125 activos **conservan el hash de `123456`** guardado. Ya no sirve para
  entrar; conviene ponerlo en `NULL` cuando se les avise.
- `config/sanctum.php` de `pw-hrbackend` tiene `expiration => null`: los tokens
  del panel Enterprise no vencen nunca. Ponerle vencimiento cierra la sesión de
  quien tenga una vieja, así que va coordinado.
- Siguen las 5 órdenes con filas `requested` duplicadas de antes del candado del
  22/09, y el backfill de cédulas fallando por cuota de Gemini.
