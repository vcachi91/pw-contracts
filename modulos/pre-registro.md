# Pre-registro asistido — Contrato del Módulo (Admin front · Admin back · App back · App · Bot WhatsApp)

> **✅ COPIA CANÓNICA.** Cualquier cambio de regla, estado o nombre de campo se
> hace aquí primero, en un commit que explique por qué.
>
> Estado: **§6, §7, §8 y §9 implementados el 2026-08-26.** Falta el camino (b),
> el flujo corto del bot de WhatsApp, y el camino (a) completo (`/complete`).
>
> Ajustes que salieron al construir, y que mandan sobre el diseño de arriba:
> - El segundo nivel es **sucursal** (`sucursales`), no sub-company: es lo que
>   usan `EmployeeController::store` y el registro de la app.
> - La firma va como **MEDIUMBLOB en la tabla**, no como archivo: admin-api
>   corre en pw-staging y app-api en pw-backend, son máquinas distintas y no
>   comparten disco. Lo único común es la MySQL. App-api copia el PNG a su
>   storage al completar el registro.
> - La tabla vive en la base **`app`** (la conexión por defecto de admin-api ya
>   es esa), así que app-api la lee sin configurar nada.
> - El kiosco vive en **`/kiosco`**, fuera del grupo `(dashboard)`, para no
>   heredar el menú del panel.
> - El texto del consentimiento es **copia literal** del de la app
>   (`registration_signature_sheet.dart`), con el nombre reemplazado por
>   "titular de la línea telefónica {teléfono}" porque el kiosco no pide nombre.
>
> Decisiones del
> dueño del producto: solo panel Admin (no Enterprise); lo opera **el captador**,
> nunca el empleado solo; al empleado se le pide **únicamente su número y su
> firma**; tres caminos para completar; la firma no se vuelve a pedir en ninguno.

---

## 1. Qué es

Un **captador** de Payway visita una empresa con una tablet. Hay empleados
interesados que **no tienen su celular a mano**. Hoy se pierden. Con este módulo
el captador los pre-registra en segundos: el empleado dice su número, lee el
consentimiento, firma y acepta. Nada más. El registro se completa después por
cualquiera de tres caminos.

Lo que este módulo NO es: no crea usuarios activos, no aprueba a nadie, no
sustituye la verificación del teléfono (OTP) ni la foto de la cédula. Solo
adelanta la captación y el consentimiento.

## 2. Reglas (se validan en backend; la UI las refleja)

| Regla | Detalle |
|---|---|
| **Al empleado solo se le pide número y firma** | Ni nombre, ni cédula, ni nada. El nombre llega después (§5). |
| **La empresa la fija el captador una vez por sesión de kiosco** | Está físicamente en esa empresa. Todos los pre-registros de la sesión heredan `company_id` (+ sucursal si aplica). El empleado no la elige. |
| **La firma se captura una sola vez** | Se guarda con el texto exacto firmado, su versión y su hash. Ningún camino posterior la vuelve a pedir. |
| **El consentimiento nombra al teléfono y a la empresa, no a una persona** | *"Yo, titular del número +507 6xxx-xxxx, colaborador de {empresa}, autorizo…"*. La identidad se ata después: el OTP prueba el teléfono y la foto de cédula prueba quién es. Las tres piezas juntas forman la evidencia. |
| **El texto lo sirve el backend y se lee antes de firmar** | El front no arma el texto: lo pide (§7 `/consent-text`), lo muestra **completo y con scroll**, y solo habilita "Acepto" cuando el empleado llegó al final — el mismo patrón que la firma de la app. |
| **Los flujos de registro actuales no cambian** | App, WhatsApp y panel siguen funcionando exactamente igual para quien no tiene pre-registro. Todo lo de este módulo es **aditivo**: si el backend no responde o no hay pre-registro, la app se comporta como hoy. |
| **El teléfono se verifica siempre por OTP** | En el kiosco no hay OTP. Se verifica en el primer contacto real: app, WhatsApp o primer login. Hasta entonces `phone_verified_at` es null. |
| **La cédula se fotografía siempre** | Por kiosco (cámara de la tablet), por app, o por WhatsApp. Sin foto de cédula no hay `pending_approval`. |
| **Un pre-registro por teléfono** | Si el teléfono ya tiene `user` activo o `pending_approval`, el kiosco lo dice ("ya está registrado") y no crea nada. Si hay un pre-registro abierto, lo reutiliza (idempotente). |
| **Modo kiosco = pantalla completa** | La tablet está autenticada como admin y se le pasa a un desconocido. Sin menú, sin datos de otros empleados, sin nada del panel. Salir pide el PIN del captador. |
| **Trazabilidad** | Cada pre-registro guarda qué captador lo hizo, cuándo, IP y dispositivo. Cada cambio de estado va a la bitácora. |

