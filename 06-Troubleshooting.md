# 🔧 Troubleshooting — Problemas y Soluciones

## Guía completa de resolución de problemas encontrados durante el despliegue

---

## Metodología de Troubleshooting

Ante cualquier problema, seguir este orden:

```
1. ¿Las interfaces están UP?
   → ip addr show (LXC)
   → get system interface physical (FortiGate)

2. ¿El routing es correcto?
   → ip route show (LXC)
   → get router info routing-table all (FortiGate)

3. ¿Hay ping entre nodos adyacentes?
   → ping al gateway directo
   → ping al siguiente hop

4. ¿Los servicios están corriendo?
   → docker compose ps (Zabbix)
   → get system status (FortiGate)

5. ¿Las políticas firewall permiten el tráfico?
   → FortiGate GUI → Log & Report → Forward Traffic

6. ¿La configuración base es correcta?
   → cat /etc/pve/lxc/104.conf (verificar typos)
   → show system interface (verificar IPs)
```

---

## 🐛 Problema 1: Typo en Gateway LXC (10.1.0.254 vs 10.0.1.254)

### Síntomas

- LXC-104 eth1 configurado pero routing no funciona
- Ping a FortiGate `10.0.1.254` falla o es intermitente
- Tráfico de respuesta sale por interfaz incorrecta
- Policy routing no se establece tras reboot

### Causa Raíz

En `/etc/pve/lxc/104.conf`, el gateway de eth1 estaba mal escrito:

```
❌ Incorrecto: net1: ...,gw=10.1.0.254,...
✅ Correcto:   net1: ...,gw=10.0.1.254,...
```

Este error generó una ruta estática a una IP que no existe, causando problemas de routing persistentes.

### Diagnóstico

```bash
# En host Proxmox
cat /etc/pve/lxc/104.conf | grep net1

# Dentro del LXC
ip route show | grep "10.1"
```

Si aparece `10.1.0.254`, hay typo.

### Solución

**Opción A — Desde Proxmox host:**

```bash
pct stop 104
nano /etc/pve/lxc/104.conf
```

Corregir la línea net1:

```
net1: name=eth1,bridge=vmbr1,gw=10.0.1.254,hwaddr=...,ip=10.0.1.10/24,...
```

```bash
pct start 104
```

**Opción B — Eliminar y recrear interfaz:**

```bash
pct set 104 -delete net1
pct set 104 -net1 name=eth1,bridge=vmbr1,ip=10.0.1.10/24,gw=10.0.1.254
pct stop 104 && pct start 104
```

### Verificación

```bash
pct enter 104
ip route show
```

**Debe mostrar:**

```
default via 192.168.0.1 dev eth0
10.0.1.0/24 dev eth1 proto kernel scope link src 10.0.1.10
```

**No debe aparecer** ninguna referencia a `10.1.0.254`.

### Lección Aprendida

> **Siempre verificar visualmente las IPs** en archivos de configuración antes de implementar soluciones complejas. Un simple typo puede parecer un problema de routing avanzado.

---

## 🐛 Problema 2: FortiGate Excede Límites de CPU

### Síntomas

```
VM is not licensed or license is invalid for current VM configuration.
```

La VM no arranca o muestra advertencia de licencia inválida.

### Causa Raíz

Licencia eval de FortiGate limita:
- **Máximo 1 vCPU**
- Máximo 2 GB RAM
- Máximo 3 interfaces

La VM se creó con 2 cores, excediendo el límite.

### Solución

```bash
# En host Proxmox
qm set 107 -cores 1
qm start 107
```

O desde Proxmox UI:

**VM-107 → Hardware → Processors → Edit → Cores: 1**

### Prevención

Crear siempre VMs de FortiGate eval con:
- Sockets: 1
- Cores: 1
- RAM: 2048 MB

---

## 🐛 Problema 3: Contraseña FortiGate Incorrecta Tras Primer Login

### Síntomas

Después de establecer la contraseña en el primer login, no es aceptada al intentar entrar de nuevo.

### Causa Raíz

El teclado de la consola usa **layout inglés** por defecto. Contraseñas con caracteres especiales se escriben diferente a como aparecen:

| Tecla presionada | Carácter obtenido |
|------------------|-------------------|
| `@` | Puede ser `"` o `'` |
| `#` | Puede ser `£` |
| `-` | Puede ser `_` |

