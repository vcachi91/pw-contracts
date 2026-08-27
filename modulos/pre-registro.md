# Pre-registro asistido — Contrato del Módulo (Admin front · Admin back · App back · App · Bot WhatsApp)

> **✅ COPIA CANÓNICA.** Cualquier cambio de regla, estado o nombre de campo se
> hace aquí primero, en un commit que explique por qué.
>
> Estado: **diseño aprobado el 2026-08-26, sin implementar.** Decisiones tomadas
> por el dueño del producto: solo panel Admin (no Enterprise); tres caminos para
> completar; la firma se captura UNA vez y no se vuelve a pedir en ningún camino.

---

## 1. Qué es

Un promotor de Payway visita una empresa. Hay empleados interesados que **no
tienen su teléfono a mano** (o no tienen datos, o no quieren instalar nada en
ese momento). Hoy se pierden. Con este módulo el promotor los capta en una
tablet: nombre, teléfono, empresa y **la firma del consentimiento**, en menos de
un minuto. El registro se completa después por cualquiera de tres caminos.

Lo que este módulo NO es: no crea usuarios activos, no aprueba a nadie, no
sustituye la verificación del teléfono (OTP) ni la foto de la cédula. Solo
adelanta la captación y el consentimiento.

## 2. Reglas (se validan en backend; la UI las refleja)

| Regla | Detalle |
|---|---|
| **La firma se captura una sola vez** | Se guarda con el texto exacto que se firmó, su versión y su hash. Ningún camino posterior vuelve a pedirla. |
| **El texto del consentimiento es el de la app** | El de `registration_signature_sheet` en `pw-mobileapp`: *"Yo, {nombre}, mayor de edad, colaborador de la empresa {empresa}…"*. Misma versión en los dos lugares; si cambia, cambia aquí. |
| **El teléfono se verifica siempre por OTP** | En el kiosco no hay OTP (no hay dispositivo). Se verifica en el primer contacto real del empleado: al completar por app, por WhatsApp, o en su primer login. Hasta entonces `phone_verified_at` es null. |
| **La cédula se fotografía siempre** | Es la prueba de identidad. Por kiosco (cámara de la tablet), por app, o por WhatsApp. Sin foto de cédula no hay `pending_approval`. |
| **Un pre-registro por teléfono** | Si el teléfono ya tiene `user` activo o `pending_approval`, el kiosco lo dice y no crea nada. Si tiene un pre-registro abierto, lo reutiliza (idempotente). |
| **Modo kiosco = pantalla completa** | La tablet está autenticada como admin y se le pasa a un desconocido. Las pantallas del kiosco no muestran menú, ni datos de otros empleados, ni nada del panel. Salir del kiosco pide el PIN del promotor. |
| **Trazabilidad** | Cada pre-registro guarda qué admin lo captó, cuándo, desde qué IP y dispositivo. Cada cambio de estado va a la bitácora. |

## 3. Estados canónicos (nadie inventa otros)

```
captured ──► completed_admin      (el promotor completó todo en el kiosco)
         ──► completed_whatsapp   (el empleado terminó por el bot)
         ──► completed_app        (el empleado terminó en la app)
         ──► expired              (N días sin completar; N configurable, sugerido 60)
         ──► cancelled            (un admin lo descartó, con motivo)
```

`completed_*` enlaza al `user` creado (`user_id`). A partir de ahí manda el
estado del `user` (`pending_approval` → `active`), como hoy.

## 4. Modelo de datos

**Tabla nueva `pre_registrations`** — NO se crean medio-usuarios en `users`.
Razones: no choca con la unicidad del teléfono, no engorda el backlog de
`pending_approval`, y la conversión (captados vs completados) se mide con un
`count`.

