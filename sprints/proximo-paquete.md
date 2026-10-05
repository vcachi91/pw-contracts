# Sprint del próximo paquete

Documento vivo: se va anotando lo que se acuerda con el dueño hasta cerrar el
sprint. Lo que necesita la app sale **junto, en una sola versión**, en vez de
varias sueltas (así se lo dijo el dueño a Pablo el 02/10/2026). Lo que es solo
de servidor y panel puede salir antes, sin esperar a la app.

Cuando un ítem pase a construirse, su regla detallada va como contrato en
`modulos/`; acá queda el resumen y el estado.

Estados: **Acordado** · **Pendiente de decisión** · **Hecho, sin publicar**.

## Resumen

| # | Ítem | Estado | ¿Necesita versión de app? |
|---|---|---|---|
| 1 | Switch de cuentas bancarias por empresa y empleado | **Backend y panel desplegados el 04/10, todo apagado**; falta la app (`modulos/cuentas-bancarias-app.md`). El bot ofrece solo cuentas verificadas | Sí |
| 2 | Sucursales por sub-company (+ bug de ids) | Acordado | Sí |
| 3 | Sucursal obligatoria con "Sin sucursal" | Acordado | Sí (el backend ya cubre a las apps viejas) |
| 4 | Menú "Marketing" + Campañas WhatsApp | **Publicado 03/10/2026**, oculto del menú hasta nuevo aviso del dueño (se entra por `/campanas-whatsapp`); sin campañas enviadas (contrato: `modulos/campanas-whatsapp.md`) | No: backend y panel |
| 5 | Cambiar de empresa a un empleado desde el panel | Acordado (espera OK para construir) | No: backend y panel |
| 8 | Notificaciones de la app: campanita con número rojo y bandeja | **Backends desplegados el 04/10**; falta la versión de la app (`modulos/notificaciones-app.md`) | Sí |
| 7 | Saldo disponible: un solo cálculo para todos los canales | **Hecho 03–04/10/2026** (detalle en `modulos/saldo-disponible.md`) | No: backends |
| 6 | Recuperar clave: traba contra doble toque | Hecho, sin publicar | Sí |

---

## 1. Switch de cuentas bancarias en la app — Acordado

Hoy la app no deja agregar ni borrar cuentas: los dos botones se quitaron el
10/03/2026 (versión 1.3.4). Todas las cuentas las carga operaciones desde la
ficha del empleado en el panel (143 en los últimos 60 días; 0 intentos desde la
app). Este switch permite **volver a habilitar** que el empleado agregue su
cuenta, por empresa o por persona.

**Reglas acordadas (03/10/2026):**

- **Empresa:** switch encendido / apagado. Arranca **apagado** en todas, que es
  como funciona hoy.
- **Empleado:** *según la empresa* (por defecto) / *permitir* / *bloquear*. La
  excepción del empleado manda sobre la empresa.
- **Apagado** = no puede agregar cuentas desde la app. Las que ya tiene se
  conservan y sigue retirando a ellas.
- **Borrar** sigue sin existir en la app, esté como esté el switch.
- Una cuenta agregada desde la app entra **sin verificar** y no aparece hasta que
  operaciones la verifica en el panel (ese candado ya existe).
- **Si la cuenta ya pertenece a otra persona, se rechaza** con un mensaje de
  contáctenos, para que dos usuarios no compartan cuenta. Hoy el backend se la
  vincula **ya verificada**, saltándose a operaciones: un error de tipeo podía
  mandar un adelanto a la cuenta de otro. Se arregla antes de reactivar nada.
- Contacto del mensaje: el mismo WhatsApp de operaciones del aviso de empresa
  pausada (`wa.me/50769958313`), salvo que el dueño diga otro.

**Qué se construye:**

| Dónde | Qué |
|---|---|
| Base (`app`) | Columna en `companies` y en `users` (empleado: hereda / permitir / bloquear) |
| app-api | Bloquear `save-bank-account` según el switch; rechazar cuenta ajena; mandarle a la app si puede agregar |
| Panel Admin | Los dos switches: ficha de empresa y ficha de empleado |
| App | Mostrar "Agregar cuenta" solo si está permitido. La pantalla no corre desde marzo: probarla de punta a punta y ponerle la traba contra doble toque |

