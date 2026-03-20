# 🖥️ Chuleta Proxmox VE

## Referencia rápida con ejemplos reales del homelab ODIN (N100 / 16GB RAM)

---

## Acceso y Conexión

```bash
# SSH al host Proxmox
ssh root@192.168.0.100

# Interfaz web
https://192.168.0.100:8006

# Ver versión de Proxmox
pveversion

# Ver estado general del cluster/nodo
pvesh get /nodes/odin/status
```

---

## Gestión de LXC Containers

### Operaciones básicas

```bash
# Listar todos los containers con estado
pct list

# Arrancar / parar / reiniciar
pct start 104
pct stop 104
pct reboot 104
pct shutdown 104        # parada limpia (espera a que los servicios cierren)

# Entrar en el container (consola)
pct enter 104

# Ejecutar un comando sin entrar
pct exec 104 -- ip route show
pct exec 104 -- bash -c "apt update && apt upgrade -y"

# Ver configuración del container
pct config 104
cat /etc/pve/lxc/104.conf
```

### Crear un LXC desde cero

```bash
# Sintaxis completa
pct create <VMID> <storage>:vztmpl/<plantilla> \
  --hostname <nombre> \
  --memory <MB> \
  --cores <núcleos> \
  --net0 name=eth0,bridge=vmbr0,ip=<IP>/24,gw=<GW> \
  --storage local-lvm \
  --rootfs local-lvm:<GB> \
  --unprivileged 1 \
  --features nesting=1

# Ejemplo real — LXC de monitorización dual-homed
pct create 104 local-lvm:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
  --hostname monitoring \
  --memory 1024 \
  --cores 1 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.0.114/24,gw=192.168.0.1 \
  --net1 name=eth1,bridge=vmbr1,ip=10.0.1.10/24,gw=10.0.1.254 \
  --storage local-lvm \
  --rootfs local-lvm:25 \
  --unprivileged 1 \
  --features nesting=1 \
  --onboot 1 \
  --startup order=5,up=10

# Arrancar tras crear
pct start 104
```

### Modificar recursos de un LXC existente

```bash
# Cambiar RAM
pct set 104 --memory 2048

# Cambiar cores
pct set 104 --cores 2

# Cambiar hostname
pct set 104 --hostname nuevo-nombre

# Añadir segunda interfaz de red (con container apagado)
pct set 104 --net1 name=eth1,bridge=vmbr1,ip=10.0.1.10/24,gw=10.0.1.254

# Añadir mountpoint (bind mount desde host)
pct set 104 --mp0 /mnt/pve/nas-data,mp=/mnt/nas/data

# Activar nesting (necesario para Docker dentro de LXC)
pct set 100 --features nesting=1

# Ver cambios aplicados
pct config 104
```

### Redimensionar disco de un LXC

```bash
# Ampliar disco (solo se puede aumentar, no reducir)
pct resize 104 rootfs +10G

# Verificar dentro del container
pct exec 104 -- df -h /
```

---

## Gestión de VMs

### Operaciones básicas

```bash
# Listar VMs
qm list

# Arrancar / parar / reiniciar
qm start 107
qm stop 107
qm reboot 107
qm shutdown 107         # parada limpia (ACPI)
qm reset 107            # equivalente a pulsar reset físico

# Ver configuración de la VM
qm config 107
cat /etc/pve/qemu-server/107.conf

# Acceder a la consola (monitor QEMU)
qm monitor 107
```

### Importar disco a una VM (FortiGate, pfSense, etc.)

```bash
# Importar imagen qcow2 a storage local-lvm
qm importdisk 107 /ruta/fortios.qcow2 local-lvm

# Ejemplo real — importar FortiGate desde NAS
qm importdisk 107 /mnt/pve/nas-tools/template/iso/FortiGate_VM64_KVM-v7.6.6/fortios.qcow2 local-lvm

# Después de importar, adjuntar el disco en la GUI:
# VM-107 → Hardware → Unused Disk 0 → Edit → Add (como scsi0)
# VM-107 → Options → Boot Order → marcar scsi0
```

### Modificar recursos de una VM

```bash
# Cambiar RAM
qm set 107 --memory 2048

# Cambiar cores (⚠️ FortiGate eval: máximo 1 core)
qm set 107 --cores 1

# Añadir interfaz de red
qm set 107 --net1 virtio,bridge=vmbr1

# Ver configuración resultante
qm config 107
```

