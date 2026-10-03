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
| 1 | Switch de cuentas bancarias por empresa y empleado | Acordado | Sí |
| 2 | Sucursales por sub-company (+ bug de ids) | Acordado | Sí |
| 3 | Sucursal obligatoria con "Sin sucursal" | Acordado | Sí (el backend ya cubre a las apps viejas) |
| 4 | Menú "Marketing" + Campañas de WhatsApp | Pendiente de decisión | No: backend y panel |
| 5 | Cambiar de empresa a un empleado desde el panel | Pendiente de decisión | No: backend y panel |
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

## 2. Sucursales por sub-company — Acordado

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
  sin sucursales, ve **"Sin sucursal"** (ver §3).

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

## 3. Sucursal obligatoria, con "Sin sucursal" — Acordado

**Reglas acordadas (03/10/2026):**

- La sucursal es **obligatoria en todos los canales**: app, kiosco, panel Admin y
  WhatsApp.
- Toda empresa o sub-company sin sucursales propias tiene una sucursal por
  defecto llamada **"Sin sucursal"**, así lo obligatorio nunca bloquea un
  registro.
- **Si un registro llega sin sucursal** (una app vieja, o un canal todavía sin
  actualizar), el backend le asigna "Sin sucursal" en vez de rechazarlo.
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

## 4. Menú "Marketing" y Campañas de WhatsApp — Pendiente de decisión

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

**Decisiones pendientes:**

1. Nombre del módulo: ¿"Campañas WhatsApp"?
2. ¿Se suman como audiencias los **172 registrados esperando aprobación** y los
   **186 pre-registros**, o solo empleados activos?
3. ¿"Descuento en planilla" va como aviso automático al aprobar la solicitud, o
   como campaña?
4. ¿Quién puede enviar? (¿solo super admin, como los textos legales?)
5. ¿Número aparte para campañas o el mismo de los códigos?

---

## 5. Cambiar de empresa a un empleado desde el panel — Pendiente de decisión

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

**Decisiones pendientes:**

1. ¿Se bloquea el cambio con deuda pendiente, o se permite y la deuda queda con
   la empresa vieja?
2. ¿Quién puede hacerlo? (¿admin de operaciones o solo super admin?)

---

## 6. Hecho, sin publicar — sale con la versión de la app

- **Recuperar clave:** traba contra doble toque y el reenvío del código ya no
  apila pantallas (pw-mobileapp `783801e`, 18/09). Parte de la respuesta al abuso
  de códigos OTP.

---

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