**Orden:** backend y panel primero (no cambia nada visible, todo apagado) →
versión de la app → forzar actualización → recién ahí encender donde haga falta.

---

## 2. Sucursales por sub-company — Backends y panel publicados (04/10/2026); app en la próxima versión

> **Estado 04/10/2026:** en vivo en pw-appbackend, pw-adminbackend y el panel
> admin. La app ya lo tiene en el código y sale con la próxima versión. Todo el
> detalle, las pruebas y cómo volver atrás: `modulos/sucursales.md`. Quedó sin
> tocar a propósito: bot de WhatsApp y panel Enterprise. El kiosco y la página
> de Sub-companies también quedaron publicados el 04/10.

Pedido de Pablo (02/10/2026), con el ejemplo de **Davissa Foods → Paul Bakery**.

Hoy las sucursales cuelgan **solo de la empresa** (`sucursales.company_id`) y una
sub-company no puede tener las suyas. Sub-company y sucursal siguen siendo
conceptos distintos (decisión de junio): sub-company es una empresa del grupo o
marca; sucursal es un local físico. Lo nuevo es que una sucursal pueda
pertenecer a una sub-company.

**Reglas acordadas:**

- Una sucursal pertenece a una empresa y, opcionalmente, a una de sus
  sub-companies.
- Quien pertenece a una sub-company que tiene sucursales propias ve **solo las
  de esa sub-company** (Pablo: "solo de la subempresa").
- Quien pertenece a una sub-company **sin** sucursales propias, o a una empresa
  sin sucursales, **no ve el campo**: el backend le asigna "Sin sucursal" por
  dentro (decisión del dueño, 04/10/2026; ver §3).

**Afecta a todos los lugares donde se elige o se muestra una sucursal:** registro
en la app, kiosco del captador, registro por WhatsApp, alta y edición de
empleado en el panel, gestión de sucursales en el panel (poder crearlas dentro de
una sub-company), alcance de usuarios Enterprise por sucursal, CSV de empleados
y conteos por sucursal del tablero.

### 2a. Bug que ya existe y se arregla en este paquete

En el registro, el desplegable mezcla empresas y sub-companies, y las dos tablas
**repiten números**. Al elegir una sub-company, la app pide las sucursales con
ese número y el backend lo busca como **empresa**:

- Un empleado de **YEM Corp** (sub-company de Grupo Piazza) ve hoy las **10
  sucursales de Cinépolis**.
- Uno de **Paul Bakery** no ve ninguna.

Todavía no causó daño (0 empleados con una sucursal ajena, verificado el
03/10/2026), porque el registro descarta en silencio la sucursal que no es de
su empresa. Se arregla pidiendo las sucursales **con el tipo** (empresa o
sub-company), igual que ya se hizo para la empresa en el registro.

---

## 3. Sucursal obligatoria, con "Sin sucursal" — Parcial (04/10/2026)

> **Estado 04/10/2026:** "Sin sucursal" quedó como `sucursal_id` vacío que los
> paneles muestran con ese texto, **no como una fila** (ver `modulos/sucursales.md`
> §3): nadie tuvo que migrarse. La app nueva ya exige elegir cuando hay
> sucursales. **Falta:** exigirla al guardar la ficha en el panel admin (se
> prende cuando las empresas tengan sus sucursales cargadas). El kiosco ya la
> exige cuando hay sucursales para elegir.

**Reglas acordadas (03/10/2026):**

- La sucursal es **obligatoria en todos los canales**: app, kiosco, panel Admin y
  WhatsApp.
- Toda empresa o sub-company sin sucursales propias tiene una sucursal por
  defecto llamada **"Sin sucursal"**, así lo obligatorio nunca bloquea un
  registro.
- **Si un registro llega sin sucursal** (una app vieja, o un canal todavía sin
  actualizar), el backend le asigna "Sin sucursal" en vez de rechazarlo.