---

## Snapshots y Backups

### Snapshots (rápidos, en caliente)

```bash
# Crear snapshot de un LXC
pct snapshot 104 nombre-snapshot
pct snapshot 104 pre-update-$(date +%Y%m%d)

# Crear snapshot de una VM
qm snapshot 107 baseline-config-ok --description "FortiGate funcional"

# Listar snapshots
pct listsnapshot 104
qm listsnapshot 107

# Restaurar snapshot
pct rollback 104 nombre-snapshot
qm rollback 107 baseline-config-ok

# Borrar snapshot
pct delsnapshot 104 nombre-snapshot
qm delsnapshot 107 nombre-snapshot
```

> ⚠️ Los snapshots NO son backups. Viven en el mismo disco — si el disco falla, pierdes todo. Úsalos para cambios rápidos que quieres poder deshacer.

### Backups (vzdump)

```bash
# Backup de un LXC al NAS
vzdump 104 --storage nas-backups --mode snapshot --compress zstd

# Backup de una VM
vzdump 107 --storage nas-backups --mode snapshot --compress zstd

# Backup de todos los containers y VMs
vzdump --all --storage nas-backups --mode snapshot --compress zstd

# Backup con exclusiones
vzdump --all --storage nas-backups --exclude 103

# Ver backups disponibles
ls /mnt/pve/nas-backups/dump/

# Restaurar desde backup
pct restore 104 /mnt/pve/nas-backups/dump/vzdump-lxc-104-*.tar.zst --storage local-lvm
qm restore 107 /mnt/pve/nas-backups/dump/vzdump-qemu-107-*.vma.zst
```

---

## Red y Bridges

### Ver configuración de red

```bash
# Ver todas las interfaces del host
ip addr show
ip link show

# Ver bridges existentes
brctl show

# Ver tabla de rutas
ip route show

# Aplicar cambios de red sin reiniciar
ifreload -a
```

### Editar configuración de red

```bash
nano /etc/network/interfaces
```

**Estructura típica del homelab ODIN:**

```bash
# Interfaz física
auto enp1s0
iface enp1s0 inet manual

# Bridge principal (LAN física)
auto vmbr0
iface vmbr0 inet static
    address 192.168.0.100/24
    gateway 192.168.0.1
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0

# Bridge interno (aislado, sin interfaz física)
auto vmbr1
iface vmbr1 inet static
    address 10.0.1.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    comment Red interna FortiGate/MSP
```

> `bridge-ports none` = bridge virtual puro, sin acceso a red física. Ideal para redes internas entre VMs/LXC.

---

## Almacenamiento

### Ver estado del almacenamiento

```bash
# Ver storages configurados
pvesm status

# Ver espacio disponible
df -h

# Ver uso por storage
pvesm list local-lvm
pvesm list nas-backups

# Ver configuración de storages
cat /etc/pve/storage.cfg
```

### Gestión de discos LVM

```bash
# Ver volúmenes LVM
lvdisplay
vgdisplay
pvdisplay

# Ver volúmenes usados por Proxmox
lvs | grep vm

# Ver espacio disponible en el grupo de volúmenes
vgs
```

### Montajes NFS

```bash
# Ver montajes NFS activos
mount | grep nfs
df -h | grep nas

# Remontar NFS si se desconectó
mount -a

# Verificar conectividad con el NAS
ping 192.168.0.101
showmount -e 192.168.0.101    # muestra exports del NAS
```

**Configuración NFS en `/etc/pve/storage.cfg` (homelab ODIN):**

```
nfs: nas-data
    export /volume1/data
    path /mnt/pve/nas-data
    server 192.168.0.101
    options vers=3,rw,hard,intr
    content images,rootdir,vztmpl

nfs: nas-backups
    export /volume1/backups
    path /mnt/pve/nas-backups
    server 192.168.0.101
    options vers=3,rw,hard,intr
    content backup

nfs: nas-tools
    export /volume1/tools
    path /mnt/pve/nas-tools
    server 192.168.0.101
    options vers=3,rw,hard,intr
    content snippets
```

---

## Hookscripts

Los hookscripts son scripts bash que Proxmox ejecuta automáticamente en momentos clave del ciclo de vida de un container (pre-start, post-start, pre-stop, post-stop).

