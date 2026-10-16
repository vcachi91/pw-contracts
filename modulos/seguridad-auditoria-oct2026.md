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

## Lo cerrado en la segunda ronda (15/10/2026, noche): los 67 que faltaban

La primera ronda cubrió los 16 graves. En la segunda se cerraron **todos los
demás** menos lo que solo puede hacer el dueño (abajo). Un commit por repo.

| Repo | Commit | Qué se cerró |
|---|---|---|
| `pw-appbackend` | `ba11b33`, `de0e9e7` | throttle en login/OTP; `register` no pisa claves; `complete-registration` ignora el teléfono del cliente y busca la cuenta bancaria solo entre las propias; fotos solo JPEG/PNG con nombre aleatorio; perfil editable solo `address` (salario, ingreso y cédula son de RH); candado por usuario al confirmar; la nota no puede valer `nuevo`; firma con nombre aleatorio, PNG real, `nosniff` y throttle; reembolso de referido atado a la orden; aviso a RH por slug; `.env.old` fuera del repo |
| `pw-adminbackend` | `38d5239` | login con 401 uniforme y límite 10/min por IP, 5 por cuenta; el middleware `admin` exige `AdminUser` (un token de la app pasaba); `change-password` con política y cierre de otras sesiones; CSV con celdas protegidas (`App\Support\Csv`); snapshots de borrado sin `password`/`fcm_token` (los 11 viejos limpiados en la base); cabeceras fuera del log; promotor solo ve sus pre-inscripciones; fotos del kiosko solo JPEG/PNG/WebP; `view-cedula`/`view-payslips` con tipo fijo, `nosniff`, nombre acotado y throttle; `storage/credentials.json` fuera del repo |
| `pw-hrbackend` | `c2a2e54` | cuentas bancarias, bandeja de adelantos y cambio de estado acotados con `AudienceResolver` (slug + sub-empresa/sucursal); `update()`/activar/desactivar/borrar respetan el alcance; `sub_company_id` validado; vacaciones con `isMyEmployee`; carga masiva resuelve la empresa de la app por slug; `forget-password` responde igual exista o no el correo y a 5/min; `reset-password` y `resend-otp` eliminados (y `password_resets` vaciada); cambio de clave cierra sesiones; `.env.example` sin secretos; webhook de Slack sin valor por defecto |
| `pw-adminfrontend` | `12aac25` | todo `[id]`/`[action]`/`[type]`/`[clave]`/`[token]` de `/api` validado antes de armar la URL del backend (`lib/ids.ts`); CSV de empleados, transacciones y adelantos con celdas protegidas; proxy de login con 401 genérico; CSV de transacciones sin volcar al log y con id validado; `?prereg=` validado; **salir del kiosco pide la contraseña del operador** |
| `pw-hrfrontend` | `5f182b2` | ídem para sus rutas `/api` y `ENTERPRISE_ENDPOINTS` con segmentos codificados; CSV protegido; recuperar contraseña con el mismo mensaje exista o no el correo |

Además, fuera de la lista de la auditoría: los tres `.env` de producción
tenían `APP_DEBUG=true` y devolvían la traza completa en cada error (se vio al
probar). Quedó en `false` en los tres, con respaldo del `.env` en `/root`.

### Lo que cambia para quien opera

- **Kiosco.** "Salir" (o "Mis captaciones") ahora pide la contraseña del panel
  del operador. Antes un empleado ajeno podía tocar Salir y quedar dentro del
  panel con la sesión del promotor. Si molesta demasiado, la salida de fondo es
  el rol propio del kiosco (§6 de `pre-registro.md`); mientras, esto.
- **Panel Admin, login.** Ya no distingue "el correo no existe" de "clave
  incorrecta". Seis intentos fallidos seguidos sobre la misma cuenta dan 429
  durante un minuto.
- **Panel Enterprise.** Quien tenga alcance de sub-empresa o sucursal ya no ve
  cuentas bancarias ni adelantos de empleados fuera de su alcance (antes veía
  los de otra empresa por una confusión de ids). `Payway SA` (hr 53) no tiene
  espejo en la app, así que sus 0 usuarios verían listas vacías.
- **Perfil en la app.** `update-profile` solo guarda la dirección. Si la app
  manda salario o fecha de ingreso, se ignoran sin error.
- **Correo de alta (Enterprise).** Ya no dice "contraseña temporal 123456";
  dice que la app pedirá crear la clave la primera vez.

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

Secretos que estuvieron escritos en los repos. El código ya no los usa o ya no
los trae, pero siguen en el **historial de git** y hay que darlos por vistos:

- **Rotar el token de Twilio.** No hay forma de saber si ya se filtró. El
  arreglo del webhook cierra la fuga hacia adelante, pero no lo que ya pasó.
  Deja el WhatsApp mudo unos minutos, así que va de madrugada. También estaba
  en `pw-appbackend/.env.old`.
- **Cambiar la clave del buzón `hello@…`** (estaba en `.env.old` de la app y
  en `.env.example` del Enterprise; la de producción es otra, pero la vieja
  sigue siendo de la misma cuenta si no se cambió).
- **Rotar las claves de almacenamiento de objetos** (`AWS_ACCESS_KEY_ID` /
  `AWS_SECRET_ACCESS_KEY`): las del `.env.example` del Enterprise son **las
  mismas que están vivas** en los `.env` de los dos paneles. Solo las usa
  `ImageUploadJob`, que no parece correr.
- **Anular el webhook de Slack** que venía fijo en `config/backend.php` del
  Enterprise (y es el mismo del `.env` en vivo). Nada lo lee hoy.
- **Revocar la cuenta de servicio de Google** de `storage/credentials.json`
  (proyecto `mitro-sheet`, ajeno a Payway; nada la referenciaba). Quedó en
  `/root/backup-seguridad-gcpkey/` del servidor del panel.
- **Anular la clave de DeepSeek** en su consola. Viaja dentro de todas las
  versiones publicadas de la app hasta la 1.3.17.
- **Decidir si se reescribe el historial** (`git filter-repo`) de
  `pw-appbackend` (`.env.old`), `pw-adminbackend` (`credentials.json`) y
  `pw-hrbackend` (`.env.example`). Rompe los clones de todo el que tenga el
  repo; si se rotan las claves de arriba, el historial deja de importar.

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