- **"Sin sucursal" nunca se le muestra al empleado** (dueño, 04/10/2026): si no
  hay sucursales reales para elegir, el campo no aparece y la asignación es
  interna. Obligatorio quiere decir que el campo se exige **cuando hay**
  sucursales reales. Sí se ve en los paneles, para reportes y reasignar.
- **Cuando una empresa reciba sus sucursales reales**, "Sin sucursal" deja de
  aparecer para registros nuevos. Quien ya esté en ella la conserva hasta que
  operaciones lo reasigne.
- **Los empleados actuales sin sucursal** (la mayoría) quedan asignados a "Sin
  sucursal" automáticamente, para que los reportes cuadren.

### 3a. Cargar sucursales a las empresas que no tienen

Con "Sin sucursal", cargar las reales ya no bloquea nada, pero el dueño quiere
hacerlo antes del cambio. Estado al 03/10/2026:

- **Se pueden cargar ya** (no tienen sub-companies): SEMM (51 empleados), Royal
  Casino (25), Smart Valet (21), BigBet (12), Petromotors (9), Bonanza 94 (7),
  HMS (5), Confecciones Shic (4), Desarrollo Bahía (2).
  - Ojo: si se cargan a **Bonanza** o **Royal Casino** antes del arreglo §2a, los
    empleados de **Deco Auto** y **Grupo Lafayette** las verán en el registro.
    Es inofensivo (lo que elijan se descarta), pero confunde.
- **Esperar al panel nuevo:** **Panafoto** (200 empleados, 3 sub-companies) y
  **Grupo Piazza** (152, 17 sub-companies). Sus sucursales probablemente son de
  cada sub-company, y hoy el panel no puede asignar una sucursal a una
  sub-company.
- Ya tienen: J. Cain Logistics (11), Cinépolis (10), Minimed (8), Manpower (4),
  La Parmigiana (2), Davissa Foods (2).

---

## 4. Menú "Marketing" y Campañas WhatsApp — Publicado (03/10/2026)

Pedido del dueño (03/10/2026).

**Menú del panel:** un grupo nuevo **"Marketing"** que contiene **Push
Notifications** (se mueve ahí, sin cambios) y debajo el módulo nuevo.
Nombre recomendado: **"Campañas WhatsApp"** — dice que es un envío masivo con
audiencia y programación, igual que las campañas de push.

**Qué es:** enviar mensajes por WhatsApp desde el panel, como hoy se mandan
pushes, pero con plantillas aprobadas por Meta.

**Lo que ya existe y se reutiliza:**

- Las campañas de push ya tienen todo el esqueleto: audiencias (todos, por
  empresa, por usuarios, por estado, filtros a medida como origen o salario),
  programación (inmediata, programada, recurrente) y estados (borrador,
  programada, enviando, enviada, pausada, cancelada, fallida).
- El backend ya manda WhatsApp con plantillas de Twilio (`WhatsAppSender`, con
  `contentSid`) y ya guarda un registro de mensajes (`wa_message_logs`).

**Lo que pidió:**

- Filtros como los de push, más uno nuevo: **con app / sin app / ambos**. Es la
  razón de ser del módulo: a quien no tiene la app no le llega un push.
- **Vista previa** antes de enviar.
- **Programar** envíos.
- Preferir plantillas de categoría **utility**.

**Lo que ya tenían pensado** (PDF "Envío masivo por WhatsApp", 02/10/2026):

- **"Sin app"** = empleados **activos** con teléfono que **no tienen token de
  push**, o sea, a los que no les llega ningún aviso por la app. Verificado
  contra la base el 03/10/2026: 650 activos, 420 con la app, **230 sin la app**
  (el PDF tenía 221 el 02/10; la diferencia son registros nuevos). "Con app" son
  los 420.
- **Costo por mensaje en Panamá:** utility **$0.0163**, marketing **$0.0790**
  (5 veces más). Una campaña a los 230 sale ~$3.75 como utility y ~$18 como
  marketing.
- **Plantillas utility** (informan algo de su propia cuenta): *Saldo
  disponible*, *Inicio de ciclo*, *Cierre de solicitudes*, *Descuento en
  planilla*.
