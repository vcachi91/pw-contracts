# Próximo paquete — sprint de la próxima versión de la app

Documento vivo: se va anotando lo que se acuerda con el dueño hasta cerrar el
sprint. Todo lo de acá sale **junto, en una sola versión de la app**, en vez de
varias sueltas (así se lo dijo el dueño a Pablo el 02/10/2026).

Cuando un ítem pase a construirse, su regla detallada va como contrato en
`modulos/`; acá queda el resumen y el estado.

Estados: **Acordado** · **Pendiente de decisión** · **Hecho, sin publicar**.

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
  contáctenos. Hoy el backend se la vincula **ya verificada**, saltándose a
  operaciones: un error de tipeo podía mandar un adelanto a la cuenta de otro.
  Se arregla antes de reactivar nada.

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
sub-company no puede tener las suyas. Sub-company y sucursal son conceptos
distintos y siguen siéndolo (decisión de junio): sub-company es una empresa del
grupo o marca; sucursal es un local físico. Lo nuevo es que una sucursal pueda
pertenecer a una sub-company.

**Reglas acordadas:**

- Una sucursal pertenece a una empresa y, opcionalmente, a una de sus
  sub-companies.
- Quien pertenece a una sub-company que tiene sucursales propias ve **solo las
  de esa sub-company** (Pablo: "solo de la subempresa").
- Quien pertenece a una sub-company **sin** sucursales propias, o a una empresa
  sin sucursales, ve la **sucursal por defecto** (ver §3).

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

## 3. Sucursal obligatoria, con sucursal por defecto — Acordado

**Reglas acordadas (03/10/2026):**

- La sucursal es **obligatoria en todos los canales**: app, kiosco, panel Admin y
  WhatsApp.
- Toda empresa o sub-company que no tenga sucursales propias tiene una
  **sucursal por defecto**, así lo obligatorio nunca bloquea un registro.

**Propuesta de diseño (a confirmar en §5):** si un registro llega sin sucursal
—una versión vieja de la app, o un canal que todavía no la pide—, el backend le
asigna la sucursal por defecto en vez de rechazarlo. Así el dato queda completo
desde el primer día y nadie se queda trabado mientras se actualizan los canales.

### 3a. Cargar sucursales a las empresas que no tienen

Con la sucursal por defecto, cargar las reales ya no bloquea nada, pero el
dueño quiere hacerlo antes del cambio. Estado al 03/10/2026:

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

## 4. Hecho, sin publicar — sale en esta versión

- **Recuperar clave:** traba contra doble toque y el reenvío del código ya no
  apila pantallas (pw-mobileapp `783801e`, 18/09). Parte de la respuesta al abuso
  de códigos OTP.

---

## 5. Pendiente de decisión del dueño

**De este paquete:**

1. **Nombre de la sucursal por defecto:** *"Sucursal Central"* o *"Sin
   sucursal"*. Recomendación: *"Sin sucursal"*. *"Central"* aparenta un local
   real, y en el tablero se leería como si cientos de personas trabajaran en una
   sola sucursal; *"Sin sucursal"* deja a la vista a quién falta ubicar.
2. **Cuando una empresa reciba sus sucursales reales**, ¿la de por defecto
   desaparece de los selectores? Recomendación: sí, se oculta para registros
   nuevos; quien ya esté en ella la conserva hasta que operaciones lo reasigne.
3. **Empleados actuales sin sucursal** (la mayoría): ¿se les asigna la de por
   defecto automáticamente? Recomendación: sí, para que los reportes cuadren y
   lo obligatorio sea parejo.
4. **Registro sin sucursal** (§3): ¿se asigna la de por defecto o se rechaza?
   Recomendación: asignar.
5. **Mensaje y contacto** cuando se bloquea agregar una cuenta (§1) o se rechaza
   una cuenta ajena. Propuesta: el mismo WhatsApp de operaciones del mensaje de
   empresa pausada (`wa.me/50769958313`).

**Anotados antes, fuera de este paquete salvo que el dueño los sume:**

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

---

## Por revisar antes de construir

- **Registro por WhatsApp:** cómo pide hoy empresa y sucursal (no revisado
  todavía).
- **Kiosco:** hoy el captador elige empresa y sucursal; con sub-companies tendría
  que elegir empresa → sub-company → sucursal.
- **Alcance Enterprise por sucursal:** confirmar que siga funcionando con
  sucursales que cuelgan de una sub-company.
