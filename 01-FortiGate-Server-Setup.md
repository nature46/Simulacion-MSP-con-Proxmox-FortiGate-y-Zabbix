# 🔥 FortiGate Server — Configuración en Proxmox

## Guía completa paso a paso para desplegar FortiGate VM en Proxmox VE

---

## Contexto

Este FortiGate actúa como **firewall central del MSP** (Virtus Informática) y es el extremo "servidor" del túnel VPN site-to-site. Gestiona:

- Conectividad WAN (acceso a internet)
- Red LAN interna del MSP (`10.0.1.0/24`)
- Túnel VPN hacia clientes remotos
- NAT para la red interna
- Firewall policies centralizadas

---

## Requisitos Previos

| Requisito | Detalle |
|-----------|---------|
| **Proxmox VE** | Versión 8.x instalado y accesible |
| **Red física** | 192.168.0.0/24 con gateway en 192.168.0.1 |
| **Cuenta Fortinet** | Registro gratuito en [support.fortinet.com](https://support.fortinet.com) |
| **Imagen FortiGate** | `FGT_VM64_KVM-v7.6.6.M-build3652-FORTINET.out.kvm.zip` |
| **Acceso Proxmox** | SSH root@192.168.0.100 |

---

## Arquitectura Objetivo

```
PROXMOX "ODIN" (192.168.0.100)
│
├─► vmbr0 (192.168.0.0/24) ───► Red física / WAN
│   │
│   └─► VM-107 port1: 192.168.0.102 (WAN)
│
└─► vmbr1 (10.0.1.0/24) ───► Red interna MSP
    │
    ├─► VM-107 port2: 10.0.1.254 (gateway LAN)
    └─► LXC-104 eth1: 10.0.1.10 (Zabbix Server)
```

---

## Paso 1 — Crear Bridge Virtual vmbr1

Este bridge aísla la red interna del MSP de la red física, proporcionando segmentación de red profesional.

### SSH al host Proxmox

```bash
ssh root@192.168.0.100
```

### Editar configuración de red

```bash
nano /etc/network/interfaces
```

### Añadir al final del archivo

```bash
# Red interna FortiGate (LAN MSP)
auto vmbr1
iface vmbr1 inet static
    address 10.0.1.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    comment Red interna MSP - FortiGate LAN
```

**Explicación parámetros:**
- `address 10.0.1.1/24`: IP del bridge (Proxmox puede acceder a esta red)
- `bridge-ports none`: Bridge virtual sin interfaz física asociada
- `bridge-stp off`: Spanning Tree Protocol desactivado (no necesario en red virtual)
- `bridge-fd 0`: Forwarding delay 0 (arranque rápido)

### Aplicar sin reiniciar

```bash
ifreload -a
```

### Verificar

```bash
ip addr show vmbr1
```

**Salida esperada:**

```
12: vmbr1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN
    inet 10.0.1.1/24 scope global vmbr1
```

> ✅ Si aparece `state UNKNOWN` o `UP`, el bridge está creado correctamente.

---

## Paso 2 — Crear VM en Proxmox

### Proxmox Web UI

Navegar a: **Datacenter → odin → Create VM**

### Pestaña General

| Campo | Valor |
|-------|-------|
| **VM ID** | `107` |
| **Name** | `fortigate-server` |
| **Start at boot** | ❌ (desmarcar por ahora) |

### Pestaña OS

| Campo | Valor |
|-------|-------|
| **Use CD/DVD disc image file** | ❌ |
| **Do not use any media** | ✅ |

> No necesitamos ISO — importaremos el disco qcow2 directamente.

### Pestaña System

| Campo | Valor |
|-------|-------|
| **BIOS** | `SeaBIOS` |
| **Machine** | `Default (i440fx)` |
| **SCSI Controller** | `VirtIO SCSI` |

> **No marcar** "Qemu Agent" ni "Nested Virtualization" — no aplican para firewall.

### Pestaña Disks

**Eliminar** el disco si aparece uno por defecto.

**No añadir ningún disco aquí** — lo importaremos manualmente después.

### Pestaña CPU

| Campo | Valor | Nota |
|-------|-------|------|
| **Sockets** | `1` | |
| **Cores** | `1` | ⚠️ **Máximo 1 core por licencia eval** |
| **Type** | `host` | Mejor rendimiento |

> **Crítico:** La licencia eval de FortiGate limita a 1 vCPU. Con 2+ cores la VM no arrancará correctamente.

### Pestaña Memory

| Campo | Valor |
|-------|-------|
| **Memory (MiB)** | `2048` (2 GB) |

### Pestaña Network

| Campo | Valor |
|-------|-------|
| **Bridge** | `vmbr0` (WAN) |
| **Model** | `VirtIO` |
| **Firewall** | ❌ (desmarcar) |

### Confirm

- **Start after created:** ❌ (NO arrancar aún)
- Click **Finish**

---

## Paso 3 — Añadir Segunda Interfaz (LAN)

**VM-107 → Hardware → Add → Network Device:**

| Campo | Valor |
|-------|-------|
| **Bridge** | `vmbr1` |
| **Model** | `VirtIO` |
| **Firewall** | ❌ |

**Resultado:** La VM tiene dos NICs

| NIC | Bridge | IP futura | Rol |
|-----|--------|-----------|-----|
| net0 | vmbr0 | 192.168.0.102 | WAN (internet) |
| net1 | vmbr1 | 10.0.1.254 | LAN (red MSP) |

---

## Paso 4 — Importar Disco FortiGate

### Ubicación del archivo

El archivo `.kvm.zip` descargado de Fortinet contiene `fortios.qcow2` (~120 MB comprimido, ~2 GB descomprimido).

**Ejemplo de ruta típica:**

```
/mnt/pve/nas-tools/template/iso/FortiGate_VM64_KVM-v7.6.6.M-build3652/fortios.qcow2
```

### Importar a la VM

```bash
qm importdisk 107 /ruta/completa/al/fortios.qcow2 local-lvm
```

**Comando real usado en el proyecto:**

```bash
qm importdisk 107 /mnt/pve/nas-tools/template/iso/FortiGate_VM64_KVM-v7.6.6.M-build3652-FORTINET.out.kvm/fortios.qcow2 local-lvm
```

**Salida esperada:**

```
importing disk '/mnt/pve/...' to VM 107 ...
transferred 2.0 GiB of 2.0 GiB (100.00%)
successfully imported disk as 'unused0:local-lvm:vm-107-disk-0'
```

> El disco se expande de ~120 MB a 2 GB gracias a thin provisioning de qcow2.

---

## Paso 5 — Adjuntar Disco y Configurar Boot

### Adjuntar disco

**VM-107 → Hardware:**

1. Seleccionar **Unused Disk 0**
2. Click **Edit** → **Add**
3. Aparece como **scsi0** (2 GB)

### Configurar boot order

**VM-107 → Options → Boot Order:**

1. Doble click en **Boot Order**
2. ✅ Marcar **solo scsi0**
3. ❌ Desmarcar net0, net1
4. Arrastrar **scsi0** a la primera posición
5. **OK**

> Esto asegura que la VM arranque desde el disco FortiGate, no intente PXE boot.

---

## Paso 6 — Primer Arranque y Login

### Arrancar VM

**VM-107 → Start**

Esperar 20-30 segundos al prompt:

```
FortiGate-VM64-KVM login:
```

### Login inicial

```
Username: admin
Password: (vacío — pulsar Enter)
```

### Cambiar contraseña

El sistema forzará cambio de contraseña:

```
New Password: Virtus2026!CS
Confirm Password: Virtus2026!CS
```

> ⚠️ **Importante:** Usar contraseña **alfanumérica simple** sin símbolos raros. El teclado de consola usa layout inglés y caracteres como `@`, `#`, `ñ` pueden causar problemas.

**Prompt tras login exitoso:**

```
fortigate-server #
```

---

## Paso 7 — Configurar Interfaces de Red

### port1 (WAN)

```bash
config system interface
    edit port1
        set mode static
        set ip 192.168.0.102 255.255.255.0
        set allowaccess ping https ssh
    next
end
```

**Explicación:**
- `mode static`: IP fija (no DHCP)
- `ip`: Dirección IP + máscara
- `allowaccess`: Servicios permitidos en esta interfaz (GUI, SSH, ping)

### Gateway por defecto

```bash
config router static
    edit 1
        set gateway 192.168.0.1
        set device port1
    next
end
```

### Verificar conectividad

```bash
execute ping 192.168.0.100
```

**Esperado:** Respuesta del Proxmox host

```bash
execute ping 8.8.8.8
```

**Esperado:** Respuesta de Google DNS (confirma internet)

### Verificar estado interfaces

```bash
get system interface physical
```

**Salida esperada:**

```
== [port1]
mode: static
ip: 192.168.0.102 255.255.255.0
status: up
speed: 10000Mbps (Duplex: full)

== [port2]
mode: static
ip: 0.0.0.0 0.0.0.0
status: up
speed: 10000Mbps (Duplex: full)
```

---

## Paso 8 — Acceso GUI Web

### Cambiar puerto HTTPS (evitar conflicto con VPN)

**Problema:** El puerto 443 (HTTPS) lo usará el túnel VPN (IKE TCP). Si dejamos la GUI en 443, habrá conflicto.

**Solución:**

```bash
config system global
    set admin-sport 8443
end
```

### Acceso desde navegador

```
https://192.168.0.102:8443
```

El navegador mostrará advertencia de certificado autofirmado — **aceptar y continuar**.

**Login:**
- Username: `admin`
- Password: `Virtus2026!CS`

---

## Paso 9 — Aplicar Licencia Eval

### Desde la GUI

**Dashboard → System Information → License:**

- Click **Upload** o **Use evaluation license**
- Login con tu cuenta Fortinet
- Aplicar licencia eval

**El sistema se reiniciará automáticamente** — esperar 1-2 minutos.

### Reconectar

Tras reinicio, acceder de nuevo a `https://192.168.0.102:8443`

---

## Paso 10 — Setup Wizard

Al entrar por primera vez tras aplicar licencia:

### Migrate Config

- Click **Skip this step**

### System Settings

| Campo | Valor |
|-------|-------|
| **Hostname** | `fortigate-server` |
| **Timezone** | `(GMT+1:00) Europe/Madrid` |
| **Idle timeout** | 60 (default) |

**Next → Finish**

### Dashboard Layout

- Seleccionar **Optimal** (recomendado para MSP)

---

## Paso 11 — Configurar port2 (LAN) desde GUI

**Network → Interfaces → port2 → Edit:**

| Campo | Valor |
|-------|-------|
| **Alias** | `LAN` |
| **Addressing mode** | Manual |
| **IP/Network Mask** | `10.0.1.254 / 255.255.255.0` |
| **Administrative Access** | HTTPS ✅, SSH ✅, PING ✅ |
| **Role** | `LAN` |
| **Device Detection** | ❌ (desmarcar) |
| **LLDP** | ❌ (desmarcar) |

**OK**

### Verificar en CLI

```bash
get system interface physical
```

**Debe mostrar:**

```
== [port1]
ip: 192.168.0.102 255.255.255.0
status: up

== [port2]
ip: 10.0.1.254 255.255.255.0
status: up
```

---

## Paso 12 — Crear Política Firewall LAN → WAN

**Policy & Objects → Firewall Policy → Create New:**

| Campo | Valor |
|-------|-------|
| **Name** | `LAN-to-WAN` |
| **Incoming Interface** | `port2` (LAN) |
| **Outgoing Interface** | `port1` (WAN) |
| **Source** | `all` |
| **Destination** | `all` |
| **Schedule** | `always` |
| **Service** | `ALL` |
| **Action** | `ACCEPT` ✅ |
| **NAT** | ✅ **Enable** (crítico) |
| **IP Pool Configuration** | `Use Outgoing Interface Address` |
| **Log Allowed Traffic** | ✅ All sessions |

**OK**

**Resultado en lista de políticas:**

```
1. LAN-to-WAN  (ACCEPT + NAT enabled)
2. Implicit Deny
```

> Esta política permite que la red `10.0.1.0/24` salga a internet con NAT usando la IP del port1.

---

## Paso 13 — Backup y Snapshot

### Snapshot en Proxmox

**VM-107 → Snapshots → Take Snapshot:**

| Campo | Valor |
|-------|-------|
| **Name** | `baseline-config-ok` |
| **Include RAM** | ❌ (opcional, no necesario) |
| **Description** | FortiGate con port1/port2 configurados, GUI en 8443, NAT activo |

### Backup a NAS

**VM-107 → Backup → Backup now:**

| Campo | Valor |
|-------|-------|
| **Storage** | `nas-backups` (o tu storage de backup) |
| **Mode** | `Snapshot` |
| **Compression** | `ZSTD` |

**Start Backup**

### Verificar backup

```bash
ls -lh /mnt/pve/nas-backups/dump/ | grep 107
```

**Esperado:**

```
vzdump-qemu-107-2026_02_XX-XX_XX_XX.vma.zst
```

---

## Verificación Final

### Desde CLI del FortiGate

```bash
# Estado interfaces
get system interface physical

# Routing
get router info routing-table all

# Políticas
show firewall policy
```

**Tabla de rutas esperada:**

```
S*      0.0.0.0/0 [10/0] via 192.168.0.1, port1
C       10.0.1.0/24 is directly connected, port2
C       192.168.0.0/24 is directly connected, port1
```

### Desde LXC-104 (tras configurar dual-homed)

```bash
ping -c 3 10.0.1.254   # FortiGate LAN
ping -c 3 8.8.8.8      # Internet vía NAT
```

**Ambos deben responder.**

---

## Resumen de Configuración

```
VM-107: FortiGate Server (Sede MSP)
├─► Hardware: 1 vCPU, 2 GB RAM, 2 GB disk
├─► Licencia: Eval (15 días, funcionalidad completa)
├─► port1 (WAN): 192.168.0.102/24 → vmbr0
│   ├─► Gateway: 192.168.0.1
│   ├─► Servicios: HTTPS (8443), SSH, PING
│   └─► Salida a internet
├─► port2 (LAN): 10.0.1.254/24 → vmbr1
│   ├─► Red interna MSP
│   ├─► Servicios: HTTPS, SSH, PING
│   └─► Gateway para 10.0.1.0/24
├─► Política: LAN → WAN (ACCEPT + NAT)
├─► Hostname: fortigate-server
├─► Timezone: Europe/Madrid
└─► GUI: https://192.168.0.102:8443
```

---

## Siguiente Paso

Con el FortiGate Server operativo, el próximo paso es configurar:

1. **[LXC-104 dual-homed](03-LXC-104-Dual-Homed.md)** — Añadir eth1 en red 10.0.1.0/24
2. **[FortiGate Cliente](02-FortiGate-Cliente-Setup.md)** — Desplegar en VMware
3. **[Túnel VPN IPsec](04-VPN-IPsec-Tunnel.md)** — Conectar ambos FortiGates

---

## Comandos de Referencia Rápida

```bash
# Estado general
get system status
get system performance status

# Interfaces
get system interface physical
show system interface

# Routing
get router info routing-table all
get router info kernel

# Sesiones activas
get system session list

# Backup config
execute backup config tftp <filename> <tftp_server>

# Reboot
execute reboot

# Factory reset (¡cuidado!)
execute factoryreset
```

---

**Documentos relacionados:**
- [→ FortiGate Cliente Setup](02-FortiGate-Cliente-Setup.md)
- [→ LXC-104 Dual-Homed](03-LXC-104-Dual-Homed.md)
- [→ Túnel VPN IPsec](04-VPN-IPsec-Tunnel.md)
- [← README Principal](README.md)

---

*Última actualización: Febrero 2025*