- **Plantillas que caerían como marketing** (invitan o piden instalar algo):
  *Activación del servicio*, *Invitación a instalar la app*, *Programa de
  referidos*.

**Lo que eso implica para construirlo:**

- **Las variables se calculan por persona al momento de enviar**, no se
  escriben a mano: el saldo con el mismo cálculo que usan la app y el bot (así
  ninguno muestra otro número), y las fechas de cierre o de descuento según la
  empresa de cada uno.
- **"Saldo disponible" solo a quien tenga saldo para pedir** (al menos el
  mínimo de B/. 25). Mandarle "tienes $0 disponibles" a alguien es peor que no
  mandar nada.
- **"Consulta el detalle de tu cuenta"**: quien no tiene la app lo consulta
  respondiendo al mismo WhatsApp, porque el bot ya contesta el saldo. Encaja
  bien con esta audiencia.
- **"Descuento en planilla" es por solicitud, no una campaña:** cada persona
  tiene su monto y su fecha. Encaja mejor como un aviso automático cuando la
  solicitud se aprueba que como un envío masivo. A decidir.
- Fuera de los 230 hay otra gente sin la app que el PDF no cubre: **172
  registrados esperando aprobación** y **186 pre-registros** del kiosco. Son el
  público natural de *Activación del servicio* e *Invitación a instalar la app*
  (las de marketing).

**Propuesta para que quede completo:**

- Elegir la plantilla de una lista de las **ya aprobadas** en Twilio, con sus
  variables (nombre, empresa…). No hay texto libre: fuera de la ventana de 24 h
  de una conversación, WhatsApp solo permite plantillas aprobadas.
- Vista previa con los datos **reales** de una persona de la audiencia, más
  cuántos destinatarios son y el **costo estimado** antes de confirmar.
- Seguimiento por destinatario: enviado, entregado, leído, fallido.
- Envío por tandas, y que quien responda que no quiere recibir más quede fuera
  de los envíos siguientes.

**A tener en cuenta, antes de decidir:**

- **La categoría la decide Meta, no nosotros.** Una plantilla utility tiene que
  ser sobre algo de la persona (su registro, su saldo, su cuenta). Si el texto
  promociona algo ("descarga la app", "aprovecha"), Meta la clasifica como
  **marketing**, que cuesta más y tiene límites de cuántos mensajes de marketing
  puede recibir cada persona.
- **El número de WhatsApp es el mismo que manda los códigos de acceso y atiende
  el bot.** Si una campaña recibe muchos bloqueos o reportes, Meta baja la
  calidad del número y puede limitarlo: dejarían de llegar los códigos OTP y
  nadie podría entrar ni recuperar su clave. Por eso las tandas, la baja
  voluntaria y empezar con audiencias chicas. Vale la pena evaluar un número
  aparte para campañas.

Con 230 destinatarios el riesgo para el número es bajo, pero crece si se suman
pre-registros y registros pendientes, que nunca escribieron a ese número.

### 4a. Medidas contra bloqueos (pedido del dueño, 03/10/2026)

Meta no banea por volumen: baja la calidad del número cuando la gente
**bloquea o reporta**, y con calidad baja limita los envíos o suspende el
número. Todo apunta a que nadie tenga motivo para bloquear.

**Lo que más protege — fuera del módulo:**

1. **Número aparte para campañas — descartado por la regla de §4b.** Aislaría
   los códigos y el bot, pero la gente le escribió al número actual: un número
   nuevo le llegaría como un desconocido, que es justo lo que más se bloquea.
   Con la audiencia de §4b conviene mandar desde el **mismo número**, y cubrir
   los códigos con el plan B del punto 2.
2. **Plan B para los códigos, ya existe:** el canal del OTP sale de
   `OTP_CHANNEL` en el `.env` de pw-appbackend (hoy no está puesto, así que va
   por `whatsapp`). Con `OTP_CHANNEL=sms` los códigos salen por SMS sin tocar
   código. Falta confirmar que el servicio de Twilio Verify tenga SMS habilitado
   y probarlo **solo con el teléfono del dueño**.