## 3. Estados canónicos (nadie inventa otros)

```
captured ──► completed_admin      (el captador completó todo en el kiosco)
         ──► completed_whatsapp   (el empleado terminó por el bot)
         ──► completed_app        (el empleado terminó en la app)
         ──► expired              (N días sin completar; configurable, sugerido 60)
         ──► cancelled            (un admin lo descartó, con motivo)
```

`completed_*` enlaza al `user` creado (`user_id`). Desde ahí manda el estado del
`user` (`pending_approval` → `active`), como hoy.

## 4. Modelo de datos

**Tabla nueva `pre_registrations`** — NO se crean medio-usuarios en `users`. No
choca con la unicidad del teléfono, no engorda el backlog de `pending_approval`,
y la conversión se mide con un `count`.

| Campo | Tipo | Nota |
|---|---|---|
| `id` | uuid | Es lo que viaja en URLs. **Nunca el teléfono.** |
| `phone` | string, único entre los abiertos | E.164 |
| `company_id`, `sub_company_id` | fk, sucursal nullable | de la sesión de kiosco |
| `signature_png` | mediumblob | El PNG de la firma, **en la tabla**, no en disco. Admin-api (pw-staging) y app-api (pw-backend) son servidores distintos: un archivo escrito por uno no existe para el otro, pero la MySQL sí es compartida. Al completar, app-api lo escribe a su `storage/signatures/` y setea `user_details.signature_path` como hoy. |
| `consent_version` | string | p. ej. `prereg_v1` |
| `consent_text_hash` | sha256 | del texto exacto renderizado con teléfono y empresa |
| `consent_signed_at` | datetime | |
| `captured_by_admin_id` | fk admin_users | el captador |
| `captured_ip`, `captured_user_agent` | string | |
| `status` | enum §3 | |
| `whatsapp_sent_at`, `whatsapp_sent_count` | datetime, int | |
| `completed_via` | enum `admin\|whatsapp\|app` | |
| `user_id` | fk users, nullable | se llena al completar |
| `expires_at` | datetime | |

No hay columnas de nombre: el nombre nace en el `user` cuando se completa.

**En `users`, columna nueva `origin`** (`app | whatsapp | admin_kiosk | admin_panel`)
para saber cómo nació cada cuenta. Hoy no se sabe.

## 5. Los tres caminos para completar

Los tres arrancan igual: el captador ya tiene número + firma + empresa. Difieren
en cómo llegan el **nombre**, la **foto de cédula**, el **talonario** y la
**contraseña**.

### (a) El captador completa en el kiosco — *empleado presente, sin celular*

1. Tras la firma, "Completar ahora".
2. El captador escribe nombre y apellido, y fotografía **cédula** y **talonario**
   con la tablet.
3. Admin back crea el `user` (§7 `/complete`) con `origin=admin_kiosk`, **sin
   contraseña**, `pending_approval`, firma heredada.
4. Al empleado: *"Listo. Cuando instales la app, entra con tu número y crea tu
   contraseña."* + WhatsApp automático con el link de descarga.
