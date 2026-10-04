# Saldo disponible (EWA) — cómo se devenga

Qué ve el empleado en el home de la app y qué le contesta el bot de WhatsApp
cuando pregunta cuánto tiene. **Es el mismo número y sale del mismo cálculo.**

Repos que lo tocan: `pw-appbackend` (lo calcula), `pw-mobileapp` (solo lo
pinta). La app **no calcula nada**: recibe `balance.amount` y lo muestra. Por eso
cualquier cambio acá se despliega —y se revierte— sin publicar la app.

## La fórmula

```
neto quincenal   = (salario / 2) − retenciones de ley − deducciones registradas
disponible       = (neto quincenal / días del período) × días devengados × (share_of_basic / 100)
                   − lo ya solicitado en el ciclo
```

**Días del período** = del cierre actual al próximo: 15 o **16**, porque los
cierres están clavados al 15 y al 30 y los meses no miden lo mismo. Era 15 fijo
hasta el 17/09/2026 (ver más abajo). **Los tres backends usan el período real
desde el 28/09/2026**; `pw-hrbackend` seguía con 15 fijo + 1 hasta ese día.

- Las retenciones de ley son 30%, salvo `honorarios` (sin retención).
- `share_of_basic` es por empresa; `percent_permit` del usuario la pisa si es > 0.
- Sobre el disponible aplica además el **tope por transacción de B/. 200** y el
  **mínimo de B/. 25**. El tope de 200 es la razón de que el mínimo de un
  adelanto extraordinario sea 201 (ver `adelantos-extraordinarios-api-app.md`).

La regla vive en `pw-appbackend/app/Support/BalanceCalculator.php` y es la
fuente de verdad. **Hoy está implementada tres veces** (cuatro contando el bot, que comparte piezas con la app; ver "Saldo unificado" al final) y hay que tocarlas
juntas o el mismo empleado ve un número distinto en cada canal:

| Dónde | Archivo | Quién lo ve |
|---|---|---|
| `pw-appbackend` | `app/Support/BalanceCalculator.php` | la app y el bot de WhatsApp |
| `pw-adminbackend` | `app/Services/Salary/BalanceCalculator.php` | panel Admin: ficha del empleado y segmentación de push |
| `pw-hrbackend` | `app/Services/EmployeeService.php` (`getEmployeeBalance`) | panel Enterprise: la empresa cliente |

El 28/09/2026 las tres estaban dando números distintos para el mismo día: la
app y Admin contaban un día de más y Enterprise uno de menos. Unificar esto en
un solo servicio compartido sigue pendiente.

## Días devengados (30/09/2026 — revertida la parte del día del cierre)

El 30/09, primer día de cierre después del cambio anterior, David reportó a las
7:26 a.m.: **"la gente tiene saldo y está pidiendo, no deberían tener"**. Tenía
razón: ese día todos amanecieron con 15/15, el 100% de su medio salario, y el
bloqueo previo al cierre no cubre el día del cierre mismo. Entraron 8
solicitudes, 6 las rechazó un admin a mano y 2 quedaron abandonadas; no se
desembolsó nada.

**Qué se revirtió:** solo la selección del ciclo. El día del cierre vuelve a
pertenecer al ciclo nuevo y por lo tanto acredita 0.

**Qué se conservó:** el conteo de días, que es lo que el dueño pidió el 28/09.
El día siguiente al cierre vale 1 y el 2 del mes vale 2.

| fecha | ciclo | días | % del medio salario |
|---|---|---|---|
| 30 sep (cierre) | cierre 30 sep | 0 | 0% |
| 1 oct | cierre 30 sep | 1 | 7% |
| 2 oct | cierre 30 sep | 2 | 13% |
| 14 oct | cierre 30 sep | 14 | 93% |
| 15 oct (cierre) | cierre 15 oct | 0 | 0% |

**Los dos pedidos se contradicen y hay que resolverlo.** El 17/09 el dueño pidió
que la quincena cerrara en 100%; el 30/09 pidió que el día del cierre no haya
saldo. Si el 100% solo se alcanza el día del cierre, no se cumplen los dos.

**Salida pendiente de aplicar:** repartir el medio salario entre los días
**usables** (14 en un período de 15, 15 en uno de 16) en vez de entre los días
calendario. Así el día anterior al cierre llega a 100%, el día del cierre queda
en 0 y el conteo diario no cambia. Con el ejemplo del dueño, si la quincena son
140 y hay 14 días usables, cada día vale 10 exacto.

