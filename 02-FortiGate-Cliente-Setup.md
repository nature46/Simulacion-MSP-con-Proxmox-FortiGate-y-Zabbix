# 🔥 FortiGate Cliente — Configuración en VMware Workstation

## Guía completa para desplegar el FortiGate del lado cliente simulando empresa remota

---

## Contexto

Este FortiGate simula el **firewall instalado en la oficina de un cliente del MSP** (empresa en Castellón). Es el extremo "cliente" del túnel VPN site-to-site y gestiona:

- Conectividad WAN (acceso a internet del cliente)
- Red LAN interna del cliente (`10.0.2.0/24`)
- Túnel VPN hacia sede MSP
- NAT para hosts internos
- Split-tunneling (VPN solo para tráfico MSP)

---

## Arquitectura Objetivo

```
PC WINDOWS (VMware Workstation)
│
├─► Red física 192.168.0.0/24 (Bridged)
│   │
│   └─► FortiGate Cliente port1: 192.168.0.103 (WAN)
│
└─► VMnet2 (10.0.2.0/24) (Host-only)
    │
    ├─► FortiGate Cliente port2: 10.0.2.254 (gateway LAN)
    └─► Ubuntu VM: 10.0.2.100 (host monitorizado)
```

---

## Requisitos Previos

| Requisito | Detalle |
|-----------|---------|
| **VMware Workstation** | Player o Pro (ambos funcionan) |
| **Sistema operativo** | Windows 10/11 |
| **Imagen FortiGate** | OVF para VMware (v7.6.6+) |
| **RAM disponible** | 4 GB libres (2 GB para FortiGate + 2 GB para Ubuntu) |
| **Cuenta Fortinet** | Para licencia eval (o funcionar sin licencia) |

---

## Paso 1 — Crear Red Virtual VMnet2

Esta red simula la LAN interna de la oficina del cliente.

### Abrir Virtual Network Editor

**VMware Workstation → Edit → Virtual Network Editor**

Si aparece UAC (Windows), aceptar permisos de administrador.

### Añadir VMnet2

1. Click **Add Network**
2. Seleccionar **VMnet2**
3. **Type:** Host-only (crea red aislada sin acceso directo a internet)
4. **Subnet IP:** `10.0.2.0`
5. **Subnet mask:** `255.255.255.0`
6. ❌ **Desmarcar** "Use local DHCP service to distribute IP addresses"
7. **Apply → OK**

**Resultado:** VMnet2 es una red interna donde los hosts solo se comunican entre sí y con el host Windows. No tiene salida a internet directa — eso lo proporcionará el FortiGate con NAT.

---

## Paso 2 — Importar FortiGate OVF

### Descomprimir archivo descargado

El archivo `.zip` de Fortinet contiene varios archivos OVF. Buscar:

```
FGT_VM64-v7.6.6.M-build3652.ovf
```

> **No confundir** con los archivos `hw13.ovf`, `hw15.ovf`, etc. — esos son para ESXi específico. El genérico `.ovf` funciona en Workstation.

### Importar

**VMware: File → Open:**

1. Navegar al `FortiGate-VM64.ovf`
2. **Name:** `FortiGate-Cliente`
3. **Storage path:** Ubicación local con espacio (el disco ocupará ~2 GB)
4. **Start** (solo importa, no arranca)

La importación puede tardar 1-2 minutos.

---

## Paso 3 — Configurar Adaptadores de Red

**Click derecho en FortiGate-Cliente → Settings → Hardware**

### Adaptador de red 1 (WAN)

| Configuración | Valor |
|---------------|-------|
| **Network connection** | Bridged: Connected directly to the physical network |
| **Replicate physical network connection state** | ❌ Desmarcar |

**¿Por qué Bridged y no NAT?**

- **Bridged:** La VM aparece como otro dispositivo en tu red física `192.168.0.x`. Puede comunicarse directamente con el FortiGate Server en Proxmox (`192.168.0.102`).
- **NAT:** La VM estaría en una red oculta `192.168.182.x` y no podría establecer el túnel VPN con el Server.

**Bridged es esencial para el túnel VPN site-to-site.**

### Adaptador de red 2 (LAN cliente)

| Configuración | Valor |
|---------------|-------|
| **Network connection** | Custom: Specific virtual network |
| **Virtual network** | VMnet2 (Host-only) |

Esta es la red interna donde conectaremos el Ubuntu con Zabbix Agent.

### Adaptadores 3 a 10

La importación OVF crea 10 adaptadores por defecto. **Desmarcar "Connect at power on"** en todos los adaptadores 3-10 (no los usaremos).