| Campo | Tipo | Nota |
|---|---|---|
| `id` | uuid | Es lo que viaja en URLs. **Nunca el teléfono.** |
| `phone` | string, único entre los abiertos | E.164 |
| `first_name`, `last_name` | string | |
| `company_id`, `sub_company_id` | fk, nullable la sucursal | |
| `signature_path` | string | PNG en `storage/app/public/signatures/` (mismo store que hoy) |
| `consent_version` | string | p. ej. `registro_v2` |
| `consent_text_hash` | sha256 | del texto exacto renderizado con nombre y empresa |
| `consent_signed_at` | datetime | |
| `captured_by_admin_id` | fk admin_users | |
| `captured_ip`, `captured_user_agent` | string | |
| `status` | enum §3 | |
| `whatsapp_sent_at`, `whatsapp_sent_count` | datetime, int | para el botón "enviar a WhatsApp" y recordatorios |
| `completed_via` | enum `admin\|whatsapp\|app` | |
| `user_id` | fk users, nullable | se llena al completar |
| `expires_at` | datetime | |

**En `users`, una columna nueva `origin`** (`app | whatsapp | admin_kiosk | admin_panel`)
para saber cómo nació cada cuenta. Hoy no se sabe.

Dinero no aplica aquí. Fechas en ISO 8601.

## 5. Los tres caminos para completar — experiencia del empleado

Los tres arrancan igual: el promotor captó al empleado (§6). Difieren en cómo
llegan la foto de cédula, el talonario y la contraseña.

### (a) El promotor completa todo en el kiosco — *empleado presente, sin dispositivo*

1. Tras la firma, el kiosco ofrece "Completar ahora".
2. El promotor fotografía la **cédula** y el **talonario** con la cámara de la tablet.
3. Admin back crea el `user` (§7) con `origin=admin_kiosk`, **sin contraseña**,
   `pending_approval`. La firma del pre-registro pasa a `user_details.signature_path`.
4. Pantalla final al empleado: *"Listo. Cuando instales la app, entra con tu
   número y crea tu contraseña."* + WhatsApp automático con el link de descarga.
5. **Primer contacto en la app:** el empleado toca "Registrarme" o "Iniciar sesión"
   con su número. App back detecta `origin=admin_kiosk` sin contraseña → manda OTP
   → la app muestra OTP → **pantalla "Crea tu contraseña"** (ya existe:
   `set-new-password`) → entra. No ve formulario, no ve firma, no sube fotos.

### (b) Enviar a WhatsApp — *el empleado termina desde su teléfono, sin app*

1. En el kiosco (o después, desde la lista §6) el promotor toca "Enviar a WhatsApp".
2. Llega un mensaje: *"Hola {nombre}, {promotor} te registró en Payway. Para
   terminar solo necesitamos una foto de tu cédula y de tu último talonario.
   Responde a este mensaje cuando estés listo."*
3. Al responder, el bot (§8) detecta el pre-registro por teléfono y **entra en un
   flujo corto**: NO pide términos, NO pide nombre ni empresa. Pide cédula, talonario,
   y lo que el bot pida hoy además (banco, etc.). Responder ya verifica el teléfono.
4. Crea el `user` con `origin=whatsapp`, `pending_approval`, firma heredada.
5. Contraseña: como en (a), la crea al primer login en la app.

### (c) Continuar por la app — *el empleado instala la app después*

Desde su punto de vista es **"el registro corto"**: número, código, dos fotos.

1. Instala la app → "Registrarme" → escribe su teléfono → siguiente.
2. App back ve el pre-registro → responde `200` con el shape de registro nuevo
   **más** un bloque `pre_registration` (§8) → manda OTP.
3. Pantalla OTP (la de siempre).
4. La app lee `pre_registration` y va **directo al paso Documentos**, mostrando
   arriba, en solo lectura: *"{nombre} · {empresa} · Consentimiento firmado el
   {fecha}"*. El bloque de firma **no se muestra**. Solo: foto de cédula, foto de
   talonario, y crear contraseña (como hoy).
5. `complete-registration` sin `signature_img` → app back la toma del
   pre-registro. Queda `pending_approval`.

Si algo del pre-registro está mal (nombre con error, empresa equivocada), la
app muestra "¿No eres tú? Corrige tus datos" → habilita el paso 1 editable. La
firma sigue valiendo si solo cambió un typo del nombre; **si cambia la empresa,
hay que volver a firmar** (el consentimiento nombra a la empresa).

## 6. DEV FRONT ADMIN — módulo "Pre-registro"

