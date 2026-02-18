# 🖥️ LXC-104 — Configuración Dual-Homed + Zabbix Stack

## Container de monitorización con doble interfaz de red y stack Zabbix en Docker

---

## Contexto

LXC-104 es el **corazón de la monitorización del MSP**. Usa una arquitectura dual-homed (dos interfaces de red) para separar:

- **eth0 (management):** Acceso SSH, GUI Zabbix, updates, salida a internet
- **eth1 (monitored):** Recepción de datos de Zabbix Agents vía VPN

Esta separación es una **best practice en MSPs** — el tráfico de monitorización va por una red dedicada, aislada del tráfico de gestión.

---

## Arquitectura

```
LXC-104: monitoring (192.168.0.114)
│
├─► eth0: 192.168.0.114/24 (vmbr0) — Management
│   ├─► Gateway: 192.168.0.1
│   ├─► SSH desde tu PC
│   ├─► GUI Zabbix accesible
│   ├─► Docker pull de imágenes
│   └─► Salida a internet para updates y Telegram
│
├─► eth1: 10.0.1.10/24 (vmbr1) — Monitored
│   ├─► Gateway: 10.0.1.254 (FortiGate Server)
│   ├─► Zabbix Agent data entra por aquí (desde VPN)
│   ├─► Sin default route (solo conoce 10.0.1.0/24)
│   └─► Tráfico monitorización segregado
│
└─► Docker Stack
    ├─► Uptime Kuma (:3001)
    └─► Zabbix (Server :10051 + Web :8080 + PostgreSQL)
```

---

## Especificaciones LXC

| Propiedad | Valor |
|-----------|-------|
| **CT ID** | 104 |
| **Hostname** | monitoring |
| **OS** | Ubuntu 22.04 LTS |
| **vCPU** | 1 |
| **RAM** | 1 GB (1024 MB) |
| **Disco** | 25 GB (local-lvm) |
| **Tipo** | Unprivileged |
| **Features** | `nesting=1` (para Docker) |
| **Start at boot** | ✅ `onboot=1` |

> **Nota:** Los recursos (1 vCPU, 1 GB RAM) son suficientes para Zabbix + Uptime Kuma en un entorno de laboratorio monitorizando pocos hosts. En producción con 50+ hosts, escalar a 2 vCPU y 2-4 GB RAM.

---

## Paso 1 — Añadir Segunda Interfaz (eth1)

> **Requisito:** vmbr1 ya debe existir en Proxmox. Ver [01-FortiGate-Server-Setup.md](01-FortiGate-Server-Setup.md) Paso 1.

### Apagar LXC-104

**Proxmox UI → LXC-104 → Shutdown**

Esperar a que el estado sea "stopped".

### Añadir eth1

**LXC-104 → Network → Add:**

| Campo | Valor |
|-------|-------|
| **Name** | `eth1` |
| **Bridge** | `vmbr1` |
| **IPv4/CIDR** | `10.0.1.10/24` |
| **Gateway (IPv4)** | `10.0.1.254` |
| **Firewall** | ❌ Desmarcar |

**Add**

### Iniciar LXC-104

**LXC-104 → Start**

---

## Paso 2 — Verificar Configuración de Red

### Acceder al container

```bash
pct enter 104
```

### Ver interfaces

```bash
ip addr show
```

**Salida esperada:**

```
1: lo: <LOOPBACK,UP,LOWER_UP>
    inet 127.0.0.1/8 scope host lo

2: eth0@if87: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.0.114/24 brd 192.168.0.255 scope global eth0

3: eth1@if88: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 10.0.1.10/24 brd 10.0.1.255 scope global eth1
```

### Ver rutas

```bash
ip route show
```

**Salida esperada:**

```
default via 192.168.0.1 dev eth0
10.0.1.0/24 dev eth1 proto kernel scope link src 10.0.1.10
192.168.0.0/24 dev eth0 proto kernel scope link src 192.168.0.114
```

> ✅ **Correcto:** Default route solo apunta a `192.168.0.1` por eth0. No hay default route en eth1.

### Verificar archivo de configuración Proxmox

```bash
# En el host Proxmox (no dentro del LXC)
cat /etc/pve/lxc/104.conf | grep net
```

**Debe mostrar:**

```
net0: name=eth0,bridge=vmbr0,gw=192.168.0.1,hwaddr=...,ip=192.168.0.114/24,...
net1: name=eth1,bridge=vmbr1,gw=10.0.1.254,hwaddr=...,ip=10.0.1.10/24,...
```

> ⚠️ **Lección aprendida:** Un typo en este archivo (`10.1.0.254` en lugar de `10.0.1.254`) causó problemas de routing graves. Siempre verificar visualmente las IPs.

---

## Paso 3 — Test de Conectividad

```bash
# Management (eth0)
ping -c 2 192.168.0.1       # Router
ping -c 2 192.168.0.100     # Proxmox
ping -c 2 8.8.8.8           # Internet

# Red FortiGate (eth1)
ping -c 2 10.0.1.254        # FortiGate LAN
ping -c 2 10.0.1.1          # Proxmox bridge

# Internet check
ping -c 2 google.com
```