3. **Consentimiento en el registro:** sumar al texto legal del registro (se edita
   en el panel y cada firma guarda versión) una línea tipo "acepto recibir por
   WhatsApp avisos sobre mi cuenta". Desde ahí queda registrado quién aceptó.
4. **Perfil del remitente completo:** nombre Payway, logo y descripción, para
   que la persona reconozca quién le escribe y no lo reporte como spam.

**Lo que el módulo hace solo, sin depender de quien envía:**

5. **Baja con un toque:** cada plantilla lleva un botón "No recibir más". Quien
   lo toque, o escriba BAJA/STOP, queda en una lista de exclusión que respetan
   todas las campañas siguientes. Es lo que más evita bloqueos: la gente bloquea
   cuando no encuentra cómo dejar de recibir.
6. **Tope de frecuencia por persona:** máximo 1 campaña por semana y 4 al mes
   (a confirmar). La audiencia descuenta a quien ya llegó al tope y la vista
   previa lo muestra.
7. **Tandas y horario:** envío en tandas (p. ej. 50 cada 10 minutos), solo de
   8:00 a 19:00 hora de Panamá. Lo programado fuera de horario espera.
8. **Freno automático:** si en una tanda fallan o rebotan más de un umbral
   (p. ej. 10 %), la campaña se pausa sola y avisa. Los números que fallan
   seguido quedan fuera de los envíos siguientes.
9. **Audiencia limpia:** solo personas con relación con Payway; nunca
   despedidos ni inactivos.
10. **Primera campaña de prueba chica:** 20–30 personas, revisar la calidad del
    número en Meta al día siguiente, y recién ahí la audiencia completa.

**Contenido de las plantillas:** sobre algo de la persona (su saldo, su ciclo),
con su nombre, sin lenguaje de promoción ni links acortados. Además de ser lo
que Meta acepta como utility, es lo que la gente no reporta.

### 4b. Audiencia: solo quien ya le escribió al número (regla del dueño, 03/10/2026)

El dueño fijó la regla: **WhatsApp solo a quien ya usó WhatsApp con Payway** —
se registró por WhatsApp, pidió un adelanto por WhatsApp, o le escribió al bot.
Es la medida más fuerte contra bloqueos: conoce el número y ya conversó con él.

**Dónde está ese dato.** La base solo sabe quién se registró
(`users.signup_source = 'whatsapp'`) o pidió (`orders.source = 'whatsapp'`)
por WhatsApp. **Los mensajes que la gente le manda al bot no se guardan**: el
bot solo usa caché para la conversación en curso. El historial completo está en
Twilio. Contado el 03/10/2026 desde Twilio (solo lectura): 9.007 mensajes
entrantes desde el 01/04/2026, de **398 teléfonos distintos**; 148 escribieron
en los últimos 30 días.

**Cruce con usuarios (03/10/2026)**, contando cualquiera de las tres formas:

| Estado | Con app | Sin app |
|---|---|---|
| Activos | 177 | **94** |
| Pendientes de aprobación | — | 34 |
| Despedidos | 21 | 9 |
| Inactivos | 9 | 5 |

Más 51 teléfonos que escribieron y no son usuarios. Ninguno es pre-registro.

**Qué cambia en las audiencias:**

- "Sin app" pasa de 230 a **94**. Los otros 136 activos sin app nunca le
  escribieron al número y quedan fuera.
- "Con app" que escribieron: 177. "Ambos": 271.
- Los 186 pre-registros quedan fuera: ninguno escribió.
- Los 34 pendientes de aprobación que escribieron podrían recibir *Activación
  del servicio*. A decidir.
- Despedidos e inactivos nunca, aunque hayan escrito.

**Cómo saberlo sin tocar el bot.** El dueño pidió no cambiar el flujo de
WhatsApp, así que el bot no se modifica. Una tarea programada lee cada noche de
Twilio quién escribió y guarda por teléfono la fecha del último mensaje
entrante (tabla nueva, primera carga desde el 01/04/2026). El módulo filtra
contra esa tabla. Sirve también para un filtro "escribió en los últimos N días".

**Decisiones del dueño (03/10/2026):**

1. Nombre: **Campañas WhatsApp**. Arriba de la tabla, un resumen de lo gastado
   a la fecha.
