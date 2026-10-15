# Transacciones por ciclo, primera gratis, gratis del mes y login en 2 pasos

Pedidos del dueño del 14/10/2026. Publicado ese día en los servidores y en el
panel; la parte de la app sale en la **1.3.18+40 (Android) / 11.0.15 (iOS)**.

## 1. Transacciones preaprobadas por ciclo (solo informativo)

- Cada empresa tiene un número de transacciones **preaprobadas por ciclo**:
  `companies.max_transactions_per_cycle`, **3 por defecto**, editable en la
  ficha de la empresa (1 a 30).
- **No bloquea.** La solicitud sigue igual y la aprueba operaciones como
  siempre. Los mensajes son solo para el empleado (decisión del dueño).
- Cuentan las transacciones **confirmadas** (`requested`, `verified`,
  `disbursed`, `repaid`) creadas desde el inicio del **ciclo actual** de su
  empresa. Las canceladas y los resúmenes sin confirmar no cuentan.
- Ciclo actual = la misma regla que el saldo (`CalculadoraSaldo`): el día del
  cierre ya es del ciclo nuevo; si hoy no cae en ningún ciclo, el último que
  empezó antes de hoy. Sin ciclos cargados, el último mes. Comprobado igual a
  `saldos.ciclo_inicio` en los 1.246 casos.

### Los avisos (textos del dueño; con otro límite se ajustan solos)

**La que llega al límite (3.ª):**
> Esta sería tu tercera transacción de este ciclo.
>
> Payway está pensado para ocasiones puntuales. Recursos humanos recomienda no
> usarlo más de 3 veces por período, para que te quede suficiente de tu pago el
> día de la quincena.
>
> ¿Estás seguro que quieres continuar?

**De ahí en adelante (4.ª, 5.ª…):**
> Ya usaste las 3 transacciones preaprobadas para este ciclo.
>
> Payway está diseñado para ocasiones puntuales, no para uso frecuente. Esta
> cuarta transacción tendría que ser enviada a recursos humanos y aprobada por
> tu empresa.
>
> ¿Quieres continuar de todas formas?

### Dónde aparece

| Lugar | Qué hace |
|---|---|
| App (≥ 1.3.18) | El resumen (`POST /requests/summary`) trae `transacciones_ciclo {numero, limite, aviso{titulo, mensaje}}`. Si hay aviso, la app lo muestra antes del resumen; "Cancelar" cancela el resumen. Las versiones viejas lo ignoran. |
| WhatsApp | Antes del resumen con botones, el bot manda el mismo texto (autorizado por el dueño el 14/10/2026). |
| Correo de cada solicitud | Fila "Transacción del ciclo: N.º X", y "supera las N preaprobadas" si pasa el límite. |
| Panel → Transactions | Columna **Ciclo actual** (en lugar de Branch): cuántas lleva el empleado en su ciclo de HOY "N de L", ámbar al llegar al límite, rojo al pasarlo; abajo "Esta: N.º X" (qué número fue esa transacción). |

`orders.cycle_number` guarda el número al confirmarse (app: `submitRequest`;
bot: al crear). Las anteriores al 14/10/2026 quedan en null.

Código: pw-appbackend `App\Support\TransaccionesDelCiclo`; pw-adminbackend
`App\Support\TransaccionesPorCiclo` (solo cuenta, para el panel: si cambia la
regla, cambiar las dos).

## 2. Primera transacción gratis por empresa

- `companies.first_transaction_free`, **encendido por defecto** (como fue
  siempre). Switch en la ficha de la empresa, tarjeta "Transacciones".
- Apagado: la primera transacción paga la tarifa normal y no lleva `note='nuevo'`
  (no sale "New" en el panel). Lo que ya se hizo no cambia.
- Vale en la app y en WhatsApp (los dos usan `App\Support\Gratuidad`).

## 3. Transacciones gratis por mes (por empleado)

- `users.monthly_free_transactions`: cuántas gratis por mes le dio operaciones.
  En la ficha del empleado: tarjeta "Transacciones gratis por mes" con −, +1,
  +2 y Guardar; muestra por mes, usadas este mes y cuántas le quedan.
