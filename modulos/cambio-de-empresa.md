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
| 3 | (05/10/2026) **El historial ya descontado se mueve con el empleado**: no bloquea. |

## 2. Qué bloquea el cambio

| Código | Qué es |
|---|---|
| `solicitud_en_tramite` | Tiene una solicitud en estado `requested`. Hay que aprobarla o cancelarla. |
| `adelantos_por_descontar` | Tiene adelantos pagados de la quincena en curso **o de la anterior**. Su empresa actual todavía tiene que descontarlos. Se cuenta desde el inicio de la quincena anterior, con un mínimo de 20 días hacia atrás; sin ciclos cargados, 45 días. |
| `extraordinario_abierto` | Tiene un adelanto extraordinario que no está rechazado, cancelado ni saldado. |
| `misma_empresa` | Ya está en esa empresa. La sub-empresa y la sucursal se cambian desde la ficha. |
| `falta_sucursal` / `sucursal_invalida` / `sub_empresa_invalida` | El destino no cumple las reglas de `sucursales.md`. |
| `destino_invalido` / `destino_inactivo` / `destino_sin_enterprise` | La empresa de destino no existe, no está activa o no tiene equipo en el panel Enterprise. |
| `sin_empresa` / `empresas_duplicadas` | El empleado no tiene empresa, o figura en dos a la vez. |

Las solicitudes en `ongoing` (borradores sin confirmar) y las canceladas no
cuentan: no movieron plata.

## 3. El historial se va con el empleado

Las solicitudes **no guardan de qué empresa fueron**: `orders` no tiene empresa
y los reportes la deducen de la empresa **actual** del empleado. Por eso, al
mover a alguien con solicitudes viejas ya descontadas, ese historial pasa a
verse en la empresa nueva y deja de verse en la anterior, en el panel admin y
en Enterprise.

La primera versión (05/10/2026, madrugada) bloqueaba a quien tuviera historial.
El dueño decidió ese mismo día que **se mueva también**. Lo que queda:

- El panel lo **avisa** antes de confirmar: cuántas solicitudes y por cuánto.
- El registro del cambio guarda en `antes.ordenes_que_se_mueven` los ids de
  esas solicitudes y de qué empresa venían. Con eso se puede reconstruir de
  qué empresa fue cada una si algún día los reportes pasan a leerlo de la
  solicitud (agregar `orders.company_id`).
- La **deuda** sigue bloqueando: solicitud en trámite, adelantos de la
  quincena en curso o de la anterior, y extraordinarios sin cerrar.

Al 05/10/2026, de 938 empleados: 227 bloqueados por deuda reciente y 3 por un
extraordinario abierto; el resto se puede mover (91 de ellos con historial).

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
- Con el historial permitido: `revisar` otra vez para los 938 (0 errores) y
  `mover` a un empleado con 18 solicitudes viejas, deshecho: sus solicitudes
  no se tocan y el registro guarda las 18 (10 comprobaciones).
- Ojo al leer el log: los ensayos dejaron dos líneas `[CAMBIO-EMPRESA] empleado
  movido` del usuario #8230 a las ~23:00 del 04/10. No son reales: la
  transacción se deshizo. La verdad está en `employee_company_changes`.

## 9. Respaldo

`/root/respaldo-antes-cambio-empresa-20261005/` en pw-staging (volcado de
`company_users`, `user_details`, `pre_registrations`, permisos, y de
`hr.employees`, `hr.workplaces`, `hr.teams`) y en pw-frontend (la ficha
anterior).
