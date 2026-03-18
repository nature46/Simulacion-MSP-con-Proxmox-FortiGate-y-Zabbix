# Ubuntu Cliente — Configuración VM en Red Cliente (VMnet2)

## Guía completa para desplegar el host cliente con Zabbix Agent en la red 10.0.2.0/24

---

## Contexto

Esta VM Ubuntu simula un **PC o servidor en la oficina de un cliente remoto** del MSP. Está conectada a la red `10.0.2.0/24` (VMnet2), que es la LAN interna del lado cliente del túnel VPN.

```
PC Windows (VMware Workstation)
├── FortiGate Cliente
│   ├── port1 (WAN): NAT VMware → acceso a Internet
│   └── port2 (LAN): VMnet2 → red cliente 10.0.2.0/24
└── Ubuntu VM (este documento)
    ├── ens33: 10.0.2.100/24
    ├── Gateway: 10.0.2.254 (FortiGate Cliente)
    └── Zabbix Agent 2 → reporta a 10.0.1.10 (Zabbix Server vía VPN)
```

**Flujo de datos de monitorización:**
```
Ubuntu (10.0.2.100)
  └─► Zabbix Agent envía métricas a 10.0.1.10:10051
       └─► FortiGate Cliente → cifra con IPsec → VPN tunnel
            └─► FortiGate Server → descifra
                 └─► LXC-104 eth1 (10.0.1.10) → Zabbix Server ✅
```

---

## Requisitos Previos

- VMware Workstation instalado en el PC
- VMnet2 configurada como Host-only (10.0.2.0/24, sin DHCP)
- FortiGate Cliente operativo con port2 en VMnet2 (IP 10.0.2.254)
- Túnel VPN IPsec activo entre ambos FortiGates
- Ubuntu 22.04 LTS disponible (ISO o imagen existente)

> **Verificación previa:** Antes de empezar, confirma que el túnel VPN está UP:
> ```bash
> ssh admin@192.168.0.103
> diagnose vpn tunnel list name VPN-Virtus-CS
> # Debe mostrar: status=up
> ```

---

## Paso 1 — Crear la VM Ubuntu en VMware

> Si ya tienes una Ubuntu instalada, ve directamente al Paso 2.

### Crear nueva VM

**Archivo → Nueva máquina virtual → Típica:**

| Campo | Valor |
|---|---|
| Instalación | ISO de Ubuntu 22.04 LTS |
| Nombre | `Ubuntu-Cliente-Castellon` |
| Disco | 20 GB (almacenamiento único) |
| RAM | 2 GB |
| CPU | 1 core |

### Configurar adaptador de red

**Ajustes de la VM → Adaptador de red:**

```
Conexión de red: Personalizada: red virtual específica
└─ Seleccionar: VMnet2
```

> ⚠️ **Importante:** NO usar modo Puente ni NAT. Debe ser VMnet2 (Host-only) para que el tráfico pase por el FortiGate Cliente y luego por la VPN.

---

## Paso 2 — Configurar Red Estática (Netplan)

### Identificar la interfaz de red

```bash
ip addr show
```

Busca la interfaz que NO sea `lo`. Normalmente es `ens33` en VMware.

```
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP>
    link/ether 00:0c:29:xx:xx:xx
```

### Editar configuración Netplan

```bash
sudo nano /etc/netplan/01-network-manager-all.yaml
```

**Reemplazar TODO el contenido con:**

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 10.0.2.100/24
      routes:
        - to: default
          via: 10.0.2.254
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

> **⚠️ Importante — renderer: networkd**
> Usar `networkd` en vez de `NetworkManager`. Si se deja `NetworkManager`, puede ignorar la configuración estática y resetear la interfaz periódicamente, causando pérdidas de red intermitentes cada pocos minutos.

### Aplicar configuración

```bash
# Corregir permisos (evita warnings de Netplan)
sudo chmod 600 /etc/netplan/01-network-manager-all.yaml

# Aplicar cambios
sudo netplan apply
```

### Verificar configuración

```bash
ip addr show ens33
ip route show
```

**Salida esperada:**

```
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 10.0.2.100/24 brd 10.0.2.255 scope global noprefixroute ens33

default via 10.0.2.254 dev ens33 proto static metric 20100
10.0.2.0/24 dev ens33 proto kernel scope link src 10.0.2.100 metric 100
```

---

## Paso 3 — Verificar Conectividad

### Test de red local (FortiGate Cliente LAN)

```bash
ping -c 3 10.0.2.254
```

**Esperado:** 0% packet loss — el FortiGate Cliente responde.

### Test de Internet (vía NAT del FortiGate Cliente)

```bash
ping -c 3 8.8.8.8
ping -c 3 google.com
```

