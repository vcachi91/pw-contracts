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

### 7.1 Interruptor del beneficio por empresa (IMPLEMENTADO)

Cada empresa decide si ofrece o no el producto. La verdad vive en **una sola
columna de la base compartida**: `companies.extraordinary_advances_enabled`
(más `_toggled_at` y `_toggled_by` para la bitácora). El panel Enterprise la
escribe; app-api la lee. No hay HTTP entre los dos.

- `GET  /api/v1/payway/extraordinary-advances/module-settings` →
  `{ enabled, toggled_at, active_advances, company_name }`
- `PUT  /api/v1/payway/extraordinary-advances/module-settings` con `{ "enabled": bool }`

Reglas:

- Solo **`com_admin`**. El resto recibe 403; `GET .../permissions` expone
  `can_toggle_module` para que el front ni siquiera pinte el switch.
- **Apagarlo NO cancela nada.** Los adelantos aprobados siguen amortizándose y
  sus cuotas se siguen descontando de planilla — cortar una amortización viva
  sería romper un acuerdo firmado. Lo único que cambia es que el empleado deja
  de ver la opción de solicitar uno nuevo. Por eso el GET devuelve
  `active_advances`: el panel lo dice explícitamente antes de apagar.
- **Por defecto viene encendido** (`DEFAULT 1`) para todas las empresas.
- Del lado de app-api, `Quoter::moduleEnabledFor()` decide en tres capas, de la
  más fuerte a la más débil: interruptor global de config → lista blanca del
  `.env` (piloto) → esta columna. Si la columna todavía no existe se asume
  encendido: un despliegue adelantado al SQL no debe apagarle el módulo a nadie.

**El panel admin de Payway escribe la MISMA columna** (§10.2), para poder
encender o apagar cualquier empresa sin depender de su RRHH. No hay dos
verdades: el último que la toca manda. Como los dos paneles guardan un id de
usuario de tablas distintas (`users` vs `admin_users`),
`extraordinary_advances_toggled_source` dice de cuál panel vino — sin eso, un
`toggled_by = 12` es ambiguo y "¿quién apagó esto?" no se puede responder.

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
7. **Interruptor del beneficio** (§7.1) — switch en la cabecera de la bandeja, visible solo
   con `can_toggle_module`. Encender es directo; **apagar pide confirmación** y avisa cuántos
   adelantos siguen en curso, porque no se cancelan. Los riesgos son asimétricos: encender de
   más se corrige apagando, pero apagar sin querer deja a toda la plantilla sin poder
   solicitar y nadie se entera hasta que alguien reclama.

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

### 10.2 Disponibilidad por empresa (IMPLEMENTADO)

Panel plegado dentro del tablero, con un switch por empresa. Es el mismo
interruptor del §7.1 y escribe la misma columna, así que apagar acá se refleja
en el panel Enterprise de esa empresa y en la app de sus empleados.

- `PUT /api/v1/admin/companies/{id}/extraordinary-advances` con `{ "enabled": bool }`
- Permiso `companies.toggle_extraordinary_advances` (super_admin y admin_ops).
  Propio, no `companies.freeze`: congelar es temporal y se levanta solo al pasar
  la fecha; esto es indefinido. Separados se puede dar uno sin el otro.
- El switch se muestra aunque el rol no pueda moverlo (queda inerte): saber qué
  empresas lo tienen apagado es información útil para cualquier rol del panel.
- El cambio entra en el `activity_log` como cualquier edición de la empresa.

El ítem del menú exige `extraordinary_advances.view`, que se creó junto con esto
— antes no existía, y el ítem estaba comentado justamente por eso. Los permisos
del panel admin viven en `localStorage` desde el login: quien ya estuviera
adentro no ve el ítem hasta volver a entrar.

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

---

## 11. Motivo de la solicitud (27/08/2026 — IMPLEMENTADO)

Al pedir un adelanto el empleado declara **por qué** lo necesita. Antes solo
viajaban monto y cuotas, y quien aprueba no tenía con qué decidir: dos
solicitudes de $500 a 6 cuotas se veían idénticas aunque una fuera una urgencia
médica y la otra no.

`POST /extraordinary-advances/submit` acepta tres campos más:

| campo | obligatorio | forma |
|---|---|---|
| `reason_category` | **sí** | key del catálogo |
| `reason_description` | **sí** | 10–500 caracteres |
| `support_document` | no | `file` (multipart) |
| `support_document_base64` + `support_document_name` | no | base64 |

**El catálogo NO está en la app ni en un ENUM de la base.** Vive en
`config/extraordinary_advances.php` de app-api y viaja en `/eligibility`:

```json
"reasons": [ {"key":"salud","label":"Salud"}, ... ],
"description_min_length": 10,
"description_max_length": 500,
"support_document": { "enabled": true, "required": false, "max_bytes": 5242880,
                      "mimes": ["jpg","jpeg","png","pdf"] }
```

Así agregar o quitar un motivo es editar un archivo: sin migración y sin
publicar una versión en las tiendas. Por eso la columna es `VARCHAR` y no
`ENUM`. Las `key` no se tocan nunca —romperían el histórico—; los `label` sí.