### Crear y registrar un hookscript

```bash
# 1. Crear el script (en el storage local o NAS)
nano /mnt/pve/nas-data/snippets/104-routes.sh

# 2. Dar permisos de ejecución
chmod +x /mnt/pve/nas-data/snippets/104-routes.sh

# 3. Registrarlo en el container
pct set 104 --hookscript local:snippets/104-routes.sh

# 4. Verificar
grep hookscript /etc/pve/lxc/104.conf
```

### Estructura de un hookscript

```bash
#!/bin/bash
# $1 = VMID del container
# $2 = fase (pre-start, post-start, pre-stop, post-stop)

if [ "$2" == "post-start" ]; then
    sleep 3    # esperar a que las interfaces estén activas

    # Ejemplo real — fijar rutas en LXC-104 dual-homed
    lxc-attach -n 104 -- ip route del default via 10.0.1.254 dev eth1 2>/dev/null
    lxc-attach -n 104 -- ip route add default via 10.0.1.254 dev eth1 metric 200 2>/dev/null
    lxc-attach -n 104 -- ip route add 10.0.2.0/24 via 10.0.1.254 dev eth1 2>/dev/null

    echo "LXC-104: rutas configuradas correctamente"
fi

if [ "$2" == "pre-stop" ]; then
    echo "LXC-104: preparando parada..."
fi
```

### Probar el hookscript manualmente

```bash
bash /mnt/pve/nas-data/snippets/104-routes.sh 104 post-start
```

---

## UID/GID Mapping (LXC + NFS)

Necesario para que containers no privilegiados puedan escribir en el NAS.

```bash
# Ver mapping actual de un container
grep idmap /etc/pve/lxc/104.conf

# Añadir mapping personalizado al final del .conf
# (Mapear UID 1000 del container al UID 1000 del host/NAS)
nano /etc/pve/lxc/104.conf
```

**Mapping completo (ejemplo LXC-104):**

```
lxc.idmap: u 0 100000 1000
lxc.idmap: g 0 100000 1000
lxc.idmap: u 1000 1000 1
lxc.idmap: g 1000 10 1
lxc.idmap: u 1001 101001 64535
lxc.idmap: g 1001 101001 64535
```

```bash
# Permitir al host usar esos UIDs en containers
echo "root:1000:1" >> /etc/subuid
echo "root:10:1" >> /etc/subgid

# Aplicar reiniciando el container
pct stop 104 && pct start 104
```

---

## Actualización del Sistema

### Proxmox host

```bash
# Actualizar lista de paquetes
apt update

# Ver qué se va a actualizar
apt list --upgradable

# Actualizar todo
apt full-upgrade -y

# Actualizar solo Proxmox (sin otros paquetes)
apt install proxmox-ve

# Reiniciar si se actualizó el kernel
reboot

# Ver versión del kernel activo
uname -r
```

> ⚠️ Hacer snapshot de los LXC críticos antes de actualizar Proxmox.

### Actualizar todos los LXC de una vez

```bash
for ct in $(pct list | awk 'NR>1 && $2=="running" {print $1}'); do
    echo "=== Actualizando LXC-$ct ==="
    
    # Detectar gestor de paquetes
    if pct exec $ct -- test -f /etc/alpine-release 2>/dev/null; then
        pct exec $ct -- apk update && pct exec $ct -- apk upgrade
    elif pct exec $ct -- test -f /etc/debian_version 2>/dev/null; then
        pct exec $ct -- bash -c "apt update && apt upgrade -y"
    elif pct exec $ct -- test -f /etc/redhat-release 2>/dev/null; then
        pct exec $ct -- bash -c "dnf upgrade -y"
    else
        echo "LXC-$ct: SO no reconocido, saltando"
    fi
done
```

### Plantillas LXC

```bash
# Ver plantillas disponibles
pveam list local
pveam list nas-data

# Descargar plantilla
pveam update
pveam available --section system | grep ubuntu
pveam download local ubuntu-24.04-standard_24.04-2_amd64.tar.zst

# Ver plantillas descargadas
ls /var/lib/vz/template/cache/
```

---

## Monitorización y Diagnóstico

### Recursos del sistema

