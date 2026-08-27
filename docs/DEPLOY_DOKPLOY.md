# Despliegue de openGym en Dokploy

VPS Contabo · Dokploy con Traefik + Let's Encrypt · repo `Llorens19/openGym`

Fichero de despliegue: **`docker-compose.dokploy.yml`** (no uses `docker-compose.yml`,
ver la nota al final).

---

## 0. Antes de empezar: lo que no se puede cambiar después

openGym autentica **solo con passkeys** (WebAuthn). No hay contraseña, ni recuperación
por email. Dos consecuencias:

- **El hostname queda grabado en cada passkey.** `RP_ID=gym.xequeproject.es` significa que
  las passkeys registradas solo valen en ese hostname exacto. Cambiarlo después las
  invalida todas y los usuarios quedan fuera sin forma de entrar.
- **`../files/data` es el único sitio donde vive todo.** Usuarios, passkeys, histórico de
  entrenos, peso corporal, secreto de sesión y claves VAPID. Si se pierde ese directorio,
  se pierden las cuentas de forma irreversible. Backup desde el día uno (sección 6).

Decide el subdominio ahora, no después de que nadie se registre.

---

## 1. DNS

**Probablemente ya está hecho.** `xequeproject.es` tiene un registro wildcard: cualquier
subdominio inventado resuelve a `75.119.157.169`, así que `gym.xequeproject.es` ya apunta al
VPS sin tocar nada. Confírmalo desde tu máquina o por SSH al VPS:

```bash
dig +short gym.xequeproject.es      # debe devolver 75.119.157.169
```

Si por lo que sea no resuelve, crea el registro explícito:

| Tipo | Nombre | Valor | TTL |
|---|---|---|---|
| A | `gym` | `75.119.157.169` | 300 (auto) |

Cloudflare (lo usas ya en XQP): el subdominio debe ir en **nube gris (DNS only)** al menos
hasta que Let's Encrypt emita el certificado. Dokploy usa desafío HTTP-01 y con el proxy
naranja delante la emisión puede fallar. Una vez emitido, activa el proxy si te interesa —
pero ojo: si lo activas, el `ORIGIN` sigue siendo `https://gym.xequeproject.es`, no cambia.

Puertos 80 y 443 abiertos en el firewall del VPS y en el panel de Contabo.

---

## 2. Crear el proyecto en Dokploy

1. **Project → Create** (o reutiliza uno existente) → dentro, **Create Service → Compose**.
2. **Provider: GitHub** (o Git genérico con la URL pública):
   - Repository: `Llorens19/openGym`
   - Branch: `main`
   - **Compose Path: `./docker-compose.dokploy.yml`** ← importante, no el por defecto
   - Compose Type: `Docker Compose`
3. Guarda. **No despliegues todavía** — faltan las variables.

---

## 3. Variables de entorno

Pestaña **Environment** del servicio Compose:

```env
RP_ID=gym.xequeproject.es
ORIGIN=https://gym.xequeproject.es
RP_NAME=openGym
SESSION_DAYS=90
```

Reglas:

- `RP_ID` = **solo el hostname**, sin `https://`, sin barra final, sin puerto.
- `ORIGIN` = **la URL completa** con `https://`, sin barra final.
- Ambos deben coincidir *exactamente* con lo que sale en la barra de direcciones del
  navegador. Un `www.` de más y las passkeys fallan con "verification failed".

`ADMIN_UIDS` e `INVITE_ONLY` se dejan **vacías en el primer deploy** — ver sección 5.

> El compose está escrito con `${RP_ID:?...}`: si olvidas definirlas, el deploy falla con un
> mensaje explícito en el log en vez de arrancar en `localhost` y romper las passkeys de forma
> silenciosa y confusa.

---

## 4. Dominio y primer deploy

1. Pestaña **Domains → Add Domain**:
   - Host: `gym.xequeproject.es`
   - **Service Name: `web`**
   - **Container Port: `80`**
   - HTTPS: **on** · Certificate provider: **Let's Encrypt**
2. **Deploy**.

Qué esperar en el log, en orden:

- Build del servicio `api` (rápido, `npm install --omit=dev`).
- Build del servicio `web`: `npm ci` + `vite build` del frontend React. Es la parte lenta,
  varios minutos la primera vez. Vigila la RAM del VPS aquí — si el build muere sin
  explicación, es OOM (ver Troubleshooting).
- Servicio `media`: descarga ~140 MB de imágenes/GIFs de ejercicios. **Solo la primera vez**;
  en deploys posteriores imprime "Media ya presente" y sale al instante.
- `web` arranca cuando `media` ha terminado con éxito.

Verificación:

```bash
curl https://gym.xequeproject.es/api/health
# {"ok":true,"users":0}
```

Si devuelve eso, el frontend, nginx, el proxy /api, el contenedor api y Traefik/TLS están
todos bien de una sola vez.

---

## 5. Tu cuenta admin + cerrar el registro

Hay un orden obligatorio, porque `INVITE_ONLY` bloquea **todos** los registros incluidos
el primero, y los códigos de invitación solo los genera un admin, y admin es quien ya tiene
un `id` listado en `ADMIN_UIDS`. Si activas invite-only antes de registrarte, te quedas
fuera de tu propia instancia.

Secuencia correcta:

1. Con `INVITE_ONLY` vacío, abre `https://gym.xequeproject.es` desde el móvil o el portátil y
   **crea tu perfil con passkey**.
2. Saca tu `id` del VPS por SSH:

   ```bash
   cat /etc/dokploy/compose/*opengym*/files/data/db.json | head -20
   ```

   (o localiza el directorio exacto con `ls -d /etc/dokploy/compose/*opengym*`)

   Apunta el valor de `users[0].id`.
