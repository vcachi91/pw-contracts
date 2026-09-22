# Producción — servidores, accesos y qué se usa de verdad

Este archivo existe porque **en los servidores y en el disco local hay más
directorios que proyectos**. Copias, respaldos, vhosts apagados y carpetas
viejas conviven con lo que está en vivo, y tienen nombres parecidos. Editar el
directorio equivocado no da error: simplemente no pasa nada, o peor, rompe algo
que nadie mira hasta que un empleado no puede retirar su salario.

Verificado contra los tres servidores el **2026-09-22**. Si algo de acá no
coincide con lo que ves, gana lo que veas: avisá y se corrige este archivo.

---

## Regla de oro

> **Producción es la única instalación que existe.** No hay ambiente de pruebas.
> Los dominios que dicen `stage` son producción (ver abajo). Cada cambio que se
> copia a un servidor lo están usando personas reales ese mismo minuto.

Tres cosas que **nunca** se hacen en un servidor:

1. **`git checkout`, `git stash`, `git reset --hard` o `git pull`** en un
   directorio de `/var/www`. El código que corre **no es** el commit de `HEAD`
   (ver *Cómo se despliega*): un checkout borra semanas de trabajo en vivo.
2. **`php artisan config:cache`**. Los `.env` de estos servidores tienen valores
   que el cache congela mal; ha dejado APIs caídas. `cache:clear` sí.
3. **Borrar un directorio** porque "parece un respaldo". Confirmar primero en la
   tabla de *Qué corre en cada servidor*.

Y una que conviene: antes de sobrescribir un archivo, copiarlo
(`cp archivo archivo.bak.$(date +%Y%m%d)`) o respaldar el directorio en `/root`.

---

## Los tres servidores

Todos Ubuntu 24.04, 4 GB de RAM, disco de 79 GB. Se entra por alias ssh ya
configurados en `~/.ssh/config`, con la llave dedicada `~/.ssh/pw_automation`
(sin passphrase, revocable). El login por contraseña está cerrado: solo llave.

| Alias | IP | Rol |
|---|---|---|
| `pw-backend` | 45.33.126.12 | API de la app del empleado |
| `pw-staging` | 45.56.127.102 | Las dos APIs de los paneles **y la base de datos** |
| `pw-frontend` | 50.116.28.28 | Los dos paneles web (Next.js con pm2) |

```sh
ssh pw-backend 'uptime'          # así de simple
scp archivo.php pw-staging:/tmp/ # subir algo puntual
```

**`stage` es producción.** Los dominios `admin-api.stage.payway.pa` y
`enterprise-api.stage.payway.pa` sirven a los paneles reales y reciben el webhook
de Twilio (el bot de WhatsApp). El nombre es herencia; renombrarlos es un
proyecto aparte (DNS, certificados, Twilio, cuatro `.env`), no un cambio rápido.

---

## Qué corre en cada servidor

Estos **cinco directorios** son todo lo que está en vivo. Cualquier otro
directorio en `/var/www` es resto de algo apagado.

| Servidor | Directorio | Dominio | Repo |
|---|---|---|---|
| `pw-backend` | `/var/www/app.api.payway.pa` | app.api.payway.pa | `pw-appbackend` |
| `pw-staging` | `/var/www/admin-api.stage.payway.pa` | admin-api.stage.payway.pa | `pw-adminbackend` |
| `pw-staging` | `/var/www/enterprise-api.stage.payway.pa` | enterprise-api.stage.payway.pa | `pw-hrbackend` |
| `pw-frontend` | `/var/www/pw-adminfrontend` | admin.payway.pa | `pw-adminfrontend` |
| `pw-frontend` | `/var/www/pw-hrfrontend` | enterprise.payway.pa | `pw-hrfrontend` |

**Restos que siguen en disco y NO se tocan** (apagados en agosto 2026, se
conservan por si acaso): en `pw-frontend`, `pw-adminfrontendbk` y
`pw-hrfrontendbk`; en `pw-backend`, los vhosts movidos a
`/root/nginx-desactivados-ago2026/` y `html/api/public_html`. Los respaldos
comprimidos están en `/root/respaldos-descartados-ago2026/`.

### Detalles que hacen falta

**Los paneles** corren con pm2 detrás de nginx: `admin.payway.pa` es el proceso
pm2 **id 0** en el puerto **5000**, `enterprise.payway.pa` el **id 1** en el
**5001**. `pm2 list` para verlos, `pm2 logs 0 --lines 50` para el error real.

**Las colas** las maneja supervisor en `pw-backend`, con tres grupos definidos
(`app-queue`, `admin-queue`, `enterprise-queue`) pero **solo `app-queue` está
corriendo** — los otros dos están en `STOPPED` a propósito. `app-queue` escucha
las colas `app` y `default`; si un job nuevo se encola en otro nombre, no lo
toma nadie y el trabajo se queda esperando sin dar error. Ver con
`supervisorctl status`.