5. **Primer contacto en la app:** pone su número → app back ve `origin=admin_kiosk`
   sin contraseña → OTP → pantalla **"Crea tu contraseña"** (`set-new-password`,
   ya existe) → entra. No ve formulario, ni firma, ni fotos.

### (b) Enviar a WhatsApp — *el empleado termina desde su teléfono, sin app*

1. Desde el kiosco o desde la lista (§6): "Enviar a WhatsApp".
2. Mensaje: *"Hola, te pre-registraron en Payway en {empresa}. Para terminar solo
   necesitamos tu nombre, una foto de tu cédula y de tu último talonario. Responde
   cuando estés listo."*
3. Al responder, el bot (§8) detecta el pre-registro y entra en el **flujo corto**:
   NO pide términos ni empresa. Pide nombre, cédula, talonario. Responder ya
   verifica el teléfono.
4. Crea el `user` con `origin=whatsapp`, `pending_approval`, firma heredada.
5. Contraseña: como en (a), al primer login en la app.

### (c) Continuar por la app — *el empleado instala la app después*

El empleado entra al **registro normal de la app**; es la app la que detecta el
pre-registro sola.

1. Instala la app → "Registrarme" → escribe su teléfono en el paso 1.
2. **Al terminar de escribir el número**, la app consulta en silencio
   (§8 `/auth/pre-registration/lookup`). Si existe: la empresa aparece ya
   seleccionada y bloqueada, con una banda *"Pre-registrado en {empresa} ·
   consentimiento firmado el {fecha}"*. Solo quedan nombre y apellido.
   Si no existe: el paso 1 es el de siempre, sin ningún cambio.
3. Registro → OTP, la pantalla de siempre.
4. Paso 2 (documentos): foto de cédula, foto de talonario. Y el bloque de firma
   **aparece ya firmado**: muestra la imagen de la firma capturada en el kiosco
   con *"Firmado el {fecha} en {empresa}"* en lugar del lienzo. No hay nada que
   dibujar ni que aceptar.
5. Crear contraseña → `complete-registration` sin `signature_img` → app back usa
   la heredada. Queda `pending_approval`.

Si la empresa está mal (el captador eligió otra sesión por error): *"¿No es tu
empresa?"* → habilita el selector, y **se vuelve a pedir la firma**, porque el
consentimiento nombra a la empresa. Cambiar solo el nombre nunca requiere re-firma.

## 6. DEV FRONT ADMIN — módulo "Pre-registro"

Ítem de menú **Pre-registro**, junto a *Employees*. Dos partes:

**Modo kiosco** (`/pre-registro/kiosco`, layout limpio, sin sidebar):

0. **Inicio de sesión de kiosco** (lo ve el captador, una vez): elige empresa y
   sucursal. Desde aquí la tablet pasa a manos del empleado.
1. **Número** — logo, *"Regístrate en Payway"*, un solo campo grande para el
   teléfono, un botón. Si el número ya está registrado, lo dice y vuelve.
2. **Consentimiento** — el texto (§7 `/consent-text`) con teléfono y empresa ya
   puestos, lienzo de firma, "Limpiar", checkbox "Acepto", botón "Firmar".
   Deslizar hasta el final habilita el checkbox (como en la app).
3. **Listo** — *"¡Gracias! Te enviamos por WhatsApp el link para completar tu
   registro."* Botón grande **"Siguiente empleado"** → vuelve a la pantalla 1 con
   la misma empresa. Botones secundarios para el captador: *Completar ahora* (a)
   y *Reenviar WhatsApp* (b).
4. "Salir del kiosco" discreto → PIN del captador.

**Lista** (`/pre-registro`, layout normal):
- Tabla: teléfono (enmascarado), empresa, captador, fecha, estado, días abierto.
  Filtros por estado, empresa, captador.
