# 🔒 Túnel VPN IPsec Site-to-Site — FortiGate ↔ FortiGate

## Configuración completa del túnel VPN entre sede MSP y cliente remoto

---

## Contexto

Este documento cubre la configuración **completa por CLI** del túnel VPN IPsec site-to-site entre:

- **FortiGate Server** (Proxmox VM-107): `192.168.0.102`
- **FortiGate Cliente** (VMware): `192.168.0.103`

El túnel conecta las redes privadas:
- **MSP Virtus:** `10.0.1.0/24` (Zabbix Server, servicios centralizados)
- **Cliente remoto:** `10.0.2.0/24` (hosts monitorizados)

---

## Arquitectura del Túnel

```
FORTIGATE SERVER (MSP)                    FORTIGATE CLIENTE (Remoto)
┌──────────────────────┐                 ┌──────────────────────┐
│ port1 (WAN)          │                 │ port1 (WAN)          │
│ 192.168.0.102        │◄────────────────┤ 192.168.0.103        │
│                      │   VPN IPsec     │                      │
│ port2 (LAN)          │   Phase 1+2     │ port2 (LAN)          │
│ 10.0.1.254           │◄────────────────┤ 10.0.2.254           │
│   │                  │                 │   │                  │
│   └─ 10.0.1.0/24     │                 │   └─ 10.0.2.0/24     │
│      (Zabbix Server) │                 │      (Zabbix Agent)  │
└──────────────────────┘                 └──────────────────────┘
```

---

## Parámetros del Túnel

### Phase 1 (IKE)

| Parámetro | Valor |
|-----------|-------|
| **Nombre** | `VPN-Virtus-CS` |
| **IKE Version** | 2 |
| **Interface** | port1 (ambos lados) |
| **Remote Gateway** | Server: `192.168.0.103` / Cliente: `192.168.0.102` |
| **Authentication** | Pre-Shared Key: `Virtus2026!CS` |
| **Mode** | Main (ID protection) |
| **DPD** | On-demand |
| **NAT-T** | Enable |

### Phase 2 (IPsec)

| Parámetro | Valor |
|-----------|-------|
| **Phase1 name** | `VPN-Virtus-CS` |
| **Encryption** | Por defecto (AES256) |
| **Authentication** | Por defecto (SHA256) |
| **PFS** | Enable |
| **Subnets Server** | Local: `10.0.1.0/24` → Remote: `10.0.2.0/24` |
| **Subnets Cliente** | Local: `10.0.2.0/24` → Remote: `10.0.1.0/24` |

---

## Configuración FortiGate Server (Proxmox VM-107)

### Acceso SSH

```bash
ssh admin@192.168.0.102
# Password: Virtus2026!CS
```

### 1. Phase 1 — IKE

```bash
config vpn ipsec phase1-interface
    edit "VPN-Virtus-CS"
        set interface "port1"
        set ike-version 2
        set peertype any
        set net-device disable
        set remote-gw 192.168.0.103
        set psksecret Virtus2026!CS
    next
end
```

**Explicación parámetros:**
- `interface "port1"`: Interfaz por donde sale el túnel (WAN)
- `ike-version 2`: Versión moderna de IKE (más segura)
- `peertype any`: Acepta cualquier tipo de peer (no require identificación específica)
- `net-device disable`: No crear interfaz de red virtual redundante
- `remote-gw`: IP del otro extremo del túnel
- `psksecret`: Clave compartida (debe ser idéntica en ambos lados)

### 2. Phase 2 — IPsec

```bash
config vpn ipsec phase2-interface
    edit "VPN-Virtus-CS"
        set phase1name "VPN-Virtus-CS"
        set src-subnet 10.0.1.0 255.255.255.0
        set dst-subnet 10.0.2.0 255.255.255.0
    next
end
```

**Explicación parámetros:**
- `phase1name`: Vincula Phase 2 con el Phase 1 creado
- `src-subnet`: Red local que se expondrá por el túnel (MSP)
- `dst-subnet`: Red remota a la que se quiere acceder (Cliente)

