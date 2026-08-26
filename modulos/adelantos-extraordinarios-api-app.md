# Adelantos Extraordinarios (financiados por el EMPLEADOR) — Contrato API para Backend

> **La app ya tiene el módulo construido** contra este contrato. Mientras el backend no
> implemente `GET /extraordinary-advances/eligibility` (o responda `enabled=false`), el módulo
> es INVISIBLE para el usuario: ni la tarjeta del Home ni el ítem del menú aparecen. No hay
> riesgo en publicar la app antes que el backend.

## Qué es

Un nuevo tipo de adelanto, distinto del estándar (EWA financiado por Payway, `/requests/*`,
que NO se toca):

1. Lo financia la **empresa** (empleador), no Payway.
2. El monto puede superar el tope del adelanto estándar.
3. Requiere **aprobación explícita del empleador** en el panel Enterprise.
4. Se amortiza por **descuento de planilla** en cuotas con calendario generado por el sistema.
5. Genera documentos legales y exige **aceptación/firma electrónica** con trazabilidad.

## Reglas legales (fuente de verdad: BACKEND, siempre)

Basado en Art. 161 del Código de Trabajo de Panamá y Ley 64 de 1961. La app NO calcula nada
de esto; solo muestra lo que el backend devuelve.

- **Cuota ≤ 15% del salario devengado** por período de pago. El backend calcula la cuota
  máxima y de ahí el plazo mínimo para el monto pedido.
- **Total de deducciones ≤ 50% del salario** (este adelanto + otras deducciones registradas,
  salvo pensiones alimenticias). Validar el margen ANTES de permitir cotizar/aprobar.
- **Diciembre excluido:** el generador del calendario salta diciembre automáticamente y la
  fecha fin se corre. Marcar `skips_december=true` cuando ocurra (la app muestra el aviso).
- **Firma obligatoria:** el descuento solo es válido con documento firmado. La app envía la
  firma (PNG base64), la versión del template aceptado y datos del dispositivo; el backend
  agrega timestamp del servidor, IP y hash del documento, y guarda la evidencia inmutable.
- **Terminación laboral:** el acuerdo incluye la cláusula de descuento del saldo sobre el
  último salario; el backend debe poder liquidar el saldo en ese evento.
- Montos y salarios en **DECIMAL, nunca float**. En el JSON viajan como **string** ("150.00");
  la app los muestra tal cual, sin convertirlos.

## Estados canónicos

```
pending_approval → approved → disbursed → in_amortization → settled
                 ↘ rejected
pending_approval → cancelled   (solo el empleado, solo mientras está pendiente)
```

Cada respuesta incluye `status` (canónico, para la lógica) y `status_label` (texto en español
listo para mostrar; la app tiene fallbacks pero el backend manda).

## Endpoints

Todos autenticados (Bearer). Prefijo `/api/v1`.

### 1. `GET /extraordinary-advances/eligibility`

Decide si el módulo se muestra y con qué parámetros.

```json
{ "data": {
    "enabled": true,
    "disabled_reason": null,
    "salary_per_period": "800.00",
    "period_type": "quincenal",
    "max_installment_amount": "120.00",
    "available_margin": "280.00",
    "min_amount": "100.00",
    "max_amount": "2000.00",
    "has_active_advance": false
}}
```

