# Adelantos Extraordinarios — Contrato General del Módulo (todas las plataformas)

> **✅ COPIA CANÓNICA.** Este archivo es la fuente única de verdad del módulo para los
> 6 repos. Si una copia en Drive, en un chat o en el `docs/` de algún repo difiere de
> esta, manda ESTA. Cualquier cambio de regla, estado o nombre de campo se hace aquí
> primero, en un commit que explique por qué — y después se alinean los repos.

> **Cómo usar este documento:** es ÚNICO y compartido. Las secciones 1–5 (núcleo) las leen
> TODOS los equipos: ahí viven las reglas legales, los estados y el modelo de datos, y nadie
> debe re-definirlos en su propio doc — si algo cambia, se cambia AQUÍ. Después, cada dev
> salta a su sección (§6–§10).
>
> El contrato detallado APP ↔ backend (endpoints con payloads JSON) ya existe aparte:
> **`modulos/adelantos-extraordinarios-api-app.md`** (en este mismo repo) — la sección §6 lo referencia.
>
> Estado actual: la APP móvil ya tiene su parte construida y dormida (se activa cuando el
> endpoint de elegibilidad responda enabled=true). Todo lo demás está por construir.

---

## 1. Qué es (núcleo — leer todos)

Un nuevo tipo de adelanto, separado del EWA estándar financiado por Payway (que queda
**intacto**):

1. Lo financia la **EMPRESA** (empleador), no Payway.
2. El monto puede superar el tope del adelanto estándar.
3. Requiere **aprobación explícita del empleador** en el panel Enterprise.
4. Se amortiza por **descuento de planilla** en cuotas, con calendario generado por el sistema.
5. Genera **documentos legales** y exige aceptación/firma electrónica del empleado con
   trazabilidad completa.

## 2. Reglas legales (núcleo — leer todos; se validan en BACKEND, la UI solo refleja)

Base: Art. 161 del Código de Trabajo de Panamá y Ley 64 de 1961.

| Regla | Detalle |
|---|---|
| **Cuota ≤ 15%** | El descuento por período de pago no supera el 15% del salario devengado en ese período. El sistema calcula cuota máxima y plazo mínimo. |
| **Total ≤ 50%** | La suma de TODAS las deducciones del empleado (este adelanto + las ya registradas, salvo pensiones alimenticias) no excede el 50% del salario. El margen se valida ANTES de cotizar y ANTES de aprobar. |
| **Diciembre excluido** | Los descuentos se suspenden en diciembre. El generador del calendario lo salta automáticamente y corre la fecha fin. |
| **Firma obligatoria** | El descuento solo es válido con documento firmado por el trabajador reconociendo la deuda. Evidencia: firma, versión del template, hash del documento, timestamp del servidor, IP, dispositivo. Inmutable. |
| **Terminación laboral** | El acuerdo incluye cláusula de descuento del saldo pendiente sobre el último salario; el sistema debe poder liquidar en ese evento. |

**Montos y salarios en DECIMAL, nunca float.** En los JSON viajan como string ("150.00").

## 3. Estados canónicos (núcleo — leer todos; nadie inventa estados propios)

```
pending_approval → approved → disbursed → in_amortization → settled
                 ↘ rejected
pending_approval → cancelled   (solo el empleado, solo mientras está pendiente)
```

Cuotas del calendario: `pending | paid | skipped` (skipped = diciembre).
Toda API expone `status` (canónico) + `status_label` (texto es-PA listo para mostrar).

## 4. Modelo de datos (núcleo — lo administra el backend app/HR; los demás lo consumen)

Sugerencia a ajustar contra el esquema real de `payway-app` / `payway-hr`:

- Solicitudes: evaluar `advance_requests.type = standard|extraordinary` +
  `funded_by = payway|employer` vs tabla nueva — decidir mirando el esquema actual.
- `amortization_schedules` + `amortization_installments` (cuota, fecha, monto, estado,
  período de planilla).
- `advance_agreements` (documento, versión de template, hash, evidencia de firma).
- **Bitácora de todo cambio de estado**: quién, cuándo, desde qué plataforma. Los tres
  frentes (app, Enterprise, Admin) escriben en la misma auditoría.

## 5. Documentos PDF y notificaciones (núcleo)