### 3. Objetos de Dirección (Address Objects)

```bash
config firewall address
    edit "VPN-CS-Local"
        set subnet 10.0.1.0 255.255.255.0
    next
    edit "VPN-CS-Remote"
        set subnet 10.0.2.0 255.255.255.0
    next
end
```

### 4. Política Firewall — Local → Remote

```bash
config firewall policy
    edit 0
        set name "VPN-Local-to-Remote"
        set srcintf "port2"
        set dstintf "VPN-Virtus-CS"
        set srcaddr "VPN-CS-Local"
        set dstaddr "VPN-CS-Remote"
        set action accept
        set schedule "always"
        set service "ALL"
    next
end
```

**Explicación:**
- `srcintf "port2"`: Tráfico que viene desde la LAN MSP
- `dstintf "VPN-Virtus-CS"`: Tráfico que sale por el túnel VPN
- `action accept`: Permitir el tráfico
- `service "ALL"`: Todos los puertos/protocolos (ajustar en producción)

### 5. Política Firewall — Remote → Local

```bash
config firewall policy
    edit 0
        set name "VPN-Remote-to-Local"
        set srcintf "VPN-Virtus-CS"
        set dstintf "port2"
        set srcaddr "VPN-CS-Remote"
        set dstaddr "VPN-CS-Local"
        set action accept
        set schedule "always"
        set service "ALL"
    next
end
```

### 6. Ruta Estática

```bash
config router static
    edit 0
        set dst 10.0.2.0 255.255.255.0
        set device "VPN-Virtus-CS"
    next
end
```

**Explicación:**
- Le dice al FortiGate: "Para llegar a `10.0.2.0/24`, usa el túnel VPN"
- Esta ruta solo se activa cuando el túnel está UP

---

## Configuración FortiGate Cliente (VMware)

### Acceso SSH

```bash
ssh admin@192.168.0.103
# Password: Virtus2026!CS
```

### 1. Phase 1 — IKE (remote-gw invertido)

```bash
config vpn ipsec phase1-interface
    edit "VPN-Virtus-CS"
        set interface "port1"
        set ike-version 2
        set peertype any
        set net-device disable
        set remote-gw 192.168.0.102
        set psksecret Virtus2026!CS
    next
end
```

⚠️ **Diferencia clave:** `remote-gw 192.168.0.102` apunta al Server

### 2. Phase 2 — IPsec (subnets invertidas)

```bash
config vpn ipsec phase2-interface
    edit "VPN-Virtus-CS"
        set phase1name "VPN-Virtus-CS"
        set src-subnet 10.0.2.0 255.255.255.0
        set dst-subnet 10.0.1.0 255.255.255.0
    next
end
```

⚠️ **Diferencia clave:** src/dst invertidos respecto al Server

### 3. Objetos de Dirección

```bash
config firewall address
    edit "VPN-CS-Local"
        set subnet 10.0.2.0 255.255.255.0
    next
    edit "VPN-CS-Remote"
        set subnet 10.0.1.0 255.255.255.0
    next
end
```

### 4. Política Firewall — Local → Remote

```bash
config firewall policy
    edit 0
        set name "VPN-Local-to-Remote"
        set srcintf "port2"
        set dstintf "VPN-Virtus-CS"
        set srcaddr "VPN-CS-Local"
        set dstaddr "VPN-CS-Remote"
        set action accept
        set schedule "always"
        set service "ALL"
    next
end
```

### 5. Política Firewall — Remote → Local

```bash
config firewall policy
    edit 0
        set name "VPN-Remote-to-Local"
        set srcintf "VPN-Virtus-CS"
        set dstintf "port2"
        set srcaddr "VPN-CS-Remote"
        set dstaddr "VPN-CS-Local"
        set action accept
        set schedule "always"
        set service "ALL"
    next
end
```

### 6. Ruta Estática

```bash
config router static
    edit 0
        set dst 10.0.1.0 255.255.255.0
        set device "VPN-Virtus-CS"
    next
end
```

---

## Verificación del Túnel