**Para revertir esta reversión:** `pw-backend:/root/backup-revert-seleccion-*.tar.gz`,
`pw-staging:/root/backup-revert-seleccion-admin-*.tar.gz` y
`pw-staging:/root/backup-revert-seleccion-ent-*.tar.gz`.

## Días devengados (28/09/2026 — el día del cierre cierra la quincena anterior)

Decisión del dueño, con su propio ejemplo: **"si gano 10 al día, el 2 de octubre
debo tener 20 disponible"**, y ese mismo día, no al cierre del día.

> **El día del cierre acredita 0 días de la quincena nueva. El día siguiente
> vale 1, el 2 del mes vale 2, el 17 vale 2.**
>
> Y el día del cierre **no se pierde**: pertenece a la quincena que cierra, así
> que ese día llega a 15/15 (o 16/16), el 100% del medio salario.

Las dos piezas van juntas y por eso se cambiaron a la vez:

1. `diasDevengados` ya no suma 1 cuando cuenta desde el cierre. Sí sigue sumando
   1 cuando cuenta desde un **alta a mitad de ciclo**, porque el día que la
   persona entró es un día trabajado.
2. **La selección del ciclo** pasa a tomar el cierre más reciente
   *estrictamente* anterior a hoy. Sin esto, el 30 arrancaba el ciclo nuevo en 0
   y el empleado nunca veía el 100%: cerraba en 14 de 15.

| fecha | ciclo | días | % del medio salario |
|---|---|---|---|
| 16 sep | cierre 15 sep | 1 | 7% |
| 17 sep | cierre 15 sep | 2 | 13% |
| 29 sep | cierre 15 sep | 14 | 93% |
| **30 sep** (cierre) | **cierre 15 sep** | **15** | **100%** |
| 1 oct | cierre 30 sep | 1 | 7% |
| 2 oct | cierre 30 sep | 2 | 13% |
| 15 nov | cierre 30 oct, período de 16 | 16 | 100% |

**Qué se acepta a cambio:** todos los empleados ven un día menos que con la
regla del 17/09, o sea unos 7 puntos porcentuales menos de disponible a media
quincena, y el día en que cruzan el mínimo de B/. 25 se corre un día.

**Efecto de borde ya conocido:** 13 de las 24 empresas bloquean solicitudes
entre 3 y 7 días antes del cierre (`log_period_before`), así que para ellas el
100% del día del cierre no es visible. Las otras 11 no tienen bloqueo y ahí sí
se usa.

**Para revertir:** `pw-backend:/root/backup-dias-devengados-*.tar.gz`,
`pw-staging:/root/backup-dias-admin-*.tar.gz` y
`pw-staging:/root/backup-dias-enterprise-*.tar.gz`.

## Días devengados (17/09/2026 — el día del cierre ya acredita)

Decisión del dueño, a pedido de David. **El día del cierre acredita 1 día** (antes
0) y el divisor pasó a ser la **duración real del período**. Las dos cosas van
juntas: con el +1 y el divisor en 15, los períodos de 16 días acreditarían 107%
del medio salario.

| | antes | ahora |
|---|---|---|
| día del cierre | 0 | **1** |
| cruza el mínimo de $25 | cierre+3 | **cierre+2** |
| cierre de quincena de 15 días | 14/15 = 93% | **15/15 = 100%** |
| cierre de quincena de 16 días | 15/15 = 100% | **16/16 = 100%** |

**Qué se acepta a cambio:** el día del cierre pertenece a la planilla que la
empresa está por pagar, no a la quincena nueva. Se adelanta sobre un salario que
el empleador deposita en días, mientras el descuento nuestro cae en la planilla
siguiente. No hay ventana que lo contenga: `log_period_after` está en **0** en
las 17 empresas activas.

**Impacto medido el día que se aplicó** (17/09, día 2 del ciclo): el disponible
total de la planilla activa pasó de B/. 10,345 a B/. 15,326 (+48%) y los
empleados que cruzan el mínimo, de 85 a 371. El salto es grande porque al inicio
del ciclo un día es mucho; al final del ciclo el efecto es ~7%.