- `enabled=false` + `disabled_reason` → la app muestra ese texto ("Tu empresa aún no habilitó
  este beneficio…"). Un **404 equivale a enabled=false** (así funciona hoy sin backend).
- `max_installment_amount` = 15% de `salary_per_period`.
- `available_margin` = lo que queda libre bajo la regla del 50% contando las deducciones
  actuales del empleado.
- `has_active_advance=true` bloquea nuevas solicitudes en la app (un adelanto a la vez).

### 2. `POST /extraordinary-advances/quote`

Request: `{ "amount": 600 }` (entero en balboas; opcionalmente `"installments_count"` si en el
futuro se deja elegir plazo mayor al mínimo).

```json
{ "data": {
    "amount": "600.00",
    "installments_count": 5,
    "installment_amount": "120.00",
    "first_discount_date": "2026-09-15",
    "last_discount_date": "2026-11-30",
    "skips_december": false,
    "schedule": [
      { "number": 1, "date": "2026-09-15", "period_label": "1ra quincena de septiembre", "amount": "120.00" }
    ],
    "agreement_body": "Yo, {{nombre}}, reconozco la deuda...",
    "agreement_version": "v3-cliente-acme"
}}
```

- Validaciones aquí (422 con `message` claro): monto fuera de rango, margen del 50%
  insuficiente, adelanto activo existente.
- `agreement_body` es el **texto legal ya renderizado** (template editable por cliente,
  aprobado por su abogado, con los datos del empleado y de esta cotización interpolados).
  La app lo muestra tal cual en la pantalla de firma. `agreement_version` identifica la
  versión del template y **vuelve en el submit** para la evidencia.

### 3. `POST /extraordinary-advances/submit`

Request:
```json
{
  "amount": 600,
  "installments_count": 5,
  "signature": "<base64 PNG>",
  "agreement_version": "v3-cliente-acme",
  "device_info": "android samsung SM-A525F",
  "app_version": "1.3.9"
}
```

- El backend re-valida TODO (nunca confiar en la app), re-genera el calendario, persiste la
  solicitud en `pending_approval` y guarda la **evidencia de firma**: signature, versión y
  hash del documento, timestamp del servidor, IP, device_info. Auditoría inmutable.
- Responde el objeto solicitud (mismo shape que el detalle, abajo).
- Notifica al aprobador del panel Enterprise (rol específico, no cualquier usuario).

### 4. `GET /extraordinary-advances` (lista) y `GET /extraordinary-advances/{id}` (detalle)

```json
{ "data": [ {
    "id": 12,
    "amount": "600.00",
    "status": "in_amortization",
    "status_label": "En amortización",
    "requested_at": "2026-08-25",
    "resolved_at": "2026-08-27",
    "rejection_reason": null,
    "installments_total": 5,
    "installments_paid": 2,
    "installment_amount": "120.00",
    "balance": "360.00",
    "next_installment_date": "2026-10-15",
    "next_installment_amount": "120.00",
    "schedule": [
      { "number": 1, "date": "2026-09-15", "period_label": "1ra quincena de septiembre", "amount": "120.00", "status": "paid" },
      { "number": 3, "date": "2026-12-15", "period_label": "diciembre (exento)", "amount": "0.00", "status": "skipped" }
    ]
}]}
```

La lista puede omitir `schedule` (la app la vuelve a pedir en el detalle).
`schedule[].status`: `pending | paid | skipped` (skipped = diciembre).

### 5. `PUT /extraordinary-advances/cancel/{id}`

Solo válido en `pending_approval`; cualquier otro estado → 422 con mensaje.

## Notificaciones (reutilizar la infra existente del flujo estándar)

Push + correo + WhatsApp en cada cambio de estado: aprobada, rechazada (con motivo),
desembolsada, recordatorio de cuota próxima, saldada. El deep link de la push puede abrir la
app en el módulo (hoy la app navega desde la tarjeta del Home).

## Datos (sugerencia, ajustar al esquema real)

- Evaluar `advance_requests.type = standard|extraordinary` + `funded_by = payway|employer`
  vs tabla nueva — decidir mirando el esquema actual de `payway-app`.
- `amortization_schedules` + `amortization_installments` (cuota, fecha, estado, período).
- `advance_agreements` (doc, versión template, hash, firma, timestamp, IP, device).
- Bitácora de TODOS los cambios de estado (quién, cuándo, desde dónde).

## PDFs que genera el backend (no la app)

1. Solicitud de anticipo. 2. Acuerdo de amortización (tabla de cuotas, validación 15%,
cláusula de último salario — template editable por cliente). 3. Constancia de aceptación
electrónica. 4. Comprobante de desembolso / estado de cuenta.

Cuando existan, exponer URLs de descarga en el detalle (`documents: [{type, url}]`) y la app
agregará la sección de documentos.