### Estado del Túnel VPN

**En ambos FortiGates:**

```bash
diagnose vpn tunnel list
```

**Salida esperada (túnel UP):**

```
name=VPN-Virtus-CS ver=2 serial=1
...
status=up
proxyid_num=1 child_num=0 refcnt=4
...
```

Buscar la línea `status=up` — confirma que el túnel está establecido.

### Estado IKE Gateway

```bash
diagnose vpn ike gateway list
```

**Salida esperada:**

```
vd: root/0
name: VPN-Virtus-CS
...
IKE SA: created 1/1  established 1/1
IPsec SA: created 1/1  established 1/1
```

`established 1/1` en ambas líneas indica éxito completo.

### Tabla de Rutas

```bash
get router info routing-table all
```

**En FortiGate Server debe aparecer:**

```
S       10.0.2.0/24 [10/0] via VPN-Virtus-CS tunnel 192.168.0.103
```

**En FortiGate Cliente debe aparecer:**

```
S       10.0.1.0/24 [10/0] via VPN-Virtus-CS tunnel 192.168.0.102
```

---

## Test de Conectividad Cross-VPN

### Desde LXC-104 (Zabbix Server)

```bash
ping -c 3 10.0.2.254
# Debe responder — ping al FortiGate Cliente LAN
```

```bash
ping -c 3 10.0.2.100
# Debe responder — ping al Ubuntu cliente
```

### Desde Ubuntu Cliente (10.0.2.100)

```bash
ping -c 3 10.0.1.254
# Debe responder — ping al FortiGate Server LAN
```

```bash
ping -c 3 10.0.1.10
# Debe responder — ping al Zabbix Server
```

Si los 4 pings funcionan, **el túnel VPN está plenamente operativo**.

---

## Logs y Troubleshooting VPN

### Ver logs en tiempo real

```bash
diagnose debug application ike -1
diagnose debug enable
```

Esto muestra el proceso de negociación IKE en vivo. Útil para detectar:
- Errores de autenticación (PSK incorrecta)
- Problemas de propuestas de cifrado incompatibles
- Timeouts de conectividad

Para detener:

```bash
diagnose debug disable
```

### Ver sesiones activas

```bash
get system session list
```

Filtra por IP de la red remota para ver tráfico atravesando el túnel.

### Ver estadísticas del túnel

```bash
diagnose vpn tunnel list | grep -A 20 VPN-Virtus-CS
```

Muestra contadores de paquetes cifrados/descifrados.

---

## Configuración de Puertos GUI (solo FortiGate Server)

**Problema original:** Puerto 443 (HTTPS) conflictúa con IKE TCP del túnel VPN.

**Solución aplicada:**

```bash
config system global
    set admin-sport 8443
end
```

Ahora la GUI del FortiGate Server es accesible en:

```
https://192.168.0.102:8443
```

> **Nota:** El FortiGate Cliente sigue en puerto 443 estándar.

---

## Split-Tunneling — Cómo Funciona

El diseño implementa **split-tunneling profesional**:

```
TRÁFICO DESDE UBUNTU CLIENTE (10.0.2.100):

Destino: 10.0.1.10 (Zabbix Server)
  └─► FortiGate revisa tabla de rutas
      └─► Coincide con ruta estática 10.0.1.0/24 → VPN
          └─► Tráfico va por VPN-Virtus-CS (CIFRADO)

Destino: 8.8.8.8 (Google DNS)
  └─► FortiGate revisa tabla de rutas
      └─► Coincide con default route → port1
          └─► Tráfico sale a internet directo (NAT, SIN CIFRAR)
```

**Ventajas:**
- Solo el tráfico MSP va cifrado (Zabbix, gestión)
- Internet normal va directo (rápido, no sobrecarga túnel)
- Transparente para el usuario final

---

## Monitorización del Túnel con Zabbix

Una vez instalado Zabbix Agent en el Ubuntu cliente, puedes monitorizar:

### Métricas recomendadas

