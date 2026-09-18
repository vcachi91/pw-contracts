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
hasta el 17/09/2026 (ver más abajo).

- Las retenciones de ley son 30%, salvo `honorarios` (sin retención).
- `share_of_basic` es por empresa; `percent_permit` del usuario la pisa si es > 0.
- Sobre el disponible aplica además el **tope por transacción de B/. 200** y el
  **mínimo de B/. 25**. El tope de 200 es la razón de que el mínimo de un
  adelanto extraordinario sea 201 (ver `adelantos-extraordinarios-api-app.md`).

Todo vive en `app/Support/BalanceCalculator.php`. **Ningún consumidor recalcula
nada por su cuenta**: el home de la app y el bot de WhatsApp llaman ahí. Si se
duplica, los dos canales terminan mostrándole al mismo usuario dos saldos
distintos.

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
