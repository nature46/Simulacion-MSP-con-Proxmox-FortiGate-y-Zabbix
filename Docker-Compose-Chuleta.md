# 🐳 Chuleta Docker Compose

## Referencia rápida con ejemplos reales del homelab

---

## Comandos Esenciales

### Arrancar y parar

```bash
# Arrancar todos los servicios en segundo plano
docker compose up -d

# Arrancar solo un servicio específico
docker compose up -d jellyfin

# Parar todos los servicios (sin borrar nada)
docker compose down

# Parar y borrar volúmenes (⚠️ DESTRUYE DATOS)
docker compose down -v

# Reiniciar un servicio
docker compose restart zabbix-server

# Reiniciar todos
docker compose restart
```

### Ver estado y logs

```bash
# Ver estado de todos los contenedores del stack
docker compose ps

# Logs en tiempo real de todos los servicios
docker compose logs -f

# Logs de un servicio específico
docker compose logs -f jellyfin

# Últimas 50 líneas de logs
docker compose logs --tail=50 zabbix-server

# Logs con timestamps
docker compose logs -f -t nextcloud
```

### Actualizar imágenes

```bash
# Descargar versiones nuevas de todas las imágenes
docker compose pull

# Descargar solo una imagen
docker compose pull immich-server

# Aplicar actualizaciones (recrea los contenedores)
docker compose up -d --force-recreate

# Pull + recreate en un solo comando
docker compose pull && docker compose up -d
```

### Entrar en un contenedor

```bash
# Abrir shell en un contenedor en ejecución
docker compose exec jellyfin bash
docker compose exec nextcloud sh       # Alpine usa sh, no bash
docker compose exec zabbix-server bash

# Ejecutar un comando sin entrar
docker compose exec nextcloud php occ status
docker compose exec immich-postgres psql -U immich -d immich
```

---

## Estructura de Archivos

### Estructura recomendada

```
/opt/nombre-stack/
├── docker-compose.yml     ← definición de servicios
├── .env                   ← variables de entorno (NO subir a GitHub)
├── .env.example           ← plantilla sin valores reales (SÍ subir)
└── data/                  ← datos persistentes (opcional, según configuración)
```

### Ejemplo real — Zabbix stack

```
/opt/zabbix/
├── docker-compose.yml
├── .env
└── .env.example
```

---

## Archivo .env

### Para qué sirve

El archivo `.env` almacena variables de entorno sensibles (contraseñas, tokens, URLs) fuera del `docker-compose.yml`. Así puedes subir el `docker-compose.yml` a GitHub sin exponer credenciales.

### Cómo se usan en docker-compose.yml

```yaml
environment:
  - POSTGRES_PASSWORD=${NEXTCLOUD_DB_PASSWORD}   # ← lee del .env
  - N8N_BASIC_AUTH_USER=${N8N_USER:-admin}        # ← con valor por defecto
```

### Ejemplo .env real (homelab)

```env
# Base de datos Nextcloud
NEXTCLOUD_DB_PASSWORD=tu_password_aqui

# Base de datos Immich
IMMICH_DB_PASSWORD=otro_password_aqui

# n8n autenticación
N8N_USER=admin
N8N_PASSWORD=password_n8n

# Icecast streaming
ICECAST_SOURCE_PASSWORD=source_pass
ICECAST_RELAY_PASSWORD=relay_pass
ICECAST_ADMIN_PASSWORD=admin_pass
```

### Ejemplo .env.example (el que subes a GitHub)

```env
# Base de datos Nextcloud
NEXTCLOUD_DB_PASSWORD=CAMBIAR_ESTO

# Base de datos Immich
IMMICH_DB_PASSWORD=CAMBIAR_ESTO

# n8n autenticación
N8N_USER=admin
N8N_PASSWORD=CAMBIAR_ESTO

# Icecast streaming
ICECAST_SOURCE_PASSWORD=CAMBIAR_ESTO
ICECAST_RELAY_PASSWORD=CAMBIAR_ESTO
ICECAST_ADMIN_PASSWORD=CAMBIAR_ESTO
```

### .gitignore — proteger el .env

Crea un `.gitignore` en la carpeta del proyecto:

```
.env
*.env
!.env.example
```

---

## Estructura docker-compose.yml

### Anatomía de un servicio