Ítem de menú **Pre-registro**, junto a *Employees*. Dos partes:

**Modo kiosco** (ruta `/pre-registro/kiosco`, layout limpio, sin sidebar):
1. **Landing** — logo, "Regístrate en Payway", teléfono + nombre + apellido +
   empresa (+ sucursal si la empresa tiene). Grande, pocos campos, un botón.
2. **Consentimiento** — el texto legal ya con nombre y empresa, lienzo de firma,
   "Limpiar", checkbox "Acepto", botón "Firmar". Deslizar hasta el final para
   habilitar el checkbox (como en la app).
3. **¿Y ahora?** — tres botones: *Completar ahora* (camino a), *Enviar a
   WhatsApp* (camino b), *Lo hará desde la app* (camino c, solo manda el WhatsApp
   con el link de descarga). Los tres terminan en "Listo, {nombre}".
4. Botón discreto "Salir del kiosco" → pide el PIN del promotor.

**Lista** (ruta `/pre-registro`, layout normal del panel):
- Tabla con estado, empresa, promotor, fecha, "días abierto". Filtros por estado.
- Acciones por fila: *Enviar/reenviar WhatsApp*, *Completar* (abre el kiosco en
  el paso de documentos), *Cancelar* (con motivo).
- Cabecera con la conversión: captados / completados / % por camino.

El panel **no calcula nada**: pinta lo que devuelve §7. `status_label` viene del
backend.

## 7. DEV BACKEND ADMIN — endpoints (todos bajo `api/v1/admin/pre-registrations`, auth admin)

| Método | Ruta | Qué hace |
|---|---|---|
| `POST` | `/` | Crea el pre-registro con firma. Body: `phone, first_name, last_name, company_id, sub_company_id?, signature (base64 png), consent_version, consent_text`. Valida que el teléfono no tenga `user` activo/pending (→ `409` con mensaje claro) y que no haya otro pre-registro abierto (→ devuelve ese, `200`). Calcula `consent_text_hash`. Registra admin, IP, UA. Bitácora. |
| `GET` | `/` | Lista paginada, filtros `status`, `company_id`, `captured_by`. Incluye `status_label`, `days_open`. |
| `GET` | `/stats` | Captados, completados por camino, expirados, % conversión. Rango de fechas. |
| `GET` | `/{id}` | Detalle + firma (URL firmada temporal, no pública). |
| `POST` | `/{id}/complete` | Camino (a). Multipart: `cedula_img, payslip_img` (+ lo que exija hoy `EmployeeController::store`). Crea el `user` reutilizando la lógica de `store`, con `origin=admin_kiosk`, sin contraseña, `pending_approval`, `signature_path` heredado. Marca `completed_admin`, `user_id`. |
| `POST` | `/{id}/send-whatsapp` | Camino (b). Envía el template de Twilio; incrementa `whatsapp_sent_count`. Límite: 3 envíos, mínimo 24 h entre ellos. |
| `POST` | `/{id}/cancel` | Body `reason` obligatorio. |
| `GET` | `/consent-text?company_id=&first_name=&last_name=` | Devuelve el texto exacto a firmar y su `consent_version`. El front NO arma el texto: lo pide aquí, lo muestra, y devuelve el hash que el backend confirmará. |

Trabajo programado: `expired` a los N días (config `pre_registrations.expiry_days`).

Escribe en `pre_registrations` y en `users` (crea el user en camino a). Comparte
MySQL con app back: **el dueño de `pre_registrations` es admin back**; app back
la **lee** (§8) y solo escribe `status`, `completed_via`, `user_id` al completar.

## 8. DEV BACKEND APP — detectar el pre-registro

**`POST /auth/register`** (el de siempre) agrega dos ramas por teléfono:

1. Existe **pre-registro `captured`** → crea el `user` como hoy (sin contraseña),
   copia nombre/apellido/empresa/firma del pre-registro, manda OTP, y responde
   `200` con el shape normal **más**:
   ```json
   "pre_registration": {
     "id": "uuid",
     "first_name": "…", "last_name": "…",
     "company_id": 74, "company_name": "…", "sub_company_id": null,
     "has_signature": true,
     "consent_signed_at": "2026-08-26T15:04:00-05:00",
     "next_step": "documents"
   }
   ```