**Todos deben responder.** Si alguno falla, revisar la sección Troubleshooting al final.

---

## Paso 4 — Stack Docker (Uptime Kuma + Zabbix)

### Verificar Docker instalado

```bash
docker --version
docker compose version
```

Si no está instalado:

```bash
curl -fsSL https://get.docker.com | sh
```

### Crear directorio de trabajo

```bash
mkdir -p /root/docker
cd /root/docker
```

### Generar password segura para PostgreSQL

```bash
openssl rand -base64 24
```

**Salida ejemplo:**

```
xR7mK9pL2nQ5vT8wY4jH6fB3cN1aZ
```

**Copiar esta password** — la usaremos en el archivo `.env`.

### Crear archivo .env

```bash
nano .env
```

**Contenido:**

```bash
DB_PASSWORD=xR7mK9pL2nQ5vT8wY4jH6fB3cN1aZ
```

*Reemplazar con la password generada*

**Guardar:** `Ctrl+O`, `Enter`, `Ctrl+X`

**Proteger:**

```bash
chmod 600 .env
```

### Crear docker-compose.yml

```bash
nano docker-compose.yml
```

**Contenido completo:**

```yaml
# MONITORING STACK - LXC-104
# Uptime Kuma + Zabbix Server 7.0 LTS
version: '3.8'

networks:
  monitoring:
    driver: bridge

volumes:
  uptime-kuma-data:
  postgres-data:
  zabbix-modules:

services:
  # Uptime Kuma (monitorización simple)
  uptime-kuma:
    image: louislam/uptime-kuma:2
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

  # PostgreSQL para Zabbix
  zabbix-db:
    image: postgres:15-alpine
    container_name: zabbix-db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=zabbix
      - POSTGRES_USER=zabbix
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - monitoring
    shm_size: 256mb

  # Zabbix Server
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
      - ZBX_HOUSEKEEPINGFREQUENCY=1
      - ZBX_MAXHOUSEKEEPERDELETE=5000
      - ZBX_HISTORYSTORAGEDATEINDEX=1
    volumes:
      - zabbix-modules:/var/lib/zabbix/modules
    depends_on:
      - zabbix-db
    networks:
      - monitoring

  # Zabbix Web Interface
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

**Guardar:** `Ctrl+O`, `Enter`, `Ctrl+X`

### Desplegar stack

```bash
docker compose up -d
```

**Salida esperada:**

```
[+] Running 5/5
 ✔ Network docker_monitoring        Created
 ✔ Volume "docker_uptime-kuma-data" Created
 ✔ Volume "docker_postgres-data"    Created
 ✔ Volume "docker_zabbix-modules"   Created
 ✔ Container uptime-kuma            Started
 ✔ Container zabbix-db              Started
 ✔ Container zabbix-server          Started
 ✔ Container zabbix-web             Started
```

### Verificar

```bash
docker compose ps
```

**Esperado — 4 containers UP:**

```
NAME            STATE    PORTS
uptime-kuma     Up       0.0.0.0:3001->3001/tcp
zabbix-db       Up       5432/tcp
zabbix-server   Up       0.0.0.0:10051->10051/tcp
zabbix-web      Up       0.0.0.0:8080->8080/tcp
```

### Ver logs

```bash
docker compose logs -f zabbix-server
```

Esperar a ver:

```
server #0 started [main process]
```

Esto indica que Zabbix Server está completamente iniciado. **Presionar Ctrl+C** para salir de los logs.

---

## Paso 5 — Configuración Inicial de Zabbix

### Acceso Web

```
http://192.168.0.114:8080
```

### Login por defecto

```
Username: Admin
Password: zabbix
```

> ⚠️ **Importante:** La 'A' de Admin es mayúscula.

### Cambiar password

**User icon (arriba derecha) → User settings → Change password**

Establecer una contraseña segura.

### Deshabilitar host "Zabbix server" por defecto

**Configuration → Hosts → Zabbix server → Disabled → Update**

> Este host auto-generado intenta monitorizar localhost pero no tiene Agent configurado. Deshabilitarlo evita alertas innecesarias.

---

## Paso 6 — Optimización Housekeeping

**Administration → General → Housekeeping:**

| Sección | Configuración |
|---------|---------------|
| **Override item history period** | ✅ Enable |
| **History** | `14d` (14 días) |
| **Override item trend period** | ✅ Enable |
| **Trends** | `180d` (6 meses) |
| **Events and alerts** | `30d` |
| **Internal data** | `14d` |

**Update**

> Esto optimiza el uso de disco — datos históricos se mantienen 14 días, tendencias 6 meses.

---

## Paso 7 — Integración con Cloudflare Tunnel (Opcional)

Si tienes Cloudflare Tunnel configurado en LXC-101:

```bash
# En LXC-101
pct enter 101
nano /etc/cloudflared/config.yml
```

**Añadir entrada:**

```yaml
ingress:
  - hostname: zabbix.tudominio.com
    service: http://192.168.0.114:8080