**Cron:** en `pw-backend` corre cada minuto `php artisan notifications:send`.
Es lo único programado.

---

## La base de datos

**MySQL vive en `pw-staging`** (45.56.127.102), y es **una sola instancia para
todo**. Dos bases:

| Base | Tablas | Quién la usa |
|---|---|---|
| `app` | 60 | `pw-appbackend` (la app) **y** `pw-adminbackend` (panel Admin) |
| `hr` | 39 | `pw-hrbackend` (panel Enterprise) |

`pw-backend` se conecta **por red** a esa IP (escucha en `0.0.0.0`); los dos
backends de `pw-staging` se conectan por `localhost`. Las credenciales están en
el `.env` de cada directorio — **no se copian a ningún archivo versionado.**

> **Esto es lo más importante de entender del sistema:** los tres backends
> **no se llaman entre sí por HTTP, se comunican por tablas compartidas.** Un
> cambio de esquema hecho desde un repo afecta a los otros dos aunque su código
> no se toque. Por eso los nombres de campos y estados se acuerdan en
> `modulos/` antes de escribirlos.

Detalle útil: `app` y `hr` usan **ids de company distintos** para la misma
empresa. El cruce se hace por `slug`, nunca por id (ver
`modulos/` y el historial de `equipoHR`).

### Consultar producción sin romper nada

La forma segura y la que más se usa: **tinker en modo lectura**, con el script
en un archivo para no pelear con las comillas.

```sh
# 1. escribir la consulta en local y subirla
scp consulta.php pw-backend:/tmp/consulta.php

# 2. ejecutarla como el usuario correcto
ssh pw-backend 'cd /var/www/app.api.payway.pa && \
  sudo -u www-data HOME=/tmp php artisan tinker --execute="require \"/tmp/consulta.php\";"'
```

`sudo -u www-data` y `HOME=/tmp` no son adorno: sin eso tinker escribe archivos
de historial como root y deja el directorio con permisos rotos.

Para SQL suelto en `pw-staging`: `ssh pw-staging 'mysql -e "select ..."'`
(entra por socket como root). Existe además un usuario **`claude_readonly`**
pensado para consultas de solo lectura.

**Antes de escribir en la base:** más de un par de filas, hacer respaldo primero
(`mysqldump` de las tablas afectadas a `/root/`) y dejar dicho en el mensaje al
usuario cuántas filas se tocaron y cómo revertirlo.

---

## Los repos