PDFs que genera el sistema: solicitud de anticipo, acuerdo de amortización (template
**editable por cliente**, lo aprueba su abogado), constancia de aceptación electrónica,
comprobante de desembolso / estado de cuenta. Expuestos como `documents: [{type, url}]`
en los detalles de cada plataforma.

Notificaciones en cada cambio de estado (reutilizar la infra existente): push (app) +
correo + WhatsApp al empleado; correo/panel al aprobador Enterprise cuando entra una
solicitud nueva.

---

## 6. DEV BACKEND APP (payway-app)

**Tu contrato detallado es `docs/adelantos-extraordinarios-backend.md`** (5 endpoints con
payloads de ejemplo). Resumen de responsabilidades:

- `GET /extraordinary-advances/eligibility` — interruptor del módulo + parámetros (15%,
  margen 50%, min/max, adelanto activo).
- `POST /extraordinary-advances/quote` — cotización + calendario sin diciembre + texto del
  acuerdo (template renderizado) + versión.
- `POST /extraordinary-advances/submit` — crea en `pending_approval`, guarda evidencia de
  firma, notifica al aprobador Enterprise.
- `GET /extraordinary-advances` y `/{id}` — lista y detalle con calendario y saldo.
- `PUT /extraordinary-advances/cancel/{id}` — solo en pendiente.
- Dueño de: cálculos legales, generador de calendario, evidencia de firma, PDFs,
  notificaciones al empleado.

## 7. DEV BACKEND HR / ENTERPRISE (payway-hr)

Endpoints para el panel Enterprise (nombres orientativos; mantener los shapes del §3):

- **Bandeja**: `GET /enterprise/extraordinary-advances?status=pending_approval` — solicitudes
  de los empleados de la empresa (y sucursal si aplica), con antigüedad de la solicitud.
- **Capacidad del empleado**: `GET /enterprise/employees/{id}/deduction-capacity` — salario
  del período, deducciones actuales registradas, margen libre bajo el 50%, y cuánto de ese
  margen consumiría ESTA solicitud. Es la pantalla con la que el aprobador decide.
- **Aprobar / Rechazar**: `POST .../{id}/approve` y `POST .../{id}/reject` (rechazo con
  `comment` obligatorio). Al aprobar se RE-VALIDA 15% y 50% con los datos del momento (pudo
  pasar tiempo desde la solicitud). Aprobación → notifica al empleado.
- **Registrar desembolso**: `POST .../{id}/disburse` — la empresa pagó; el sistema genera el
  calendario de amortización DEFINITIVO (sin diciembre) y pasa a `disbursed`/`in_amortization`.
- **Descuentos por período (planilla)**: `GET /enterprise/payroll-deductions?period=...` —
  todas las cuotas a aplicar en ese período de pago, para que RRHH las baje a planilla.
  Marcar cuota como aplicada → `paid`.
- **Terminación laboral**: acción de liquidación que calcula el saldo pendiente y lo aplica
  contra el último salario (cláusula del acuerdo).
- **Roles**: el aprobador es un ROL específico del panel (p. ej. `advance_approver`), no
  cualquier usuario de la empresa. Toda acción queda en la bitácora del §4.
- **Sync**: definir con el dev backend app dónde vive la verdad (payway-app vs payway-hr) y
  el mecanismo de sincronización — un solo dueño por dato, el otro lee.

## 8. DEV FRONT HR / ENTERPRISE (panel de la empresa)

Pantallas contra los endpoints del §7:

1. **Bandeja de solicitudes** — filtros por estado; badge con pendientes; ordenar por
   antigüedad (evitar el backlog silencioso que ya vimos en las aprobaciones de registro:
   hoy hay solicitudes de alta con 100+ días esperando — este módulo nace con recordatorios).
2. **Detalle de solicitud** — datos del empleado, monto, cuotas propuestas, y la **vista de
   capacidad**: salario, deducciones actuales, margen del 50%, qué consume esta solicitud.
   El panel NO calcula nada: pinta lo que devuelve el backend.
3. **Aprobar / Rechazar** — rechazo exige comentario (el empleado lo ve en la app).
4. **Registrar desembolso** — confirma el pago; muestra el calendario definitivo generado.
5. **Descuentos del período** — lista exportable de cuotas a aplicar en la planilla en curso;
   acción de marcar aplicadas.
