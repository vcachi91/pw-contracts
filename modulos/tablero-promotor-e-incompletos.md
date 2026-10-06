# Mi tablero (promotor) y registros incompletos

Pantalla `/mi-tablero` del panel admin y la bandeja de registros incompletos,
que también se usa en `/pre-registro`. Definido con David el 15/09/2026;
ajustado el 05/10/2026 con tres pedidos suyos (notas de voz al dueño).

**Repos:** pw-adminbackend (`PromoterDashboardController`,
`IncompleteRegistrationController`) y pw-adminfrontend
(`app/(dashboard)/mi-tablero/page.tsx`,
`components/pre-registrations/IncompleteRegistrationsPanel.tsx`).

## 1. Las tres categorías de incompletos

| # | Categoría | Quién entra |
|---|---|---|
| 1 | Solo el número de teléfono | Números importados que aún no son usuarios + usuarios que dejaron su número y nunca su nombre. |
| 2 | **Sin código de empleado** | Pre-registros del kiosco que siguen abiertos (todavía no son empleados) + usuarios **en proceso**, con nombre, sin `user_details.employee_code`. |
| 3 | Pendiente firma o cuenta bancaria | Usuarios con nombre a los que les falta la firma o una cuenta activa. Incluye a los activos sin cuenta. |

- **El código** es el campo "Employee Code" de la ficha del empleado. Solo lo
  carga operaciones (ficha o carga masiva); ni la app, ni WhatsApp, ni el
  kiosco lo piden. No confundir con "Employee Code in Company"
  (`company_code`).
- **En proceso** = `pending_profile`, `pending_approval`, o `inactive` sin
  empresa. Un **activo sin código no entra** en la 2: ya está operando.
- **Cada persona sale en una sola categoría: la primera que le falta.** Quien
  no tiene código y además no firmó se ve en la 2, con las dos cosas en "le
  falta", y no se repite en la 3.
- Hasta el 05/10/2026 la 2 era "pre-registro, por descargar la app". Se cambió
  porque bajar la app no es el dato que importa (también se registran por
  WhatsApp).
- Hay empleados con el vínculo o la ficha repetidos en la base: la bandeja
  cuenta y muestra a cada persona una vez (`distinct` + `unique('key')`).

Al cambiar (05/10/2026): 1 = 30, 2 = 146 (8 pre-registros + 138 usuarios; a 9
solo les falta el código), 3 = 179. Antes: 30 / 8 / 320.

## 2. El tablero

1. **Empresa:** tarjetas (total, mes, quincena, hoy) y una tabla por empresa
   con registrados, mes, quincena, hoy y penetración. **Ordenada de más
   registrados a menos**; las que están en cero quedan al final. Ya no muestra
   sucursales debajo de cada empresa.
2. **Por sucursal:** se elige una empresa y recién ahí se abre su detalle:
   registrados, mes, quincena y hoy por sucursal, más "Sin sucursal" para que
   la suma dé el total. Acá es donde se va a ir sumando información por
   sucursal. No hay penetración por sucursal: el total de empleados solo se
   carga por empresa.
3. **Registros incompletos:** la bandeja de arriba.

`GET /admin/promoter-dashboard` → `empresas.items[].sucursales[]` trae
`sucursal_id`, `nombre`, `sub_empresa`, `registros`, `mes`, `quincena`, `hoy`.

## 3. En el teléfono

Se usa sobre todo desde el celular. En pantalla chica (menos de 768px) las
tablas pasan a ser **tarjetas**: no hay que deslizar de lado, los nombres no se
cortan, el texto no baja de 14px y los botones y filtros son altos para el
dedo. En pantalla grande siguen siendo tablas, con los números alineados a la
derecha.

## 4. Pruebas (05/10/2026)

Ensayo de las clases nuevas contra las que estaban en vivo, solo lectura, 25
comprobaciones: la categoría 1 queda idéntica; nadie desaparece de la bandeja
ni sale dos veces; en la 2 nadie tiene código ni está activo; el resumen
coincide con el listado con y sin filtros; las tarjetas y los números por
empresa del tablero no cambian; el detalle por sucursal nunca suma más que la
empresa. Las pantallas compilaron y se publicaron, pero **no se revisaron a la
vista en un teléfono**.

Respaldo: `/root/respaldo-antes-tablero-categorias-20261005/` en pw-staging y
pw-frontend.
