# Arquitectura Payway — mapa de los repos

Todos son hermanos en `d:\NUEVOS BK 2026\htdocs\`. Rutas relativas entre ellos:
`../pw-appbackend`, `../pw-hrfrontend`, etc.

Dónde corre cada uno, cómo se entra y cómo se despliega está en
[`produccion.md`](produccion.md). Ahí también está la lista de carpetas locales
que **parecen** el proyecto y no lo son (`pw-adminfrontend - copia`,
`pw-mobileappOLD`, …).

## Los 6 repos de producto (+ este, `pw-contracts`)

| Repo | Qué es | Stack | Rama |
|---|---|---|---|
| `pw-mobileapp` | App del **empleado** (Android/iOS) | Flutter 3.47 · Dart · GetX | `master` |
| `pw-appbackend` | API que sirve a la app | Laravel 11 · PHP 8.3 | `stage` |
| `pw-hrfrontend` | Panel **Enterprise** (la empresa cliente) | Next.js 15 · React 18 · TS · Tailwind | `master` |
| `pw-hrbackend` | API del panel Enterprise | Laravel 8 | `stage` |
| `pw-adminfrontend` | Panel **Admin** (operación Payway) | Next.js 15 · React 18 · TS · Tailwind | `master` |
| `pw-adminbackend` | API del panel Admin | Laravel 8 | `stage` |

**Todos viven en GitHub (`vcachi91`)** desde el 2026-08-26. GitLab (`pwmvp`)
quedó como archivo de solo consulta y sigue presente en varios clones como
remoto `gitlab`: no se pushea ahí.

Ojo con la asimetría de ramas: **los backends trabajan sobre `stage`, la app y
los frontends sobre `master`.** Un PR a la rama equivocada no falla, solo no
llega a producción.

## Quién habla con quién

```
  EMPLEADO              EMPRESA CLIENTE           OPERACIÓN PAYWAY
     │                        │                          │
 pw-mobileapp          pw-hrfrontend            pw-adminfrontend
     │                        │                          │
 pw-appbackend          pw-hrbackend             pw-adminbackend
     └────────────────────────┴──────────────────────────┘
                              │
                      MySQL COMPARTIDA
```

**El punto clave:** los tres backends comparten la misma base de datos MySQL. No
se llaman entre sí por HTTP — se comunican por las tablas. Por eso los nombres de
campos y estados tienen que coincidir: un `status` mal nombrado no falla en un
test, falla en producción cuando el otro panel lo lee.

Corolario: un cambio de esquema (columna nueva, enum nuevo) **afecta a los tres
backends aunque lo haga uno solo**. Se anuncia en el contrato del módulo.

## Convenciones que aplican a todos

- **Dinero como string** en JSON (`"120.00"`), nunca float. Redondeo y coma
  decimal se deciden en backend; los clientes solo muestran.
- **Estados canónicos**: los define el contrato del módulo. Ningún repo inventa
  estados propios ni traduce nombres "para que se lea mejor en mi panel".
- **Las reglas de negocio se validan en backend.** Los clientes (app y paneles)
  las reflejan en la UI, pero nunca son la única barrera. Aplica en especial a
  los topes legales del 15% y 50%.
- **`status_label`**: los backends mandan el texto listo para mostrar junto al
  `status` crudo. Los clientes prefieren el label y usan el crudo solo para
  lógica (color, ícono, permisos).

## Deuda técnica conocida (transversal)

- Los mensajes de commit de los backends son casi todos `fix`. Encontrar cuándo
  entró un cambio es arqueología. Vale la pena arreglarlo de aquí en adelante.
- Los dos frontends salen de la misma plantilla (WowDash) y arrastran secciones
  que no son de Payway: `chat`, `email`, `calendar`, `form-validation`,
  `basic-table`, `users-grid`. Es código muerto que confunde al buscar.
- `pw-hrfrontend` y `pw-adminfrontend` son casi idénticos salvo por las secciones
  propias (`payway/` en HR; `companies/`, `sub-companies/`, `transactions/` en
  Admin). Un cambio de plataforma normalmente hay que hacerlo dos veces.