2. No se manda nada fuera del módulo: toda campaña la arma y la confirma una
   persona desde el panel (incluidos los 34 pendientes que escribieron).
3. "Descuento en planilla" **queda fuera**: no es marketing. Venía en el PDF del
   02/10 como plantilla utility.
4. Lo usa **solo super_admin**, desde el panel.
5. **Mismo número** de los códigos. Si la audiencia incluye a quien nunca usó
   WhatsApp con Payway, se muestra un **aviso de riesgo** antes de enviar, y
   cada campaña lleva un **nivel de riesgo verde / amarillo / rojo**.
6. Tope: **1 campaña por persona por semana**.
7. **Sin** línea de consentimiento nueva en el registro.

Orden fijado por el dueño: Campañas WhatsApp primero, por ser lo menos
invasivo; el resto espera su autorización. Las reglas completas están en
`modulos/campanas-whatsapp.md`.

**Hallazgo al arrancar (03/10/2026):** en Twilio no existe ninguna plantilla de
campaña. Solo están las de avisos sueltos (firma, cuenta aprobada, orden
aprobada, extraordinario aprobado), las del bot (sin enviar a aprobación) y las
de los códigos. Las de campaña se crean desde el módulo y esperan la
aprobación de Meta. Aparte: Meta aprobó `payway_account_approved` (el aviso de
cuenta aprobada) como **marketing**, así que cada aviso cuesta $0.079 en vez de
$0.016.

---

## 5. Cambiar de empresa a un empleado desde el panel — Publicado (05/10/2026), solo sin historial

> **Estado 05/10/2026:** en vivo en el panel admin. Mueve a quien no tiene
> deuda **ni solicitudes pagadas**: quien tiene historial queda bloqueado hasta
> la fase 2 (que cada solicitud guarde su empresa). Detalle, pruebas y cómo
> deshacer: `modulos/cambio-de-empresa.md`.

Pedido del dueño (03/10/2026): se lo piden seguido y hoy lo hace a mano.

**Qué implica hacerlo bien** (es lo que se hizo a mano con Ana Mejía, Grupo
Piazza → Minimed, en septiembre de 2026):

- Empresa del empleado (`company_users.company_id`) y su sub-company.
- La empresa y la sucursal en su ficha (`user_details`).
- Su ficha del panel Enterprise: el equipo de la empresa nueva, cruzado **por
  slug** (los ids de empresa de `app` y `hr` son distintos).
- Su pre-registro, si tiene.
- Dejar registro en la auditoría de quién lo movió, desde dónde y cuándo.

A Ana se la pudo mover a mano porque **no tenía salario cargado, ni cuentas, ni
solicitudes**. Un empleado con historia es otra cosa, y eso es lo que hay que
decidir.

**El punto delicado: las solicitudes no guardan de qué empresa eran.** La tabla
de solicitudes no tiene empresa; se deduce de la empresa **actual** del
empleado. Si hoy se mueve a alguien con historia:

- Todas sus solicitudes pasadas pasan a figurar en la empresa nueva: la empresa
  vieja deja de verlas en su panel Enterprise y en sus reportes, y la nueva ve
  transacciones que no le corresponden.
- Si tiene **adelantos sin descontar**, la empresa vieja es la que tiene que
  descontarlos de su planilla, pero deja de verlos.
- Lo mismo, más grave, con **adelantos extraordinarios** en cuotas.
- Su saldo pasa a calcularse con el ciclo de pago de la empresa nueva, desde el
  día del cambio.

**Propuesta:**

1. Guardar la empresa en cada solicitud al crearla, y completarla para las
   existentes con la empresa actual de cada empleado. Así el historial queda
   donde ocurrió aunque la persona cambie de empresa.
2. **No permitir el cambio mientras tenga deuda pendiente** (adelantos sin
   descontar o cuotas de extraordinarios), con un aviso claro en el panel; o, si
   se permite, que la deuda siga a cargo de la empresa vieja.
3. Al mover: elegir empresa, sub-company y sucursal de destino (con la regla de
   §2 y §3), y mostrar un resumen de lo que va a cambiar antes de confirmar.