- **Vencen a fin de mes calendario** y el mes siguiente vuelve a tener las
  mismas, sin que nadie haga nada.
- Se gastan solas en la próxima transacción, **antes que los premios de
  referidos** (porque vencen). La orden queda con `orders.monthly_free = 1`,
  tarifa 0, "Tarifa (Gratis del mes)" en el resumen y **"Gratis"** en el panel.
- Solo cuentan como usadas las confirmadas: un resumen abandonado o una
  solicitud cancelada no le gasta ninguna.
- Permiso `employees.free_transactions` (super_admin y admin_ops).
- Endpoints: `GET/PUT /admin/employee/{id}/free-transactions {per_month}`.

### Orden de las gratis (`Gratuidad::decidir`)
1. Primera transacción, si la empresa la tiene encendida → `note='nuevo'`.
2. Gratis del mes, si le queda alguna → `monthly_free=1`.
3. Premio de referidos (solo en la app) → `referral_reward_id`, **"Ref"** en el panel.

El resumen de comisiones del panel cuenta 0 a las que usaron un premio de
referidos o una gratis del mes (antes contaba la tarifa fija). No se cambió
cómo cuenta las demás.

## 4. Referidos en la app

El último paso del seguimiento ya no es "Premio" con un regalo: muestra
**"Tienes X acumuladas"**, con X = premios de referidos disponibles + gratis
del mes que le quedan (`free_transactions_available` en `GET /referrals/program`,
que trae también `monthly_free {per_month, used, available}`). La tarjeta del
inicio dice "Tienes X transacciones gratis acumuladas".

## 5. Login en 2 pasos (app ≥ 1.3.18)

`POST /auth/login/check {phone}` (público, 15 por minuto). Mismas reglas y
mensajes que `/auth/login`, en el mismo orden. Responde siempre 200 con
`data.estado` y `data.mensaje`, nunca datos del usuario:

| estado | La app |
|---|---|
| `con_contrasena` | Paso 2: el número con "Cambiar" y la contraseña. |
| `sin_contrasena` | "Crea tu contraseña": código por WhatsApp (el flujo de "olvidé mi contraseña"). Solo activos. |
| `no_registrado` | "No tienes cuenta todavía" → Crear cuenta con el número precargado. |
| `en_revision` | Pantalla "en revisión", como hoy. |
| `registro_incompleto` / `en_aprobacion` | El mensaje, como hoy. |
| otro / sin respuesta | Pasa al paso 2 (login de siempre). |

**Contraseña automática 123456** (creados por WhatsApp o el panel): desde el
15/10/2026 el paso 1 la trata como `sin_contrasena` y los manda a crear la
suya con el código (el dueño cambió la decisión del 14/10). Al cambiarla no se
puede elegir 123456 de nuevo. Eran 54 cuentas activas. Si un número tiene
varias cuentas, el login toma la activa (antes tomaba cualquiera).

## 6. Pruebas (14/10/2026)

- Tarifa igual que antes para los 974 activos (todas las empresas con primera
  gratis y nadie con gratis del mes); ciclo igual al del saldo en 1.246 casos.
- En vivo, con transacciones que se deshicieron: resumen con número y aviso
  desde la 3.ª; gratis del mes con tarifa 0; empresa con primera gratis
  apagada cobra; reglas y gratis del panel guardan, validan y quedan en la
  auditoría; el correo muestra la fila nueva.
- Login paso 1 con 300 teléfonos reales sin errores; en el emulador con la APK
  de release: número inexistente → registro precargado; usuario de prueba
  #7882 → paso 2. **No probado:** "sin contraseña" (mandaría un código real),
  el aviso y referidos dentro de la app (requieren entrar con una cuenta).
- Al publicar, 148 empleados verían el aviso en su próxima transacción (115 en
  la 3.ª, 33 en la 4.ª o más).

Respaldos: `/root/respaldo-antes-transacciones-ciclo-20261014/` en pw-backend,
pw-staging (incluye volcado de companies, users, orders y permisos) y pw-frontend.
