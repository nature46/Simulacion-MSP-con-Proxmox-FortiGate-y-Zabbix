# Zabbix — Alertas y Dashboard MSP

## Configuración de notificaciones y panel de monitorización centralizada

---

## Contexto

Con el Zabbix Agent funcionando en el cliente remoto y reportando métricas a través de la VPN, el siguiente paso es configurar el sistema de alertas para que el equipo del MSP sea notificado automáticamente cuando ocurra cualquier incidencia en la infraestructura del cliente.

```
Problema detectado en Ubuntu-Cliente-Castellon
  └─► Zabbix Server evalúa trigger
       └─► Action dispara notificación
            ├─► Telegram (inmediato, operativo)
            └─► Email Gmail (formal, trazabilidad)
```

---

## Triggers — Estado Inicial

El template `Linux by Zabbix agent` incluye **25 triggers predefinidos** que cubren los casos de uso más comunes en un MSP. No es necesario crear triggers desde cero.

**Triggers más relevantes para MSP:**

| Trigger | Severidad | Cuándo activa |
|---|---|---|
| Zabbix agent is not available | Average | Agente sin respuesta 3 min |
| High CPU utilization | Warning | CPU > 90% durante 5 min |
| High memory utilization | Average | RAM > 90% durante 5 min |
| FS [/]: Space is critically low | Average | Disco > 90% usado |
| Interface ens33: Link down | Average | Interfaz de red caída |
| Host has been restarted | Warning | Uptime < 10 minutos |
| Number of installed packages changed | Warning | Software instalado/eliminado |

**Verificar triggers activos:**

**Data collection → Hosts → Ubuntu-Cliente-Castellon → Triggers**

Todos deben aparecer en estado **OK** en condiciones normales.

### Ajustar umbrales por host (opcional)

Si quieres cambiar un umbral solo para este host sin tocar el template:

**Data collection → Hosts → Ubuntu-Cliente-Castellon → Macros → Add:**

| Macro | Valor por defecto | Descripción |
|---|---|---|
| `{$CPU.UTIL.CRIT}` | 90 | % CPU para alerta |
| `{$MEMORY.UTIL.MAX}` | 90 | % RAM para alerta |
| `{$VFS.FS.PUSED.MAX.CRIT:/}` | 90 | % disco crítico |
| `{$VFS.FS.PUSED.MAX.WARN:/}` | 80 | % disco warning |

---

## Alertas Telegram

### Paso 1 — Crear bot en Telegram

1. Abrir Telegram → buscar `@BotFather`
2. Enviar `/newbot`
3. Nombre del bot: `Zabbix MSP Alerts` (o similar)
4. Username: `zabbix_msp_bot` (debe terminar en `bot`)
5. BotFather devuelve el **token** — guardarlo de forma segura

### Obtener Chat ID

1. Abrir el bot recién creado → click **Start**
2. Enviar cualquier mensaje al bot
3. Abrir en navegador:

```
https://api.telegram.org/bot<TU_TOKEN>/getUpdates
```

4. Buscar el campo `"chat": {"id": XXXXXXXXX}` — ese es tu Chat ID

### Paso 2 — Configurar Media Type en Zabbix

**Alerts → Media types → Telegram → Edit:**

| Campo | Valor |
|---|---|
| Token | `<tu_token_completo>` |
| Status | Enabled |

Click **Update**.

> ⚠️ **Seguridad:** Nunca publiques el token completo en repositorios públicos. Usa `<TOKEN_CENSURADO>` en la documentación.

### Paso 3 — Asignar Telegram al usuario Admin

**Settings → Users → Admin → Media → Add:**

| Campo | Valor |
|---|---|
| Type | `Telegram` |
| Send to | `<tu_chat_id>` |
| When active | `1-7,00:00-24:00` |
| Use if severity | Warning ✅, Average ✅, High ✅, Disaster ✅ |

Click **Update**.

---

## Alertas Email (Gmail)

### Paso 1 — Crear contraseña de aplicación en Gmail

1. Cuenta Google → **Seguridad → Verificación en dos pasos** (debe estar activa)
2. **Seguridad → Contraseñas de aplicación**
3. Seleccionar: `Correo` + `Otro (nombre personalizado)` → `Zabbix`
4. Google genera una contraseña de 16 caracteres — guardarla

### Paso 2 — Configurar Media Type Gmail en Zabbix

**Alerts → Media types → Gmail → Edit:**

| Campo | Valor |
|---|---|
| Email | `tu_cuenta@gmail.com` |
| Password | `<contraseña de aplicación>` |
| Status | Enabled |

Click **Update**.

### Paso 3 — Asignar Gmail al usuario Admin