**Decisiones del dueño (03/10/2026):**

1. Con deuda pendiente **se bloquea el cambio** y el panel le avisa al admin
   qué deuda lo impide.
2. Lo pueden hacer **super_admin y admin_ops**.

---

## 7. Saldo disponible: un solo cálculo — Hecho (03–04/10/2026); falta borrar las copias de respaldo

Pedido del dueño (03/10/2026): el saldo está repartido en varios ambientes y
hay que unificarlo. Preguntó si guardarlo en la base sería lo más estable.

**Dónde se calcula hoy (03/10/2026): cuatro copias en tres backends.**

| # | Dónde | Lo usa |
|---|---|---|
| 1 | pw-appbackend `HomeController::setBalance` | La app: el home y el rechazo de montos al pedir |
| 2 | pw-appbackend `WhatsAppChatbotController::getUserBalance` | El bot, cuando le preguntan el saldo |
| 3 | pw-adminbackend `Services/Salary/BalanceCalculator` | El panel, los filtros de push y Campañas WhatsApp |
| 4 | pw-hrbackend `EmployeeService::getEmployeeBalance` | El panel Enterprise |

(Una versión anterior de este documento contaba una quinta copia en el login
de pw-appbackend. No existe: era un comentario.)

Las copias 1 y 2 comparten piezas de `Support/BalanceCalculator`; la 3 y la 4
cuentan los días por su cuenta. Cada cambio de regla (17/09, 28/09, 30/09) hubo
que repetirlo en cada copia, y una copia olvidada da otro número.

**Foto de saldos del 03/10/2026, 23:00** (958 usuarios, 651 activos), tomada
con las cuatro copias. Es la línea de base: después de cada cambio se vuelve a
tomar y se compara.

| Copia | Activos que coinciden con la app (±1) | Suma del disponible |
|---|---|---|
| App | — | B/. 18.377 |
| Bot | 651 de 651 | B/. 18.377 |
| Panel | 649 de 651 | B/. 18.325 |
| Enterprise | 648 de 651 | B/. 18.313 |

- Los tres activos que difieren: #999 (app 39, panel y Enterprise 0), #4634
  (app 45, los otros 32) y #7472 (app 40, Enterprise 28). Hay que explicar cada
  uno antes de unificar.
- **No activos con saldo mayor que cero:** la app muestra 52, el bot 83, el
  panel y Enterprise 80. Las copias no tratan igual a despedidos, inactivos y
  pendientes.
- Las deducciones salen de un solo lugar para todos: la ficha de `hr`
  (`salary_component`, 310 empleados las tienen). La diferencia está en cómo
  se suman: la app descarta la clave `Basic` y los componentes de prueba; el
  panel y Enterprise no. De ahí salen #999 y #4634. #7472 es de honorarios y
  Enterprise le retiene igual el 30 %. En los tres, el número bueno es el de
  la app.

**Respaldo previo (03/10/2026, 22:58), pedido por el dueño:**
`/root/respaldo-antes-saldo-unificado-20261003-2258/` en pw-staging (código de
admin-api y enterprise-api, bases `app` y `hr`, fotos de saldos del panel y de
Enterprise) y en pw-backend (código de app-api, foto de la app y del bot).
Probado: el código extraído es idéntico al que está en vivo, y la copia de las
bases se restauró en bases de prueba con las 104 tablas y las mismas filas.

**Regla de trabajo (dueño, 03/10/2026):** probar varias veces y por todos los
canales, y guardar los saldos antes de cada cambio para comparar después.

**Propuesta: una sola fórmula, y un saldo guardado que todos leen.**

1. **Una sola fórmula, en pw-appbackend**, que es la que autoriza el retiro.
   Las otras copias se borran.
2. **Tabla `saldos`** en la base `app`, una fila por empleado: disponible,
   devengado, ya pedido, tope aplicado, días, ciclo, si está bloqueado y por
   qué (cierre de planilla, empresa congelada, despedido), `calculado_at` y la
   versión de la regla.
