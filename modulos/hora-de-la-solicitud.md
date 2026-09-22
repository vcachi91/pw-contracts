# Hora de una solicitud — cuál es "cuándo la hizo"

Una solicitud de la app se guarda en **dos momentos distintos**, y hasta hoy cada
canal mostraba uno diferente. El panel Admin mostraba el primero; el correo a
operaciones sale en el segundo. Este contrato fija cuál es la hora buena y cómo
se llama el dato.

Repos que lo tocan: `pw-appbackend` (la escribe), `pw-adminbackend` (la expone),
`pw-adminfrontend` (la muestra). `pw-mobileapp` no la usa.

## Los dos momentos

| | Cuándo ocurre | Dónde queda | Estado |
|---|---|---|---|
| **Inicio** | el empleado entra al resumen y ve el monto | `orders.created_at` | `ongoing` |
| **Firma** | el empleado firma y confirma | fila de `order_statuses` con `status='requested'` | `requested` |

El correo a operaciones se manda **solo en la firma**, dentro del mismo request
(`Mail::send`, sincrónico). Su hora de llegada es la hora de la firma.

Las solicitudes del bot de WhatsApp nacen ya en `requested`: los dos momentos
son el mismo segundo y este contrato no las afecta.

## La regla

> **"Cuándo hizo la transacción" es la hora de la FIRMA**, no la del inicio.
>
> Es el instante en que la persona aceptó el contrato y nació la obligación de
> desembolsar. Antes de la firma hay una intención, no una solicitud.

Los tres repos la llaman **`requested_at`**. Ningún panel muestra
`orders.created_at` como si fuera la hora de la solicitud.

- `pw-adminbackend` agrega `requested_at` a la respuesta de transacciones,
  tomándolo de la fila `requested` de `order_statuses`. Si no existe, va `null`.
- `pw-adminfrontend` muestra `requested_at` y cae a `created_at` solo si viene
  `null`, sin inventar etiquetas nuevas.
- No se agrega columna a `orders`: el dato ya existe en `order_statuses`. Un
  cambio de esquema tocaría los tres backends para nada.

**El inicio no se borra**: sigue sirviendo para medir cuánto tarda la gente en
firmar. Si un panel lo muestra, lo rotula como "inicio", nunca como la fecha de
la solicitud.

## Por qué apareció el 22/09/2026

Operaciones reportó dos casos donde el correo y el panel no coincidían:

| Orden | Inicio (lo que mostraba el panel) | Firma (hora del correo) | Brecha |
|---|---|---|---|
| 2831 | 21/09 21:25:08 | 22/09 00:21:59 | 2 h 56 m |
| 2917 | 22/09 13:55:12 | 22/09 14:48:11 | 53 m |

No era un problema de zona horaria: `pw-appbackend` y `pw-adminbackend` están
los dos en `America/Panama` y el panel convierte a esa zona.

**Por qué no se había visto antes.** En las 528 firmas de la app de los 30 días
previos, la brecha promedio es de **1 m 43 s** y solo **3** pasaron de una hora.
Las dos reportadas son parte de esas 3, y la 2831 es el máximo del mes. Con una
brecha de segundos las dos horas se ven iguales y el error no se nota.

## Confirmaciones duplicadas (defecto aparte, mismo endpoint)

`submitRequest` valida que la orden esté en `ongoing` y después la guarda como
`requested`, sin candado en medio. **Dos llamadas simultáneas pasan las dos.**

La orden 2831 se confirmó **3 veces en el mismo segundo** (00:21:59): quedaron 3
filas `requested` en `order_statuses` y los 4 correos de administración salieron
3 veces cada uno, 12 en total por una sola solicitud. En 60 días hay **5**
órdenes con confirmación duplicada.

La regla del contrato:

> Una orden tiene **una sola** fila por estado alcanzado. El historial de
> estados es auditoría: una fila repetida es un dato falso, no un duplicado
> inocente.

`pw-appbackend` cierra la carrera al confirmar (bloqueo de fila en la
transacción, o condición sobre el estado en el propio `update`) y **no manda el
correo si no fue esta llamada la que cambió el estado**. El consumidor no puede
arreglar esto: el correo ya salió.

Consecuencia para quien lee `order_statuses`: hasta que se limpien las filas
viejas, **la hora de la firma se toma con `min(created_at)`** de las filas
`requested`, no con `count()` ni asumiendo que hay una sola.

## El correo lleva la fecha sin hora

`mail/request_summary.blade.php` imprime `now()->format('d/m/Y')`. Operaciones
no tiene con qué comparar salvo la hora de llegada a la bandeja, que es lo que
originó el reporte. El correo pasa a llevar **fecha y hora de la firma**, en
horario de Panamá.

## Preguntas abiertas

- Las 5 órdenes con filas `requested` duplicadas: ¿se limpian dejando la más
  antigua, o se conservan como evidencia del defecto? Hay que decidirlo antes de
  que algún reporte cuente estados.
- `enterprise-api` corre sin `TIMEZONE` en su `.env`, o sea en **UTC**, mientras
  los otros dos backends están en `America/Panama`. No afecta a este módulo
  porque el panel Enterprise no muestra estas órdenes, pero cualquier fecha que
  ese backend escriba o serialice queda corrida 5 horas. Revisar aparte.

---

## Estado: implementado el 22/09/2026

Desplegado por copia de archivos, con respaldo previo en cada servidor.

| Repo / servidor | Archivo | Qué cambió |
|---|---|---|
| `pw-appbackend` en `pw-backend` | `RequestController.php` | candado atómico en `submitRequest` + hora en `$fecha` |
| `pw-appbackend` en `pw-backend` | `mail/request_summary.blade.php` | la etiqueta pasa a "Fecha y hora" |
| `pw-adminbackend` en `pw-staging` | `Models/Order.php` | relación `requestedStatus` + `requested_at` en `$appends` |
| `pw-adminbackend` en `pw-staging` | `Admin/TransactionController.php` | eager load de `requestedStatus` en listado y detalle |
| `pw-adminfrontend` en `pw-frontend` | `transactions/TransactionsTable.tsx` | `fechaSolicitud()` en la tabla, la tarjeta y el CSV |

Verificado en producción: la 2831 devuelve `requested_at` 00:21:59 y la 2917
14:48:11 (hora de Panamá), el correo renderiza "22/09/2026 4:41 p.m.", y de las
30 filas de la primera página 25 traen `requested_at`. Las 5 restantes son
canceladas, que nunca se firmaron — en 30 días son 150, todas `source=app`.

**Trampa encontrada al implementar:** `->where('status','requested')->oldestOfMany()`
devuelve siempre `null`. El `where` encadenado afuera no entra en la subconsulta
de agregación, que termina eligiendo la fila `ongoing` y el filtro externo la
descarta. Va `->ofMany(['created_at' => 'min'], fn($q) => $q->where(...))`.

**Para revertir:** `pw-backend:/root/backup-hora-solicitud-*.tar.gz`,
`pw-staging:/root/backup-requested-at-*.tar.gz`,
`pw-frontend:/root/TransactionsTable.tsx.bak.*` y, para el panel compilado,
`/var/backups/adminfe/anterior`.
