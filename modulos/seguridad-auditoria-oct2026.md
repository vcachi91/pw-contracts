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
| El `com_admin` nacía con `123456` y la clave iba por correo | `pw-adminbackend` | `109e444` |
| Registro público de `admin_users` | `pw-adminbackend` | `c649b30` |
| La gratis del mes no se revisaba al confirmar | `pw-appbackend` | `e31f951` |
| Los tokens del panel Enterprise no vencían nunca | `pw-hrbackend` | `0167fc7` |
| El token del link de firma era el teléfono en base64 | `pw-appbackend`, `pw-adminbackend` | `e387db2`, `12bcc4a` |

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

### Solo lo puede hacer el dueño

- **Rotar el token de Twilio.** No hay forma de saber si ya se filtró. El
  arreglo del webhook cierra la fuga hacia adelante, pero no lo que ya pasó.
  Deja el WhatsApp mudo unos minutos, así que va de madrugada.
- **Anular la clave de DeepSeek** en su consola. Viaja dentro de todas las
  versiones publicadas de la app hasta la 1.3.17.

### Decidido no hacer, con motivo

- **La página `/auth/register` del panel Admin.** El endpoint ya se quitó
  (`c649b30`), y el enlace del login ya estaba oculto con `display: none`, así
  que nadie llega por la interfaz y la página apunta a una ruta que da 404.
  Recompilar el panel solo para borrar archivos muertos es riesgo sin beneficio.
  Los archivos quedan: `app/auth/register/page.tsx`,
  `components/auth/register-form.tsx`, `components/auth/actions/register.ts`.

### Menores que quedan

- Los tokens de firma gastados y vencidos se acumulan en `signature_tokens`.
  Conviene un `prune` cuando la tabla crezca.
- Siguen las 5 órdenes con filas `requested` duplicadas de antes del candado del
  22/09, y el backfill de cédulas fallando por cuota de Gemini.
- `admin_view` y `admin_finance` están definidos con 10 y 14 permisos y **nadie
  los tiene**. Si algún día se los asignan a alguien, conviene revisar primero
  que no hereden nada que no deba.