3. **Se recalcula por eventos:** cuando se crea, aprueba, cancela o rechaza una
   solicitud; cuando cambian el salario, las deducciones, la empresa o el
   porcentaje; cuando se congela o descongela la empresa. Además hay una pasada
   a las 00:05 para todos, porque el saldo sube cada día aunque nadie haga nada.
4. **Panel, Enterprise, bot, push y Campañas WhatsApp leen la tabla.** Ninguno
   calcula.
5. **Al pedir un adelanto, el backend recalcula en vivo** y no confía en la
   tabla, porque es el único momento en que un número viejo cuesta plata. Si el
   número en vivo difiere del guardado, se registra la diferencia: es la
   alarma de que alguna actualización se perdió.
6. **Historial diario** (`saldos_historial`, una foto por empleado y día) para
   contestar reclamos ("ayer la app me decía 80") y ver el efecto de cada
   cambio de regla.

**Por qué no alcanza con guardarlo:** guardar el saldo sin una fórmula única
solo mueve el problema. Si dos procesos escriben la tabla con reglas
distintas, el número depende de quién escribió último. La estabilidad la da la
fórmula única; la tabla suma velocidad, una sola lectura para todos y
trazabilidad.

**Alternativas descartadas:**

- *Que panel y Enterprise le pregunten a pw-appbackend por API.* Ata los
  servidores entre sí: si pw-backend se cae, el panel se queda sin saldos. La
  tabla los desacopla.
- *Un paquete PHP compartido por los tres repos.* Cada cambio de regla sigue
  siendo tres despliegues, y un servidor sin actualizar vuelve a dar otro
  número.

**Orden sugerido:** (1) la tabla y el llenado, sin que nadie la lea todavía;
(2) una semana comparando la tabla contra el cálculo en vivo de cada canal;
(3) pasar panel, Enterprise, bot y campañas a leer la tabla; (4) borrar las
copias. Es solo backend: no necesita versión de la app.

**Decisiones pendientes:**

1. ¿Entra en este sprint o en el siguiente?
2. ¿Se resuelve antes la contradicción del 93 % de la quincena? Conviene: si
   no, se guarda en la tabla una regla que después hay que volver a cambiar.

---

## 6. Hecho, sin publicar — sale con la versión de la app

- **Recuperar clave:** traba contra doble toque y el reenvío del código ya no
  apila pantallas (pw-mobileapp `783801e`, 18/09). Parte de la respuesta al abuso
  de códigos OTP.

---

## Pendientes de la app anotados por el dueño

- **Botón de iniciar sesión (05/10/2026):** al tocarlo debe decir "Iniciando
  sesión…" y quedar deshabilitado hasta que responda el servidor, para que no
  se toque dos veces. Va con la próxima versión de la app.

## Por revisar antes de construir

- **Registro por WhatsApp:** cómo pide hoy empresa y sucursal.
- **Kiosco:** hoy el captador elige empresa y sucursal; con sub-companies tendría
  que elegir empresa → sub-company → sucursal.
- **Alcance Enterprise por sucursal:** confirmar que siga funcionando con
  sucursales que cuelgan de una sub-company.
- **Campañas WhatsApp:** qué plantillas hay aprobadas hoy en Twilio y de qué
  categoría; si los empleados aceptaron recibir mensajes por WhatsApp al
  registrarse.

---

## Anotados antes, fuera de este paquete salvo que el dueño los sume

- Saldo: la quincena llega a 93 %, no a 100 % (contradicción entre lo pedido el
  17/09 y el 30/09). Hay una salida propuesta en `modulos/saldo-disponible.md`.
  Es solo backend.
- Varias solicitudes vivas a la vez del mismo empleado (caso Vivian, 21/09):
  nada lo impide hoy. El tope de saldo evita que retire de más, pero cobra una
  comisión por cada una.
- Resúmenes abandonados en estado `ongoing` (34 empleados, algunos desde marzo):
  limpiarlos.
- Referidos (anotado el 14/09): campo *"¿Quién te invitó?"* en el kiosco, push
  por FCM de los avisos, anunciar el programa.
- Textos legales del acuerdo de extraordinarios y del consentimiento: pendientes
  de revisión con abogado.