**Para revertir:** respaldos en `pw-backend:/root/bk-saldo-diaA-*.tgz` y
`pw-staging:/root/bk-saldo-admin-*.tgz`. No hace falta publicar la app: el saldo
lo calcula el backend y la app solo lo pinta.

**Tercer consumidor:** `pw-adminbackend/app/Services/Salary/BalanceCalculator.php`
se había quedado con la regla vieja desde el 27/08 y mostraba un día menos que
la app (lo usan la ficha del empleado y la segmentación de push). Quedó alineado
el 17/09.

## Días devengados (27/08/2026 — se corrigió un off-by-one)

El período de una quincena va **desde el día siguiente al cierre hasta el
próximo cierre**: 15 días. El conteo es:

| día del ciclo | ejemplo (cierre 15 ago) | días devengados |
|---|---|---|
| d0 — cierre | 15 ago | 0 |
| d1 | 16 ago | 1 |
| d2 | 17 ago | 2 |
| **d3** | 18 ago | **3** ← acá el saldo cruza el mínimo de $25 |
| … | | |
| d14 | 29 ago | 14 |

Antes de esta corrección, d0 **y d1** daban 0 y el máximo era 13 de 15: dos días
muertos al inicio, y el empleado nunca alcanzaba a devengar su quincena
completa. La causa era un `cycleStart->addDay()` agregado para que el día de
pago diera 0 — cosa que ya resolvía el `$absolute=false` de `diffInDays`, así
que era redundante y costaba un día entero durante todo el ciclo.

**Efecto medido:** el volumen de solicitudes arranca a los 3 días del cierre en
vez de 4. Sobre junio–agosto 2026 el salto era siempre en cierre+4 (154
transacciones contra 26 en cierre+3), sin una sola excepción. El disponible
promedio de la planilla activa subió **~9%**.

El empleado que entra a mitad de ciclo devenga desde su fecha de alta y solo por
días completos. Eso **no** cambió.

## Por qué "el día 3 del mes" no era el problema

Se investigó porque la quincena del 1 al 15 de agosto parecía arrancar antes que
las demás. No arrancaba antes: el comportamiento era idéntico en todos los
ciclos (siempre cierre+4). Lo que cambiaba era **en qué día del calendario caía
eso**, porque el cierre está clavado al día 30 y los meses no miden lo mismo:

- cierre 30 jun (junio tiene 30 días) → cierre+4 = **4 de julio**
- cierre 30 jul (julio tiene 31 días) → cierre+4 = **3 de agosto**

Se descartó mover las fechas de cierre para emparejarlo: `log_period_before`
cuenta hacia atrás desde el **próximo** cierre, así que adelantar el cierre
adelanta también la ventana de bloqueo. Se gana un día al inicio y se pierde
uno al final — se mueve la ventana, no se agranda.

## Ventana de bloqueo

`companies.log_period_before` bloquea los N días **previos** al próximo cierre;
`log_period_after`, los N días **posteriores** al cierre anterior. Hoy
`log_period_after` está en 0 en todas las empresas activas; `log_period_before`
va de 3 a 8 según la empresa. Por eso las solicitudes se apagan hacia el d11–d14
y no por un problema de acumulación.

## Saldo unificado: una sola fórmula y la tabla `saldos` (desde 03/10/2026)

Pedido del dueño (03/10/2026): el saldo estaba calculado en cuatro lugares (el
Home de la app, el bot, el panel y Enterprise) y cada cambio de regla había
que repetirlo en todos. Se unifica **por etapas**, sin publicar la app y sin
que cambie ningún número hasta haberlo comparado.

**La fórmula única** vive en `pw-appbackend/app/Support/Saldo/CalculadoraSaldo.php`.
Reproduce rama por rama lo que hoy devuelve el Home, incluso lo raro (ver
"Rarezas heredadas"). Si una regla del saldo cambia, se cambia ahí y se sube
`CalculadoraSaldo::REGLA`.

### Etapas

| # | Qué | Estado |
|---|---|---|
| 1 | `CalculadoraSaldo`, tablas `saldos` y `saldos_historial`, `saldos:recalcular` cada 10 minutos. **Nadie lee la tabla.** | Construida el 03/10/2026 |
| 2 | Una semana de `saldos:verificar` (cada hora) sin diferencias, más la comparación contra el panel y Enterprise. | Pendiente |
| 3 | El Home y el bot llaman a `CalculadoraSaldo`; el panel, Enterprise, push y Campañas WhatsApp leen `saldos`. Recálculo al crear o cambiar una solicitud. | Pendiente |
| 4 | Borrar las copias viejas. | Pendiente |