```

```bash
systemctl restart cloudflared
```

**En Cloudflare Dashboard → DNS:**

- Type: `CNAME`
- Name: `zabbix`
- Target: `<TUNNEL_ID>.cfargotunnel.com`
- Proxy: ✅

**Acceso externo:** `https://zabbix.tudominio.com`

---

## Comandos Útiles de Mantenimiento

### Ver logs en tiempo real

```bash
cd /root/docker
docker compose logs -f zabbix-server
```

### Reiniciar solo Zabbix (mantener Uptime Kuma)

```bash
docker compose restart zabbix-db zabbix-server zabbix-web
```

### Ver recursos

```bash
docker stats
```

### Tamaño de base de datos

```bash
docker exec zabbix-db psql -U zabbix -d zabbix \
  -c "SELECT pg_size_pretty(pg_database_size('zabbix'));"
```

### Backup PostgreSQL

```bash
docker exec zabbix-db pg_dump -U zabbix zabbix | gzip > \
  /root/backups/zabbix-db-$(date +%Y%m%d).sql.gz
```

---

## Reset de Emergencia

### Reset contraseña Admin (sin perder datos)

```bash
docker exec -it zabbix-db psql -U zabbix -d zabbix
```

```sql
UPDATE users SET passwd='$2y$10$L1l1VJ3E5HwJEm8UYVl4DOz.9d5R5E5QkXFPW9K3z1q1Q9j4VqB1C'
WHERE username='Admin';
\q
```

Contraseña reseteada a: `zabbix`

### Reset completo Zabbix (mantiene Uptime Kuma)

```bash
cd /root/docker
docker stop zabbix-web zabbix-server zabbix-db
docker rm zabbix-web zabbix-server zabbix-db
docker volume rm docker_postgres-data docker_zabbix-modules

# Verificar Uptime Kuma sigue corriendo
docker ps | grep uptime-kuma

# Recrear Zabbix
docker compose up -d zabbix-db zabbix-server zabbix-web
```

---

## Troubleshooting

### eth1 no aparece o está DOWN

**Verificar en Proxmox:**

```bash
# En host Proxmox
cat /etc/pve/lxc/104.conf | grep net1
```

Si no aparece, añadir manualmente:

```bash
pct set 104 -net1 name=eth1,bridge=vmbr1,ip=10.0.1.10/24,gw=10.0.1.254
```

Reiniciar LXC:

```bash
pct stop 104 && pct start 104
```

### Conflicto de default routes

**Síntoma:** Ping a internet falla, o tráfico sale por interfaz incorrecta.

**Diagnóstico:**

```bash
ip route show | grep default
```

Si aparecen **dos default routes**, hay conflicto.

**Solución:** Verificar que el gateway de eth1 en `/etc/pve/lxc/104.conf` es correcto (`10.0.1.254` no `10.1.0.254`).

### Zabbix Web no carga

**Causa:** Zabbix Server aún no terminó de inicializar la base de datos.

**Solución:**

```bash
docker compose logs -f zabbix-server
```

Esperar 2-3 minutos hasta ver `server #0 started [main process]`.

### Uptime Kuma perdió alertas por Telegram

**Causa:** Uptime Kuma intentó usar eth1 (que no tiene salida directa a internet) para conectar a Telegram.

**Solución:** Las notificaciones Telegram van por internet, deben salir por eth0. Si Uptime Kuma no puede alcanzar Telegram, verificar routing:

```bash
ip route get 149.154.167.99  # IP de Telegram
```

Debe mostrar que sale por eth0 vía `192.168.0.1`.

Si el problema persiste, reiniciar Uptime Kuma:

```bash
docker compose restart uptime-kuma
```

---

## Recursos del Sistema

```bash
# Uso CPU/RAM en tiempo real
htop

# Logs del sistema
journalctl -u docker -f

# Espacio en disco
df -h

# Ver puertos escuchando
ss -tunlp | grep -E '3001|8080|10051'
```

---

## Resumen de Configuración

```
LXC-104: monitoring
├─► Recursos: 1 vCPU, 1 GB RAM, 25 GB disk
├─► eth0: 192.168.0.114/24 (management)
│   └─► Gateway: 192.168.0.1 (salida internet)
├─► eth1: 10.0.1.10/24 (monitored)
│   └─► Gateway: 10.0.1.254 (FortiGate Server)
├─► Docker Compose:
│   ├─► Uptime Kuma :3001
│   ├─► Zabbix Server :10051
│   ├─► Zabbix Web :8080
│   └─► PostgreSQL 15 (interno)
└─► Acceso:
    ├─► SSH: ssh root@192.168.0.114
    ├─► Uptime Kuma: http://192.168.0.114:3001
    └─► Zabbix: http://192.168.0.114:8080
```

---

**Documentos relacionados:**
- [← FortiGate Server Setup](01-FortiGate-Server-Setup.md)
- [→ Túnel VPN IPsec](04-VPN-IPsec-Tunnel.md)
- [→ Ubuntu Cliente Setup](05-Ubuntu-Cliente-Setup.md)
- [→ Troubleshooting](06-Troubleshooting.md)
- [← README Principal](README.md)

---

*Última actualización: Febrero 2025*