---

## Paso 4 — Primer Arranque y Login

### Encender VM

**Power On**

Esperar al prompt (puede tardar 1-2 minutos en primer arranque):

```
FortiGate-VM64 login:
```

### Login inicial

```
Username: admin
Password: (vacío — pulsar Enter)
```

### Cambiar contraseña

```
New Password: Virtus2026!CS
Confirm Password: Virtus2026!CS
```

**Prompt tras login:**

```
FortiGate-VM64 #
```

---

## Paso 5 — Configurar Interfaces

### port1 (WAN) — IP estática en red física

**Decisión de diseño:** Usamos IP estática `192.168.0.103` en lugar de DHCP para que la configuración sea reproducible y el túnel VPN tenga un remote gateway fijo.

```bash
config system interface
    edit port1
        set mode static
        set ip 192.168.0.103 255.255.255.0
        set allowaccess ping https ssh http
        set role wan
    next
end
```

### Gateway por defecto

```bash
config router static
    edit 1
        set gateway 192.168.0.1
        set device port1
    next
end
```

### Verificar conectividad WAN

```bash
execute ping 192.168.0.1       # Router
execute ping 192.168.0.100     # Proxmox
execute ping 192.168.0.102     # FortiGate Server
execute ping 8.8.8.8           # Internet
```

**Todos deben responder.**

### port2 (LAN) — Red interna cliente

```bash
config system interface
    edit port2
        set mode static
        set ip 10.0.2.254 255.255.255.0
        set allowaccess ping https ssh
        set role lan
    next
end
```

### Verificar estado interfaces

```bash
get system interface physical
```

**Salida esperada:**

```
== [port1]
mode: static
ip: 192.168.0.103 255.255.255.0
status: up

== [port2]
mode: static
ip: 10.0.2.254 255.255.255.0
status: up
```

---

## Paso 6 — Acceso GUI Web

**Desde navegador Windows:**

```
https://192.168.0.103
```

**Aceptar certificado autofirmado:**

| Navegador | Acción |
|-----------|--------|
| Chrome/Edge | "Advanced" → "Proceed to 192.168.0.103 (unsafe)" |
| Firefox | "Advanced" → "Accept the Risk and Continue" |
| Brave | Desactivar shields para el sitio, o usar Chrome |

**Login:**
- Username: `admin`
- Password: `Virtus2026!CS`

---

## Paso 7 — Licencia (Opcional)

### Opción A: Segunda licencia eval

Si tienes una segunda cuenta Fortinet con licencia eval no usada:

**Dashboard → System Information → License → Upload/Apply**

### Opción B: Funcionar sin licencia (laboratorio)

La GUI mostrará un popup de licencia — **cerrar o hacer clic en "Later"**.

**Funcionalidad disponible sin licencia:**
- ✅ Configuración VPN IPsec completa por CLI
- ✅ Routing y firewall policies
- ✅ NAT
- ✅ Acceso GUI limitado pero funcional

**Limitaciones sin licencia:**
- ❌ Algunas features avanzadas de GUI
- ❌ Updates automáticos
- ❌ Soporte oficial

Para este laboratorio educativo, **funcionar sin licencia es totalmente aceptable**.

---

## Paso 8 — Setup Wizard

Si aparece el wizard tras login:

### Migrate Config

- **Skip this step**

### System Settings

| Campo | Valor |
|-------|-------|
| **Hostname** | `fortigate-cliente` |
| **Timezone** | `(GMT+1:00) Europe/Madrid` |

**Next → Finish**

### Dashboard Layout

- Seleccionar **Optimal**

---

## Paso 9 — Crear Política Firewall LAN → WAN (NAT)

Esta política permite que los hosts de la red `10.0.2.0/24` (como el Ubuntu) salgan a internet.

### Por GUI

**Policy & Objects → Firewall Policy → Create New:**

| Campo | Valor |
|-------|-------|
| **Name** | `LAN-to-Internet` |
| **Incoming Interface** | `port2` (LAN) |
| **Outgoing Interface** | `port1` (WAN) |
| **Source** | `all` |
| **Destination** | `all` |
| **Service** | `ALL` |
| **Action** | `ACCEPT` ✅ |
| **NAT** | ✅ **Enable** |
| **Log** | ✅ ON |

**OK**

### Por CLI (alternativa)

```bash
config firewall policy
    edit 0
        set name "LAN-to-Internet"
        set srcintf "port2"
        set dstintf "port1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
end
```

### Verificar

```bash
show firewall policy | grep -A5 "LAN-to-Internet"
```

