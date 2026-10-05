# Campañas WhatsApp

Módulo del panel admin para mandar mensajes de WhatsApp a empleados con
plantillas aprobadas por Meta, con audiencia, vista previa, nivel de riesgo y
programación. Pedido del dueño el 03/10/2026; reglas cerradas el mismo día.
Resumen y contexto en `sprints/proximo-paquete.md` §4.

**Repos:** pw-adminbackend (API, envío, tareas programadas) y pw-adminfrontend
(pantallas). **No toca la app, ni pw-appbackend, ni el bot de WhatsApp.** El
dueño pidió no cambiar el flujo de WhatsApp.

## 1. Reglas del dueño (03/10/2026)

| # | Regla |
|---|---|
| 1 | Se llama **Campañas WhatsApp** y va en el grupo **Marketing** del menú, debajo de Push Notifications. |
| 2 | Arriba de la tabla, un **resumen de lo gastado** en campañas a la fecha. |
| 3 | Lo usa **solo super_admin**. Nadie más lo ve en el menú ni puede llamar a la API. |
| 4 | **Nada sale solo**: toda campaña la arma y la confirma una persona desde el módulo. |
| 5 | Se manda desde el **mismo número** de los códigos y del bot. |
| 6 | Audiencia por defecto: **quien ya usó WhatsApp con Payway**. Si se incluye a quien nunca escribió, la campaña es **roja** y exige confirmar un aviso de riesgo. |
| 7 | Antes de enviar, un **nivel de riesgo** verde / amarillo / rojo con sus motivos. |
| 8 | **Una campaña por persona por semana**, como máximo. |
| 9 | Sin línea de consentimiento nueva en el registro. |
| 10 | "Descuento en planilla" **no** es parte del módulo. |

## 2. Quién "usó WhatsApp"

Una persona usó WhatsApp si se cumple cualquiera de estas tres condiciones:

- se registró por WhatsApp (`users.signup_source = 'whatsapp'`);
- pidió un adelanto por WhatsApp (`orders.source = 'whatsapp'`);
- le escribió al número alguna vez (`wa_contactos.ultimo_entrante_at`).

El bot no guarda los mensajes que recibe. Por eso la tarea
`wa:sincronizar-entrantes` lee de Twilio, cada 10 minutos, los mensajes que
llegaron al número y guarda por teléfono el primero y el último. La primera
carga arranca el 01/04/2026, que es desde cuando Twilio tiene historial.

## 3. Audiencia

Se resuelve **al momento de enviar**, no al crear la campaña.

1. **Base:** los mismos tipos que push: todos, por empresa, usuarios
   específicos, por estado, o filtro personalizado (`PushAudienceResolver`).
2. **App:** `sin_app` (sin `fcm_token`), `con_app` o `ambos`.
3. **Historial de WhatsApp:** `solo_usaron` (por defecto) o `incluir_nunca`.
   Opcional: "escribió en los últimos N días".
4. **Exclusiones fijas**, que no se pueden apagar. Cada persona excluida queda
   contada con su motivo en la vista previa:

| Motivo | Regla |
|---|---|
| `estado` | Despedidos (`dismissed`) e inactivos (`inactive`) nunca reciben. |
| `sin_telefono` | Sin teléfono o con un número que no se puede normalizar. |
| `baja` | Pidió no recibir más (ver §6). |
| `tope_semanal` | Ya recibió una campaña en los últimos 7 días. |
| `duplicado` | Otro usuario con el mismo teléfono ya está en la lista. |
| `sin_saldo` | La plantilla usa el saldo y la persona no llega al mínimo de B/. 25, o su empresa está congelada. |
| `variable_vacia` | Alguna variable de la plantilla sale vacía para esa persona. WhatsApp rechaza variables vacías. |

## 4. Plantillas

- Se **sincronizan desde Twilio** (Content API) a `wa_plantillas`, con estado
  de aprobación y categoría. Las de autenticación (códigos OTP) no se muestran.
