# Troubleshooting — Nuevas Entradas (añadir al 04-Troubleshooting.md existente)

---

## Problema 9: Ubuntu VM pierde red cada pocos minutos

### Síntomas

- La red funciona al arrancar pero se pierde periódicamente
- `ping 8.8.8.8` falla
- Ejecutar `sudo netplan apply` lo restaura temporalmente
- En Ubuntu aparece una notificación de "Fallo de activación de red"
- El archivo `/etc/netplan/01-network-manager-all.yaml` muestra warnings de permisos

### Causa raíz

El renderer configurado es `NetworkManager`, pero la configuración es estática. NetworkManager intenta gestionar la interfaz activamente y la resetea, entrando en conflicto con la configuración estática de Netplan. Adicionalmente, los permisos incorrectos del archivo (`644` en vez de `600`) provocan que Netplan rechace la configuración con warnings.

### Solución

**Paso 1 — Corregir permisos del archivo:**

```bash
sudo chmod 600 /etc/netplan/01-network-manager-all.yaml
```

**Paso 2 — Cambiar el renderer a `networkd`:**

```bash
sudo nano /etc/netplan/01-network-manager-all.yaml
```

Cambiar:
```yaml
renderer: NetworkManager
```

Por:
```yaml
renderer: networkd
```

**Paso 3 — Aplicar:**

```bash
sudo netplan apply
```

### Verificación

```bash
ping -c 5 8.8.8.8
# Debe mantener conectividad estable sin pérdidas
```

### Lección aprendida

En Ubuntu con configuración de red estática en Netplan, usar siempre `renderer: networkd`. El renderer `NetworkManager` está pensado para entornos dinámicos (laptops con WiFi) donde la red cambia. Para servidores o VMs con IP fija, `networkd` es más estable y predecible.

---

## Problema 10: LXC-104 no alcanza la red del cliente remoto (10.0.2.0/24)

### Síntomas

- El túnel VPN está UP y funciona en ambas direcciones según los FortiGates
- Desde el Ubuntu cliente (`10.0.2.100`), el ping a `10.0.1.10` falla
- Desde LXC-104, el ping a `10.0.2.100` también falla
- `diagnose vpn tunnel list` muestra `rxp>0` pero `enc:pkts/bytes=0/0` en el FortiGate Server

### Causa raíz

El túnel VPN está operativo, pero LXC-104 (Zabbix Server) no tiene una ruta hacia `10.0.2.0/24`. Cuando el agente en `10.0.2.100` intenta enviar datos a `10.0.1.10`, el paquete llega correctamente al LXC-104. Sin embargo, la respuesta de LXC-104 sale por `eth0` (internet) en vez de por `eth1` (red interna hacia el FortiGate), porque Linux no sabe que debe usar `eth1` para responder a `10.0.2.x`.

**Flujo del problema:**
```
Ubuntu (10.0.2.100) → VPN → FortiGate Server → LXC-104 eth1 ✅ (llega)
LXC-104 eth0 → 192.168.0.1 → internet ❌ (respuesta sale por interfaz incorrecta)
```

### Solución

Añadir ruta estática en LXC-104 apuntando a la red del cliente vía FortiGate Server:

```bash
# Aplicación inmediata (no persiste tras reinicio)
lxc-attach -n 104 -- ip route add 10.0.2.0/24 via 10.0.1.254 dev eth1
```

**Para que persista tras reinicios**, añadir al hookscript:

```bash
nano /mnt/pve/nas-data/snippets/104-routes.sh
```

```bash
# Añadir esta línea dentro del bloque if [ "$2" == "post-start" ]:
lxc-attach -n 104 -- ip route add 10.0.2.0/24 via 10.0.1.254 dev eth1 2>/dev/null
```

### Verificación

```bash
pct exec 104 -- ip route show | grep 10.0.2
# Debe mostrar: 10.0.2.0/24 via 10.0.1.254 dev eth1

pct exec 104 -- ping -c 3 10.0.2.100
# Debe responder con 0% packet loss
```

### Lección aprendida

En arquitecturas dual-homed donde el servidor tiene múltiples interfaces, Linux solo conoce las redes directamente conectadas. Las redes remotas accesibles a través de VPN requieren rutas estáticas explícitas para que las respuestas salgan por la interfaz correcta. Esta es una diferencia clave respecto a un host con una sola interfaz.

---

## Problema 11: LXC pierde rutas personalizadas tras reinicio

### Síntomas

- Después de un reinicio del LXC-104, la ruta a `10.0.2.0/24` desaparece
- El hookscript existe pero no aplicó la ruta
- Hay que añadir la ruta manualmente tras cada reinicio

### Causa raíz

El hookscript no tenía la línea de la ruta VPN cuando se hizo el reinicio. Se editó el script después del reinicio pero antes de verificar si persiste.

### Solución

Verificar que el contenido actual del hookscript incluye la línea de la ruta:

```bash
cat /mnt/pve/nas-data/snippets/104-routes.sh
```

El script debe contener las tres líneas funcionales:

```bash
lxc-attach -n 104 -- ip route del default via 10.0.1.254 dev eth1 2>/dev/null
lxc-attach -n 104 -- ip route add default via 10.0.1.254 dev eth1 metric 200 2>/dev/null
lxc-attach -n 104 -- ip route add 10.0.2.0/24 via 10.0.1.254 dev eth1 2>/dev/null
```

Después de editar, reiniciar el LXC para confirmar que persiste:

```bash
pct reboot 104
sleep 15
pct exec 104 -- ip route show | grep 10.0.2
```

### Lección aprendida

Siempre verificar que los cambios a scripts de persistencia (hookscripts, systemd units, etc.) están guardados ANTES de reiniciar para probarlos. El flujo correcto es: editar → guardar → reiniciar → verificar.
