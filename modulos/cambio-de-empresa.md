# Cambiar de empresa a un empleado

Mover a un empleado de una empresa a otra desde el panel admin. Pedido del
dueño el 03/10/2026 (se lo piden seguido y lo hacía a mano); construido el
05/10/2026.

**Repos:** pw-adminbackend y pw-adminfrontend. No toca pw-appbackend ni la app.

## 1. Decisiones del dueño (03/10/2026)

| # | Decisión |
|---|---|
| 1 | Con **deuda pendiente** el cambio se bloquea y el panel dice qué deuda lo impide. |
| 2 | Lo pueden hacer **super_admin y admin_ops** (permiso `employees.change_company`). |

## 2. Qué bloquea el cambio

| Código | Qué es |
|---|---|
| `solicitud_en_tramite` | Tiene una solicitud en estado `requested`. Hay que aprobarla o cancelarla. |
| `adelantos_por_descontar` | Tiene adelantos pagados de la quincena en curso **o de la anterior**. Su empresa actual todavía tiene que descontarlos. Se cuenta desde el inicio de la quincena anterior, con un mínimo de 20 días hacia atrás; sin ciclos cargados, 45 días. |
| `extraordinario_abierto` | Tiene un adelanto extraordinario que no está rechazado, cancelado ni saldado. |
| `tiene_historial` | Tiene **cualquier** solicitud pagada, aunque sea vieja. Ver §3. |
| `misma_empresa` | Ya está en esa empresa. La sub-empresa y la sucursal se cambian desde la ficha. |
| `falta_sucursal` / `sucursal_invalida` / `sub_empresa_invalida` | El destino no cumple las reglas de `sucursales.md`. |
| `destino_invalido` / `destino_inactivo` / `destino_sin_enterprise` | La empresa de destino no existe, no está activa o no tiene equipo en el panel Enterprise. |
| `sin_empresa` / `empresas_duplicadas` | El empleado no tiene empresa, o figura en dos a la vez. |

Las solicitudes en `ongoing` (borradores sin confirmar) y las canceladas no
cuentan: no movieron plata.

## 3. Por qué el historial bloquea (por ahora)

Las solicitudes **no guardan de qué empresa fueron**: `orders` no tiene empresa
y los reportes la deducen de la empresa **actual** del empleado (17 archivos
entre el panel admin y Enterprise). Si se mueve a alguien con solicitudes, ese
historial pasa a verse en la empresa nueva y desaparece de la vieja: la empresa
vieja pierde sus registros y la nueva ve descuentos que no hizo.

Hasta que cada solicitud guarde su empresa y los reportes la lean de ahí, quien
tiene historial no se mueve. Al 05/10/2026: de 938 empleados, 291 quedan
bloqueados solo por esto y 227 por deuda reciente; se pueden mover los que
nunca recibieron un adelanto, que incluye a **todos los pendientes de
aprobación** (193), que es el caso típico de "se registró en la empresa
equivocada".

**Fase 2 (pendiente, necesita OK):** agregar `orders.company_id`, llenarlo con
la empresa actual de cada empleado, grabarlo al crear cada solicitud, y pasar
los reportes a leerlo. Con eso se quita `tiene_historial`.

## 4. Qué mueve

Todo dentro de una sola transacción (las bases `app` y `hr` están en el mismo
servidor): o se hace todo o no se hace nada.

1. `company_users`: empresa y sub-empresa (todas las filas vivas del empleado).
2. `user_details`: `company` (respeta el formato que tenía, id o slug) y
   `sucursal_id`.
3. **Panel Enterprise** (`hr`), cruzando por **slug**: `hr.employees.team_id`
   pasa al equipo con el mismo nombre en la empresa nueva (si no existe,
   "Operations"; si no, el primero), y `hr.workplaces.branch_id` a una sede de
   la empresa nueva.
4. `pre_registrations` del empleado: empresa, sub-empresa y sucursal. El texto
   firmado y su hash **no** se tocan.
5. Registro en `employee_company_changes` y en la auditoría del panel
   (`activity_log`, `log_name = CambioDeEmpresa`).

**No se toca:** su salario, su fecha de ingreso, sus cuentas bancarias, su
estado, su código de referido, ni la tabla `saldos` (al cambiar
`user_details.updated_at` el panel y Enterprise calculan en vivo hasta que el
proceso de saldos lo rehaga; la app y el bot siempre calculan en vivo).

**Avisos que muestra el panel antes de confirmar:** el saldo pasa a calcularse
con el ciclo de la empresa nueva; hay que revisar salario y fecha de ingreso;
si está activo podrá solicitar de inmediato; si la empresa de destino está
congelada.

## 5. API (pw-adminbackend)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/admin/employee/{id}/company-change?company_id=&sub_company_id=&sucursal_id=` | **No escribe.** Devuelve `puede`, `bloqueos[]`, `avisos[]`, `de`, `a`, `cambios` e `historial`. Sin `company_id`, solo el historial. |
| POST | `/admin/employee/{id}/company-change` | `{ company_id, sub_company_id?, sucursal_id?, motivo }`. Vuelve a revisar dentro de la transacción y mueve. 422 `no_se_puede_mover` si ya no se puede; 409 si hay otro cambio en curso para el mismo empleado. |

El motivo es obligatorio (5 a 500 caracteres) y queda guardado con el admin, la
IP y la fecha.

La lógica vive en `App\Services\Empleados\CambioDeEmpresa` (`revisar` y `mover`).

## 6. Panel

Ficha del empleado → junto a "Company" → **Cambiar de empresa** (solo con el
permiso). Se elige empresa, sub-empresa y sucursal; el panel revisa al instante
y muestra en rojo lo que lo impide o en verde el resumen de lo que va a
cambiar. Con el resumen a la vista y el motivo escrito se habilita **Mover
empleado**, que pide una confirmación más. Al terminar la ficha se recarga.

Los permisos se guardan en el navegador: quien ya tenía sesión abierta tiene
que salir y volver a entrar para ver el botón.

## 7. Deshacer un cambio

Cada fila de `employee_company_changes` guarda en `antes` cómo estaba todo
(`company_users`, `user_details`, equipo y sedes de `hr`, pre-registros). Para
devolverlo: usar el mismo botón hacia la empresa anterior (mientras no haya
pedido un adelanto en la nueva), o restaurar a mano con esa foto.

## 8. Pruebas (05/10/2026)

- `revisar` corrido para los **938 empleados** hacia Minimed: 0 errores, y no
  escribió nada.
- Ensayo de `mover` con un empleado real sin historial (Cinépolis → Minimed),
  dentro de una transacción que se deshizo: empresa, ficha, equipo de
  Enterprise, registro y auditoría quedaron como debían, y al deshacer todo
  volvió a estar idéntico (28 comprobaciones).
- Los dos endpoints ya desplegados, llamados como los llama el panel: 12
  comprobaciones, entre ellas que con deuda responde 422 y no mueve nada, y que
  sin motivo se rechaza.
- Ojo al leer el log: los ensayos dejaron dos líneas `[CAMBIO-EMPRESA] empleado
  movido` del usuario #8230 a las ~23:00 del 04/10. No son reales: la
  transacción se deshizo. La verdad está en `employee_company_changes`.

## 9. Respaldo

`/root/respaldo-antes-cambio-empresa-20261005/` en pw-staging (volcado de
`company_users`, `user_details`, `pre_registrations`, permisos, y de
`hr.employees`, `hr.workplaces`, `hr.teams`) y en pw-frontend (la ficha
anterior).