- Una campaña solo puede usar una plantilla **aprobada**.
- Se pueden **crear desde el módulo**: nombre, categoría pedida (utility o
  marketing), texto con `{{1}}`, `{{2}}`…, ejemplos y botón opcional "No
  recibir más". Se crea en Twilio y se manda a aprobación de WhatsApp. **La
  categoría final la decide Meta**: el módulo muestra la que Meta aprobó.
- Cada `{{n}}` se asocia a un dato de Payway o a un texto fijo:

| Dato | Ejemplo | Nota |
|---|---|---|
| `first_name` | Juan | |
| `full_name` | Juan Pérez | |
| `company_name` | Panafoto | |
| `balance_available` | B/. 120.00 | Mismo cálculo que la app, recortado al tope de B/. 200 por solicitud. Activa la exclusión `sin_saldo`. |
| `next_payday` | Octubre 15 | Próximo cierre de ciclo de su empresa. |
| `texto_fijo` | (lo que escriba el admin) | Igual para todos. |

El salario no se ofrece como variable: no se manda por WhatsApp.

## 5. Nivel de riesgo

Se calcula en la vista previa y se guarda con la campaña al activarla.

| Nivel | Cuándo |
|---|---|
| **Rojo** | Incluye personas que **nunca usaron WhatsApp con Payway**; o la última campaña de los últimos 7 días tuvo más de 10 % de fallas; o la plantilla no está aprobada. |
| **Amarillo** | Plantilla de categoría **marketing**; o más de la mitad no escribió en los últimos 90 días; o más de 500 destinatarios. |
| **Verde** | Todo lo demás: plantilla utility, personas que ya usaron WhatsApp y una audiencia mediana. |

Una campaña **roja** solo se activa con `riesgo_aceptado = true`. El panel pide
marcar un aviso: *"Esta campaña incluye personas que nunca escribieron a
Payway por WhatsApp. Es probable que la reporten como spam y Meta puede
limitar o bloquear el número, que es el mismo de los códigos de acceso."*

## 6. Baja ("No recibir más")

Sin tocar el bot. La misma tarea `wa:sincronizar-entrantes` revisa el texto de
cada mensaje entrante. Si el mensaje entero, sin mayúsculas ni tildes, es
`baja`, `stop` o `no recibir mas`, ese teléfono entra en `wa_bajas`. Ahí entra
también quien toca el botón "No recibir más", porque el botón llega como ese
texto.

Si después escribe `alta`, sale de la lista.

**Limitación conocida:** el bot sigue contestando ese mensaje con su respuesta
habitual, porque no se modifica. Sumar una respuesta de confirmación al bot es
un cambio aparte que necesita el OK del dueño.

## 7. Envío

- Tarea `wa-campanas:despachar`, cada minuto, con `withoutOverlapping`.
- Al arrancar una corrida se resuelve la audiencia y se crean las filas de
  `wa_campana_envios` en estado `pendiente`.
- **Tandas de 50, cada 10 minutos, solo de 8:00 a 19:00 hora de Panamá.** Fuera
  de horario, la campaña espera.
- Cada envío pasa por `WhatsAppSender` con `kind = 'wa_campana'`. Así queda en
  `wa_message_logs` y el webhook de estado que ya existe lo actualiza.
- **Freno automático:** después de cada tanda, si se intentaron al menos 20 y
  falló más del 10 %, la campaña pasa a `paused` con
  `motivo_pausa = 'freno_automatico'`.
- El tope semanal se vuelve a revisar justo antes de cada envío, no solo al
  armar la lista.
- Estados y transiciones: los mismos que push (`draft`, `scheduled`, `running`,
  `sent`, `paused`, `canceled`, `failed`).

## 8. Costos

Precio por mensaje entregado, en Panamá, configurable en
`config/wa_campanas.php`:

| Categoría | Meta | Twilio | Total |
|---|---|---|---|
| Utility | $0.0163 | $0.005 | $0.0213 |
| Marketing | $0.0790 | $0.005 | $0.0840 |

- **Estimado** (vista previa): destinatarios × total de la categoría.
- **A la fecha** (resumen arriba de la tabla): Meta cobra los mensajes
  entregados o leídos; Twilio cobra cada mensaje que aceptó. Se muestra el total
  histórico, el del mes en curso, los mensajes enviados y entregados, y el
  desglose por categoría.

Son estimados con la tarifa publicada. La factura real es la de Twilio.

## 9. Tablas (base `app`, SQL en `pw-adminbackend/database/sql/`)

- `wa_plantillas`: espejo de Twilio, más el mapeo por defecto de variables.
- `wa_campanas`: la campaña, igual que `push_campaigns`, más plantilla, mapeo,
  filtros de app y de historial, riesgo y costo estimado.
- `wa_campana_envios`: una fila por persona y corrida, con el texto resuelto,
  el `message_sid` y el estado.
- `wa_contactos`: teléfono → primer y último mensaje entrante.
- `wa_bajas`: teléfonos que pidieron no recibir más.

## 10. API (pw-adminbackend, prefijo `/api/v1/admin/wa-campaigns`)

Todo exige `auth:sanctum`, `admin` y `permission:wa_campaigns.manage`. Ese
permiso lo tiene solo `super_admin`.

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/` | Lista, con filtros `status` y `search`. |
| GET | `/costos` | Resumen de gasto para el widget. |
| POST | `/preview` | Audiencia, exclusiones, riesgo, costo estimado y 3 mensajes de ejemplo con datos reales. |
| POST | `/` | Crea la campaña. Rojo sin `riesgo_aceptado` → 422 `riesgo_no_aceptado`. |
| GET | `/{id}` | Detalle con envíos paginados y conteo por estado. |
| PUT | `/{id}` | Edita (solo `draft` o `paused`). |
| DELETE | `/{id}` | Borrado lógico. |
| POST | `/{id}/pause` · `/resume` · `/cancel` · `/activate` · `/duplicate` | Igual que push. |
| GET | `/plantillas` | Plantillas sincronizadas. |
| POST | `/plantillas/sincronizar` | Trae de Twilio el estado actual. |
| POST | `/plantillas` | Crea una plantilla en Twilio y la manda a aprobación. |
| PUT | `/plantillas/{id}` | Guarda el mapeo por defecto de variables. |

## Cambios del 05/10/2026 (dueño)

- El módulo volvió al menú, como **WhatsApp** (dentro de Marketing).
- La lista de plantillas muestra **solo las creadas para campañas**
  (`creada_desde_panel = 1`). Las del bot y las de avisos siguen en Twilio y se
  siguen sincronizando, pero no se ven acá: no sirven para marketing. No se
  borraron de Twilio porque el bot y los avisos las usan.
- Tres plantillas nuevas, pedidas como **UTILITY** y sin botón de baja, sacadas
  de los push que manda Pablo (se dejaron fuera los de "congelado"):

| Nombre | Texto | Variables |
|---|---|---|
| `payway_disponible_v1` | Hola {{1}}, Payway está nuevamente disponible. / Tienes ${{2}} para acceder de inmediato. | nombre, saldo |
| `payway_saldo_disponible_v1` | Hola {{1}}, tienes ${{2}} disponibles en Payway. / Puedes acceder a tu salario en cualquier momento de tu quincena. | nombre, saldo |
| `payway_cierre_de_ciclo_v1` | Hola {{1}}, tu ciclo de Payway de esta quincena cierra el {{2}}. / Tienes ${{3}} disponibles hasta esa fecha. | nombre, próximo cierre, saldo |

  Meta decide la categoría final: puede aprobarlas como MARKETING aunque se
  pidan como UTILITY (cuesta casi 5 veces más por mensaje).
