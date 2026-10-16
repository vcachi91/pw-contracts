# El token del link de firma

El empleado firma el acuerdo de su adelanto abriendo un link que le llega por
WhatsApp. Ese link identifica a la persona, así que lo que lleve dentro **es la
credencial**: con él se abre su pantalla de firma y se guarda su firma.

Repos que lo tocan: `pw-appbackend` (lo emite el bot y lo verifica),
`pw-adminbackend` (lo emite el panel). La tabla vive en la base `app`.

## La regla

> El token es **aleatorio, de un solo uso y con vencimiento**. Nunca se deriva
> del teléfono, del id ni de ningún dato que alguien pueda adivinar o deducir.

- 48 caracteres de `Str::random`, que son 286 bits.
- Vence a las **72 horas**.
- Se gasta en cuanto la firma queda guardada.
- Emitir uno nuevo **mata los que esa persona tuviera sin usar**.
- En la tabla se guarda el **SHA-256**, nunca el token: quien lea la tabla no
  puede armar links con lo que ve.

## Por qué una tabla y no un HMAC

El link lo generan **dos repos que no comparten código ni `APP_KEY`**: el bot de
WhatsApp en `pw-appbackend` y el panel en `pw-adminbackend`. Un HMAC obligaría a
un secreto compartido en dos `.env`, que es una cosa más que mantener sincronizada
y que se rota mal.

Una tabla encaja con cómo se comunican estos backends —por tablas, no por HTTP,
ver `arquitectura.md`— y además da dos cosas que un HMAC por sí solo no da:
**vencimiento** y **un solo uso de verdad**.

## La tabla `signature_tokens` (base `app`)

| Columna | Qué es |
|---|---|
| `token_hash` | SHA-256 del token crudo. Único. |
| `user_id` | A quién pertenece el link. |
| `origen` | `bot` o `panel`. Para saber de dónde salió cuando algo no cuadra. |
| `expires_at` | Emisión + 72 h. |
| `used_at` | Cuándo se gastó. `null` mientras sirve. |

Un token sirve solo si existe, `used_at` es `null` y `expires_at` es futuro.

## Quién hace qué

- **`pw-appbackend`** tiene `App\Support\TokenFirma`: `emitir`, `usuario`,
  `fila` y `marcarUsado`. Es el único que **verifica**.
- **`pw-adminbackend`** emite escribiendo la misma fila desde
  `EmployeeController::sendSignatureLink`. No verifica nada.

> **Si cambia el formato, cambia en los dos.** El panel no puede llamar a
> `TokenFirma` porque son repos distintos: lo que los mantiene de acuerdo es
> este contrato y que los dos escriben `hash('sha256', $crudo)` en la misma
> tabla.

## Qué había antes (15/10/2026)

El token era **el teléfono en base64**. Ocho dígitos, enumerables. Quien supiera
el número de un empleado podía abrir su pantalla de firma y, si esa persona
todavía no había firmado, dejar una firma a su nombre.

Además `store()` no comprobaba si ya había firmado, así que se podía **pisar la
firma de un adelanto ya otorgado** — eso se cerró el mismo día, aparte.

## Cómo se desplegó, para la próxima vez que haya que cambiarlo

Por etapas, para no romper ningún link ya enviado:

1. El verificador acepta **los dos** formatos, anotando en el log cada uso del
   viejo.
2. Se cambian los **dos generadores**.
3. Recién entonces se quita el formato viejo.

Se midió antes: había **un solo link pendiente** en 14 días, del 8 de octubre.

Verificado entre los dos servidores: un token escrito por el panel en
`pw-staging` lo resuelve el app backend en `pw-backend`; el teléfono en base64
ya no sirve ni por GET ni por POST; un token inventado tampoco; y emitir uno
nuevo invalida el anterior.

## Pendiente

- Los tokens gastados y vencidos se acumulan. No estorban, pero conviene un
  `prune` cuando la tabla crezca.
- El `?force=1` del panel sigue siendo solo un flag de auditoría: el reseteo real
  de la firma lo hace el panel NULLeando las columnas antes de mandar el link.