```yaml
services:
  nombre-servicio:                          # nombre interno del stack
    image: imagen:tag                       # imagen de Docker Hub o registry
    container_name: nombre-contenedor      # nombre visible en docker ps
    restart: unless-stopped                # política de reinicio
    ports:
      - "HOST:CONTENEDOR"                  # puerto_host:puerto_contenedor
    volumes:
      - volumen-nombrado:/ruta/interna     # volumen gestionado por Docker
      - /ruta/host:/ruta/contenedor        # bind mount (ruta real del servidor)
      - /ruta/host:/ruta/contenedor:ro     # bind mount solo lectura
    networks:
      - nombre-red                         # red interna del stack
    environment:
      - VARIABLE=valor
      - VARIABLE=${VARIABLE_DEL_ENV}
    depends_on:
      - otro-servicio                      # espera a que este servicio arranque
```

### Políticas de reinicio

| Valor | Cuándo reinicia |
|---|---|
| `no` | Nunca |
| `always` | Siempre, incluso si se para manualmente |
| `unless-stopped` | Siempre, excepto si se paró manualmente ← **recomendado** |
| `on-failure` | Solo si falla (exit code != 0) |

### Volúmenes: nombrados vs bind mounts

```yaml
volumes:
  # Volumen nombrado — Docker gestiona dónde se guarda
  - jellyfin-config:/config

  # Bind mount — tú decides la ruta exacta en el host
  - /mnt/nas/media/movies:/media/movies:ro

  # Bind mount del socket Docker (para acceso al daemon)
  - /var/run/docker.sock:/var/run/docker.sock:ro
```

```yaml
# Declarar volúmenes nombrados al final del archivo
volumes:
  jellyfin-config:
  nextcloud-db-data:
  n8n-data:
```

### Redes

```yaml
# Definir la red
networks:
  homelab:
    driver: bridge

# Asignar a servicios
services:
  jellyfin:
    networks:
      - homelab
```

> Los servicios en la misma red se comunican por nombre de contenedor:
> `nextcloud` puede conectar a `nextcloud-db` usando el hostname `nextcloud-db`

---

## Comandos de Mantenimiento

### Limpieza

```bash
# Borrar contenedores parados, imágenes sin uso, redes huérfanas
docker system prune

# Incluir volúmenes sin usar (⚠️ cuidado)
docker system prune --volumes

# Ver espacio usado por Docker
docker system df

# Borrar imágenes sin usar (las que no usa ningún contenedor activo)
docker image prune -a
```

### Inspección

```bash
# Ver todos los contenedores (activos + parados)
docker ps -a

# Ver uso de recursos en tiempo real
docker stats

# Inspeccionar un contenedor (red, volúmenes, variables...)
docker inspect jellyfin

# Ver IP de un contenedor dentro de la red Docker
docker inspect jellyfin | grep IPAddress

# Ver logs del sistema Docker
journalctl -u docker -f
```

### Gestión de volúmenes

```bash
# Listar todos los volúmenes
docker volume ls

# Inspeccionar un volumen (ver dónde está en disco)
docker volume inspect zabbix_zabbix-db

# Borrar un volumen específico (⚠️ DESTRUYE DATOS)
docker volume rm nombre_volumen

# Borrar volúmenes huérfanos
docker volume prune
```

---

## Ejemplos Reales del Homelab

### Stack Monitoring — LXC-104 (Zabbix + Uptime Kuma)

```yaml
# /opt/zabbix/docker-compose.yml
networks:
  monitoring:
    driver: bridge

volumes:
  uptime-kuma-data:
  postgres-data:
  zabbix-modules:

services:
  uptime-kuma:
    image: louislam/uptime-kuma:2          # versión major fija
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - uptime-kuma-data:/app/data
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - monitoring
    environment:
      - TZ=Europe/Madrid

  zabbix-db:
    image: postgres:15-alpine              # versión LTS fija
    container_name: zabbix-db
    restart: unless-stopped
    shm_size: 256mb                        # necesario para PostgreSQL bajo carga
    environment:
      - POSTGRES_DB=zabbix
      - POSTGRES_USER=zabbix
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - monitoring

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:7.0-alpine-latest
    container_name: zabbix-server
    restart: unless-stopped
    ports:
      - "10051:10051"
    environment:
      - DB_SERVER_HOST=zabbix-db
      - POSTGRES_DB=zabbix
      - POSTGRES_USER=zabbix
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - ZBX_HOUSEKEEPINGFREQUENCY=1        # limpieza cada hora
      - ZBX_MAXHOUSEKEEPERDELETE=5000      # máx registros borrados por ciclo
      - ZBX_HISTORYSTORAGEDATEINDEX=1      # índices por fecha (rendimiento)
    volumes:
      - zabbix-modules:/var/lib/zabbix/modules
    depends_on:
      - zabbix-db
    networks:
      - monitoring

  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:7.0-alpine-latest
    container_name: zabbix-web
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - DB_SERVER_HOST=zabbix-db
      - POSTGRES_DB=zabbix
      - POSTGRES_USER=zabbix
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - ZBX_SERVER_HOST=zabbix-server
      - PHP_TZ=Europe/Madrid
    depends_on:
      - zabbix-db
      - zabbix-server
    networks:
      - monitoring
```