FortiGate pide la contraseña **sin confirmación visual**, así que no hay forma de verificar lo que se escribió.

### Solución

**Factory reset:**

```bash
execute factoryreset
```

Confirmar con `y`. La VM reinicia con contraseña vacía.

### Prevención

Usar **contraseñas alfanuméricas simples** en consola:

✅ Bueno: `Virtus2026!CS` (símbolos comunes)  
❌ Evitar: `Vírtüs@2026#Pr0x` (acentos, símbolos complejos)

Una vez dentro de la GUI (con teclado correcto), cambiar a contraseña más compleja si se desea.

---

## 🐛 Problema 4: No Se Puede Acceder a GUI FortiGate por HTTPS

### Síntomas

- Página en blanco en `https://<IP_FortiGate>`
- Error "Connection refused"
- Certificado rechazado sin opción de continuar

### Causas y Soluciones

#### Causa A: `allowaccess` no incluye HTTPS

```bash
config system interface
    edit port1
        set allowaccess ping https ssh http
    next
end
```

#### Causa B: Navegador bloquea certificado autofirmado

| Navegador | Solución |
|-----------|----------|
| **Chrome/Edge** | "Advanced" → "Proceed to... (unsafe)" |
| **Firefox** | "Advanced" → "Accept the Risk and Continue" |
| **Brave** | Desactivar shields para el sitio |
| **Chrome (sin botón)** | Escribir `thisisunsafe` en la página |

#### Causa C: Firewall Windows bloqueando

```powershell
# PowerShell como administrador
New-NetFirewallRule -DisplayName "FortiGate Lab" -Direction Inbound -LocalPort 443,8443 -Protocol TCP -Action Allow
```

#### Causa D: FortiGate no escucha en esa interfaz

```bash
get system interface physical
```

Verificar que la interfaz tiene `status: up` e IP asignada.

```bash
execute ping <IP_tu_PC>
```

Verificar conectividad bidireccional.

---

## 🐛 Problema 5: port1 FortiGate Cliente en DOWN

### Síntomas

`get system interface physical` muestra port1 como **DOWN** o sin IP.

### Causa Raíz

El adaptador de red en VMware no está conectado o el tipo es incorrecto.

### Solución

**Desde VMware:**

1. VM → Network Adapter → ✅ Verificar "Connected"
2. Verificar tipo: **Bridged** (no NAT)
3. Desconectar y reconectar el adaptador desde el menú VM

**Desde FortiGate CLI:**

```bash
config system interface
    edit port1
        set status up
        set mode static
        set ip 192.168.0.103 255.255.255.0
        set allowaccess ping https ssh http
    next
end
```

Esperar 10 segundos y verificar:

```bash
get system interface physical
```

---

## 🐛 Problema 6: Túnel VPN No Sube (status=down)

### Síntomas

```bash
diagnose vpn tunnel list
```

Muestra `status=down` o `status=connecting`.

### Diagnóstico

```bash
# Ver estado IKE
diagnose vpn ike gateway list

# Ver logs en tiempo real
diagnose debug application ike -1
diagnose debug enable
```

### Causas Comunes

#### Causa A: Pre-Shared Key diferente

**Verificar en ambos FortiGates:**

```bash
show vpn ipsec phase1-interface VPN-Virtus-CS | grep psksecret
```

Debe ser **idéntica** en ambos lados (case-sensitive).

**Solución:**

```bash
config vpn ipsec phase1-interface
    edit "VPN-Virtus-CS"
        set psksecret Virtus2026!CS
    next
end
```

#### Causa B: Remote Gateway IP incorrecta

**FortiGate Server debe apuntar a Cliente:**

```bash
show vpn ipsec phase1-interface VPN-Virtus-CS | grep remote-gw
```

Debe mostrar: `set remote-gw 192.168.0.103`

**FortiGate Cliente debe apuntar a Server:**

Debe mostrar: `set remote-gw 192.168.0.102`

#### Causa C: Subnets Phase 2 no coinciden

**Server:**

```
src-subnet 10.0.1.0 255.255.255.0
dst-subnet 10.0.2.0 255.255.255.0
```

**Cliente:**

```
src-subnet 10.0.2.0 255.255.255.0  ← Invertido
dst-subnet 10.0.1.0 255.255.255.0  ← Invertido
```