**Settings → Users → Admin → Media → Add:**

| Campo | Valor |
|---|---|
| Type | `Gmail` |
| Send to | `tu_cuenta@gmail.com` |
| When active | `1-7,00:00-24:00` |
| Use if severity | Warning ✅, Average ✅, High ✅, Disaster ✅ |

Click **Update**.

---

## Action — Conectar Triggers con Notificaciones

La Action define qué hacer cuando se activa un trigger. Sin Action, los triggers se detectan pero no se notifica a nadie.

**Alerts → Actions → Trigger actions → Create action:**

### Pestaña Action

| Campo | Valor |
|---|---|
| Name | `Alertas MSP` |
| Conditions | (dejar por defecto — aplica a todos los triggers) |
| Enabled | ✅ |

### Pestaña Operations

**Operations → Add:**

| Campo | Valor |
|---|---|
| Send to users | `Admin` |
| Send only to | `All` (envía por Telegram Y Gmail) |

**Recovery operations → Add:**

| Campo | Valor |
|---|---|
| Send to users | `Admin` |
| Send only to | `All` |

> ⚠️ **Importante:** Configurar también Recovery operations para recibir notificación cuando el problema se resuelve. Sin esto, solo recibes alertas de problema pero no de resolución.

Click **Update**.

---

## Verificación de Alertas

### Test rápido — parar el agente

```bash
# En el Ubuntu cliente
sudo systemctl stop zabbix-agent2
```

Esperar **3-5 minutos** — debe llegar alerta a Telegram y Gmail:

```
🔴 PROBLEM: Linux: Zabbix agent is not available (for 3m)
Host: Ubuntu-Cliente-Castellon
Severity: Average
```

Arrancar de nuevo:

```bash
sudo systemctl start zabbix-agent2
```

Debe llegar mensaje de resolución:

```
✅ Resolved in Xm Xs: Linux: Zabbix agent is not available
```

### Test de carga de CPU

```bash
# Instalar si no está disponible
sudo apt install -y stress

# Generar carga (ajustar umbral en macros si CPU tiene muchos cores)
stress --cpu 4 --timeout 300
```

---

## Dashboard MSP — Vista Global

El dashboard proporciona una vista de un solo vistazo del estado de todos los clientes monitorizados.

**Monitoring → Dashboards → Create dashboard:**

- Name: `MSP - Vista Global`

### Widgets recomendados

| Widget | Type | Descripción |
|---|---|---|
| Estado de Hosts | `Host availability` | Resumen verde/rojo de todos los hosts |
| Problemas Activos | `Problems` | Lista de problemas en tiempo real |
| CPU Ubuntu Cliente | `Graph` | Gráfica CPU utilization |
| Memoria Ubuntu Cliente | `Graph` | Gráfica Memory utilization |
| Disco Ubuntu Cliente | `Graph` | Gráfica FS [/]: Space utilization |
| Red ens33 | `Graph` | Gráfica Interface bits received/sent |
| Uptime Ubuntu Cliente | `Item value` | System uptime del host |
| Hora Local | `Clock` | Reloj en tiempo real |

### Configuración Widget — Host availability

| Campo | Valor |
|---|---|
| Type | `Host availability` |
| Interface type | Zabbix agent (passive checks) ✅ |
| Refresh interval | 1 minute |

### Configuración Widget — Problems

| Campo | Valor |
|---|---|
| Type | `Problems` |
| Severity | Warning, Average, High, Disaster |
| Refresh interval | 1 minute |

Click **Save changes** cuando termines de añadir todos los widgets.

---

## Resultado Final

Con esta configuración el MSP tiene:

```
MONITORIZACIÓN COMPLETA DEL CLIENTE REMOTO
├── 25 triggers activos (CPU, RAM, disco, red, seguridad)
├── Alertas inmediatas por Telegram (operativo)
├── Alertas formales por Gmail (trazabilidad)
├── Notificación de resolución automática
└── Dashboard MSP con vista global en tiempo real
```

**Flujo completo verificado:**
```
Ubuntu-Cliente-Castellon (10.0.2.100)
  └─► Zabbix Agent → VPN IPsec → Zabbix Server (10.0.1.10)
       └─► Trigger activado
            └─► Action: notificación
                 ├─► Telegram: mensaje instantáneo ✅
                 └─► Gmail: email formal ✅
```

---

## Siguiente Paso

**SNMP Monitoring de FortiGates** — monitorizar los propios firewalls desde Zabbix usando el protocolo SNMP. Esto permite ver el estado de las interfaces, el tráfico VPN, la CPU y memoria de los FortiGates directamente en el dashboard MSP.
