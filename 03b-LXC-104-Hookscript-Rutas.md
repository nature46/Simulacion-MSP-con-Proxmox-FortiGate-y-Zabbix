# LXC-104 — Hookscript y Rutas Persistentes

## Gestión de rutas de red que deben sobrevivir reinicios

---

## Contexto

LXC-104 tiene dos interfaces de red (dual-homed). Al arrancar, Linux puede asignar rutas por defecto en ambas interfaces, lo que provoca que el tráfico salga por la interfaz incorrecta. Para resolverlo de forma profesional y persistente se usa un **hookscript de Proxmox**: un script bash que Proxmox ejecuta automáticamente en momentos clave del ciclo de vida del container (arranque, parada, etc.).

---

## Problema que resuelve

Sin el hookscript, al arrancar LXC-104:

```
ip route show
default via 192.168.0.1 dev eth0   ← correcto
default via 10.0.1.254 dev eth1    ← problemático (interfiere con Internet)
```

Consecuencias:
- Zabbix no puede enviar notificaciones (sale por eth1 sin internet)
- Docker no puede hacer pull de imágenes
- Uptime Kuma reporta servicios como caídos
- Las rutas a redes remotas vía VPN (`10.0.2.0/24`) no existen

---

## Solución Implementada

### Archivo del hookscript

**Ubicación:** `/mnt/pve/nas-data/snippets/104-routes.sh`

```bash
#!/bin/bash
# Hookscript para LXC-104: gestión de rutas post-arranque
# Ejecutado automáticamente por Proxmox después de iniciar el container

if [ "$2" == "post-start" ]; then
    sleep 3  # Esperar a que las interfaces estén completamente activas

    # 1. Eliminar la ruta por defecto de eth1 (que interfiere con Internet)
    lxc-attach -n 104 -- ip route del default via 10.0.1.254 dev eth1 2>/dev/null

    # 2. Volver a añadirla con métrica alta (eth0 tiene prioridad para salida)
    lxc-attach -n 104 -- ip route add default via 10.0.1.254 dev eth1 metric 200 2>/dev/null

    # 3. Añadir ruta específica a la red del cliente remoto vía VPN
    lxc-attach -n 104 -- ip route add 10.0.2.0/24 via 10.0.1.254 dev eth1 2>/dev/null

    echo "LXC-104: eth0 preferred, eth1 metric 200, ruta VPN 10.0.2.0/24 añadida"
fi
```

**Explicación de cada línea:**

| Línea | Qué hace |
|---|---|
| `sleep 3` | Espera a que networkd termine de configurar las interfaces |
| `ip route del default via 10.0.1.254` | Elimina la ruta por defecto que añade Proxmox por eth1 |
| `ip route add default ... metric 200` | La vuelve a añadir con métrica 200 (eth0 tiene métrica 100, gana siempre) |
| `ip route add 10.0.2.0/24 via 10.0.1.254` | Añade ruta a la red del cliente remoto para que las respuestas de Zabbix lleguen bien |

---

## Configuración en Proxmox

### Referenciar el hookscript en la configuración del LXC

El hookscript se referencia en `/etc/pve/lxc/104.conf`:

```
hookscript: local:snippets/104-routes.sh
```

> **Nota:** `local` aquí se refiere al storage `local` de Proxmox. Si tus snippets están en el NAS (como en este caso), la ruta completa en disco es `/mnt/pve/nas-data/snippets/104-routes.sh` pero Proxmox lo referencia como `local:snippets/104-routes.sh`.

### Verificar que el hookscript está referenciado

```bash
grep hookscript /etc/pve/lxc/104.conf
# Debe mostrar: hookscript: local:snippets/104-routes.sh
```

---

## Verificación Post-Arranque

Después de reiniciar LXC-104, verificar que las rutas son correctas:

```bash
# Opción 1: desde el host Proxmox
pct exec 104 -- ip route show

# Opción 2: entrando en el container
pct enter 104
ip route show
```

**Salida esperada:**

```
default via 10.0.1.254 dev eth1 proto static metric 200
default via 192.168.0.1 dev eth0 proto static metric 100
10.0.1.0/24 dev eth1 proto kernel scope link src 10.0.1.10
10.0.2.0/24 via 10.0.1.254 dev eth1          ← ruta a red cliente vía VPN
172.17.0.0/16 dev docker0 ...
172.18.0.0/16 dev br-... ...
192.168.0.0/24 dev eth0 proto kernel scope link src 192.168.0.114
```

**Comprobaciones clave:**
- ✅ `eth0` tiene métrica 100 (prioridad sobre eth1 para salida a internet)
- ✅ `eth1` tiene métrica 200 (solo se usa para tráfico de monitorización)
- ✅ `10.0.2.0/24` está presente vía eth1 (permite llegar al cliente remoto)

---

## Prueba de Conectividad Completa

```bash
pct enter 104

# Internet (debe salir por eth0)
ping -c 2 8.8.8.8 && echo "Internet OK ✅"

# FortiGate Server LAN
ping -c 2 10.0.1.254 && echo "FortiGate Server OK ✅"

# FortiGate Cliente (extremo remoto del túnel VPN)
ping -c 2 10.0.2.254 && echo "FortiGate Cliente OK ✅"

# Ubuntu cliente (host monitorizado)
ping -c 2 10.0.2.100 && echo "Ubuntu Cliente OK ✅"
```

---

## Modificar el Hookscript

Si en el futuro añades más clientes (más redes remotas), edita el script:

```bash
nano /mnt/pve/nas-data/snippets/104-routes.sh
```

Añade una línea por cada nueva red cliente:

```bash
# Cliente 1 (Castellón) - ya existe
lxc-attach -n 104 -- ip route add 10.0.2.0/24 via 10.0.1.254 dev eth1 2>/dev/null

# Cliente 2 (Valencia) - ejemplo futuro
lxc-attach -n 104 -- ip route add 10.0.3.0/24 via 10.0.1.254 dev eth1 2>/dev/null
```

Los cambios se aplican **en el próximo arranque** del container. Para aplicarlos inmediatamente sin reiniciar:

```bash
lxc-attach -n 104 -- ip route add 10.0.X.0/24 via 10.0.1.254 dev eth1 2>/dev/null
```

---

## Troubleshooting del Hookscript

### La ruta 10.0.2.0/24 no aparece tras reiniciar

**Verificar que el hookscript existe y tiene permisos de ejecución:**

```bash
ls -la /mnt/pve/nas-data/snippets/104-routes.sh
# Debe tener: -rwxr-xr-x (ejecutable)

# Si no tiene permisos de ejecución:
chmod +x /mnt/pve/nas-data/snippets/104-routes.sh
```

**Verificar que está referenciado en la config del LXC:**

```bash
grep hookscript /etc/pve/lxc/104.conf
```

**Ejecutar manualmente para probar:**

```bash
bash /mnt/pve/nas-data/snippets/104-routes.sh 104 post-start
```

### El hookscript falla silenciosamente

Añadir logging al script para depuración:

```bash
# Añadir al inicio del bloque if:
exec >> /var/log/lxc-104-hookscript.log 2>&1
echo "$(date): Ejecutando hookscript post-start para LXC-104"
```