- Acciones por fila: *Enviar/reenviar WhatsApp*, *Completar* (abre el kiosco en
  el paso de nombre + documentos), *Cancelar* (con motivo).
- Cabecera con la conversión: captados / completados por camino / % .

El panel **no calcula nada**: pinta lo que devuelve §7. `status_label` viene del
backend.

## 7. DEV BACKEND ADMIN — endpoints (`api/v1/admin/pre-registrations`, auth admin)

| Método | Ruta | Qué hace |
|---|---|---|
| `GET` | `/consent-text?company_id=&phone=` | Devuelve `{consent_version, consent_text}` ya renderizado con teléfono y empresa. Es lo que se firma. |
| `POST` | `/` | Body: `phone, company_id, sub_company_id?, signature (base64 png), consent_version`. Re-renderiza el texto con los mismos datos y guarda su hash (el front no manda el texto). Si el teléfono tiene `user` activo/pending → `409` con `message` claro. Si hay pre-registro abierto → lo devuelve, `200`. Registra captador, IP, UA. Bitácora. |
| `GET` | `/` | Lista paginada, filtros `status`, `company_id`, `captured_by`. Incluye `status_label`, `days_open`, `phone_masked`. |
| `GET` | `/stats` | Captados, completados por camino, expirados, % conversión. Rango de fechas. |
| `GET` | `/{id}` | Detalle + firma (URL firmada temporal, no pública). |
| `POST` | `/{id}/complete` | Camino (a). Multipart: `first_name, last_name, cedula_img, payslip_img` (+ lo que exija hoy `EmployeeController::store`). Crea el `user` reutilizando `store`, con `origin=admin_kiosk`, sin contraseña, `pending_approval`, `signature_path` heredado. Marca `completed_admin`, `user_id`. |
| `POST` | `/{id}/send-whatsapp` | Camino (b). Template de Twilio; incrementa `whatsapp_sent_count`. Límite: 3 envíos, mínimo 24 h entre ellos. |
| `POST` | `/{id}/cancel` | Body `reason` obligatorio. |

Al crear (`POST /`) se envía **automáticamente** el WhatsApp con el link de
descarga (template `payway_app_download_v1`). El botón "Enviar a WhatsApp" manda
el otro template (`payway_pre_registration_v1`, camino b).

Job programado: `expired` a los N días (`pre_registrations.expiry_days`).

**Dueño de `pre_registrations` es admin back.** App back la **lee** (§8) y solo
escribe `status`, `completed_via`, `user_id` al completar. Comparten MySQL.

## 8. DEV BACKEND APP — detectar el pre-registro

**`GET /auth/pre-registration/lookup?phone=`** — nuevo, público, con el mismo
`throttle` que `register`. Es lo que la app llama al escribir el número.
Devuelve **lo mínimo** (antes del OTP no se entrega la firma ni nada
enumerable):
```json
{ "exists": true, "company_id": 74, "company_name": "…", "sub_company_id": null,
  "consent_signed_at": "2026-08-26T15:04:00-05:00" }
```
o `{ "exists": false }`. Si el backend aún no lo tiene (404), la app sigue como hoy.

**`POST /auth/register`** agrega dos ramas por teléfono:

1. Existe **pre-registro `captured`** → crea el `user` como hoy (sin contraseña),
   copia empresa y firma, manda OTP, responde `200` con el shape normal **más**:
   ```json
   "pre_registration": {
     "id": "uuid",
     "company_id": 74, "company_name": "…", "sub_company_id": null,
     "has_signature": true,
     "signature_url": "https://…/signatures/sig_…png?expires=…&signature=…",
     "consent_signed_at": "2026-08-26T15:04:00-05:00",
     "next_step": "profile"
   }
   ```
   `signature_url` es una URL firmada temporal (la app la muestra en el paso 2
   como "ya firmado"). Solo se entrega aquí, después de que el teléfono pidió
   su OTP — nunca en el `lookup`.
