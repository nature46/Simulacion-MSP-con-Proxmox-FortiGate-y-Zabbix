# 🔥 Proyecto ODIN — VPN Site-to-Site FortiGate + Zabbix MSP

## Simulación profesional de infraestructura MSP con Proxmox, FortiGate y Zabbix

[![Estado](https://img.shields.io/badge/estado-funcional-success)](https://github.com/nature46/Simulacion-MSP-con-Proxmox-FortiGate-y-Zabbix/blob/main)
[![FortiGate](https://img.shields.io/badge/FortiGate-7.6.6-red)](https://github.com/nature46/Simulacion-MSP-con-Proxmox-FortiGate-y-Zabbix/blob/main)
[![Zabbix](https://img.shields.io/badge/Zabbix-7.0%20LTS-blue)](https://github.com/nature46/Simulacion-MSP-con-Proxmox-FortiGate-y-Zabbix/blob/main)
[![Proxmox](https://img.shields.io/badge/Proxmox-9.x-orange)](https://github.com/nature46/Simulacion-MSP-con-Proxmox-FortiGate-y-Zabbix/blob/main)

---

## 📋 Descripción

Este proyecto replica la **infraestructura completa de un Managed Service Provider (MSP)** real como Virtus Informática, donde una empresa central gestiona y monitoriza remotamente la infraestructura de múltiples clientes a través de túneles VPN seguros.

**Objetivo educativo:** Preparación para prácticas profesionales en empresa MSP, dominando el stack tecnológico FortiGate + Zabbix + Proxmox usado en producción real.

### Escenario simulado

```
SEDE CENTRAL MSP               TÚNEL VPN CIFRADO              EMPRESA CLIENTE
(Proxmox "ODIN")              IPsec Site-to-Site          (VMware en PC Windows)
┌─────────────────┐                 ⟺                       ┌──────────────────┐
│ FortiGate Server│ ◄─────────────────────────────────────► │ FortiGate Cliente│
│ 192.168.0.102   │         10.0.1.0/24 ↔ 10.0.2.0/24       │ 192.168.0.103    │
│                 │                                          │                  │
│ Zabbix Server   │ ──── Monitorización remota ───────────► │ Ubuntu 24.04 VM  │
│ 10.0.1.10       │         (por VPN cifrada)               │ 10.0.2.100       │
└─────────────────┘                                          └──────────────────┘
```

---

## 🎯 Estado del Proyecto

### ✅ FASE 1 — FortiGate + VPN (COMPLETADA)

- Red física `vmbr0` (192.168.0.0/24) y red interna `vmbr1` (10.0.1.0/24) operativas
- VM-107 FortiGate Server en Proxmox — port1 WAN + port2 LAN + NAT
- FortiGate Cliente en VMware — Bridged WAN + VMnet2 LAN
- Túnel VPN IPsec Phase 1 + Phase 2 establecido y estable
- Políticas firewall bidireccionales (LAN↔VPN) activas
- Split-tunneling — solo tráfico MSP va por VPN, internet sale directo
- GUI FortiGate Server movida a puerto 8443 (evita conflicto con IKE en 443)

### ✅ FASE 2 — Zabbix + Monitorización (COMPLETADA)

- LXC-104 dual-homed (eth0 management 192.168.0.114 + eth1 monitored 10.0.1.10)
- Hookscript Proxmox para rutas persistentes (eth0 preferida + ruta VPN 10.0.2.0/24)
- Zabbix Server 7.0 LTS en Docker Compose con PostgreSQL 15 Alpine
- Uptime Kuma coexistiendo en el mismo LXC-104
- Cloudflare Tunnel → `zabbix.nature46.uk` (acceso externo sin puertos abiertos)
- Housekeeping optimizado (14d history / 180d trends)
- VM Ubuntu 24.04 LTS cliente (10.0.2.100) en VMnet2 con red estable (renderer: networkd)
- Zabbix Agent 2 instalado, configurado y reportando métricas a través de la VPN
- 25 triggers activos (CPU, RAM, disco, red, seguridad, disponibilidad)
- Alertas Telegram — notificación de problema y resolución automática
- Alertas Email (Gmail) — canal formal con contraseña de aplicación
- Dashboard MSP "Vista Global" con 8 widgets en tiempo real

### 🔄 PENDIENTE

- SNMP monitoring de los propios FortiGates desde Zabbix
- Zabbix Proxy en red cliente (arquitectura distribuida)
- Automatización con Ansible
- Runbooks y procedimientos operativos MSP
- NSE 1 + NSE 2 (certificaciones gratuitas Fortinet)

---

## 🗺️ Arquitectura de Red Completa

```
RED FÍSICA 192.168.0.0/24 (vmbr0)
├── 192.168.0.1        Router/Gateway (salida internet)
├── 192.168.0.100      Proxmox "ODIN" (hipervisor)
├── 192.168.0.101      NAS "Thoth" (almacenamiento NFS)
├── 192.168.0.102      FortiGate Server port1 (WAN)  ← GUI: :8443
├── 192.168.0.103      FortiGate Cliente port1 (WAN) ← VMware Bridged
└── 192.168.0.114      LXC-104 eth0 (management)

RED INTERNA MSP 10.0.1.0/24 (vmbr1)
├── 10.0.1.1           Proxmox bridge gateway
├── 10.0.1.10          LXC-104 eth1 (Zabbix Server — interface monitored)
└── 10.0.1.254         FortiGate Server port2 (gateway LAN MSP)
         │
         └─────────────────────────────────────┐
                  VPN IPsec Site-to-Site        │
                  (cifrado AES/SHA)             │
         ┌─────────────────────────────────────┘
         │
RED INTERNA CLIENTE 10.0.2.0/24 (VMnet2 — Host-only)
├── 10.0.2.100         Ubuntu 24.04 LTS VM (Zabbix Agent 2)
└── 10.0.2.254         FortiGate Cliente port2 (gateway LAN cliente)
```

---

## 📚 Documentación

| Documento | Descripción |
|---|---|
| [01-FortiGate-Server-Setup.md](01-FortiGate-Server-Setup.md) | Despliegue FortiGate en Proxmox — VM, interfaces, NAT, GUI |
| [02-FortiGate-Cliente-Setup.md](02-FortiGate-Cliente-Setup.md) | Despliegue FortiGate en VMware — OVF, VMnet2, políticas |
| [03-LXC-104-Dual-Homed.md](03-LXC-104-Dual-Homed.md) | LXC-104 dual-interface + Stack Zabbix Docker Compose |
| [03b-LXC-104-Hookscript-Rutas.md](03b-LXC-104-Hookscript-Rutas.md) | Hookscript Proxmox — rutas persistentes tras reinicio |
| [04-VPN-IPsec-Tunnel.md](04-VPN-IPsec-Tunnel.md) | Túnel VPN IPsec completo — Phase 1, Phase 2, políticas, verificación |
| [05-Ubuntu-Cliente-Setup.md](05-Ubuntu-Cliente-Setup.md) | VM Ubuntu 24.04 cliente — red, Zabbix Agent 2, registro en servidor |
| [06-Zabbix-Alertas-Dashboard.md](06-Zabbix-Alertas-Dashboard.md) | Triggers, alertas Telegram/Gmail, dashboard MSP |
| [06-Troubleshooting.md](06-Troubleshooting.md) | Problemas reales encontrados y soluciones documentadas |

---

## 🔧 Stack Tecnológico

| Tecnología | Versión | Rol |
|---|---|---|
| **Proxmox VE** | 9.x | Hipervisor principal (sede MSP) |
| **FortiGate VM** | 7.6.6 build 3652 | Firewalls VPN (KVM + OVF) |
| **Zabbix Server** | 7.0 LTS | Monitorización centralizada MSP |
| **PostgreSQL** | 15 Alpine | Base de datos Zabbix |
| **Docker Compose** | v2.x | Orquestación servicios LXC-104 |
| **VMware Workstation** | 17.x | Host de simulación sede cliente |
| **Ubuntu LXC** | 22.04 LTS | OS containers Proxmox |
| **Ubuntu VM** | 24.04 LTS | VM cliente con Zabbix Agent 2 |
| **Uptime Kuma** | 2.x | Monitorización complementaria |
| **Cloudflare Tunnel** | — | Acceso externo sin puertos abiertos |

---

## 💻 Hardware

### Servidor "ODIN" (sede MSP)
```
CPU:     Intel N100 (4 cores, 3.4 GHz boost, Alder Lake-N)
RAM:     16 GB DDR4
Storage: 256 GB NVMe SSD
Red:     1 GbE RJ45
Consumo: ~15W idle / ~25W carga
SO:      Proxmox VE 9.x
```

### NAS "Thoth"
```
Modelo:    UGREEN DH2300 (2-bay)
Capacidad: 3.6 TB usable (BTRFS RAID1)
Protocolo: NFS v3
Función:   Snippets, backups y datos persistentes
```

### PC Windows (sede cliente simulada)
```
SO:      Windows 11
VMware:  VMware Workstation Pro 17.x
VMs:     FortiGate Cliente (OVF) + Ubuntu 24.04 LTS
Rol:     Simula la sede remota de un cliente del MSP
```

---

## 📊 Direccionamiento IP

| Dispositivo | IP | Red | Rol |
|---|---|---|---|
| Router físico | `192.168.0.1` | vmbr0 | Gateway internet |
| Proxmox ODIN | `192.168.0.100` | vmbr0 | Hipervisor |
| NAS Thoth | `192.168.0.101` | vmbr0 | Almacenamiento NFS |
| FortiGate Server port1 | `192.168.0.102` | vmbr0 | WAN firewall MSP |
| FortiGate Cliente port1 | `192.168.0.103` | vmbr0 | WAN firewall cliente (Bridged) |
| LXC-104 eth0 | `192.168.0.114` | vmbr0 | Management Zabbix |
| LXC-104 eth1 | `10.0.1.10` | vmbr1 | Zabbix monitored interface |
| FortiGate Server port2 | `10.0.1.254` | vmbr1 | Gateway LAN MSP |
| Ubuntu 24.04 VM | `10.0.2.100` | VMnet2 | Host monitorizado (Zabbix Agent 2) |
| FortiGate Cliente port2 | `10.0.2.254` | VMnet2 | Gateway LAN cliente |

---

## 🚀 Cómo Replicar

### Requisitos
- Servidor con Proxmox VE (mínimo 8GB RAM, recomendado 16GB)
- PC Windows con VMware Workstation (mínimo 8GB RAM libres)
- Cuenta en [support.fortinet.com](https://support.fortinet.com) (licencias eval gratuitas)
- Imagen FortiGate VM KVM (para Proxmox) y OVF (para VMware)
- ISO Ubuntu 24.04 LTS para la VM cliente

### Orden de despliegue

```
1. Crear vmbr1 en Proxmox
2. Desplegar FortiGate Server (VM-107)          → guía 01
3. Configurar LXC-104 dual-homed + Zabbix       → guía 03
4. Crear VMnet2 en VMware + FortiGate Cliente   → guía 02
5. Crear VM Ubuntu 24.04 en VMnet2              → guía 05
6. Configurar túnel VPN IPsec site-to-site      → guía 04
7. Instalar Zabbix Agent + alertas + dashboard  → guías 05 y 06
```

---

## 🔐 Accesos del Laboratorio

> ⚠️ Entorno de laboratorio aislado. No exponer credenciales reales en repositorios públicos.

| Servicio | Usuario | URL |
|---|---|---|
| Proxmox VE | `root` | `https://192.168.0.100:8006` |
| FortiGate Server | `admin` | `https://192.168.0.102:8443` |
| FortiGate Cliente | `admin` | `https://192.168.0.103` |
| Zabbix Web (LAN) | `Admin` | `http://192.168.0.114:8080` |
| Zabbix Web (externo) | `Admin` | `https://zabbix.nature46.uk` |
| Ubuntu VM | `pablo` | Consola VMware |

---

## 🎓 Conceptos Cubiertos

**Networking y Seguridad**
- VPN IPsec site-to-site con FortiGate (Phase 1 + Phase 2)
- Split-tunneling — tráfico selectivo por VPN
- NAT/PAT en firewalls empresariales
- Dual-homed hosts con rutas persistentes via hookscript Proxmox
- Network bridges virtuales (vmbr0/vmbr1) y redes VMware (VMnet2/VMnet8)

**Virtualización**
- KVM/QEMU en Proxmox (VM FortiGate)
- LXC containers unprivileged (Zabbix Server)
- VMware Workstation — simulación sede cliente con múltiples VMs
- Hookscripts Proxmox para automatización post-arranque

**Monitorización**
- Arquitectura Zabbix Server + Agent a través de VPN cifrada
- Docker Compose — orquestación de Zabbix + PostgreSQL + Uptime Kuma
- Triggers, actions y notificaciones multi-canal (Telegram + Gmail)
- Dashboards MSP profesionales con widgets en tiempo real
- Housekeeping y retención de datos optimizados para hardware limitado

**Troubleshooting**
- Diagnóstico de routing en hosts dual-homed
- Diagnóstico VPN IPsec con `diagnose vpn tunnel list`
- Resolución de conflictos NetworkManager/networkd en Ubuntu
- Rutas persistentes con hookscripts en Proxmox

---

## 📝 Notas

- FortiGate eval: 1 vCPU, 2GB RAM, funcionalidad básica completa para laboratorio
- Una licencia eval por cuenta Fortinet — segunda instancia funciona sin licencia para VPN básica
- Zabbix: GNU GPL v2 | Proxmox VE: AGPL v3 | Proyecto: uso educativo no comercial

---

## 🙏 Agradecimientos

- **Virtus Informática** — inspiración del proyecto y oportunidad de prácticas
- **Fortinet** — licencias eval y documentación técnica
- **Comunidades** — r/Proxmox, r/homelab, r/zabbix

---

**Autor:** Nat  
**Contexto:** Preparación para prácticas en Virtus Informática (Castellón)  
**Última actualización:** Marzo 2026