**Esperado:** respuesta correcta — el FortiGate hace NAT para el tráfico de Internet.

### Test de conectividad por VPN (Zabbix Server)

```bash
ping -c 3 10.0.1.10
```

**Esperado:** respuesta correcta — el tráfico va por la VPN hasta el Zabbix Server.

> Si este ping falla, ver sección de Troubleshooting al final del documento.

---

## Paso 4 — Instalar Zabbix Agent 2

### Actualizar sistema

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl wget net-tools
```

### Añadir repositorio oficial Zabbix

> **⚠️ Ajusta la versión de Ubuntu si es necesario.** Este comando es para Ubuntu 24.04. Para Ubuntu 22.04, cambia `ubuntu24.04` por `ubuntu22.04`.

```bash
# Para Ubuntu 24.04:
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest+ubuntu24.04_all.deb

# Para Ubuntu 22.04:
# wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest+ubuntu22.04_all.deb

sudo dpkg -i zabbix-release_latest+ubuntu*.deb
sudo apt update
```

### Instalar Zabbix Agent 2

```bash
sudo apt install -y zabbix-agent2
```

### Verificar instalación

```bash
zabbix_agent2 --version
```

Salida esperada:
```
zabbix_agent2 (Zabbix) 7.0.x
```

---

## Paso 5 — Configurar Zabbix Agent 2

### Editar archivo de configuración

```bash
sudo nano /etc/zabbix/zabbix_agent2.conf
```

**Buscar y modificar exactamente estas tres líneas:**

```ini
# IP del Zabbix Server (monitorización pasiva - el server consulta al agente)
Server=10.0.1.10

# IP del Zabbix Server (monitorización activa - el agente envía datos)
ServerActive=10.0.1.10

# Nombre del host (debe coincidir EXACTAMENTE con el nombre en la GUI de Zabbix)
Hostname=Ubuntu-Cliente-Castellon
```

> **Nota sobre Server vs ServerActive:**
> - `Server`: el Zabbix Server se conecta al agente en el puerto 10050 para consultar métricas (modo pasivo)
> - `ServerActive`: el agente se conecta al Zabbix Server en el puerto 10051 para enviar métricas (modo activo)
> Configura ambos para máxima compatibilidad con cualquier template.

### Activar e iniciar el servicio

```bash
sudo systemctl enable zabbix-agent2
sudo systemctl restart zabbix-agent2
```

### Verificar estado del servicio

```bash
sudo systemctl status zabbix-agent2
```

**Salida esperada:**

```
● zabbix-agent2.service - Zabbix Agent 2
     Loaded: loaded (/usr/lib/systemd/system/zabbix-agent2.service; enabled)
     Active: active (running) since ...
             └─ /usr/sbin/zabbix_agent2 -c /etc/zabbix/zabbix_agent2.conf
             Zabbix Agent2 hostname: [Ubuntu-Cliente-Castellon]
```

> El hostname `[Ubuntu-Cliente-Castellon]` en los logs confirma que la configuración se leyó correctamente.

---

## Paso 6 — Añadir Host en Zabbix Server (GUI)

### Acceder a Zabbix Web

```
http://192.168.0.114:8080
```

O si tienes Cloudflare configurado: `https://zabbix.tudominio.com`

Login: `Admin` / `<tu_password>`

### Crear el host

**Data collection → Hosts → Create host**

Rellenar los campos:

| Campo | Valor |
|---|---|
| **Host name** | `Ubuntu-Cliente-Castellon` |
| **Visible name** | Ubuntu Cliente (Castellón) |
| **Templates** | `Linux by Zabbix agent` |
| **Host groups** | `Linux servers` |

En la sección **Interfaces → Add → Agent:**

| Campo | Valor |
|---|---|
| **IP address** | `10.0.2.100` |
| **Port** | `10050` |

Click **Add** para guardar el host.

> **⚠️ El campo "Host name" debe coincidir EXACTAMENTE con el `Hostname` del archivo `zabbix_agent2.conf`** — incluyendo mayúsculas, minúsculas y guiones.

### Verificar que el host aparece como disponible

Ir a **Data collection → Hosts** y esperar 1-2 minutos.

El indicador **ZBX** debe aparecer en **verde** ✅

Si aparece en rojo, ver sección Troubleshooting.

---

## Paso 7 — Verificar Métricas en Zabbix

### Ver datos en tiempo real

**Monitoring → Latest data**

- Filtrar por: **Hosts** = `Ubuntu-Cliente-Castellon`
- Debe aparecer una lista de métricas con valores recientes:
  - CPU utilization
  - Memory utilization
  - Filesystem usage
  - Network traffic

### Ver gráficas

**Monitoring → Hosts → Ubuntu-Cliente-Castellon → Graphs**

