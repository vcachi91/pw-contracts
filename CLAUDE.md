# pw-contracts

Repo de **documentación pura** — no hay código, no hay build, no hay tests.

Es la fuente única de verdad entre los 6 repos de Payway. Antes de tocar nada,
leer `README.md` (la regla) y `arquitectura.md` (el mapa de los 6 repos).

## Al trabajar aquí

- **Un cambio en `modulos/` puede romper hasta 6 repos.** Renombrar un campo o
  agregar un estado no es una edición de texto: es un cambio de API. Al proponer
  uno, decir explícitamente a qué repos impacta.
- **No editar `historicos/`.** Son contratos ya implementados; explican por qué el
  código es como es. Si algo cambió, va en `modulos/` como contrato nuevo.
- Cuando se cierra una "pregunta abierta", mover la respuesta a la sección que
  corresponda del contrato y borrarla de la lista. No dejar las dos.
- Mensajes de commit descriptivos: el valor de este repo es su historial. `fix`
  no sirve.

## Al leer desde otro repo

Si llegaste aquí desde el `CLAUDE.md` de otro repo, lo que buscás está en
`modulos/`. La sección que te toca depende del repo desde el que venís — el
contrato de adelantos extraordinarios está partido en §1–§5 (núcleo, lo leen
todos) y §6–§10 (una por plataforma).
