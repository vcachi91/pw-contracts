# Registro con menos campos: `cedula`, `joining_date` y `bank_accounts` dejan de enviarse — Brief para Backend

> **✅ RESUELTO por backend (2026-08-25).** `cedula` pasó a nullable (era el único 422);
> `joining_date` ya lo era y `bank_accounts` nunca estuvo en la validación. Además:
> el email sintético dejó de derivar de la cédula (chocaba con UNIQUE al ser opcional),
> el número de cédula ahora se extrae por **OCR de `cedula_img`** (~90% de acierto, admin
> cubre el resto), `joining_date` guarda NULL en vez de now() (antes dejaba el cupo en $0
> el día del alta), y se eliminó el literal 'PENDIENTE' que WhatsApp imprimía como número
> de documento en contratos. **Compromiso de la app:** `cedula_img` se comprime en modo
> OCR (2048px, calidad 88 con degradado solo si supera 2MB) — ver `_compressImage`.
>
> Pendiente aparte (operación/Enterprise, no de este cambio): el "32% sin salario"
> reportado inicialmente resultó ser un dato mezclado — entre usuarios ACTIVOS solo el
> 0.5% (2/415) tiene salario en 0; el salario sí se carga bien al aprobar. Lo que ese
> número medía en realidad es un **backlog de 113 usuarios en `pending_approval`**
> (algunos con 116 días de espera): los empleadores no están aprobando las solicitudes.
> Eso es gestión con las empresas (recordatorios/escalación desde Enterprise), no código.

> **La app ya cambió.** A partir de esta versión, el formulario de registro pide menos datos y
> `POST /auth/complete-registration` llega **sin** `cedula`, **sin** `joining_date` y **sin**
> `bank_accounts[]`. Si esos campos son `required` en el FormRequest, **toda alta nueva va a
> fallar con 422** y el usuario queda atascado después de haber verificado su OTP.

## Por qué el cambio

Reducir la fricción del alta. Se quitaron los datos que el usuario todavía no necesita dar, o que
ya se validan por otra vía:

| Campo eliminado | Por qué se puede quitar |
|---|---|
| `cedula` (número) | La identidad se valida con la **foto** (`cedula_img`), que se sigue enviando y es obligatoria en la app. El número lo transcribe/valida el admin al aprobar. |
| `joining_date` | La antigüedad la confirma el **empleador** al aprobar la solicitud; el dato autodeclarado no aportaba control real. |
| `bank_accounts[]` | La cuenta de destino se registra al pedir el **primer adelanto**, con el flujo que ya existe (`/accounts/save-bank-account` y `/accounts/save-mfs-account`). Pedirla en el alta era anticipar información. |
| `email` | **Nunca se envió.** La app lo recogía y lo descartaba: `complete-registration` jamás incluyó el campo. Quitarlo del formulario no cambia nada del lado del backend. |

## Qué cambia en el request

`POST /auth/complete-registration` (multipart, con el token del verify-otp).

**Se siguen enviando, igual que antes:**

```
company_id, first_name, last_name, phone, password,
sucursal_id            (opcional, solo si la empresa tiene sucursales)
signature_img          (PNG, <= 2MB)
cedula_img             (JPG comprimido, <= 2MB)   <- obligatorio en la app
payslip_img            (JPG comprimido, <= 2MB)   <- obligatorio en la app
detail[gross_salary]   (ver nota abajo)
salary_component[n][key] / [value]                 (ver nota abajo)
```

**Dejan de enviarse:**

```
cedula
joining_date
bank_accounts[0][bank_id]
bank_accounts[0][account_number]
bank_accounts[0][account_type]
```

## Cambio pedido

1. **Hacer `cedula`, `joining_date` y `bank_accounts` opcionales** (`nullable` / `sometimes`) en la
   validación de `complete-registration`. No romper si llegan (una app vieja en un teléfono sin
   actualizar los seguirá mandando): la idea es **aceptar ambos casos**.
2. **Columnas nullable**: verificar que `users.cedula` (o donde viva) y
   `user_details.joining_date` acepten `NULL`. Si hay un índice único sobre la cédula, que tolere
   nulos múltiples.
3. **Panel de admin**: la ficha del empleado va a llegar sin número de cédula ni fecha de ingreso.
   Conviene que el admin pueda **completar esos campos a mano** al revisar la solicitud, leyéndolos
   de la foto de la cédula que sí se sube.
4. **Aprobación**: confirmar que el flujo de aprobación (`pending_approval` → `active`) no dependa
   de que exista una cuenta bancaria. El usuario ahora registra la cuenta después, al pedir su
   primer adelanto.

## Nota sobre el salario (deuda técnica preexistente, NO la introduce este cambio)

`detail[gross_salary]` **siempre se envía como `"0"`** y `salary_component[]` siempre va vacío:
la app declara los observables pero ninguna pantalla los llena. Ya era así antes de este cambio.
Si el backend usa `gross_salary` para calcular el cupo de adelanto, hoy está recibiendo `0` en toda
alta y el dato real debe estar saliendo de otro lado (nómina del empleador). Vale confirmarlo.

## Sobre el contrato firmado

El texto de la autorización que el usuario firma en el registro **ya no menciona el número de
documento**. Antes decía "con documento de identidad Cédula No. {número}"; ahora identifica al
firmante por **nombre completo + empresa**. Se quitó la frase entera en vez de dejar un "N/A"
impreso en un documento legal.

El contrato del **resumen de adelanto** (`/requests/summary`) **no cambia**: ese sigue mostrando
`user_document_type` y `user_document_number` tal como los devuelve el backend en la orden. Si el
backend deja de tener el número de cédula del usuario, ese contrato mostrará lo que haya en la BD
— otra razón para que el admin pueda completarlo al aprobar (punto 3).
