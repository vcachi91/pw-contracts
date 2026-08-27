# pw-contracts — la fuente única de verdad entre los repos de Payway

Payway vive en **6 repos separados**. Cuando un módulo cruza varios (y casi todos
cruzan), el riesgo no es escribir el código: es que cada repo entienda algo
distinto por la misma palabra. Un `status` que en la app se llama `paid` y en el
panel `applied` cuesta un bug por cada cambio futuro.

Este repo existe para que **eso se defina una sola vez, en un solo archivo**.

## Regla

> Si un dato, un estado o un endpoint lo tocan **dos o más repos**, se define
> aquí ANTES de escribirlo en cualquiera de ellos.
>
> Los repos **implementan** el contrato. No lo negocian por su cuenta ni lo
> re-documentan en su propio `docs/`.

Cuando algo cambia —un estado nuevo, un campo que se renombra— se cambia **aquí
primero**, en un commit que explique por qué. Los repos se alinean después.

## Qué hay

| Carpeta | Qué contiene |
|---|---|
| `arquitectura.md` | Mapa de los 6 repos: qué es cada uno, con quién habla, dónde deploya. **Empezar por aquí.** |
| `modulos/` | Contratos vivos. Módulos en construcción o en producción que se siguen tocando. |
| `modulos/saldo-disponible.md` | Cómo se devenga el saldo EWA y por qué. **Leer antes de tocar cualquier cosa que mueva el disponible del empleado.** |
| `historicos/` | Contratos ya implementados y cerrados. Se conservan porque explican *por qué* el código es como es. No se editan. |

## Cómo se usa con Claude Code

Cada uno de los 6 repos tiene un `CLAUDE.md` que apunta a este repo. Al abrir una
sesión en cualquiera de ellos, Claude ya sabe que estos contratos existen y dónde
leerlos — sin que haya que explicárselo ni pegarle texto a mano.

Para trabajo que cruza repos (como adelantos extraordinarios, que vive en los 6),
conviene **una sola sesión** con `/add-dir` apuntando a los repos involucrados,
en vez de una sesión por repo relayando mensajes.

## Por qué en git y no en Drive

Drive sirve para compartir con gente que no es dev. Para un contrato que define
reglas legales y estados canónicos, git gana en lo que importa: **historial de
quién cambió qué regla y cuándo**, diffs revisables, y cero fricción de
autenticación para las herramientas que lo leen.
