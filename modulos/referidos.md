# Programa de referidos

Un empleado invita a compañeros de **su misma empresa**. Cuando el invitado
recibe su primer adelanto, el que invitó suma progreso hacia un premio.

Estado: **implementado, sin desplegar** (09/09/2026).

---

## 1. La regla que manda sobre todas

**Ni las reglas ni los textos del programa viven en la app.** Cuántos referidos
hacen falta, qué se gana, el título de la pantalla, la descripción, el mensaje
que se comparte y hasta la frase del progreso salen de `GET /referrals/program`.

El requisito era que operaciones pudiera cambiar *"2 referidos = 1 transacción
gratis"* a cualquier otra combinación **sin publicar la app**. Si el cliente
calculara o redactara algo por su cuenta, cambiar el programa exigiría pasar por
revisión de Apple.

Si algún día la app empieza a mostrar un número que no vino del backend, eso es
un bug, no una optimización.

---

## 2. Decisiones de negocio congeladas

Confirmadas con el dueño el 09/09/2026. Están anotadas en el código porque
varias son contraintuitivas.

| Decisión | Valor | Por qué |
|---|---|---|
| Qué activa un referido | La primera orden del invitado en **`verified`** | El requisito decía "desembolsado", pero `disbursed` está muerto en producción desde 2025-11-28 (21 órdenes históricas, ninguna posterior): operaciones deja todo en `verified`. Atarlo a `disbursed` no habría activado a nadie jamás. |
| Empresa | Referidor y referido de la **misma** empresa | El programa es entre compañeros de trabajo. |
| Consumo del premio | **Automático** en el siguiente adelanto | Más simple de construir y de explicar. |
| Tope mensual | Mes **calendario**, hora de Panamá | Un promotor/empleado no está atado al ciclo de una empresa. |
| Qué hace el tope | **Acota, no descarta** | Quien trae 8 en un mes con tope 5 aporta 5 y **no pierde los otros 3**: ese mes topea y el siguiente vuelve a haber cupo. Anularlos habría castigado al que mejor funciona. |
| Cuándo se aplica el código | **Solo al registrarse** | No hay repesca posterior, por diseño. |

---

## 3. Dónde vive cada cosa

| Pieza | Repo | Nota |
|---|---|---|
| Migración (6 tablas + `orders.referral_reward_id`) | **pw-appbackend** | ⚠️ **Solo este repo las corre.** No duplicar en admin. |
| Motor (atribución, activación, premios) | pw-appbackend | `app/Modules/Referrals/` |
| API de la app | pw-appbackend | `/v1/referrals/*` |
| Enlace corto `/r/{slug}` | pw-appbackend | `routes/web.php`, devuelve HTML |
| Administración del programa | pw-adminbackend | `/admin/referrals/*` |
| Modelos espejo | pw-adminbackend | Mismas tablas. **Un cambio va en los dos.** |
| Pantallas | pw-adminfrontend | `/referrals` |
| App | pw-mobileapp | Bloque en el home + pantalla + campo en el registro |

Las dos apps Laravel comparten la MySQL (`DB_DATABASE=app` en ambas) pero **no
el código**. Por eso el barrido de activación corre en app-api aunque el
`verified` lo marque admin-api: duplicar la lógica sería la forma más segura de
que las dos mitades se desincronicen con el tiempo.

---

## 4. El premio NO reusa `note='nuevo'`

Esa marca ya significa "primera orden del cliente", y **se compara distinto en
cada repo**:

- `OrderPricing` y `RequestController` (app): `$order->note === 'nuevo'`
- `Order` model (admin): `Str::contains(Str::lower($this->note), 'nuevo')`

Colgar los referidos de ahí habría hecho que el panel y la app calcularan
montos **distintos para la misma orden**. Va una columna propia:
`orders.referral_reward_id`.

---

## 5. Atribución diferida

El caso difícil: alguien toca el enlace **sin tener la app**, instala, abre… y
el código se perdió.

| Plataforma | Cómo | Confiabilidad |
|---|---|---|
| Android | **Play Install Referrer**: el enlace corto manda a Play Store con `&referrer=payway_ref=CODIGO`; Google lo guarda y la app lo lee en el primer arranque | Determinista |
| iOS | **Portapapeles**: la página del enlace copia el código y la app lo lee al abrir | Parcial — iOS 14+ muestra "pegó desde Safari" |
| Ambas | El **código en texto plano** dentro del mensaje compartido | Siempre funciona, si la persona lo escribe |

Se guarda de dónde vino en `referrals.attribution_source`, y el panel lo
muestra. **Ese es el número que decide** si algún día vale la pena pagar un SDK
de atribución para iOS.

⚠️ **Firebase Dynamic Links no es opción**: Google lo apagó el **25/08/2025**.

⚠️ Los códigos se generan sobre un alfabeto **sin caracteres ambiguos**
(`ACDEFGHJKMNPQRTUVWXY34679` — sin `0/O`, `1/I/L`, `5/S`, `8/B`, `2/Z`) porque
la última vía de atribución es que alguien los lea de un WhatsApp y los escriba.

---

## 6. Endpoints

### App (`/api/v1`)

| Método | Ruta | Auth | Qué devuelve |
|---|---|---|---|
| GET | `/referrals/program` | sanctum | Todo lo que la app pinta. `enabled:false` = no hay programa, **200 y no error** |
| GET | `/referrals/my-referrals` | sanctum | Invitados y su estado. Solo el nombre, ningún dato personal más |
| POST | `/referrals/validate-code` | **pública** | Valida y devuelve el nombre de quien invita |
| GET | `/r/{slug}` | pública | Enlace corto (web.php, HTML) |

`validate-code` va **fuera de `auth:sanctum` a propósito**: la persona escribe
el código durante el registro, cuando todavía no existe token. Mismo criterio
que `reportar-foto-registro`, donde exigir token dejó el diagnóstico sin
enviarse nunca.

El registro (`/auth/complete-registration`) acepta `referral_code` y
`referral_source` opcionales. **Un código inválido no bloquea el registro.**

### Admin (`/api/v1/admin/referrals`)

`GET /`, `/stats`, `/export`, `/rewards`, `/rewards/export`, `/{id}/audit`,
`GET|POST|PUT /program`, `POST /rewards`, `DELETE /rewards/{id}`,
`DELETE /{id}`.

**Permisos** (los crea `database/sql/2026_09_09_referrals_permissions.sql`,
aplicar **antes** de desplegar o todo devuelve 403):

- `referrals.view` — leer
- `referrals.manage` — editar el programa (mueve dinero futuro)
- `referrals.adjust` — otorgar/anular premios a mano (mueve dinero **ya**, exige motivo)

Los promotores **no reciben ninguno**: captar registros no tiene que ver con
ver premios ajenos.

---

## 7. Formatos de respuesta — son DOS

| Backend | Forma |
|---|---|
| app-api | `{status, response, message, data}` |
| admin-api | `{response_code, message, data}` |

No son intercambiables. Cada lado respeta el suyo.

---

## 8. Qué falta

- **Push al teléfono.** El aviso se escribe en `push_notifications` (la app lo
  ve en su lista), pero el envío por FCM no: el paquete de Firebase está en
  admin-api, no en app-api. Se engancha desde `App\Services\Push\PushDispatcher`,
  que ya lee de esa tabla.
- **`REFERRAL_LINK_BASE`** en el `.env` de app-api. Sin eso el enlace sale como
  `http://localhost/r/xxxx`, porque cae a `APP_URL`.
- Aplicar el SQL de permisos y correr la migración.