⚠️ **La lista actual la puso el desarrollo como punto de partida, no el
negocio.** Revisarla con el dueño del producto.

Se aceptan dos formas para el soporte porque **el cliente HTTP de la app no arma
multipart** y la firma ya viajaba como base64. En base64 la extensión se deduce
de los primeros bytes del archivo, nunca del nombre que manda el cliente:
confiar en el nombre deja subir un `.php` llamado `.jpg`.

Las columnas son NULL: las solicitudes anteriores a esta fecha no tienen motivo
y no se les puede inventar uno. La obligatoriedad se valida en la app y en la
API, no con `NOT NULL`.

## 12. Aviso al empleado por WhatsApp (IMPLEMENTADO)

Hasta el 27/08/2026 aprobar un adelanto **no le avisaba nada al empleado**:
`notifyEmployee()` en hr-backend era un stub que solo escribía en el log.

El panel Enterprise no tiene credenciales de Twilio —viven solo en app-api— así
que **encola** en `employee_notifications` de la base compartida y app-api
envía con `php artisan notifications:send` (cron cada minuto).

Se eligió la cola sobre las alternativas: un HTTP interno hr→app pierde el aviso
si app-api está caído en ese instante, y duplicar las credenciales pone el mismo
secreto en dos servidores. La cola además permite reintentar (3 intentos antes
de marcar `failed`).

Índice único por `(user_id, event, subject_type, subject_id)`: si el aprobador
toca dos veces el botón, al empleado no le llegan dos WhatsApps.

## 13. PDF de la solicitud (IMPLEMENTADO)

`GET /api/v1/payway/extraordinary-advances/{id}/pdf` — documento imprimible para
que **RRHH lo firme junto al colaborador**. Sale del panel Enterprise porque es
RRHH quien firma. Incluye datos de la empresa y del colaborador, monto y cuotas,
el motivo, el calendario de descuentos y dos líneas de firma.

El logo de la empresa se incrusta **como data URI**: dompdf no sale a internet a
buscar imágenes. El archivo lo escribió admin-api en su propio disco; los dos
corren en el mismo servidor, así que se lee por ruta (`config/payway.php`,
configurable). Si no está, el documento sale con el nombre de la empresa en
texto — **nunca se cae por un logo**.

`laravel-dompdf` ya estaba instalado en hr-backend; no se agregó dependencia.

## 14. Documentos (27/08/2026 — §5 COMPLETADO)

La sección "Documentos" del detalle llegaba SIEMPRE vacía. No era un bug: los
PDFs del §5 nunca se construyeron. Pero además había tres cosas que **sí
existían** y vivían dispersas en otros rincones de la pantalla, así que decir
"sin documentos disponibles" era peor que no tener la sección.

Ahora la lista es una sola e incluye:

| documento | origen |
|---|---|
| Solicitud de adelanto (PDF para firmar) | generado |
| **Acuerdo de amortización firmado** | generado (§5) |
| **Constancia de aceptación electrónica** | generado (§5) |
| **Estado de cuenta** | generado (§5) |
| Soporte adjuntado por el colaborador | lo sube el empleado |
| Firma del acuerdo | evidencia |
| **Comprobante de desembolso** | lo sube RRHH |

`GET /api/v1/payway/extraordinary-advances/{id}/document/{acuerdo|constancia|estado}`

El **acuerdo** imprime el texto TAL CUAL quedó guardado, sin regenerarlo ni
reformatearlo: es lo que hace válido el descuento si alguien lo discute.

La **constancia** lleva los cinco datos que sostienen la aceptación electrónica
—versión del documento, huella SHA-256, timestamp del servidor, IP y
dispositivo—. La huella permite verificar que el texto aceptado no se modificó
después de la firma.

### Comprobantes que sube RRHH

`POST /api/v1/payway/extraordinary-advances/{id}/documents` (multipart, ≤15 MB)

Resuelve un hueco real: la empresa le paga al colaborador **por fuera del
sistema** y ese pago no dejaba rastro. Si el colaborador después decía que no lo
recibió, no había con qué responder.

Los archivos van al **disco privado** de enterprise-api y se sirven por un
endpoint que valida el alcance. Nunca por URL pública: un comprobante lleva el
nombre y el monto de un empleado. El borrado es **lógico** — un comprobante
borrado por error no se recupera, y su ausencia es justo lo que se discutiría.

Requirió subir `client_max_body_size` a 20 MB en el nginx de enterprise-api, que
estaba en el default de 1 MB. Sin eso nginx CIERRA la conexión sin devolver 413
y el panel muestra un error de red engañoso — la misma trampa del kiosco.

### Lo que el panel admin NO muestra

Los tres PDFs generados no aparecen en `admin.payway.pa`: los arma
enterprise-api con dompdf, que admin-api no tiene instalado. Payway sí ve la
firma, el soporte y los comprobantes, que es lo que necesita para atender un
reclamo. Para los generados, se consulta el panel Enterprise.