Siete, todos hermanos en `d:\NUEVOS BK 2026\htdocs\` y todos en **GitHub
(`vcachi91`)**. GitLab (`pwmvp`) quedó como archivo de solo consulta; en varios
clones sigue como remoto `gitlab` y no se pushea ahí.

| Repo | Qué es | Stack | Rama |
|---|---|---|---|
| `pw-mobileapp` | App del **empleado** | Flutter 3.47 · Dart · GetX | `master` |
| `pw-appbackend` | API de la app | Laravel 11 · PHP 8.3 | `stage` |
| `pw-adminfrontend` | Panel **Admin** (operación Payway) | Next.js 15 · TS | `master` |
| `pw-adminbackend` | API del panel Admin | Laravel 8 | `stage` |
| `pw-hrfrontend` | Panel **Enterprise** (empresa cliente) | Next.js 15 · TS | `master` |
| `pw-hrbackend` | API del panel Enterprise | Laravel 8 | `stage` |
| `pw-contracts` | Estos contratos (solo docs) | — | `main` |

Ojo con la asimetría de ramas: **los backends trabajan en `stage`, la app y los
frontends en `master`.** Un PR al branch equivocado no falla, solo no llega.

### Carpetas locales que NO son el proyecto

En `htdocs/` hay copias viejas con nombres casi idénticos. Antes de editar,
confirmar que la ruta es exactamente la de la tabla de arriba:

| Carpeta | Peso | Qué es |
|---|---|---|
| `pw-adminfrontend - copia` | 750 MB | copia vieja, tiene `.git` propio |
| `pw-hrfrontend - copia` | 626 MB | copia vieja, tiene `.git` propio |
| `pw-mobileappOLD` | 845 MB | app anterior |
| `public_html`, `archivosapi`, `nuevo` | ~630 MB | restos del hosting viejo |
| `crm`, `landing_page`, `prospe_landing`, `dashboard` | ~1.4 GB | otros productos, fuera de Payway |
| `estaticos-2026` | — | los tres sitios WordPress convertidos a HTML (no es Payway) |

---

## Cómo se despliega (y por qué git en el servidor miente)

**El `git` de los servidores está desactualizado a propósito.** Ejemplo real a
hoy: en `pw-backend`, `HEAD` es un commit del **27/08** y hay **48 archivos
modificados** en el working tree — esos 48 archivos son el código de septiembre
que está corriendo ahora mismo. El repo del servidor quedó como referencia; los
cambios llegan **copiando archivos**, no con `git pull`.

Por eso un `git checkout .` en el servidor borraría un mes de producción. Y por
eso, para saber qué código corre de verdad, se lee **el archivo**, no `git log`.

### Backends (los tres)

El patrón que se usa, paso a paso:

1. Comparar el archivo local contra el del servidor (`md5sum`) para ver qué
   cambió realmente.
2. Respaldar: `tar czf /root/backup-<algo>-$(date +%Y%m%d%H%M%S).tar.gz <rutas>`.
3. Copiar los archivos con `scp`.
4. **`php -l`** sobre cada archivo PHP copiado. No es opcional.
5. `chown www-data:www-data` a lo copiado.
6. Probar el endpoint de verdad (curl, o el flujo en la app) antes de dar por
   hecho que funciona.

> `php -l` solo detecta errores de sintaxis. **No** detecta una clase mal
> referenciada: un `Carbon::` sin `\` pasó el lint y rompió la consulta de saldo
> del bot de WhatsApp para 19 personas durante 17 horas. Si el cambio toca una
> ruta de código con clases, ejecutarla una vez contra un usuario real.

### Frontends (los dos paneles)

**Nunca compilar a mano en el servidor.** Hay un script que hace los pasos
siempre igual y no toca producción si el build falla:

```sh
ssh pw-frontend 'bash /var/www/pw-adminfrontend/scripts/build-y-publicar.sh'
ssh pw-frontend 'bash /var/www/pw-hrfrontend/scripts/build-y-publicar.sh'
```

Existe porque el servidor tiene 4 GB de RAM y sirve los dos paneles: `next build`
necesita ~3.5 GB, así que compila fuera del directorio en vivo, respalda el
`.next` anterior y solo publica si terminó bien. Sin él, un build a medias deja
el panel caído.

Dos trampas propias de acá: **capturar el exit code del build, no el de un
pipe** (`npm run build > log 2>&1; echo $?` — con `| grep` se lee el código del
grep y un build muerto parece exitoso, eso tumbó el panel Admin una vez), y
**no dejar copias viejas de `.next*` dentro del árbol**: Tailwind v4 las escanea
y el build se queda girando en swap sin terminar nunca.

### App móvil

No se "despliega": se publica. Android sale de `flutter build appbundle` y se
sube a Play Console; iOS sale por Codemagic y se envía a mano desde App Store
Connect. Subir `version:` en `pubspec.yaml` antes de cada build (Play rechaza un
`versionCode` repetido). Ver el `CLAUDE.md` de `pw-mobileapp`, que tiene las
reglas que ya costaron caro (plugins nativos, R8, fotos).

**Forzar que la gente actualice** es aparte: Firebase Remote Config
(`min_version_ios`, `min_version_android`). Hay un script listo en
`pw-staging:/root/rc_min_version.php`. Nunca subir el mínimo antes de que la
versión esté visible en la tienda, o se bloquea a todos sin nada que descargar.

---

## Cosas que sorprenden (y que conviene saber antes)

- **`APP_ENV=local` y `APP_DEBUG=true` en los tres backends de producción.** Está
  así hoy; significa que un error puede devolver trazas al cliente. Cambiarlo no
  es trivial (hay código que depende de esa config), pero hay que saberlo.
- **El panel Admin y la app leen la misma base `app`.** Un cambio en el panel se
  ve en la app al instante, sin deploy de la app.
- **Los dos frontends salen de la misma plantilla (WowDash)** y arrastran
  secciones que no son de Payway (`chat`, `email`, `calendar`, `basic-table`…).
  Es código muerto que confunde al buscar.
- **Son casi idénticos entre sí**: un cambio de plataforma normalmente hay que
  hacerlo dos veces, una en cada panel.
- **Hay un módulo en producción cuyo código no está en ninguna rama**
  (adelantos extraordinarios). Antes de tocarlo, leer su contrato en `modulos/`
  y mirar el archivo en el servidor, no el repo.

---

## Convenciones que aplican a todos los repos

- **Dinero como string** en JSON (`"120.00"`), nunca float. El redondeo se decide
  en backend; los clientes solo muestran.
- **Las reglas de negocio se validan en el backend.** La UI las refleja, nunca es
  la única barrera. Vale en especial para los topes legales (15 % y 50 %) y el
  saldo disponible.
- **Estados canónicos**: los define el contrato del módulo. Ningún repo inventa
  estados ni los traduce "para que se lea mejor en mi panel".
- **Todo el texto de usuario va en español panameño.**

---

## Antes de cambiar algo en producción

1. ¿Está en la tabla de los cinco directorios? Si no, es un resto.
2. ¿El cambio cruza repos (campo, estado, endpoint)? Entonces primero el
   contrato en `modulos/`.
3. ¿Hay respaldo, y sé cómo revertir en un comando?
4. ¿Lo probé contra un caso real, no solo `php -l` o un test?
5. Al terminar, decir qué se tocó, en qué servidor, y cómo se revierte.
