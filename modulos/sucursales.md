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

- **Página Sub-companies:** columna "Sucursales" y botón **Sucursales** en cada
  fila, que abre la lista de esa sub-empresa para agregar, renombrar o quitar.
  `PUT /admin/sub-company/{id}/sucursales` con `{ sucursales: [{ id?, name }] }`
  (permiso `sucursales.manage`). Solo toca las de esa sub-empresa: las de la
  empresa madre y las de otras sub-empresas no se modifican. Quitar una deja a
  sus empleados sin sucursal.

## 5b. Kiosco del captador

- `GET /admin/pre-registrations/companies` trae `sub_companies` (id y nombre)
  en cada empresa.
- Si la empresa tiene sub-empresas, **hay que elegir una** para comenzar, igual
  que en la app. Las sucursales que se ofrecen son solo las de esa sub-empresa.
- Si hay sucursales para elegir, **hay que elegir una**. Sin ellas no se pide.
- `GET /admin/pre-registrations/sucursales?company_id=&sub_company_id=`: con la
  clave `sub_company_id` presente (aunque vacía) aplica la regla; sin la clave
  devuelve todas las de la empresa, que es lo que pide un kiosco viejo que
  quedó abierto en la tablet.
- `pre_registrations.sub_company_id` (nullable) guarda la sub-empresa. Si el
  kiosco no la manda, se toma la de la sucursal elegida. Se rechaza con 422 si
  la sub-empresa no es de la empresa (`invalid_sub_company`) o la sucursal no
  es de esa sub-empresa (`invalid_sucursal`).
- **Al completar el registro**, por la app o desde el panel, el empleado nace
  en la sub-empresa del pre-registro. En la app (`completeRegistration`), si no
  llega una sucursal válida se usa la que eligió el captador: las apps hasta la
  1.3.16 la perdían y el empleado quedaba sin sucursal.
- `GET pre-registration/lookup` (pw-appbackend) devuelve `sub_company_id`, que
  la app nueva usa para pedir las sucursales de esa sub-empresa.
- El texto del consentimiento no cambió: sigue nombrando empresa y sucursal.

## 5c. Bot de WhatsApp (04/10/2026, pedido expreso del dueño)

- Paso nuevo `WAITING_SUCURSAL`, justo después de confirmar la empresa y antes
  de los términos. Solo existe si la empresa o sub-empresa **tiene** sucursales:
  el bot manda la lista numerada y la persona responde con el número (también
  acepta el nombre). Es texto libre, no necesita plantilla.
- Sin sucursales el paso no aparece y el registro sigue igual que antes.
- Usa la misma regla que la app (`App\Support\Sucursales`), con el tipo de la
  empresa que el bot ya guardaba: una sub-empresa ve solo las suyas.
- La sucursal viaja en la sesión y se manda como `sucursal_id` al mismo
  service del registro, que la valida igual que la de la app.
- Si la lista no se puede cargar, el registro sigue sin sucursal.
- Tres respuestas inválidas mandan al agente, como en los demás pasos.
- Fue lo único que se tocó del bot.

## 6. App (pw-mobileapp) — sale con la próxima versión

- Pide la lista con `type`.
- Sin sucursales: el campo no aparece.
- Con sucursales: el campo dice "Sucursal" y hay que elegir una para continuar.
- Con pre-registro del kiosco, se conserva la sucursal que eligió el captador.

**Apps viejas (hasta 1.3.16):** siguen funcionando igual. No mandan `type`, la
sucursal les sigue apareciendo como opcional, y el caso YEM Corp/Cinépolis les
sigue pasando hasta que actualicen; lo que elijan de más se descarta.

## 7. Qué NO cambió (a propósito)

- **Panel Enterprise:** el alcance de usuarios por sucursal funciona igual.
  Desde el 05/10/2026 `GET /sucursales` y las `sucursales` de cada usuario
  (`/auth/me`, listado de usuarios) traen además `sub_company_name` (`null` si
  la sucursal cuelga directo de la empresa), y el panel la muestra junto al
  nombre: "Casa Matriz · Deco Auto". Es solo presentación: ids, nombres y
  alcance no cambiaron (foto de los 34 usuarios idéntica antes y después).
  La bandeja de adelantos extraordinarios ya tenía su propia columna de
  sub-empresa; sus filas no se tocaron.

## 7b. Obligatoria en el panel y "Sin sucursal" en reportes (04/10/2026)

- **Ficha y alta de empleado:** si su empresa / sub-empresa tiene sucursales
  para elegir, no se puede guardar sin una (el panel avisa y el backend
  responde 422). Si no tiene, queda "Sin sucursal" y no se pide nada.
- Al activarse había 73 empleados sin sucursal en empresas que sí tienen
  (Minimed 41, Cinépolis 24, J. Cain 4, Manpower 4): la próxima vez que se
  guarde su ficha habrá que elegirles una. Los otros 493 sin sucursal están en
  empresas o sub-empresas sin sucursales y no se les pide.
- Cubre también al que entró por WhatsApp antes de que el bot preguntara la
  sucursal: operaciones no puede aprobarlo desde la ficha sin elegirla.
- **Reportes:** el CSV de empleados y el de transacciones dicen "Sin sucursal"
  en vez de dejar el espacio vacío, y el tablero suma "Sin sucursal: N" para
  que el desglose cuadre con el total.

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
- Guardado desde "Editar empresa" (13 comprobaciones) y kiosco más página de
  Sub-companies (31 comprobaciones), sobre el código desplegado y dentro de
  transacciones que se deshicieron. Incluye el kiosco viejo sin el dato nuevo.
- Primer registro real por el código nuevo: usuario #8261 (J. Cain Logistics),
  20:34, completado con su sucursal.

## 9. Volver atrás

Respaldo en `/root/respaldo-antes-sucursales-20261004/` (los tres servidores):
los archivos anteriores y un volcado de `sucursales` y `sub_companies`. La
columna `sub_company_id` puede quedarse aunque se vuelva al código anterior: el
código viejo la ignora (verificado).