#### Causa D: No hay conectividad básica

```bash
execute ping <IP_remota>
```

Si falla, revisar routing básico antes de debuggear VPN.

### Solución General

1. Verificar PSK idéntica
2. Verificar remote-gw correctas
3. Verificar Phase 2 subnets inversas
4. Verificar políticas firewall permiten tráfico VPN
5. Reiniciar negociación:

```bash
diagnose vpn ike gateway clear
```

---

## 🐛 Problema 7: Túnel Sube Pero No Pasa Tráfico

### Síntomas

- `diagnose vpn tunnel list` muestra `status=up`
- Pero `ping` a través del túnel falla

### Diagnóstico

```bash
# Ver políticas firewall
show firewall policy | grep VPN

# Ver rutas estáticas
get router info routing-table all

# Ver sesiones activas
get system session list
```

### Causas Comunes

#### Causa A: Falta política firewall

Las políticas `VPN-Local-to-Remote` y `VPN-Remote-to-Local` deben existir en ambos FortiGates.

**Verificar:**

```bash
show firewall policy | grep -A10 "VPN-Local-to-Remote"
```

Si no existe, crear según [04-VPN-IPsec-Tunnel.md](04-VPN-IPsec-Tunnel.md).

#### Causa B: Falta ruta estática

**En Server:**

```bash
get router info routing-table all | grep 10.0.2.0
```

Debe mostrar:

```
S       10.0.2.0/24 [10/0] via VPN-Virtus-CS tunnel 192.168.0.103
```

Si no aparece, crear:

```bash
config router static
    edit 0
        set dst 10.0.2.0 255.255.255.0
        set device "VPN-Virtus-CS"
    next
end
```

#### Causa C: Subnets Phase 2 incorrectas

Verificar que src/dst subnets coinciden (invertidas) entre Server y Cliente.

---

## 🐛 Problema 8: Ubuntu Cliente No Tiene Internet

### Síntomas

- `ping 8.8.8.8` falla desde Ubuntu
- `ping 10.0.2.254` funciona (FortiGate alcanzable)

### Causa Raíz

Falta política NAT en FortiGate Cliente.

### Solución

```bash
# En FortiGate Cliente
show firewall policy | grep -i "LAN-to-Internet"
```

Si no existe:

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

**Verificar desde Ubuntu:**

```bash
ping -c 3 8.8.8.8
```

---

## 🐛 Problema 9: Zabbix Agent No Conecta con Server

### Síntomas

Host en Zabbix muestra "ZBX" rojo (no disponible).

### Diagnóstico

**En Ubuntu cliente:**

```bash
sudo systemctl status zabbix-agent2
sudo tail -f /var/log/zabbix/zabbix_agent2.log
```

**Desde Zabbix Server:**

```bash
zabbix_get -s 10.0.2.100 -k agent.ping
```

### Causas Comunes

#### Causa A: Firewall bloqueando puerto 10050

```bash
# En Ubuntu
sudo ufw status
sudo ufw allow 10050/tcp
sudo systemctl restart zabbix-agent2
```

#### Causa B: IP incorrecta en configuración

```bash
grep "^Server=" /etc/zabbix/zabbix_agent2.conf
```

Debe mostrar: `Server=10.0.1.10`

#### Causa C: VPN caída

```bash
ping -c 3 10.0.1.10
```

Si falla, revisar túnel VPN en FortiGates.

---

## 🐛 Problema 10: Zabbix Web No Carga Tras Deploy

### Síntomas

Acceder a `http://192.168.0.114:8080` da error de conexión justo después de `docker compose up`.

### Causa Raíz

Zabbix Server necesita **2-3 minutos** para inicializar la base de datos (crear tablas, cargar datos).

### Solución

```bash
docker compose logs -f zabbix-server
```

Esperar a ver:

```
server #0 started [main process]
```

Una vez aparezca, la web cargará correctamente.

---

## 🐛 Problema 11: Segunda Licencia FortiGate Bloqueada

### Síntomas

Al aplicar licencia eval en FortiGate Cliente, indica que la cuenta ya tiene licencia activa.

### Soluciones

#### Opción A: Segunda cuenta Fortinet

Crear nueva cuenta con email diferente:
- Email real (recibirá verificación)
- Diferentes datos de registro

