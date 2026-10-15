# Auditoría de seguridad de Payway — octubre 2026

Propuesta aprobada por el dueño el 14/10/2026. Corre sola, 2 veces por día
hábil (8:00 y 13:00, hora de Panamá), una fase por corrida, hasta terminar.

**Reglas (no negociables):**
- Solo lectura. No se modifica código de ningún repo salvo esta carpeta.
- No se llama a producción, no se ataca nada, no se manda ningún mensaje ni
  correo a empleados ni clientes.
- No se corrige nada: cada hallazgo propone la corrección y su costo, y el
  dueño decide.
- Ningún secreto se copia completo en los informes: se cita archivo y línea, y
  como mucho los primeros 4 caracteres.

**Cómo se usa este archivo:** cada corrida toma la PRIMERA fase en estado
`pendiente`, la pasa a `en curso`, la hace, escribe `fase-N.md` en esta
carpeta, la marca `hecha` con la fecha y suma sus hallazgos a la tabla de
abajo. Si una fase no entra en una corrida, queda `en curso` con una nota de
dónde siguió, y la próxima corrida la continúa.

## Fases

| # | Qué | Estado | Fecha | Notas |
|---|---|---|---|---|
| 0 | Mapa de riesgos y lista de lo ya conocido | pendiente | | |
| 1 | API de la app (pw-appbackend) | pendiente | | |
| 2 | API del panel admin (pw-adminbackend) | pendiente | | |
| 3 | API de Enterprise (pw-hrbackend) | pendiente | | |
| 4 | Paneles web (pw-adminfrontend, pw-hrfrontend) | pendiente | | |
| 5 | App móvil (pw-mobileapp) | pendiente | | |
| 6 | Servidores (solo lectura, por SSH) | **local** | | No corre en la nube: la hace la sesión local de Claude Code con el dueño. La nube la salta. |
| 7 | Informe final por gravedad | pendiente | | Se hace cuando 0 a 5 estén `hechas`. |

## Hallazgos

| ID | Gravedad | Dónde | Qué | Fase |
|---|---|---|---|---|

Gravedad: **Crítica** (se puede explotar hoy y toca plata o datos de
empleados) · **Alta** · **Media** · **Baja** · **Info**.

## Ya conocido antes de empezar (entra en la fase 0)

- Clave de DeepSeek escrita en `pw-mobileapp/lib/controllers/ia_assistant_controller.dart`: se puede sacar del APK.
- `pw-mobileapp/upload_certificate.pem` está en el repo.
- Usuarios creados por WhatsApp o el panel quedan con la contraseña 123456 (el dueño decidió no forzar el cambio, 14/10/2026).
- El token de Codemagic quedó expuesto en una conversación; falta rotarlo.
- Abuso de códigos OTP el 17/09/2026 (se puso límite por número).
- `POST /auth/login/check` y `GET /auth/pre-registration/lookup` permiten saber si un número tiene cuenta.