Selecciona cualquier métrica para ver su gráfica histórica.

---

## Verificación Final del Escenario Completo

Ejecutar desde LXC-104 (Zabbix Server) para confirmar conectividad end-to-end:

```bash
pct enter 104

# Ping al FortiGate Cliente (extremo remoto del túnel)
ping -c 3 10.0.2.254

# Ping al Ubuntu Cliente (host monitorizado)
ping -c 3 10.0.2.100

# Test directo del agente Zabbix
zabbix_get -s 10.0.2.100 -k agent.ping
```

**Salida esperada del último comando:**
```
1
```

El valor `1` confirma que el Zabbix Server puede consultar al agente a través de la VPN cifrada.

---

## Troubleshooting

### Ubuntu pierde red cada pocos minutos

**Síntoma:** `ping 8.8.8.8` falla intermitentemente. Ejecutar `sudo netplan apply` lo arregla temporalmente.

**Causa:** El renderer `NetworkManager` está en conflicto con la configuración estática de Netplan. NetworkManager gestiona la interfaz y la resetea periódicamente.

**Solución:** Cambiar a renderer `networkd` en el archivo netplan:

```yaml
network:
  version: 2
  renderer: networkd   # ← Cambiar NetworkManager por networkd
  ethernets:
    ens33:
      ...
```

```bash
sudo chmod 600 /etc/netplan/01-network-manager-all.yaml
sudo netplan apply
```

---

### Ping a 10.0.1.10 falla (La red es inaccesible)

**Síntoma:** `ping 10.0.1.10` devuelve `connect: La red es inaccesible`

**Verificación 1 — ¿La interfaz tiene IP?**

```bash
ip addr show ens33
```

Si no tiene IP: `sudo netplan apply`

**Verificación 2 — ¿El FortiGate Cliente está encendido?**

El gateway `10.0.2.254` debe estar disponible antes de intentar rutas más lejanas:

```bash
ping -c 3 10.0.2.254
```

Si no responde → encender la VM del FortiGate Cliente en VMware y esperar 30 segundos.

**Verificación 3 — ¿El túnel VPN está activo?**

```bash
ssh admin@192.168.0.103
diagnose vpn tunnel list name VPN-Virtus-CS
# status=up significa túnel activo
```

**Verificación 4 — ¿LXC-104 tiene la ruta de vuelta?**

El LXC-104 necesita saber cómo llegar a `10.0.2.0/24`. Esta ruta se añade automáticamente via hookscript al arrancar. Verificar:

```bash
# Desde el host Proxmox
pct exec 104 -- ip route show | grep 10.0.2
# Debe mostrar: 10.0.2.0/24 via 10.0.1.254 dev eth1
```

Si no aparece, añadir manualmente:

```bash
lxc-attach -n 104 -- ip route add 10.0.2.0/24 via 10.0.1.254 dev eth1
```

---

### Zabbix Agent no conecta — ZBX rojo en la GUI

**Verificación 1 — Estado del servicio:**

```bash
sudo systemctl status zabbix-agent2
sudo tail -20 /var/log/zabbix/zabbix_agent2.log
```

**Verificación 2 — Hostname correcto:**

```bash
grep "^Hostname=" /etc/zabbix/zabbix_agent2.conf
# Debe coincidir EXACTAMENTE con el hostname en la GUI de Zabbix
```

**Verificación 3 — Firewall local:**

```bash
sudo ufw status
# Si está activo y bloqueando el puerto 10050:
sudo ufw allow 10050/tcp
```

**Verificación 4 — Puerto accesible desde el servidor:**

```bash
# Desde LXC-104
zabbix_get -s 10.0.2.100 -k agent.ping
# Debe devolver: 1
```

---

## Resumen de Configuración

```
VM Ubuntu: Ubuntu-Cliente-Castellon
├── OS: Ubuntu 22.04/24.04 LTS
├── Adaptador VMware: VMnet2 (Host-only, 10.0.2.0/24)
├── IP: 10.0.2.100/24
├── Gateway: 10.0.2.254 (FortiGate Cliente)
├── DNS: 8.8.8.8 / 1.1.1.1
├── Netplan renderer: networkd (estable, sin pérdidas)
└── Zabbix Agent 2
    ├── Server: 10.0.1.10
    ├── ServerActive: 10.0.1.10
    └── Hostname: Ubuntu-Cliente-Castellon
```

---

## Siguiente Paso

Con el agente funcionando y reportando métricas, el siguiente paso es configurar **alertas y triggers** en Zabbix para ser notificado cuando el cliente tenga problemas:

- Triggers de CPU > 80%
- Triggers de disco > 90%
- Alertas por Telegram o Email
- Dashboard tipo MSP con visión global de todos los clientes
