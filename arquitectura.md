# Arquitectura Payway — mapa de los 6 repos

Todos son hermanos en `d:\NUEVOS BK 2026\htdocs\`. Rutas relativas entre ellos:
`../pw-appbackend`, `../pw-hrfrontend`, etc.

## Los 6 repos

| Repo | Qué es | Stack | Rama | Remoto |
|---|---|---|---|---|
| `pw-mobileapp` | App del **empleado** (Android/iOS) | Flutter 3.47 · Dart · GetX | `master` | GitLab `pwmvp` |
| `pw-appbackend` | API que sirve a la app | Laravel 11 · PHP 8.2 | `stage` | GitLab `pwmvp` |
| `pw-hrfrontend` | Panel **Enterprise** (la empresa cliente) | Next.js 15 · React 18 · TS · Tailwind | `master` | GitHub `vcachi91` |
| `pw-hrbackend` | API del panel Enterprise | Laravel 8 · PHP 7.3/8 | `stage` | GitLab `pwmvp` |
| `pw-adminfrontend` | Panel **Admin** (operación Payway) | Next.js 15 · React 18 · TS · Tailwind | `master` | GitHub `vcachi91` |
| `pw-adminbackend` | API del panel Admin | Laravel 8 · PHP 7.3/8 | `api_v1.php` · `stage` | GitLab `pwmvp` |

Ojo con la asimetría: **backends en GitLab, frontends en GitHub**, y los backends
trabajan sobre `stage` mientras app y fronts sobre `master`.

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