3. Vuelve a Dokploy → **Environment**, añade:

   ```env
   ADMIN_UIDS=elIdQueAcabasDeCopiar
   INVITE_ONLY=1
   ```

4. **Redeploy**. Ahora tienes enlace **Admin** en Ajustes (quién está entrenando, histórico
   por usuario, desactivar cuentas, generar y revocar códigos de invitación) y nadie puede
   crearse perfil sin código.

Las cuentas ya existentes siguen funcionando al activar invite-only. El panel de admin va
protegido por tu passkey y se valida en servidor, no necesita login aparte.

---

## 6. Backup

Todo lo irrecuperable vive en `../files/data`: `db.json` (usuarios y passkeys), `secret`
(clave de firma de sesiones), `vapid.json` (claves push) y un `state-<uid>.json` por usuario
con **el histórico de entrenos y el peso corporal**. Sin passkeys no hay "he olvidado mi
contraseña": si eso se pierde, se pierde.

### Lo que hay montado: tarea programada de Dokploy

Pestaña **Schedules** del servicio Compose, tarea *Backup diario de data*:

- Service: `api` · Shell: `sh` · Cron: `0 4 * * *` (UTC)
- Comando:

```sh
mkdir -p /data/_backups && \
tar czf /tmp/og.tgz -C /data --exclude=_backups . && \
mv /tmp/og.tgz /data/_backups/opengym-$(date +%Y%m%d).tar.gz && \
find /data/_backups -name 'opengym-*.tar.gz' -mtime +14 -delete && \
ls -la /data/_backups
```

Se construye en `/tmp` y se mueve después: así el archivo no se incluye a sí mismo. El
`--exclude=_backups` evita que cada copia arrastre las anteriores. Rotación de 14 días.
Los `.tar.gz` acaban en `/etc/dokploy/compose/<appName>/files/data/_backups/` en el host, que
sobrevive a los redeploys.

Puedes lanzarla a mano con el botón ▶ de la tarea y verificar el contenido desde
**Open Terminal** (contenedor `api`, shell `/bin/sh`):

```sh
ls -la /data/_backups
tar tzf /data/_backups/opengym-*.tar.gz     # deben salir db.json, secret, vapid.json y state-*.json
```

### Lo que falta: sacarlo del VPS

Un backup que solo vive en el mismo disco que el original no protege contra fallo de disco,
borrado del directorio ni pérdida del VPS. Protege contra corrupción de `db.json` y poco más.
Para cerrarlo del todo, por SSH al VPS:

```bash
# copia diaria fuera del servidor (rclone a un bucket, o scp a casa)
0 5 * * * rclone copy /etc/dokploy/compose/$(ls /etc/dokploy/compose | grep opengym | head -1)/files/data/_backups remoto:opengym-backups
```

O configura un **S3 Destination** en Dokploy (Settings → S3 Destinations) y usa la pestaña
**Backups** del servicio.

### Restaurar

Parar el compose, desempaquetar el `.tar.gz` sobre `files/data/`, arrancar. Los usuarios
además pueden exportar sus propios datos en JSON desde Ajustes.

---

## 7. Actualizar

`git push` a `main` → **Deploy** en Dokploy (o activa Auto Deploy con el webhook que te da
el panel). `../files/data` y `../files/media` no se tocan: viven fuera del directorio que
Dokploy re-clona.

---

## Troubleshooting

| Síntoma | Causa y arreglo |
|---|---|
| Deploy falla con `falta RP_ID...` | Las variables no están puestas en Environment. Sección 3. |
| El build de `web` muere sin error claro | OOM durante `vite build`. Comprueba con `free -h` y `dmesg \| grep -i oom`. Arreglo: añade swap (`fallocate -l 2G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile`) o construye la imagen en local y súbela a un registry. |
| No sale el diálogo de passkey en el móvil | Estás entrando por `http://` o por IP. Solo funciona sobre HTTPS con hostname real. |
| "verification failed" al entrar | `RP_ID`/`ORIGIN` no coinciden literalmente con la barra de direcciones. Revisa `www.`, barra final, `https://`. |
| Certificado no se emite | Puerto 80 cerrado, DNS sin propagar, o Cloudflare en nube naranja. Sección 1. |
| El servicio `web` no arranca | Espera a `media`. Mira `docker compose logs media` — normalmente es fallo de red al clonar el dataset. |
| 502 desde Traefik | El dominio apunta al servicio o puerto equivocado. Debe ser servicio `web`, puerto `80`. |
| Media no aparece (huecos en los ejercicios) | `ls /etc/dokploy/compose/*opengym*/files/media/img \| wc -l` debe dar >1000. Si da 0, redeploy para relanzar el servicio `media`. |

---

## Nota: por qué NO usar `docker-compose.yml`

El compose original está pensado para self-hosting manual y monta `./data:/data` — una ruta
**dentro del directorio que Dokploy clona con git en cada deploy**. Dos problemas:

1. Dokploy vacía y re-clona ese directorio en cada despliegue → los datos desaparecen o
   vuelven a un estado antiguo.
2. Ese `data/` está versionado en el repo e incluye un fichero `secret` público — la clave
   HMAC con la que se firman las cookies de sesión (`api/server.js:34-36`). Arrancar con él
   permitiría a cualquiera que lea el repo falsificar una cookie de sesión válida para
   cualquier usuario, incluido el admin, sin passkey.

`docker-compose.dokploy.yml` monta `../files/data`, hermano del directorio clonado, que
persiste entre deploys y arranca vacío generando un secreto nuevo.
