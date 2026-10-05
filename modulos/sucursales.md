# Sucursales por sub-empresa

Una sucursal puede pertenecer a una sub-empresa. Pedido de Pablo (02/10/2026)
con el ejemplo **Davissa Foods → Paul Bakery**; reglas cerradas con el dueño el
03/10 y el 04/10/2026.

**Repos:** pw-appbackend (lista del registro y validación), pw-adminbackend y
pw-adminfrontend (carga y ficha del empleado), pw-mobileapp (registro).

## 1. Conceptos

- **Sub-empresa** (`sub_companies`): una empresa o marca del grupo. Paul Bakery
  dentro de Davissa Foods; Deco Auto dentro de Grupo Piazza.
- **Sucursal** (`sucursales`): un local físico.

Son cosas distintas. Lo nuevo es que una sucursal puede ser **de** una
sub-empresa.

## 2. Reglas

| # | Regla |
|---|---|
| 1 | Una sucursal pertenece a una empresa y, opcionalmente, a **una** de sus sub-empresas. |
| 2 | Quien está en una sub-empresa ve **solo las sucursales de esa sub-empresa** (Pablo: "solo de la subempresa"). |
| 3 | Quien está directo en la empresa ve solo las que **no son de ninguna** sub-empresa. Nunca se mezclan. |
| 4 | Si no hay sucursales para elegir, **el campo no aparece** y el empleado queda "Sin sucursal" por dentro (dueño, 04/10/2026). |
| 5 | Si hay sucursales para elegir, elegir una es **obligatorio**. |
| 6 | Un registro que llega sin sucursal, o con una que no le corresponde, **nunca se rechaza**: queda "Sin sucursal". |

## 3. Datos

- `sucursales.sub_company_id` (nullable, sin FK): `NULL` = de la empresa a
  secas; un id = de esa sub-empresa. Que la sub-empresa sea de la misma empresa
  lo valida el backend al guardar.
- **"Sin sucursal" no es una fila**: es `user_details.sucursal_id = NULL`, y los
  paneles lo muestran con ese texto. Así no existe una sucursal de mentira que
  haya que esconder en cada lista, y los 581 empleados que hoy no tienen
  sucursal ya están en ese estado sin migrar nada.
- SQL: `pw-adminbackend/database/sql/2026_10_04_sucursales_sub_company.sql`.

## 4. API del registro (pw-appbackend)

`GET /api/v1/companies/{id}/sucursales?type=company|subcompany` — público.

| `type` | Qué devuelve |
|---|---|
| `subcompany` | Las sucursales de esa sub-empresa. |
| `company` | Las de la empresa que no son de ninguna sub-empresa. |
| *(sin `type`)* | Igual que `company`. Es lo que mandan las apps publicadas hasta la 1.3.16. |

`type` es el mismo `tipo` que trae `/companies/register-list`. Hace falta porque
`companies` y `sub_companies` repiten ids: sin él, alguien de **YEM Corp**
(sub-empresa #26) veía las sucursales de **Cinépolis** (empresa #26).

Al registrarse (`completeRegistration`), la sucursal se acepta solo si le
corresponde a la empresa y sub-empresa finales. Si no, se descarta en silencio
y queda en el log, igual que antes.

La regla vive en un solo lugar: `App\Support\Sucursales`.

## 5. Panel admin

- **Editar empresa:** cada sucursal tiene un desplegable de sub-empresa (solo
  si la empresa tiene sub-empresas). `PUT` manda `sucursales[].sub_company_id`.
  Si la clave no viene (un panel viejo en caché), el backend **no toca** la que
  ya tenía.
- **Detalle de empresa:** `sub_companies` (id y nombre) viaja en el detalle; las
  sucursales de una sub-empresa se muestran como "Paul Bakery · Pacific Center".
- **Ficha y alta de empleado:** el desplegable de sucursal muestra solo las de
  su sub-empresa. Al cambiarle la sub-empresa, la sucursal se limpia.
- **Guardar la ficha:** la sucursal tiene que corresponder a su sub-empresa,
  pero solo se exige cuando en ese guardado cambia la sucursal o la
  sub-empresa. Un empleado que quedó descuadrado (porque le pasaron su sucursal
  a una sub-empresa) puede seguir editándose; la ficha le muestra su sucursal
  con "(no es de su sub-empresa)".

## 6. App (pw-mobileapp) — sale con la próxima versión

- Pide la lista con `type`.
- Sin sucursales: el campo no aparece.
- Con sucursales: el campo dice "Sucursal" y hay que elegir una para continuar.
- Con pre-registro del kiosco, se conserva la sucursal que eligió el captador.

**Apps viejas (hasta 1.3.16):** siguen funcionando igual. No mandan `type`, la
sucursal les sigue apareciendo como opcional, y el caso YEM Corp/Cinépolis les
sigue pasando hasta que actualicen; lo que elijan de más se descarta.

## 7. Qué NO cambió (a propósito)

- **Bot de WhatsApp:** no se tocó. Quien se registra por ahí queda con la
  sucursal que el bot ya manejaba, o sin sucursal.
- **Kiosco del captador:** sigue eligiendo empresa y sucursal como antes, y
  lista todas las sucursales de la empresa. Si se elige una que es de una
  sub-empresa, al completar el registro en la app se descarta (el pre-registro
  no guarda sub-empresa). Pendiente: que el kiosco elija también la sub-empresa.
- **Panel Enterprise:** el alcance de usuarios por sucursal funciona igual.
- **Obligatoria en el panel admin:** todavía no se exige al guardar la ficha.
  Se prende al final, cuando las empresas tengan sus sucursales cargadas, para
  no frenar a operaciones al editar a los 581 empleados sin sucursal.

## 8. Pruebas (04/10/2026)

- Foto de la lista pública de los 90 ids de empresa antes del cambio. Igual
  después de agregar la columna, después de desplegar el código nuevo sin
  `type`, y con `type=company`. Con `type=subcompany`: todas vacías, como debe
  ser hoy.
- Ensayo del código nuevo sobre una copia temporal de la tabla: 132
  comprobaciones, entre ellas el caso Davissa → Paul Bakery y el choque
  YEM Corp / Cinépolis.
- La regla nueva acepta a los 390 empleados que hoy tienen sucursal y a los 130
  pre-registros del kiosco con sucursal: nadie pierde la suya.

## 9. Volver atrás

Respaldo en `/root/respaldo-antes-sucursales-20261004/` (los tres servidores):
los archivos anteriores y un volcado de `sucursales` y `sub_companies`. La
columna `sub_company_id` puede quedarse aunque se vuelva al código anterior: el
código viejo la ignora (verificado).