```bash
# Ver uso de CPU y memoria en tiempo real
top
htop    # más visual (instalar con apt install htop)

# Ver uso de CPU por container
pct list   # luego entrar en cada uno

# Ver temperatura (si está disponible)
sensors

# Ver carga del sistema
uptime

# Ver memoria disponible
free -h

# Ver disco
df -h
du -sh /var/lib/vz/*    # espacio usado por imágenes locales
```

### Logs del sistema

```bash
# Logs del kernel (errores hardware, red)
dmesg | tail -50
dmesg | grep -i error

# Logs de Proxmox
journalctl -u pvedaemon -f          # daemon principal
journalctl -u pvestatd -f           # estadísticas
journalctl -u pve-cluster -f        # cluster

# Logs de un container específico
journalctl -u lxc@104 -f

# Logs del sistema en general
tail -f /var/log/syslog
```

### Diagnóstico de red

```bash
# Ver interfaces y sus IPs
ip addr show

# Ver tabla de rutas
ip route show

# Ver bridges y qué interfaces tienen
brctl show

# Ver conexiones activas
ss -tlnp                  # puertos en escucha
ss -s                     # resumen de sockets

# Test de conectividad
ping 192.168.0.1          # gateway
ping 8.8.8.8              # internet
ping 192.168.0.101        # NAS

# Traceroute
traceroute 8.8.8.8
```

---

## GPU Passthrough a LXC (Intel QuickSync)

Para pasar la GPU a un container que ejecuta Jellyfin o Immich:

```bash
# 1. Verificar GPU en el host
ls -la /dev/dri/
getent group render     # obtener GID del grupo render (ej: 993)

# 2. Añadir al .conf del LXC
nano /etc/pve/lxc/100.conf
```

**Líneas a añadir en el .conf:**

```
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```

```bash
# 3. Reiniciar el container y verificar
pct reboot 100
pct exec 100 -- ls -la /dev/dri/

# 4. Test dentro del container
pct exec 100 -- vainfo
```

---

## Ficheros de Configuración Clave

| Fichero | Qué contiene |
|---|---|
| `/etc/network/interfaces` | Bridges y configuración de red del host |
| `/etc/pve/storage.cfg` | Storages configurados (NFS, LVM, local) |
| `/etc/pve/lxc/<ID>.conf` | Configuración de cada LXC |
| `/etc/pve/qemu-server/<ID>.conf` | Configuración de cada VM |
| `/etc/pve/nodes/odin/lrm.conf` | Configuración del nodo |
| `/var/lib/vz/template/cache/` | Plantillas LXC descargadas |
| `/mnt/pve/nas-data/snippets/` | Hookscripts almacenados en NAS |
| `/etc/subuid` y `/etc/subgid` | Mappings UID/GID para containers |

---

## Comandos de Emergencia

```bash
# Container no arranca — ver por qué
pct start 104 && journalctl -u lxc@104 -f

# VM no arranca — ver log de QEMU
qm start 107
tail -f /var/log/pve/qemu-server/107.log

# Recuperar container con rootfs dañado
# (montar el disco del container desde el host)
lvdisplay | grep 104
mount /dev/pve/vm-104-disk-0 /mnt/recovery

# Forzar parada si no responde
pct stop 104 --skiplock
qm stop 107 --skiplock

# NFS bloqueado / container colgado esperando NFS
# (desmontar NFS forzado)
umount -f -l /mnt/pve/nas-data

# Ver qué procesos usan un mountpoint
lsof +D /mnt/pve/nas-data
fuser -m /mnt/pve/nas-data
```

---

## Referencia Rápida de IDs — Homelab ODIN

| ID | Tipo | Hostname | IP | Rol |
|---|---|---|---|---|
| 100 | LXC | docker-host | 192.168.0.110 | Docker stack principal |
| 101 | LXC | cloudflare | 192.168.0.111 | Cloudflare Tunnel |
| 102 | LXC | guacamole | 192.168.0.112 | Remote desktop gateway |
| 103 | LXC | ollama | 192.168.0.113 | IA local (LLMs) |
| 104 | LXC | monitoring | 192.168.0.114 | Zabbix + Uptime Kuma |
| 105 | LXC | wordpress | 192.168.0.115 | Blog |
| 106 | LXC | ansible | 192.168.0.116 | Automatización |
| 107 | VM | fortigate-server | 192.168.0.102 | Firewall VPN MSP |
