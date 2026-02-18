# 🖥️ Ubuntu Cliente — Configuración en VMware

## VM Ubuntu simulando host monitorizado en sede cliente remota

---

## Contexto

Esta VM Ubuntu simula un **servidor o estación de trabajo** en la oficina de un cliente del MSP. Se conecta a la red interna del cliente (`10.0.2.0/24`) y su tráfico:

- **Tráfico MSP (Zabbix)** → Sale por VPN cifrada hacia `10.0.1.0/24`
- **Tráfico internet normal** → Sale directo por FortiGate Cliente con NAT

Este es el **split-tunneling** profesional que implementan los MSPs reales.

---

## Arquitectura

```
UBUNTU VM (10.0.2.100)
    │
    └─► VMnet2 (red interna cliente)
         │
         └─► FortiGate Cliente port2 (10.0.2.254)
              │
              ├─► Destino 10.0.1.0/24 → VPN túnel cifrado
              └─► Resto destinos → Internet directo (NAT port1)
```

---

## Requisitos Previos

- VMware Workstation instalado en tu PC Windows
- VMnet2 ya creada (ver [02-FortiGate-Cliente-Setup.md](02-FortiGate-Cliente-Setup.md))
- FortiGate Cliente operativo con política NAT activa
- Ubuntu Desktop 22.04 ISO descargada

---

## Paso 1 — Crear VM en VMware

### Configuración VM

**File → New Virtual Machine:**

| Parámetro | Valor |
|-----------|-------|
| **Configuration** | Typical |
| **Installer disc image** | Ubuntu 22.04 Desktop ISO |
| **Full name** | pablo (o tu usuario) |
| **Username** | pablo |
| **Password** | *(tu contraseña)* |
| **VM name** | Ubuntu-Cliente-Castellon |
| **Location** | Directorio local |
| **Disk size** | 25 GB (single file) |

**Finalizar creación** pero **no arrancar aún**.

---

## Paso 2 — Configurar Adaptador de Red

**VM Settings → Hardware → Network Adapter:**

| Configuración | Valor |
|---------------|-------|
| **Network connection** | Custom: Specific virtual network |
| **Virtual network** | VMnet2 (Host-only) |
| **Replicate physical network...** | ❌ Desmarcar |

**Importante:** La VM debe tener **un solo adaptador** en VMnet2.

> ⚠️ Si tiene dos adaptadores (uno en Bridged y otro en VMnet2), eliminar el de Bridged. La VM solo debe tener salida a través del FortiGate Cliente.

---

## Paso 3 — Instalación Ubuntu

**Arrancar la VM** y seguir el instalador gráfico:

1. **Welcome:** Español / English (según preferencia)
2. **Keyboard layout:** Spanish / US (según preferencia)
3. **Updates and software:** 
   - Normal installation
   - ✅ Download updates while installing
4. **Installation type:** Erase disk and install Ubuntu
5. **Who are you:**
   - Name: Pablo
   - Computer name: ubuntu-cliente
   - Username: pablo
   - Password: *(segura pero memorable)*
6. **Finish** → Reiniciar cuando pida

---

## Paso 4 — Configurar Red Estática

Tras reiniciar e iniciar sesión:

### Verificar nombre de interfaz

```bash
ip link show
```

Buscar el nombre de la interfaz (típicamente `ens33`, `ens160`, o `enp0s3`).

### Editar Netplan

```bash
sudo nano /etc/netplan/01-network-manager-all.yaml
```

**Contenido completo:**

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    ens33:  # Ajustar al nombre real de tu interfaz
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

**Guardar:** `Ctrl+O`, `Enter`, `Ctrl+X`

### Aplicar configuración

```bash
sudo netplan apply
```

### Verificar

```bash
ip addr show
```

Debe mostrar:

```
ens33: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 10.0.2.100/24 brd 10.0.2.255 scope global ens33
```

---

## Paso 5 — Test de Conectividad

### Ping al FortiGate Cliente (gateway local)

```bash
ping -c 3 10.0.2.254
```

**Esperado:** 0% packet loss

### Ping a Zabbix Server (a través de VPN)

```bash
ping -c 3 10.0.1.10
```

**Esperado:** 0% packet loss, ~1ms RTT

> ✅ Si esto funciona, el túnel VPN está operativo y el split-tunneling funciona.