- **Latencia del túnel:** RTT entre `10.0.1.10` ↔ `10.0.2.100`
- **Estado del túnel:** Script que hace `diagnose vpn tunnel list` y verifica `status=up`
- **Throughput:** Bytes cifrados/segundo en el túnel
- **Packet loss:** Pings periódicos entre extremos

### Template personalizado (futuro)

Crear un template Zabbix específico para "FortiGate VPN Tunnel" que incluya estos items.

---

## Escenario de Producción Real

### Diferencias con el laboratorio

| Aspecto | Laboratorio | Producción Real |
|---------|-------------|-----------------|
| **IPs WAN** | 192.168.0.102 / .103 (LAN) | IPs públicas reales |
| **Port forwarding** | No necesario (misma LAN) | UDP 500 y 4500 en routers |
| **DDNS** | No usado | Común si IP dinámica |
| **Certificados** | Autofirmados | CA privada o Let's Encrypt |
| **Monitoring** | Básico | SNMP + FortiAnalyzer |
| **Redundancia** | Túnel único | Dual-WAN + failover |

### Puertos a abrir en routers (producción)

```
Router MSP:
  UDP 500  → FortiGate Server (IKE)
  UDP 4500 → FortiGate Server (NAT-T)

Router Cliente:
  UDP 500  → FortiGate Cliente (IKE)
  UDP 4500 → FortiGate Cliente (NAT-T)
```

---

## Backup de Configuración VPN

### Backup por CLI

**FortiGate Server:**

```bash
execute backup config ftp backup-vpn-config.conf <ftp_server> <user> <password>
```

O exportar desde GUI:
```
System → Configuration → Backup → Download
```

### Restore rápido

Si necesitas recrear el túnel, copia-pega los comandos de este documento en orden.

---

## Mejoras Futuras

### Corto plazo
- [ ] Habilitar logs detallados de VPN
- [ ] Crear dashboard Zabbix para estado túnel
- [ ] Documentar procedimiento de renovación PSK

### Mediano plazo
- [ ] Implementar VPN con certificados (PKI)
- [ ] Configurar BGP over VPN para routing dinámico
- [ ] Añadir segundo túnel (redundancia)

### Largo plazo
- [ ] ADVPN (Auto Discovery VPN) para multi-site
- [ ] SD-WAN con path selection inteligente
- [ ] FortiAnalyzer para análisis de tráfico VPN

---

## Comandos de Referencia Rápida

```bash
# Estado general VPN
diagnose vpn tunnel list
diagnose vpn ike gateway list

# Ver políticas firewall
show firewall policy | grep VPN

# Ver rutas estáticas
get router info routing-table static

# Test conectividad
execute ping-options source 10.0.1.254
execute ping 10.0.2.254

# Restart VPN (si hay problemas)
diagnose vpn ike gateway clear
```

---

## Resolución de Problemas Comunes

### Túnel no sube (status=down)

1. **Verificar PSK idéntica** en ambos lados
2. **Verificar IPs remote-gw** correctas
3. **Verificar conectividad básica:** `execute ping <IP_remota>`
4. **Revisar logs:** `diagnose debug application ike -1`

### Túnel sube pero no pasa tráfico

1. **Verificar políticas firewall** existen y están enabled
2. **Verificar rutas estáticas** configuradas
3. **Verificar subnets Phase 2** coinciden (src ↔ dst invertidos)
4. **Check NAT no interfiere** con tráfico VPN

### Rendimiento bajo del túnel

1. **Verificar MTU:** Puede necesitar ajuste (1400 típico para VPN)
2. **Revisar cifrado:** Usar AES-NI si disponible
3. **Check latencia base:** `execute ping` sin VPN primero

---

**Documentos relacionados:**
- [← FortiGate Server Setup](01-FortiGate-Server-Setup.md)
- [← FortiGate Cliente Setup](02-FortiGate-Cliente-Setup.md)
- [→ Ubuntu Cliente Setup](05-Ubuntu-Cliente-Setup.md)
- [→ Troubleshooting](06-Troubleshooting.md)

---

*Última actualización: Febrero 2025*