2. Existe **`user` con `origin` en (`admin_kiosk`, `whatsapp`) y sin contraseña**
   → NO es duplicado. Manda OTP, responde `200` con `"next_step": "password"`.

**`POST /auth/complete-registration`**: si el `user` viene de pre-registro,
`signature_img` es **opcional**; si no viene, usa la heredada. Al terminar marca
el pre-registro `completed_app`, `user_id`.

**Bot de WhatsApp** (`WhatsAppChatbotController::handleNewUserRegistration`):
antes de arrancar el onboarding, buscar pre-registro `captured` por teléfono. Si
existe → sesión precargada (`company_id`, `terms_accepted=true`, `signature_path`)
y saltar a: nombre → cédula → talonario. Al finalizar, `origin=whatsapp`,
`completed_whatsapp`.

## 9. DEV APP (Flutter) — el registro corto

- Paso 1 de `SignupScreen`: al perder el foco el campo de teléfono (o al tener
  8 dígitos válidos), llamar `lookup` con debounce. Si `exists`: seleccionar y
  bloquear la empresa, mostrar la banda de pre-registro. Si no, o si el endpoint
  falla: **nada cambia**, el formulario es el de hoy.
- `AuthController.register`: si la respuesta trae `pre_registration`, guardarla
  y, tras el OTP, navegar según `next_step`:
  - `profile` → `SignupScreen` paso 2 con `preRegistration` cargado.
  - `password` → pantalla de crear contraseña (reusar `set-new-password`).
- Paso 2 con `preRegistration`: el bloque de firma muestra la imagen de
  `signature_url` (con `cacheWidth` y `errorBuilder`, como toda imagen) y el
  texto *"Firmado el {fecha} en {empresa}"*. Sin lienzo, sin checkbox. El botón
  final se habilita con solo las dos fotos y la contraseña.
- Link *"¿No es tu empresa?"* en el paso 1 habilita el selector; si la cambia,
  el paso 2 vuelve a mostrar el lienzo y el consentimiento normal (regla §2).
- `complete-registration` no manda `signature_img` cuando hay `preRegistration`
  y no se re-firmó.
- Guard anti doble-submit y `cacheWidth` en previews, como en todo el registro.

## 10. Lo que este módulo deja escrito de paso (fuera de alcance, pero visto)

- La página pública de firma por WhatsApp (`/signature/show/{phone}`) usa el
  teléfono en base64 como token: **cualquiera puede firmar por cualquier número**.
  Y no guarda texto ni versión del consentimiento. Merece su propio contrato.
- `users` no registra el origen de la cuenta. La columna `origin` lo arregla
  hacia adelante; las cuentas viejas quedan `null`.

## Decisiones cerradas (2026-08-26, dueño del producto)

- **Solo número y firma.** Se construye así. La firma es sobre "titular del
  número + empresa"; la identidad se ata después con OTP y cédula. Si en el
  futuro un abogado pide firma nominal, el kiosco agrega el nombre y re-versiona
  el texto — el modelo (versión + hash) ya lo soporta sin migración.
- **El texto del consentimiento se muestra completo, con scroll, antes de
  firmar.** Lo sirve el backend (§7). El texto de la app sigue donde está por
  ahora; unificarlos queda como mejora, no como bloqueo.

## Preguntas abiertas

**C. Campos del bot.** El bot todavía pide banco y fecha de ingreso; el registro
de la app ya no (`registro-campos-reducidos.md`). Alinear el flujo corto del bot
al registro reducido en la misma pasada.

## Orden de construcción sugerido

1. §7 admin back: tabla, `/consent-text`, `POST /`, `GET /` — con esto §6 ya
   capta y lista.
2. §6 admin front: kiosco + lista. **Ya sirve solo**: se capta desde el día uno.
3. §8 app back + §9 app: camino (c), el que escala.
4. §7 `/complete` + §8 rama `password`: camino (a).
5. §8 bot: camino (b). Último, porque toca la máquina de estados más frágil.