### Ping a internet (a través de NAT del FortiGate)

```bash
ping -c 3 8.8.8.8
```

**Esperado:** 0% packet loss

### Resolución DNS

```bash
ping -c 3 google.com
```

**Esperado:** Resuelve la IP y responde

---

## Paso 6 — Actualizar Sistema

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y vim curl wget net-tools traceroute
```

---

## Paso 7 — Instalar Zabbix Agent 2

### Añadir repositorio oficial Zabbix

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_latest+ubuntu22.04_all.deb
sudo apt update
```

### Instalar Zabbix Agent 2

```bash
sudo apt install -y zabbix-agent2
```

### Configurar Agent

```bash
sudo nano /etc/zabbix/zabbix_agent2.conf
```

**Líneas clave a modificar:**

```ini
Server=10.0.1.10
ServerActive=10.0.1.10
Hostname=Ubuntu-Cliente-Castellon
```

**Explicación:**
- `Server`: IP del Zabbix Server (comunicación pasiva)
- `ServerActive`: IP del Zabbix Server (comunicación activa)
- `Hostname`: Identificador del host en Zabbix (debe coincidir con el nombre en la GUI)

### Reiniciar servicio

```bash
sudo systemctl restart zabbix-agent2
sudo systemctl enable zabbix-agent2
```

### Verificar estado

```bash
sudo systemctl status zabbix-agent2
```

**Esperado:**

```
● zabbix-agent2.service - Zabbix Agent 2
   Loaded: loaded
   Active: active (running)
```

### Test de conectividad Agent → Server

```bash
zabbix_agent2 -t agent.ping
```

**Esperado:**

```
agent.ping                                    [u|1]
```

---

## Paso 8 — Agregar Host en Zabbix Server

### Desde Zabbix Web GUI

**Acceso:** `http://192.168.0.114:8080` (o `https://zabbix.nature46.uk` si Cloudflare está activo)

**Login:** `Admin` / `<tu_password>`

### Crear host

**Configuration → Hosts → Create host:**

| Campo | Valor |
|-------|-------|
| **Host name** | `Ubuntu-Cliente-Castellon` |
| **Visible name** | Ubuntu Cliente (Castellón) |
| **Templates** | Linux by Zabbix agent |
| **Groups** | Linux servers, Clientes MSP *(crear grupo)* |
| **Interfaces** | Agent: `10.0.2.100`, Port: `10050` |

**Add → Apply**

### Verificar monitorización

**Monitoring → Latest data:**

Filtrar por host `Ubuntu-Cliente-Castellon`

Debe aparecer métricas:
- CPU utilization
- Memory utilization
- Disk usage
- Network traffic

---

## Verificación Completa del Escenario

### Diagrama del flujo de datos

```
UBUNTU CLIENTE (10.0.2.100)
  └─► Zabbix Agent 2 recopila métricas
       └─► Envía datos a 10.0.1.10:10051
            └─► FortiGate Cliente detecta destino 10.0.1.0/24
                 └─► Enruta por VPN-Virtus-CS (CIFRADO)
                      └─► Llega a FortiGate Server
                           └─► Enruta a 10.0.1.10 (LXC-104 eth1)
                                └─► Zabbix Server recibe datos ✅
```

### Test manual desde Zabbix Server

```bash
# Desde LXC-104
pct enter 104
zabbix_get -s 10.0.2.100 -k agent.ping
```

**Esperado:**

```
1
```

Si devuelve `1`, el Zabbix Server puede consultar al Agent a través de la VPN.

---

## Troubleshooting

### Agent no conecta con Server

**Síntoma:** Host en Zabbix muestra "ZBX" rojo (no disponible)

**Verificar:**

```bash
# Desde el Ubuntu cliente
sudo systemctl status zabbix-agent2
sudo tail -f /var/log/zabbix/zabbix_agent2.log
```

**Causas comunes:**

1. **Firewall en Ubuntu bloqueando puerto 10050**

```bash
sudo ufw status
# Si está activo:
sudo ufw allow 10050/tcp
```

2. **IP incorrecta en zabbix_agent2.conf**

```bash
grep "^Server=" /etc/zabbix/zabbix_agent2.conf
# Debe mostrar: Server=10.0.1.10
```

3. **VPN caída**