Debe mostrar `set nat enable`.

---

## Paso 10 — Backup y Snapshot

### Snapshot en VMware

**VM → Snapshot → Take Snapshot:**

| Campo | Valor |
|-------|-------|
| **Name** | `baseline-config-ok` |
| **Description** | FortiGate con port1/port2, NAT activo, preparado para VPN |
| **Snapshot the virtual machine's memory** | ❌ |

### Backup de configuración

**Desde GUI:** System → Configuration → Backup → **Download**

Guarda el archivo `.conf` en un lugar seguro.

---

## Verificación Final

### Desde CLI

```bash
# Estado general
get system status

# Interfaces
get system interface physical

# Routing
get router info routing-table all
```

**Tabla de rutas esperada:**

```
S*      0.0.0.0/0 [10/0] via 192.168.0.1, port1
C       10.0.2.0/24 is directly connected, port2
C       192.168.0.0/24 is directly connected, port1
```

### Desde Ubuntu (después de crearlo)

```bash
ping -c 3 10.0.2.254   # FortiGate LAN
ping -c 3 8.8.8.8      # Internet vía NAT
```

---

## Resumen de Configuración

```
VMware: FortiGate Cliente (Oficina Castellón simulada)
├─► Hardware: 1 vCPU, 2 GB RAM, 2 GB disk
├─► Licencia: Eval o sin licencia (lab)
├─► port1 (WAN): 192.168.0.103/24 → Bridged
│   ├─► Gateway: 192.168.0.1
│   ├─► Servicios: HTTPS, SSH, PING, HTTP
│   └─► Salida a internet + túnel VPN
├─► port2 (LAN): 10.0.2.254/24 → VMnet2 (Host-only)
│   ├─► Red interna cliente
│   ├─► Servicios: HTTPS, SSH, PING
│   └─► Gateway para 10.0.2.0/24
├─► Política: LAN → WAN (ACCEPT + NAT)
├─► Hostname: fortigate-cliente
├─► Timezone: Europe/Madrid
└─► GUI: https://192.168.0.103
```

---

## Troubleshooting Común

### port1 aparece DOWN

**Causa:** Adaptador de red no conectado o tipo incorrecto.

**Solución:**

1. En VMware → VM menu → Network Adapter → ✅ Verificar "Connected"
2. Verificar que es **Bridged** (no NAT)
3. Desconectar y reconectar el adaptador
4. Dentro del FortiGate:

```bash
config system interface
    edit port1
        set status up
    next
end
```

### No puedo acceder a la GUI

**Causa A: Firewall de Windows bloqueando**

```powershell
# En Windows PowerShell (admin)
New-NetFirewallRule -DisplayName "FortiGate-Cliente" -Direction Inbound -LocalPort 443 -Protocol TCP -Action Allow
```

**Causa B: `allowaccess` no incluye https**

```bash
config system interface
    edit port1
        set allowaccess ping https ssh http
    next
end
```

**Causa C: Navegador bloqueando certificado**

Usar Chrome/Edge en lugar de Brave. Si Chrome no muestra botón "Proceed", escribir `thisisunsafe` en la página (sin campo visible).

### Ubuntu no tiene internet tras conectarse a VMnet2

**Causa:** Política NAT no configurada en FortiGate Cliente.

**Solución:** Ver Paso 9 arriba.

---

## Siguiente Paso

Con ambos FortiGates operativos, el siguiente paso es:

**[Configurar Túnel VPN IPsec](04-VPN-IPsec-Tunnel.md)** — Conectar FortiGate Server ↔ FortiGate Cliente

Esto permitirá que las redes `10.0.1.0/24` (MSP) y `10.0.2.0/24` (Cliente) se comuniquen de forma segura y cifrada.

---

## Comandos de Referencia Rápida

```bash
# Estado interfaces
get system interface physical

# Test conectividad
execute ping 192.168.0.102   # FortiGate Server
execute ping 192.168.0.100   # Proxmox
execute ping 8.8.8.8         # Internet

# Ver políticas firewall
show firewall policy

# Ver sesiones activas
get system session list

# Reboot
execute reboot

# Cambiar hostname
config system global
    set hostname fortigate-cliente
end
```

---

**Documentos relacionados:**
- [← FortiGate Server Setup](01-FortiGate-Server-Setup.md)
- [→ Túnel VPN IPsec](04-VPN-IPsec-Tunnel.md)
- [→ Ubuntu Cliente Setup](05-Ubuntu-Cliente-Setup.md)
- [← README Principal](README.md)

---

*Última actualización: Febrero 2025*