**Archivo .env del stack monitoring:**

```env
DB_PASSWORD=tu_password_seguro_aqui
```

> **Nota importante:** `zabbix-server` y `zabbix-web` deben usar exactamente la **misma versión** (`7.0-alpine-latest` en ambos). Mezclar versiones causa errores de compatibilidad.

---

## Cómo Elegir Versiones para Producción

### El principio básico

```
latest       ← ❌ nunca en producción (puede romperse sin aviso)
x.y.z        ← ✅ versión exacta, máximo control
x.y-alpine   ← ✅ versión minor fija, parches automáticos
x-alpine     ← ⚠️ versión major fija, aceptable para servicios estables
```

### Tipos de versiones

| Tipo | Ejemplo | Cuándo usar |
|---|---|---|
| **Latest** | `postgres:latest` | Solo en desarrollo local |
| **Major** | `uptime-kuma:2` | Servicios estables con buen historial |
| **Minor** | `postgres:15-alpine` | Bases de datos, servicios críticos |
| **Patch** | `immich-server:v2.3.1` | Cuando necesitas reproducibilidad exacta |
| **LTS** | `zabbix-server-pgsql:7.0-alpine-latest` | Servicios empresariales con soporte largo |

### Dónde investigar versiones

**1. Docker Hub — página oficial de la imagen:**
```
https://hub.docker.com/_/postgres
https://hub.docker.com/r/zabbix/zabbix-server-pgsql/tags
```
Busca tags marcados como `stable`, `lts`, o con fechas de release.

**2. GitHub Releases del proyecto:**
```
https://github.com/louislam/uptime-kuma/releases
https://github.com/immich-app/immich/releases
```
Fíjate en:
- ¿Cuántos bugs se reportaron después del release?
- ¿Hay un hotfix (x.y.1, x.y.2) lanzado poco después? → espera al hotfix
- ¿Lleva 2-4 semanas sin issues críticos? → ya es seguro

**3. Changelogs — qué cambió entre versiones:**
Busca `CHANGELOG.md` en el repo de GitHub. Señales de alerta:
```
⚠️  "breaking change"     ← puede requerir migración de datos
⚠️  "database migration"  ← haz backup antes de actualizar
✅  "bug fix"             ← generalmente seguro
✅  "security patch"      ← actualiza pronto
```

**4. Reddit y foros de la comunidad:**
```
r/selfhosted    ← experiencias reales de usuarios
r/zabbix
r/Proxmox
```
Busca `"immich v2.4 issues"` o `"zabbix 7.2 upgrade problems"` antes de actualizar.

### Estrategia práctica para este homelab

```
Bases de datos (PostgreSQL, Redis)
└── Fijar versión minor: postgres:15-alpine
    Actualizar solo cuando Zabbix/Nextcloud/Immich lo requiera explícitamente

Zabbix Server + Web
└── Usar rama LTS: 7.0-alpine-latest
    Ambos contenedores SIEMPRE la misma versión

Immich
└── Fijar versión exact: v2.3.1
    Actualizar siguiendo sus release notes (tienen migraciones de DB frecuentes)

Jellyfin, Uptime Kuma, n8n
└── Major fija: louislam/uptime-kuma:2
    Watchtower puede gestionar actualizaciones automáticas

Watchtower (el que actualiza los demás)
└── latest es aceptable aquí
```

### Flujo de actualización seguro

```bash
# 1. Leer el changelog antes de actualizar
#    https://github.com/proyecto/releases

# 2. Backup de la base de datos
docker compose exec zabbix-db pg_dump -U zabbix zabbix > backup-$(date +%Y%m%d).sql

# 3. Snapshot del LXC en Proxmox (desde el host)
pct snapshot 104 pre-update-$(date +%Y%m%d)

# 4. Actualizar la imagen en docker-compose.yml
nano docker-compose.yml

# 5. Pull + recreate
docker compose pull nombre-servicio
docker compose up -d nombre-servicio

# 6. Verificar que funciona
docker compose logs -f nombre-servicio
# Esperar 2-3 minutos y comprobar la web

# 7. Si algo falla → rollback
docker compose down nombre-servicio
# Editar docker-compose.yml con la versión anterior
docker compose up -d nombre-servicio
# O restaurar snapshot de Proxmox si el problema es grave
```

---

## Comandos Específicos por Servicio

### PostgreSQL (Nextcloud, Zabbix, Immich)

