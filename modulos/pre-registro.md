# Pre-registro asistido — Contrato del Módulo (Admin front · Admin back · App back · App · Bot WhatsApp)

> **✅ COPIA CANÓNICA.** Cualquier cambio de regla, estado o nombre de campo se
> hace aquí primero, en un commit que explique por qué.
>
> Estado: **diseño aprobado el 2026-08-26, sin implementar.** Decisiones del
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
| **El texto lo sirve el backend** | El front no arma el texto: lo pide (§7 `/consent-text`), lo muestra y firma. Misma versión que use la app cuando adopte este endpoint (pregunta abierta A). |
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
| `signature_path` | string | PNG en `storage/app/public/signatures/` (mismo store que hoy) |
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

Para el empleado es el **registro corto**: número, código, nombre, dos fotos.

1. Instala la app → "Registrarme" → su teléfono.
2. App back ve el pre-registro → responde `200` con el shape normal **más** el
   bloque `pre_registration` (§8) → manda OTP.
3. Pantalla OTP, la de siempre.
4. La app muestra el paso 1 **reducido**: la empresa ya puesta y en solo lectura
   (*"{empresa} · consentimiento firmado el {fecha}"*), y solo nombre y apellido
   editables. Sin selector de empresa.
5. Paso 2: foto de cédula, foto de talonario, crear contraseña. **Sin bloque de
   firma.**
6. `complete-registration` sin `signature_img` → app back usa la del pre-registro.
   Queda `pending_approval`.

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

**`POST /auth/register`** agrega dos ramas por teléfono:

1. Existe **pre-registro `captured`** → crea el `user` como hoy (sin contraseña),
   copia empresa y firma, manda OTP, responde `200` con el shape normal **más**:
   ```json
   "pre_registration": {
     "id": "uuid",
     "company_id": 74, "company_name": "…", "sub_company_id": null,
     "has_signature": true,
     "consent_signed_at": "2026-08-26T15:04:00-05:00",
     "next_step": "profile"
   }
   ```
   `next_step: profile` = la app pide nombre y luego documentos, sin firma.
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

- `AuthController.register`: si la respuesta trae `pre_registration`, guardarla
  y, tras el OTP, navegar según `next_step`:
  - `profile` → `SignupScreen` con `prefilled=true`.
  - `password` → pantalla de crear contraseña (reusar `set-new-password`).
- `SignupScreen` con `prefilled=true`: paso 1 muestra la empresa en solo lectura
  con *"consentimiento firmado el {fecha}"* y solo nombre + apellido editables;
  paso 2 **sin bloque de firma**. Link *"¿No es tu empresa?"* habilita el selector
  y vuelve a mostrar la firma (regla §2).
- `complete-registration` no manda `signature_img` cuando `prefilled=true` y no
  se re-firmó.
- Guard anti doble-submit y `cacheWidth` en previews, como en todo el registro.

## 10. Lo que este módulo deja escrito de paso (fuera de alcance, pero visto)

- La página pública de firma por WhatsApp (`/signature/show/{phone}`) usa el
  teléfono en base64 como token: **cualquiera puede firmar por cualquier número**.
  Y no guarda texto ni versión del consentimiento. Merece su propio contrato.
- `users` no registra el origen de la cuenta. La columna `origin` lo arregla
  hacia adelante; las cuentas viejas quedan `null`.

## Preguntas abiertas

**A. Un solo lugar para el texto del consentimiento.** Hoy el de la app vive
hardcodeado en `registration_signature_sheet.dart` y nombra a la persona. El del
kiosco nombra al teléfono. Propuesta: que la app también pida su texto a un
endpoint (mismo mecanismo que §7 `/consent-text`, en app back), y que existan
dos versiones del texto —persona y teléfono— administradas en un solo sitio.
Impacta §9. Decisión del dueño del producto.

**B. Validación legal.** ¿Firma sobre "titular del número" + OTP posterior +
foto de cédula equivale a la firma nominal en la app? El módulo asume que sí.
Si el abogado dice que no, el camino (c) tendría que re-firmar y el módulo
pierde la mitad de su gracia. **Preguntar antes de construir.**

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