6. Visibilidad solo del rol aprobador para aprobar/rechazar/desembolsar; lectura para el
   resto según permisos existentes del panel.

## 9. DEV BACKEND ADMIN (operación Payway)

- **Vista global**: `GET /admin/extraordinary-advances` con filtros por company/subcompany,
  estado, rango de fechas, monto. Incluye saldos vivos y cuotas vencidas/aplicadas.
- **Reportería exportable** (CSV/Excel): solicitudes, aprobaciones, desembolsos, cartera
  viva por empresa, cuotas por período.
- **Solo lectura + soporte**: Admin NO aprueba ni desembolsa (eso es del empleador). Sí puede
  ver la evidencia de firma y documentos de cualquier solicitud para soporte/disputas.
- Métricas de salud sugeridas: tiempo medio de aprobación, % rechazo, solicitudes estancadas
  en `pending_approval` > N días (alerta temprana del patrón de backlog).

## 10. DEV FRONT ADMIN (panel Payway)

1. **Tablero global** por company/subcompany: solicitudes por estado, montos, saldos.
2. **Detalle** de cualquier solicitud: línea de tiempo de estados (de la bitácora), calendario,
   evidencia de firma y PDFs (solo lectura).
3. **Exportes** de la reportería del §9.
4. **Alertas**: pendientes de aprobación envejecidos, cuotas vencidas sin aplicar.

---

## Orden de construcción sugerido

1. §6 backend app (`eligibility` + `quote` + `submit`) — con solo esto la APP ya funciona
   hasta "solicitud enviada".
2. §7 backend HR (bandeja + aprobar/rechazar + desembolso + calendario) + §8 front HR — cierra
   el ciclo completo empleado↔empresa.
3. §9/§10 Admin — visibilidad y reportería, puede ir en paralelo desde que existan datos.
4. WhatsApp (consultas y solicitud por chat, sobre la infra existente) — al final; la app y
   el panel ya cubren el flujo completo sin él.

---

## Preguntas abiertas (bloquean a más de un repo)

Se resuelven AQUÍ. Cuando una se cierre, se mueve la respuesta a la sección que
corresponda (§3, §7, etc.) y se borra de esta lista.

### A. Cuota que no se pudo aplicar — falta un estado
**Dueño de la respuesta:** backend HR/Enterprise (§7).

Hoy una cuota solo puede ser `pending | paid | skipped`, y `skipped` está
reservado para diciembre. Falta el caso real: la quincena pasó y el descuento
**no se pudo aplicar** (incapacidad, vacaciones, salario insuficiente ese
período, ingreso a mitad de quincena).

Preguntas concretas:
1. ¿La cuota se queda `pending` y el calendario se corre una quincena, o hace
   falta un estado nuevo (`failed`, `deferred`)?
2. Si el calendario se corre, **¿cambia la fecha fin del acuerdo firmado?** El
   empleado firmó un documento que dice N cuotas hasta una fecha concreta. Hay
   que decidir si se re-emite el acuerdo o queda como estaba. *Este es el punto
   con peso legal — diciembre y la liquidación por terminación ya están cubiertos
   en el acuerdo; este no.*

A quién impacta cuando se responda:
- **Front HR (§8, pantalla 5)** — hoy solo tiene la acción "marcar aplicadas".
  Necesitaría una segunda: "no se pudo aplicar" + motivo.
- **App (§6)** — pinta el ícono/color del estado nuevo. Cambio de minutos, pero
  hay que avisarle: **cualquier estado que la app no conozca hoy cae al ícono
  gris de "pendiente"**, que sería engañoso para el empleado.
- **Front Admin (§10)** — su alerta de "cuotas vencidas sin aplicar" debería
  distinguir "nadie la tocó" de "se intentó y falló". No es lo mismo para
  operación.

### B. Fuente única de las deducciones para la regla del 50%
**Dueño de la respuesta:** backend HR + backend app, de común acuerdo.

Pendiente decidir si la verdad vive en `salary_component` o en una tabla propia
del módulo. Mientras no se decida, el margen del 50% puede calcularse distinto en
cada backend — y ambos comparten la misma MySQL.

### C. Almacenamiento compartido de las firmas
Los PNG de firma que sube la app tienen que ser legibles por los paneles HR y
Admin para mostrar la evidencia. Falta definir dónde viven (S3 / Spaces / URL
firmada) y quién los sirve.