2. Existe **`user` con `origin=admin_kiosk`** (o `whatsapp`) **sin contraseña**
   → NO es duplicado. Manda OTP, responde `200` con `"next_step": "password"`.
   La app va a OTP y luego a "Crea tu contraseña" (`set-new-password`).

**`POST /auth/complete-registration`**: si el `user` viene de pre-registro,
`signature_img` es **opcional**; si no viene, usa la del pre-registro. Al
terminar marca el pre-registro `completed_app`, `user_id`.

**Bot de WhatsApp** (`WhatsAppChatbotController`): en `handleNewUserRegistration`,
antes de arrancar el onboarding, buscar pre-registro `captured` por teléfono. Si
existe → sesión de onboarding **precargada** (nombre, empresa, `terms_accepted=true`,
`signature_path`) y saltar directo al paso de documentos. Al finalizar, `origin=whatsapp`,
`completed_whatsapp`.

Template de Twilio nuevo: `payway_pre_registration_v1` (camino b) y
`payway_app_download_v1` (caminos a y c). Texto en §5.

## 9. DEV APP (Flutter) — el registro corto

- `AuthController.register`: si la respuesta trae `pre_registration`, guardarla
  en el controller y, tras el OTP, navegar según `next_step`:
  - `documents` → `SignupScreen` en el paso 2 con `prefilled=true`.
  - `password` → pantalla de crear contraseña (reusar la de `set-new-password`).
- `SignupScreen` con `prefilled=true`: paso 1 oculto; cabecera de solo lectura con
  nombre · empresa · "Consentimiento firmado el {fecha}"; **sin bloque de firma**;
  botón "¿No eres tú? Corrige tus datos" que habilita el paso 1.
- Si el usuario cambia la **empresa** en ese paso, se vuelve a mostrar la firma
  (regla §2). Si solo cambia el nombre, no.
- `complete-registration` no manda `signature_img` cuando `prefilled=true` y no
  se re-firmó.
- Guard anti doble-submit y `cacheWidth` en previews, como en todo el registro.

## 10. Lo que este módulo deja escrito de paso (fuera de alcance, pero visto)

- La página pública de firma por WhatsApp (`/signature/show/{phone}`) usa el
  teléfono en base64 como token: **cualquiera puede firmar por cualquier número**.
  Y no guarda texto ni versión del consentimiento. Merece su propio contrato.
- `users` no registra el origen de la cuenta. La columna `origin` de §4 lo arregla
  hacia adelante; las cuentas viejas quedan `null`.

## Preguntas abiertas

**A. Texto exacto del consentimiento y su versión.** Hoy vive en el código de la
app (`registration_signature_sheet.dart`). Para que el kiosco y la app firmen lo
mismo, el texto tiene que vivir en **un** lugar: propuesta, en admin back
(`/consent-text`), y la app lo pide en el registro en vez de tenerlo hardcodeado.
Decisión del dueño del producto. Impacta §7 y §9.

**B. ¿El abogado valida que la firma del kiosco + OTP posterior + foto de cédula
equivale a la firma en la app?** El módulo asume que sí. Si no, el camino (c)
tendría que re-firmar y el módulo pierde la mitad de su gracia.

**C. Los campos que hoy pide el bot de WhatsApp y la app no** (banco, fecha de
ingreso). ¿Se mantienen en el camino (b)? El registro de la app ya no los pide
(ver `registro-campos-reducidos.md`). Conviene que el bot se alinee al registro
reducido en la misma pasada.

## Orden de construcción sugerido

1. §7 admin back: tabla, `POST /`, `/consent-text`, `GET /` — con esto §6 puede
   captar y listar.
2. §6 admin front: kiosco completo + lista. **Ya sirve solo**: captar hoy,
   completar cuando exista lo demás.
3. §8 app back + §9 app: camino (c). Es el que escala.
4. §7 `/complete` + §8 rama `password`: camino (a).
5. §8 bot: camino (b). Último, porque toca la máquina de estados más frágil.
