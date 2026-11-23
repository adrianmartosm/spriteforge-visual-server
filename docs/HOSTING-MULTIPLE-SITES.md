# Alojar varias webs en el mismo servidor

Guía operativa para desplegar nuevos proyectos junto a SpriteForge usando
Docker Compose y el reverse proxy Traefik/Coolify que ya gestiona HTTPS.

> Esta guía es deliberadamente segura para un repositorio público. Los valores
> reales de servidor, usuario, dominio, clave SSH y secretos no se escriben
> aquí. Se obtienen de las notas privadas del entorno o del gestor de secretos.

## 1. Modelo recomendado

Cada web vive en su propio directorio y en su propio proyecto Compose:

```text
/srv/sites/
  spriteforge/                  # proyecto existente
  nuevo-proyecto/               # un proyecto Compose independiente
    compose.yml
    .env                         # sólo en el servidor, nunca en Git
    app/                          # checkout o release desplegada
```

Cada proyecto debe tener:

- nombres únicos de servicio, router y middleware de Traefik;
- un dominio o subdominio propio;
- una red interna propia para sus servicios;
- conexión a la red externa del proxy (`coolify`, si esa es la red existente);
- volúmenes persistentes fuera del checkout;
- ningún puerto publicado directamente a Internet salvo el reverse proxy.

El proxy termina TLS y envía el tráfico al contenedor web. El API, la base de
datos y los servicios auxiliares sólo deben usar redes Docker internas.

## 2. Acceso SSH

Usar una cuenta de despliegue sin privilegios de root y una clave privada
gestionada localmente. Sustituir los placeholders por valores de las notas
privadas del entorno:

### Linux/macOS/WSL

```bash
chmod 600 "$SPRITEFORGE_SSH_KEY"
ssh -o IdentitiesOnly=yes \
  -i "$SPRITEFORGE_SSH_KEY" \
  "$SPRITEFORGE_DEPLOY_USER@$SPRITEFORGE_SERVER_HOST"
```

### PowerShell

```powershell
$env:SPRITEFORGE_SSH_KEY = "C:\Users\<usuario>\.ssh\spriteforge_deploy"
$env:SPRITEFORGE_DEPLOY_USER = "<deploy-user>"
$env:SPRITEFORGE_SERVER_HOST = "<server-host-or-ip>"

ssh -o IdentitiesOnly=yes `
  -i $env:SPRITEFORGE_SSH_KEY `
  "$env:SPRITEFORGE_DEPLOY_USER@$env:SPRITEFORGE_SERVER_HOST"
```

No copiar la clave privada al servidor, no pegarla en tickets/chats y no
guardarla en Git. Si el acceso falla, comprobar primero DNS, puerto 22,
permisos de la clave y que la clave pública esté en `authorized_keys` del
usuario de despliegue. No habilitar login SSH de root ni autenticación por
contraseña para resolver un fallo temporal.

Una vez dentro, confirmar que se está en el host correcto antes de tocar nada:

```bash
hostname
whoami
docker version
docker network ls
```

## 3. DNS y dominio

Para `nuevo.example.com`:

1. Crear un registro `A` hacia la IP pública del servidor.
2. Crear `AAAA` sólo si el servidor tiene IPv6 operativo y firewall configurado.
3. Esperar a que el DNS resuelva antes de probar Traefik.
4. Mantener abiertos únicamente los puertos 80 y 443 al tráfico web; SSH debe
   estar restringido según la política del servidor.

```bash
dig +short nuevo.example.com
curl -I http://nuevo.example.com
```

El certificado lo obtiene Traefik con el `certresolver` ya configurado. No
subir certificados ni claves TLS al repositorio. El router HTTP debe redirigir
a HTTPS y el router HTTPS debe declarar `tls=true`.

## 4. Compose mínimo para una web estática

Crear un `compose.yml` independiente. No reutilizar `container_name` ni
etiquetas de otro proyecto:

```yaml
name: nuevo-proyecto

services:
  web:
    build: .
    restart: unless-stopped
    networks:
      - default
      - coolify
    labels:
      - traefik.enable=true
      - traefik.docker.network=coolify
      - traefik.http.services.nuevo-web.loadbalancer.server.port=80
      - traefik.http.routers.nuevo-web.rule=Host(`nuevo.example.com`)
      - traefik.http.routers.nuevo-web.entrypoints=https
      - traefik.http.routers.nuevo-web.tls=true
      - traefik.http.routers.nuevo-web.tls.certresolver=letsencrypt
      - traefik.http.routers.nuevo-web.service=nuevo-web
      - traefik.http.routers.nuevo-web-http.rule=Host(`nuevo.example.com`)
      - traefik.http.routers.nuevo-web-http.entrypoints=http
      - traefik.http.routers.nuevo-web-http.middlewares=nuevo-https
      - traefik.http.routers.nuevo-web-http.service=nuevo-web
      - traefik.http.middlewares.nuevo-https.redirectscheme.scheme=https

networks:
  coolify:
    external: true
```

El nombre `coolify` debe coincidir con la red existente del proxy. Si el
servidor usa otra red, comprobarla con `docker network ls` y cambiar sólo ese
valor. No crear una segunda red con el mismo propósito sin entender cómo está
configurado Traefik.

## 5. Compose para una web con API

La API no debe tener `ports:`. Sólo se publica mediante `expose:` y se conecta
a la red interna del proyecto:

```yaml
services:
  web:
    build: ./web
    restart: unless-stopped
    networks: [default, coolify]
    depends_on: [api]
    labels:
      # mismas etiquetas de Traefik que en el ejemplo anterior

  api:
    build: ./api
    restart: unless-stopped
    env_file: ./.env
    expose:
      - "3000"
    volumes:
      - ./persistent/data:/data
    networks: [default]

networks:
  coolify:
    external: true
```

El Nginx del servicio `web` puede hacer proxy de `/api/` a `http://api:3000`.
Así el navegador usa el mismo origen, se simplifica CORS y la API queda fuera
de Internet. Si una API debe tener dominio propio, crear otro router Traefik
con nombre y reglas diferentes, pero mantener la base de datos sin exposición.

## 6. Primer despliegue

Ejecutar los pasos desde el servidor, dentro del directorio del proyecto:

```bash
cd /srv/sites/nuevo-proyecto

# Validar el Compose antes de construir
docker compose config

# Crear o revisar el fichero privado de entorno; nunca usar valores del Git
test -f .env

docker compose build --pull
docker compose up -d
docker compose ps
```

Si se despliega desde Git:

```bash
git fetch origin --prune
git checkout --detach <commit-revisado>
docker compose config
docker compose build --pull
docker compose up -d
```

No ejecutar `docker compose down -v`: elimina volúmenes y puede borrar datos.
Para cambios de frontend, normalmente basta reconstruir el servicio web. Si
cambia la API, migraciones o variables de entorno, reconstruir el conjunto y
validar el almacenamiento persistente antes de sustituir contenedores.

## 7. Validación después del despliegue

```bash
docker compose ps
docker compose logs --tail=100 web
docker compose logs --tail=100 api       # si existe
curl -fsS https://nuevo.example.com/ > /dev/null
curl -I https://nuevo.example.com/
```

Comprobar también:

- certificado válido y redirección HTTP → HTTPS;
- ausencia de errores 404/502 en Traefik;
- que la API responde sólo por el origen esperado;
- que no se han publicado puertos con `docker ps`;
- que los volúmenes y permisos de datos siguen intactos;
- que los logs no contienen tokens, contraseñas ni datos personales.

Para una aplicación con autenticación, probar login, logout, CSRF, subida de
archivos, límites de tamaño y recuperación ante reinicio antes de darla por
publicada.

## 8. Actualizaciones y rollback

Guardar cada despliegue como una revisión identificable por commit. Antes de
actualizar:

1. Confirmar que existe un backup reciente de la base de datos y assets.
2. Anotar el commit actualmente activo.
3. Construir la nueva imagen y ejecutar tests fuera del tráfico público.
4. Ejecutar `docker compose up -d` y revisar salud/logs.

Para volver atrás, recuperar el commit anterior y reconstruir:

```bash
git checkout --detach <commit-anterior>
docker compose build --pull=false
docker compose up -d
docker compose ps
```

El rollback de código no debe restaurar automáticamente la base de datos a una
versión anterior. Las migraciones deben ser compatibles hacia atrás o tener un
procedimiento de restauración probado.

## 9. Seguridad y mantenimiento

- Nunca subir `.env`, SQLite, backups, assets privados, claves SSH o tokens.
- Usar una cuenta de despliegue dedicada y permisos mínimos.
- Mantener Docker, Traefik, Nginx y las imágenes base actualizados.
- Publicar sólo 80/443; usar `expose` y redes internas para APIs y bases de
  datos.
- No montar `/var/run/docker.sock` en una aplicación; si Traefik lo necesita,
  usar el mecanismo gestionado por el host o un socket-proxy.
- Aplicar cabeceras de seguridad, CSP, `X-Content-Type-Options` y protección
  contra clickjacking en cada web.
- Configurar límites de subida, rate limiting y validación server-side.
- Redactar secretos y datos personales de los logs.
- Mantener backups fuera del checkout y probar periódicamente una restauración.
- No reutilizar nombres de router, servicio, middleware o contenedor entre
  proyectos.

## 10. Checklist para futuros proyectos

Antes de pedir el despliegue a un agente:

- [ ] Repositorio y commit exactos definidos.
- [ ] Dominio y registros DNS creados.
- [ ] `compose.yml` con nombres únicos.
- [ ] Red externa del proxy confirmada.
- [ ] `.env` preparado sólo en el servidor o gestor de secretos.
- [ ] Volúmenes persistentes identificados y respaldados.
- [ ] Health check y ruta de rollback documentados.
- [ ] Puertos directos cerrados.
- [ ] Tests locales y `docker compose config` correctos.
- [ ] No hay secretos ni datos privados en Git.

### Variables que debe recibir el agente

Entregar estos valores por un canal privado, nunca dentro del repositorio:

```text
SERVER_HOST=<host o IP del servidor>
DEPLOY_USER=<usuario SSH sin privilegios de root>
SSH_KEY_PATH=<ruta local a la clave privada>
DEPLOY_ROOT=/srv/sites/<slug>
PUBLIC_DOMAIN=<dominio del nuevo proyecto>
PROXY_NETWORK=coolify
COMPOSE_PROJECT_NAME=<slug único>
```

El agente debe comprobar el host, usuario y dominio antes de ejecutar cambios,
hacer backup antes de tocar datos persistentes y no asumir que una publicación
en GitHub equivale a un despliegue en producción.