```bash
# Entrar al cliente psql
docker compose exec nextcloud-db psql -U nextcloud -d nextcloud

# Backup de base de datos
docker compose exec nextcloud-db pg_dump -U nextcloud nextcloud > backup.sql

# Restaurar backup
docker compose exec -T nextcloud-db psql -U nextcloud -d nextcloud < backup.sql

# Ver tamaño de las bases de datos
docker compose exec nextcloud-db psql -U nextcloud -c "\l+"
```

### Nextcloud

```bash
# Comandos de administración (occ)
docker compose exec nextcloud php occ status
docker compose exec nextcloud php occ maintenance:mode --on
docker compose exec nextcloud php occ maintenance:mode --off
docker compose exec nextcloud php occ files:scan --all
docker compose exec nextcloud php occ upgrade
```

### Zabbix

```bash
# Ver logs del servidor Zabbix
docker compose logs -f zabbix-server

# Verificar conectividad con la base de datos
docker compose exec zabbix-server bash
# Dentro: zbx_db_check

# Reiniciar solo el servidor (sin tocar la DB)
docker compose restart zabbix-server
```

---

## Tips y Buenas Prácticas

### Fijar versiones de imágenes

```yaml
# ❌ Malo — puede romperse en cualquier actualización
image: immich-server:latest

# ✅ Bueno — versión fija, actualizas cuando tú decides
image: ghcr.io/immich-app/immich-server:v2.3.1
```

> Excepción: servicios estables como Jellyfin o Dozzle pueden usar `latest`
> si Watchtower gestiona las actualizaciones automáticas.

### Healthchecks

```yaml
# Añadir healthcheck para que depends_on espere de verdad
nextcloud-db:
  image: postgres:15-alpine
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U nextcloud"]
    interval: 10s
    timeout: 5s
    retries: 5

nextcloud:
  depends_on:
    nextcloud-db:
      condition: service_healthy    # espera a que esté healthy
```

### Variables de entorno con valor por defecto

```yaml
environment:
  # Si N8N_USER no está en .env, usa "admin"
  - N8N_BASIC_AUTH_USER=${N8N_USER:-admin}

  # Si no está definida, usa cadena vacía
  - VARIABLE=${VARIABLE:-}
```

### Múltiples archivos compose (override)

```bash
# Útil para tener configuración base + overrides por entorno
docker compose -f docker-compose.yml -f docker-compose.override.yml up -d
```

---

## Flujo de Trabajo Habitual

### Añadir un nuevo servicio

```bash
# 1. Editar docker-compose.yml y añadir el servicio
nano docker-compose.yml

# 2. Añadir variables necesarias al .env
nano .env

# 3. Arrancar solo el nuevo servicio (sin tocar los demás)
docker compose up -d nombre-servicio

# 4. Verificar que arrancó bien
docker compose ps
docker compose logs -f nombre-servicio
```

### Actualizar un servicio

```bash
# 1. Descargar nueva imagen
docker compose pull nombre-servicio

# 2. Recrear el contenedor con la nueva imagen
docker compose up -d nombre-servicio

# 3. Verificar
docker compose logs -f nombre-servicio

# 4. Limpiar imagen antigua
docker image prune -f
```

### Backup antes de actualizar (recomendado)

```bash
# Snapshot del volumen (si está en Proxmox LXC)
# Desde el host Proxmox:
pct snapshot 100 pre-update-$(date +%Y%m%d)

# O backup manual de los volúmenes importantes:
docker run --rm \
  -v nextcloud-db-data:/data \
  -v /mnt/nas/backups:/backup \
  alpine tar czf /backup/nextcloud-db-$(date +%Y%m%d).tar.gz /data
```

---

## Troubleshooting Rápido

### El contenedor no arranca

```bash
# Ver por qué falló
docker compose logs nombre-servicio

# Ver eventos del contenedor
docker events --filter container=nombre-contenedor
```

### Puerto ya en uso

```bash
# Ver qué proceso usa el puerto
ss -tlnp | grep :8080
# o
lsof -i :8080
```

### Contenedor en bucle de reinicios (restart loop)

```bash
# Ver cuántas veces ha reiniciado
docker ps   # columna RESTARTS

# Ver los últimos logs antes de morir
docker logs --tail=50 nombre-contenedor
```

### No puede conectar a la base de datos

```bash
# Verificar que la DB está en la misma red
docker network inspect nombre-red

# Probar conectividad desde el contenedor de la app
docker compose exec nextcloud ping nextcloud-db
```

### Watchtower actualizó y rompió algo

```bash
# Volver a la versión anterior (si tienes la imagen en caché)
docker images | grep nombre-servicio

# Forzar versión específica en docker-compose.yml y recrear
docker compose up -d --force-recreate nombre-servicio
```