#### Opción B: Contactar Fortinet

Enviar email a Business Development explicando uso educativo:

```
Asunto: Request for Second Evaluation License - Educational Lab

Hola,
Soy estudiante preparando prácticas profesionales en empresa MSP.
Necesito segunda licencia eval para laboratorio VPN site-to-site.
Uso educativo, no comercial. ¿Es posible obtener segunda licencia?

Gracias,
[Tu nombre]
```

#### Opción C: Funcionar sin licencia

El FortiGate Cliente puede operar sin licencia para funciones básicas de VPN. Ver [02-FortiGate-Cliente-Setup.md](02-FortiGate-Cliente-Setup.md) Paso 7.

---

## 🐛 Problema 12: Conflicto Puerto 443 (GUI vs VPN)

### Síntomas

Al configurar VPN, aparece advertencia:

```
Administrative access on port1 will no longer be available because IKE TCP and HTTPS are configured to use the same port.
```

### Causa Raíz

IKE TCP (usado por VPN) y HTTPS (GUI) ambos usan puerto 443.

### Solución

**Cambiar puerto GUI a 8443:**

```bash
config system global
    set admin-sport 8443
end
```

**Acceso GUI ahora:**

```
https://192.168.0.102:8443
```

---

## Checklist Rápido de Diagnóstico

```bash
# 1. Interfaces UP?
ip addr show                    # LXC
get system interface physical   # FortiGate

# 2. Routing correcto?
ip route show                   # LXC
get router info routing-table all  # FortiGate

# 3. Ping básico?
ping 192.168.0.1               # Gateway
ping 10.0.1.254                # FortiGate LAN

# 4. Servicios corriendo?
docker compose ps              # Zabbix
get system status              # FortiGate

# 5. VPN UP?
diagnose vpn tunnel list       # FortiGate

# 6. Políticas permiten tráfico?
# FortiGate GUI → Log & Report → Forward Traffic
```

---

## Comandos de Diagnóstico por Componente

### LXC-104 (Zabbix Server)

```bash
# Estado red completo
ip addr show && echo "---" && ip route show

# Conectividad a nodos clave
ping -c 1 192.168.0.1 && echo "Router OK" || echo "FAIL"
ping -c 1 10.0.1.254 && echo "FortiGate OK" || echo "FAIL"
ping -c 1 8.8.8.8 && echo "Internet OK" || echo "FAIL"

# Docker
docker compose ps
docker compose logs --tail=20 zabbix-server
```

### FortiGate (ambos)

```bash
# Estado general
get system status
get system interface physical

# Routing
get router info routing-table all

# VPN
diagnose vpn tunnel list
diagnose vpn ike gateway list

# Sesiones activas
get system session list | head -20

# Políticas
show firewall policy
```

### Ubuntu Cliente

```bash
# Red
ip addr && ip route show

# Conectividad
ping -c 2 10.0.2.254   # Gateway local
ping -c 2 10.0.1.10    # Zabbix Server (vía VPN)
ping -c 2 8.8.8.8      # Internet

# Zabbix Agent
sudo systemctl status zabbix-agent2
zabbix_agent2 -t agent.ping
```

---

## Tips Profesionales

### Usar traceroute para ver path

```bash
traceroute -n 10.0.1.10
```

Muestra cada hop — útil para identificar dónde se pierde el tráfico.

### Capturar tráfico con tcpdump

```bash
# En LXC-104
tcpdump -i eth1 -n host 10.0.2.100
```

Muestra paquetes de Zabbix Agent en tiempo real.

### Ver logs de FortiGate en vivo

```bash
# En FortiGate
execute log filter category 0  # All categories
execute log display
```

### Forzar renegociación VPN

```bash
diagnose vpn ike gateway clear
```

Reinicia el túnel sin tocar la configuración.

---

**Documentos relacionados:**
- [← README Principal](README.md)
- [← FortiGate Server](01-FortiGate-Server-Setup.md)
- [← FortiGate Cliente](02-FortiGate-Cliente-Setup.md)
- [← LXC-104 Setup](03-LXC-104-Dual-Homed.md)
- [← Túnel VPN](04-VPN-IPsec-Tunnel.md)
- [← Ubuntu Cliente](05-Ubuntu-Cliente-Setup.md)

---

*Última actualización: Febrero 2025*
