# 🔥 Proyecto ODIN — VPN Site-to-Site FortiGate + Zabbix MSP

## Simulación profesional de infraestructura MSP con Proxmox, FortiGate y Zabbix

[![Estado](https://img.shields.io/badge/estado-funcional-success)]()
[![FortiGate](https://img.shields.io/badge/FortiGate-7.6.6-red)]()
[![Zabbix](https://img.shields.io/badge/Zabbix-7.0%20LTS-blue)]()
[![Proxmox](https://img.shields.io/badge/Proxmox-9.x-orange)]()

---

## 📋 Descripción

Este proyecto replica la **infraestructura completa de un Managed Service Provider (MSP)** real como Virtus Informática, donde una empresa central gestiona y monitoriza remotamente la infraestructura de múltiples clientes a través de túneles VPN seguros.

**Objetivo educativo:** Preparación para prácticas profesionales en empresa MSP, dominando el stack tecnológico FortiGate + Zabbix + Proxmox usado en producción.

### Escenario simulado

```
SEDE CENTRAL MSP               TÚNEL VPN CIFRADO              EMPRESA CLIENTE
(Proxmox "ODIN")              IPsec Site-to-Site            (VMware Workstation)
┌─────────────────┐                 ⟺                       ┌──────────────────┐
│ FortiGate Server│ ◄─────────────────────────────────────►  │ FortiGate Cliente│
│ 192.168.0.102   │         10.0.1.0/24 ↔ 10.0.2.0/24        │ 192.168.0.103    │
│                 │                                          │                  │
│ Zabbix Server   │ ──── Monitorización remota ───────────►  │ Ubuntu + Agent   │
│ 10.0.1.10       │         (por VPN cifrada)                │ 10.0.2.100       │
└─────────────────┘                                          └──────────────────┘
```

---

## 🎯 Logros Completados

### ✅ Infraestructura base
- [x] Red física vmbr0 (192.168.0.0/24) operativa
- [x] Red interna vmbr1 (10.0.1.0/24) aislada para FortiGate
- [x] VM-107 FortiGate Server en Proxmox funcional
- [x] LXC-104 dual-homed (eth0 management + eth1 monitoring)

### ✅ FortiGate VPN Site-to-Site
- [x] FortiGate Server configurado (port1 WAN + port2 LAN + NAT)
- [x] FortiGate Cliente en VMware (Bridged + VMnet2)
- [x] Túnel VPN IPsec Phase 1 y Phase 2 establecido
- [x] **Políticas firewall bidireccionales activas**
- [x] **Routing entre redes 10.0.1.0/24 ↔ 10.0.2.0/24 funcional**
- [x] Split-tunneling profesional (VPN solo para tráfico MSP, resto por internet)

### ✅ Monitorización Zabbix
- [x] Zabbix Server 7.0 LTS en Docker Compose
- [x] PostgreSQL 15 Alpine optimizado para Zabbix
- [x] Uptime Kuma coexistiendo en LXC-104
- [x] Cloudflare Tunnel para acceso externo (zabbix.nature46.uk)
- [x] Housekeeping configurado (14d history / 180d trends)

### ✅ Cliente remoto
- [x] VM Ubuntu en VMware conectada a VMnet2 (10.0.2.100)
- [x] Internet funcional vía NAT del FortiGate Cliente
- [x] Conectividad bidireccional Zabbix Server ↔ Cliente por VPN
- [ ] Zabbix Agent 2 instalado y reportando métricas *(próximo paso)*

---

## 🗺️ Arquitectura de Red Completa

```
RED FÍSICA 192.168.0.0/24 (vmbr0)
├── 192.168.0.1        Router/Gateway (salida internet)
├── 192.168.0.100      Proxmox "ODIN" (hipervisor)
├── 192.168.0.101      NAS "Thoth" (almacenamiento NFS)
├── 192.168.0.102      FortiGate Server port1 (WAN)
├── 192.168.0.103      FortiGate Cliente port1 (WAN) ← VMware Bridged
└── 192.168.0.114      LXC-104 eth0 (management)

RED INTERNA MSP 10.0.1.0/24 (vmbr1)
├── 10.0.1.1           Proxmox bridge gateway
├── 10.0.1.10          LXC-104 eth1 (Zabbix monitored interface)
└── 10.0.1.254         FortiGate Server port2 (gateway LAN MSP)
         │
         └─────────────────────────────────────┐
                  VPN IPsec Site-to-Site        │
         ┌─────────────────────────────────────┘
         │
RED INTERNA CLIENTE 10.0.2.0/24 (VMnet2)
├── 10.0.2.100         Ubuntu VM (Zabbix Agent)
└── 10.0.2.254         FortiGate Cliente port2 (gateway LAN cliente)
```

---

## 📚 Documentación del Proyecto

| Documento | Descripción | Enlace |
|-----------|-------------|--------|
| **README.md** | Este archivo — Visión general del proyecto | *(estás aquí)* |
| **01-FortiGate-Server-Setup.md** | Despliegue completo FortiGate en Proxmox (VM-107) | [Ver guía](01-FortiGate-Server-Setup.md) |
| **02-FortiGate-Cliente-Setup.md** | Despliegue FortiGate en VMware Workstation | [Ver guía](02-FortiGate-Cliente-Setup.md) |
| **03-LXC-104-Dual-Homed.md** | Configuración dual-interface + Stack Zabbix Docker | [Ver guía](03-LXC-104-Dual-Homed.md) |
| **04-VPN-IPsec-Tunnel.md** | Configuración completa túnel VPN site-to-site | [Ver guía](04-VPN-IPsec-Tunnel.md) |
| **05-Ubuntu-Cliente-Setup.md** | Configuración VM Ubuntu en red cliente | [Ver guía](05-Ubuntu-Cliente-Setup.md) |
| **06-Troubleshooting.md** | Problemas comunes y soluciones | [Ver guía](06-Troubleshooting.md) |
| **07-Diagrama-Red.md** | Diagramas de red (Mermaid + ASCII) | [Ver diagramas](07-Diagrama-Red.md) |

---

## 🔧 Stack Tecnológico

| Tecnología | Versión | Propósito |
|------------|---------|-----------|
| **Proxmox VE** | 9.x | Hipervisor principal (sede MSP) |
| **FortiGate VM** | 7.6.6 build 3652 | Firewalls VPN (KVM + OVF) |
| **Zabbix Server** | 7.0 LTS | Monitorización centralizada MSP |
| **PostgreSQL** | 15 Alpine | Base de datos Zabbix |
| **Docker Compose** | v2.x | Orquestación servicios LXC-104 |
| **VMware Workstation** | 17.x | Simulación sede cliente remota |
| **Ubuntu LXC** | 22.04 LTS | Sistema operativo containers |
| **Ubuntu Desktop** | 22.04 LTS | Cliente con Zabbix Agent |
| **Uptime Kuma** | 2.x | Monitorización complementaria |

---

## 💻 Hardware del Laboratorio

### Servidor Principal "ODIN"
```
Procesador: Intel N100 (4 cores, Alder Lake-N, 3.4 GHz boost)
RAM:        16 GB DDR4
Storage:    256 GB NVMe SSD (sistema + containers)
Network:    1 GbE RJ45
Power:      15W idle / 25W load (fanless)
```

### NAS "Thoth"
```
Modelo:     UGREEN DH2300 (2-bay)
Capacidad:  3.6 TB usable (BTRFS)
Protocolo:  NFS v3
Función:    Backups y datos persistentes
```

### PC Cliente (Simulación)
```
SO:         Windows 11
Software:   VMware Workstation Pro/Player
Rol:        Simula sede cliente remota
```

---

## 📊 Tabla de Direccionamiento IP Completa

### Red WAN — vmbr0 (192.168.0.0/24)

| Dispositivo | IP | Gateway | Rol |
|-------------|-------|---------|-----|
| Router físico | `192.168.0.1` | — | Salida internet |
| Proxmox "ODIN" | `192.168.0.100` | 192.168.0.1 | Hipervisor |
| NAS "Thoth" | `192.168.0.101` | 192.168.0.1 | Almacenamiento NFS |
| **FortiGate Server port1** | `192.168.0.102` | 192.168.0.1 | WAN firewall MSP |
| **FortiGate Cliente port1** | `192.168.0.103` | 192.168.0.1 | WAN firewall cliente |
| LXC-104 eth0 | `192.168.0.114` | 192.168.0.1 | Management Zabbix |

### Red LAN MSP — vmbr1 (10.0.1.0/24)

| Dispositivo | IP | Gateway | Rol |
|-------------|-------|---------|-----|
| Proxmox bridge | `10.0.1.1` | — | Bridge virtual |
| LXC-104 eth1 | `10.0.1.10` | 10.0.1.254 | Zabbix monitored interface |
| **FortiGate Server port2** | `10.0.1.254` | — | Gateway LAN MSP |

### Red LAN Cliente — VMnet2 (10.0.2.0/24)

| Dispositivo | IP | Gateway | Rol |
|-------------|-------|---------|-----|
| **Ubuntu VM** | `10.0.2.100` | 10.0.2.254 | Host monitorizado (Zabbix Agent) |
| **FortiGate Cliente port2** | `10.0.2.254` | — | Gateway LAN cliente |

---

## 🚀 Cómo Replicar Este Proyecto

### Prerrequisitos

1. **Hardware mínimo:**
   - Servidor con Proxmox VE instalado (16GB RAM recomendado)
   - PC con VMware Workstation (8GB RAM libres)
   - Conexión de red estable

2. **Cuentas necesarias:**
   - Cuenta gratuita en [Fortinet Support Portal](https://support.fortinet.com) (para licencias eval)
   - *(Opcional)* Cuenta Cloudflare para túnel externo

3. **Descargas:**
   - FortiGate VM KVM para Proxmox (v7.6.6+)
   - FortiGate VM OVF para VMware (v7.6.6+)
   - Plantillas LXC Ubuntu 22.04

### Orden de despliegue recomendado

```
Día 1-2: Infraestructura Base
├─ 1. Crear vmbr1 en Proxmox
├─ 2. Desplegar FortiGate Server (VM-107)
├─ 3. Configurar LXC-104 dual-homed
└─ 4. Desplegar stack Docker Zabbix

Día 3-4: Lado Cliente
├─ 5. Crear VMnet2 en VMware
├─ 6. Desplegar FortiGate Cliente
├─ 7. Crear VM Ubuntu en VMnet2
└─ 8. Configurar red cliente

Día 5: VPN Site-to-Site
├─ 9. Configurar Phase 1 en ambos FortiGates
├─ 10. Configurar Phase 2 en ambos FortiGates
├─ 11. Crear políticas firewall VPN
├─ 12. Verificar túnel UP
└─ 13. Probar conectividad cross-VPN

Día 6: Monitorización
├─ 14. Instalar Zabbix Agent en Ubuntu
├─ 15. Agregar host en Zabbix Server
├─ 16. Configurar templates
└─ 17. Crear dashboards
```

Sigue las guías en orden numérico en la carpeta de documentación.

---

## 🔐 Credenciales del Laboratorio

> ⚠️ **Nota de seguridad:** Este es un entorno de laboratorio aislado. En producción, usa contraseñas únicas y rotación periódica.

| Servicio | Usuario | Contraseña | Acceso |
|----------|---------|------------|--------|
| **Proxmox VE** | `root` | *(tu password)* | https://192.168.0.100:8006 |
| **FortiGate Server** | `admin` | `Virtus2026!CS` | https://192.168.0.102:8443 |
| **FortiGate Cliente** | `admin` | `Virtus2026!CS` | https://192.168.0.103 |
| **Zabbix Web** | `Admin` | *(cambiar tras login)* | http://192.168.0.114:8080 |
| **Zabbix DB** | `zabbix` | *(ver `.env`)* | PostgreSQL interno |
| **Ubuntu VM** | `pablo` | *(tu password)* | Consola VMware |

---

## 🎓 Conceptos Aprendidos

Este proyecto enseña habilidades de nivel profesional MSP:

### Networking
- ✅ VPN IPsec site-to-site con FortiGate
- ✅ Dual-homed hosts con policy routing
- ✅ Split-tunneling (tráfico selectivo por VPN)
- ✅ NAT/PAT en firewalls empresariales
- ✅ VLAN segmentation (vmbr0 vs vmbr1)

### Virtualización
- ✅ KVM en Proxmox (VM FortiGate)
- ✅ LXC containers (Zabbix Server)
- ✅ VMware Workstation (sede cliente)
- ✅ Network bridges virtuales

### Monitorización
- ✅ Zabbix Server + Agent arquitectura
- ✅ PostgreSQL tuning para monitorización
- ✅ Docker Compose orquestación
- ✅ Housekeeping y retención de datos

### Troubleshooting
- ✅ Routing debugging (ip route, traceroute)
- ✅ VPN diagnostics (diagnose vpn tunnel list)
- ✅ Firewall policy verification
- ✅ Análisis de logs en tiempo real

---

## 🐛 Problemas Conocidos y Soluciones

### Licencia FortiGate eval limitada
**Problema:** Una licencia eval por cuenta Fortinet.  
**Solución aplicada:** Segunda cuenta con email diferente + contacto con Fortinet España.

### Typo en configuración red LXC (10.1.0.254)
**Problema:** Gateway mal escrito causó routing incorrecto.  
**Solución:** Corrección en `/etc/pve/lxc/104.conf` y reinicio del container.

### FortiGate GUI puerto 443 conflicto con VPN
**Problema:** IKE TCP y HTTPS usan mismo puerto.  
**Solución:** Cambiar GUI a puerto 8443 en FortiGate Server.

Ver [06-Troubleshooting.md](06-Troubleshooting.md) para lista completa.

---

## 📈 Próximos Pasos

### Corto plazo
- [ ] Instalar y configurar Zabbix Agent 2 en Ubuntu cliente
- [ ] Agregar host en Zabbix Server y verificar métricas
- [ ] Configurar plantillas para monitorización Linux
- [ ] Crear triggers y alertas (Telegram/Email)

### Mediano plazo
- [ ] SNMP monitoring de los propios FortiGates
- [ ] Zabbix Proxy en red cliente (arquitectura distribuida)
- [ ] Dashboards tipo MSP profesional
- [ ] Automatización con Ansible

### Largo plazo
- [ ] Simular segundo cliente (tercer FortiGate)
- [ ] VPN hub-and-spoke (ADVPN)
- [ ] FortiAnalyzer para logs centralizados
- [ ] Integración con ticketing system

---

## 📝 Licencias y Disclaimer

- **FortiGate eval:** 1 vCPU, 2GB RAM, funcionalidad completa limitada a 15 días
- **Zabbix:** GNU GPL v2 (open source)
- **Proxmox VE:** AGPL v3 (community edition)
- **Este proyecto:** Uso educativo y no comercial

> ⚠️ **Disclaimer:** Este es un laboratorio educativo. No usar en producción sin las licencias comerciales apropiadas y hardening de seguridad.

---

## 🙏 Agradecimientos

- **Virtus Informática** — Inspiración del proyecto y oportunidad de prácticas profesionales
- **Fortinet** — Licencias eval y documentación técnica
- **Comunidades:** r/Proxmox, r/homelab, r/zabbix, FortiGate NSE forums

---

## 📞 Contacto

**Proyecto creado por:** Nat  
**Contexto:** Preparación para prácticas en Virtus Informática (marzo 2025)  
**Duración desarrollo:** 80 días (roadmap "Hoja de Ruta 80 Días")  

---

**🔗 Enlaces rápidos:**
- [Guía FortiGate Server](01-FortiGate-Server-Setup.md)
- [Guía FortiGate Cliente](02-FortiGate-Cliente-Setup.md)
- [Configuración VPN](04-VPN-IPsec-Tunnel.md)
- [Troubleshooting](06-Troubleshooting.md)

---

*Última actualización: Febrero 2026*