**Regla de trabajo del dueño:** antes de cada cambio se guarda una foto de
los saldos de todos los empleados en todos los canales, y después se compara.

### Tabla `saldos` (base `app`), una fila por empleado

| Columna | Qué es |
|---|---|
| `disponible` | Lo que la app muestra como saldo. 0 en cierre de planilla y para un despedido. |
| `para_solicitud` | `disponible` recortado al tope por solicitud (B/. 200). |
| `puede` | Puede pedir ahora: cuenta activa, sin ningún bloqueo y con el mínimo (B/. 25). **Es la columna que deben usar los lectores.** |
| `puede_app` | `can_progress` tal como lo devuelve hoy el Home. Solo sirve para comparar en la etapa 2. |
| `bloqueo` | Por qué no puede, en una palabra (ver abajo). `null` si puede. |
| `aviso` | El texto que muestra la app. |
| `dias_devengados`, `dias_periodo`, `ciclo_inicio`, `proximo_cierre` | Con qué días se calculó. |
| `salario_bruto`, `neto_quincenal`, `deducciones`, `porcentaje`, `ya_solicitado` | Con qué plata se calculó. |
| `regla` | Versión de la regla. |
| `calculado_at` | Cuándo. Un lector que encuentre un valor de hace más de 30 minutos debe desconfiar. |

`bloqueo`, en orden de prioridad: `despedido` · `no_activo` (inactivo o
pendiente) · `sin_aprobar` (sin empresa asignada) · `sin_perfil` ·
`sin_ciclo` · `cierre_planilla` · `empresa_congelada` · `bajo_minimo` · `otro`.

`saldos_historial` guarda una fila por empleado y día: el primer saldo del día
(`disponible_inicio`) y el último (`disponible`).

### Al pedir un adelanto se calcula en vivo

La tabla es para **mostrar**. `RequestController` sigue calculando en el
momento (`HomeController::limitesParaSolicitar`) y nunca autoriza plata con un
valor guardado.

### Rarezas heredadas (se reproducen igual; candidatas a limpiarse en la etapa 4)

- **Bloqueo viejo** (`Controller::checkLockPeriodBeforeAfter`): arma una
  ventana alrededor del último ciclo cargado más un mes. Con 24 meses de ciclos
  cargados casi nunca cae en hoy. El bloqueo real es el de cierre de planilla.
- **"Ya solicitado"** suma las solicitudes desde el cierre hasta un mes
  después, no hasta el próximo cierre.
- **El Home no mira la empresa congelada**: muestra saldo y deja entrar a
  pedir, y recién `RequestController` rechaza. En la tabla ya sale como
  `bloqueo = empresa_congelada` y `puede = 0`.
- **Inactivos y pendientes con empresa asignada** tienen un saldo calculado
  (38 inactivos y 14 pendientes el 03/10/2026) aunque no pueden entrar a la
  app. En la tabla quedan con `puede = 0` y `bloqueo = no_activo`.

### Qué mostró la línea de base (03/10/2026, 958 usuarios, 651 activos)

- App y bot: iguales en los 651 activos.
- Panel: 649 de 651. Enterprise: 648 de 651.
- Las tres diferencias son errores del panel y de Enterprise, no de la app:
  dos empleados (#999, #4634) tienen en `salary_component` la clave `Basic`,
  que es un ingreso, y el panel y Enterprise la restan como deducción; uno
  (#7472) es de honorarios y Enterprise le retiene igual el 30 %. La app ya
  maneja los dos casos (`BalanceCalculator::deduccionesDesdeComponentes` e
  `isHonorarios`). Al pasar a leer la tabla, esos tres suben al número de la app.
- El bot le calcula saldo a 31 despedidos, pero nunca se lo muestra: solo
  atiende a cuentas activas.

### Pruebas de la etapa 1 (03/10/2026), antes de desplegar

`CalculadoraSaldo` contra el Home y contra el bot, para los 958 usuarios:
0 diferencias hoy y 0 en cada una de diez fechas simuladas (16/09, 29/09,
30/09, 01/10, 14/10, 15/10, 16/10, 30/10, 31/10 y 15/11), que cubren días de
cierre, el día siguiente y quincenas de 15 y de 16 días.