```bash
ping -c 3 10.0.1.10
# Si no responde, revisar FortiGate Cliente
```

### No tiene internet

**Síntoma:** `ping 8.8.8.8` falla

**Verificar:**

1. **Política NAT en FortiGate Cliente**

```bash
# En FortiGate Cliente
ssh admin@192.168.0.103
show firewall policy | grep -A5 "LAN-to-Internet"
```

Debe mostrar `set nat enable`.

2. **Rutas en Ubuntu**

```bash
ip route show
```

Debe mostrar:

```
default via 10.0.2.254 dev ens33
```

3. **DNS funcionando**

```bash
cat /etc/resolv.conf
```

Debe contener `nameserver 8.8.8.8` o similar.

### Latencia alta en VPN

**Síntoma:** Ping a `10.0.1.10` tarda >50ms

**Diagnóstico:**

```bash
# Desde Ubuntu cliente
traceroute -n 10.0.1.10
```

**Esperado:**

```
1  10.0.2.254      0.3 ms   (FortiGate Cliente)
2  10.0.1.10       1.2 ms   (Zabbix Server)
```

Si hay más saltos o latencias altas, revisar túnel VPN:

```bash
# En FortiGate Cliente
diagnose vpn tunnel list | grep -i "time\|rtt"
```

---

## Comandos de Mantenimiento

### Ver logs Zabbix Agent

```bash
sudo tail -f /var/log/zabbix/zabbix_agent2.log
```

### Reiniciar Agent tras cambios config

```bash
sudo systemctl restart zabbix-agent2
```

### Test manual de métricas

```bash
zabbix_agent2 -t system.cpu.load[percpu,avg1]
zabbix_agent2 -t vm.memory.size[available]
zabbix_agent2 -t vfs.fs.size[/,used]
```

### Verificar conectividad Zabbix

```bash
# Test desde el propio Ubuntu (loopback)
zabbix_get -s 127.0.0.1 -k agent.ping
```

---

## Simulación de Alertas

Para probar que Zabbix detecta problemas:

### Simular CPU alta

```bash
# Genera carga CPU
stress --cpu 4 --timeout 60s
# Instalar stress si no existe:
sudo apt install -y stress
```

En Zabbix GUI, tras 1-2 minutos debe aparecer trigger:

```
High CPU utilization on Ubuntu-Cliente-Castellon
```

### Simular disco lleno

```bash
dd if=/dev/zero of=/tmp/bigfile bs=1M count=5000
```

Debe generar alerta de espacio en disco bajo.

### Limpiar

```bash
rm /tmp/bigfile
```

---

## Mejoras Futuras

### Corto plazo
- [ ] Configurar triggers personalizados en Zabbix
- [ ] Alertas por Telegram cuando el host cae
- [ ] Items personalizados (monitorizar aplicaciones específicas)

### Mediano plazo
- [ ] Template personalizado "Ubuntu Cliente MSP"
- [ ] Scripts de auto-discovery de servicios
- [ ] Integración con ticketing (crear ticket automático si alerta)

### Largo plazo
- [ ] Zabbix Proxy en red cliente (arquitectura distribuida)
- [ ] Monitoring de aplicaciones Docker en el cliente
- [ ] Zabbix sender para métricas custom

---

## Snapshot de Respaldo

Tras tener todo funcional, crear snapshot en VMware:

**VM → Snapshot → Take Snapshot:**

| Campo | Valor |
|-------|-------|
| **Name** | `baseline-zabbix-agent-ok` |
| **Description** | Ubuntu con red, VPN, Zabbix Agent funcional |
| **Snapshot memory** | ❌ (no necesario) |

---

## Recursos del Sistema

```bash
# Ver uso de recursos
htop

# Ver conexiones de red activas
sudo ss -tunap | grep zabbix

# Ver logs del sistema
journalctl -u zabbix-agent2 -f
```

---

**Documentos relacionados:**
- [← FortiGate Cliente Setup](02-FortiGate-Cliente-Setup.md)
- [← Túnel VPN IPsec](04-VPN-IPsec-Tunnel.md)
- [← LXC-104 Zabbix Server](03-LXC-104-Dual-Homed.md)
- [→ Troubleshooting](06-Troubleshooting.md)

---

*Última actualización: Febrero 2025*
