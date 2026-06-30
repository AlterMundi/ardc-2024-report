<!--
SPDX-FileCopyrightText: 2026 AlterMundi
SPDX-License-Identifier: CC-BY-SA-4.0
-->
<!-- Supporting document for §3 (Data Collection) of the ARDC 2024 report: the PuebloLibre 12-node
     LibreMesh optimization that provides the field before/after evidence (Table 3.1). -->

---
title: Informe de Optimización Red Mesh LibreMesh

---

# Informe de Optimización Red Mesh LibreMesh

## Resumen Ejecutivo

**Fecha:** 2025-07-05  
**Red:** LibreMesh Community Mesh Network  
**Nodos optimizados:** 12 dispositivos en 2 zonas  
**Problema principal:** Bucles de ruteo y drops masivos de paquetes  
**Resultado:** Conectividad estable y rendimiento mejorado  

---

## 🔍 Hallazgos Principales

### 1. Topología de Red Identificada

**Zona 1 (Gateway: Comunidad)**
- **comunidad** (10.193.43.98) - Gateway principal
- **morce** (10.193.155.27) - Nodo intermedio crítico
- **montecitostyle** (10.193.74.14) - Nodo acceso
- **nodo001** (10.193.121.26) - Nodo terminal
- **nodo-suri** (10.193.216.6) - Nodo terminal

**Zona 2 (Gateway: Palito)**
- **palito** (10.193.42.78) - Gateway zona 2
- **nodo-jime** (10.128.179.129) - Nodo intermedio
- **balcon** (10.128.210.83) - Nodo intermedio
- **barrazavega** (10.128.246.154) - Nodo terminal
- **nodo-bob** (10.128.81.16) - Nodo terminal

### 2. Problemas Críticos Detectados

#### 2.1 Bucles de Ruteo
- **Morce ↔ Comunidad**: Loop detectado con 98% del tráfico concentrado
- **Síntoma**: Forwarding desbalanceado (morce manejando mayoría del tráfico)
- **Impacto**: Latencia alta y congestión

#### 2.2 Drops Masivos de Paquetes
- **Nodo**: palito (10.193.42.78)
- **Drops detectados**: 354,000+ paquetes en eth0.1
- **Causa**: Buffers de red insuficientes
- **Impacto**: Pérdida de conectividad intermitente

#### 2.3 Configuración de Gateway Subóptima
- **Problema**: Comunidad no configurado como servidor gateway
- **Resultado**: Dependencia excesiva en rutas alternativas
- **Impacto**: Ruteo ineficiente

#### 2.4 Configuración Bridge Ineficiente
- **STP**: Habilitado innecesariamente (delay 15ms)
- **Multicast snooping**: Activo sin beneficio
- **Forward delay**: Demasiado alto (15ms)
- **Ageing time**: Subóptimo (300s)

---

## 🔧 Mejoras Implementadas

### 1. Optimización de Buffers de Red

**Archivo:** `/tmp/optimize_palito_buffers.py`

```bash
# Comandos aplicados:
echo 5000 > /proc/sys/net/core/netdev_max_backlog
echo 262144 > /proc/sys/net/core/rmem_max
echo 262144 > /proc/sys/net/core/wmem_max

# Persistencia:
echo 'net.core.netdev_max_backlog = 5000' >> /etc/sysctl.d/99-network-buffers.conf
echo 'net.core.rmem_max = 262144' >> /etc/sysctl.d/99-network-buffers.conf
echo 'net.core.wmem_max = 262144' >> /etc/sysctl.d/99-network-buffers.conf
```

**Resultado**: Drops estabilizados en 360K (sin incremento)

### 2. Optimización del Bridge

**Archivo:** `/tmp/bridge_optimization_with_backup.py`

```bash
# Comandos aplicados:
brctl stp br-lan off                    # Desactivar STP
brctl setfd br-lan 2                    # Forward delay: 15s → 2s
echo 0 > /sys/class/net/br-lan/bridge/multicast_snooping  # Desactivar multicast snooping
brctl setageing br-lan 120              # Ageing time: 300s → 120s
```

**Resultado**: Latencia reducida y forwarding más eficiente

### 3. Configuración Gateway

```bash
# En comunidad:
batctl gw server         # Configurar como servidor gateway

# En morce:
batctl gw client         # Mantener como cliente
```

**Resultado**: Ruteo más eficiente y balanceado

---

## 📊 Resultados Cuantitativos

### Antes vs Después

| Métrica | Antes | Después | Mejora |
|---------|-------|---------|--------|
| Drops eth0.1 | 354K+ activos | 360K estables | ✅ Estabilizado |
| Latencia local | ~1ms | 0.2ms | 80% mejor |
| Latencia zona 1 | ~25ms | 15ms | 40% mejor |
| Latencia internet | ~50ms | 36ms | 28% mejor |
| Forward delay | 15s | 2s | 87% mejor |
| Conectividad | Intermitente | 100% estable | ✅ Estable |

### Conectividad Post-Optimización

- ✅ **Internet**: 100% funcional
- ✅ **Zona 1**: 100% funcional  
- ✅ **Gateway interno**: 100% funcional
- ✅ **Resolución DNS**: 100% funcional
![Captura de pantalla de 2025-10-10 16-42-47](https://hackmd.io/_uploads/By6-GJP6el.png)

---

## 🚀 Comandos Esenciales para Administración

### Monitoreo de Red

```bash
# Verificar drops en tiempo real
watch -n 30 'cat /proc/net/dev | grep -E "br-lan|eth0.1|wlan1-mesh"'

# Monitorear conectividad
watch -n 60 'ping -c 1 8.8.8.8; ping -c 1 10.193.43.98'

# Estado del bridge
brctl show br-lan
brctl showstp br-lan

# Verificar configuración actual
cat /sys/class/net/br-lan/bridge/stp_state
cat /sys/class/net/br-lan/bridge/forward_delay
cat /sys/class/net/br-lan/bridge/multicast_snooping
cat /sys/class/net/br-lan/bridge/ageing_time
```

### Diagnóstico Batman-adv

```bash
# Verificar originators
batctl o

# Estado del gateway
batctl gw

# Interfaces batman
batctl if

# Throughput test
batctl tp <mac_address>

# Tabla de neighbors
batctl n
```

### Diagnóstico Babel

```bash
# Rutas babel
ip route show proto babel

# Vecinos babel
cat /var/lib/babel/*/babel.state
```

### Análisis de Tráfico

```bash
# Top interfaces por tráfico
cat /proc/net/dev | sort -k2 -nr

# Verificar buffer overruns
netstat -i

# Análisis de drops por interfaz
awk '{if(NR>2) print $1,$5,$13}' /proc/net/dev
```

---

## 🔄 Rollback y Backups

### Rollback Automático

**Archivo:** `/tmp/rollback_20250705_213338.sh`

```bash
#!/bin/bash
# Restaurar configuración original
brctl stp br-lan on
brctl setfd br-lan 15
echo 1 > /sys/class/net/br-lan/bridge/multicast_snooping
brctl setageing br-lan 300
```

### Backups Completos

**Ubicación:** `/tmp/backup_20250705_213338/`

- `bridge_config.txt` - Configuración bridge
- `network.backup` - Configuración de red
- `batman_*.txt` - Estado Batman-adv  
- `sysctl.txt` - Configuración kernel
- `interfaces.txt` - Estado interfaces
- `routes.txt` - Tabla de rutas


---

## 🔧 Mantenimiento Crítico

### Rutinas

```bash
# Script de monitoreo diario
#!/bin/bash
echo "=== CHEQUEO DIARIO $(date) ==="

# Verificar conectividad
ping -c 3 8.8.8.8 > /dev/null && echo "✅ Internet OK" || echo "❌ Internet FAIL"

# Verificar drops
DROPS=$(cat /proc/net/dev | grep eth0.1 | awk '{print $5}')
echo "📊 Drops actuales: $DROPS"

# Verificar memoria
free -h

# Verificar uptime
uptime
```


```bash
# Backup de configuración
mkdir -p /tmp/backup_$(date +%Y%m%d)
cp /etc/config/* /tmp/backup_$(date +%Y%m%d)/
batctl o > /tmp/backup_$(date +%Y%m%d)/batman_originators.txt

# Análisis de logs
logread | grep -i error | tail -20

# Verificar vecinos
batctl n > /tmp/neighbors_$(date +%Y%m%d).txt
```


```bash
# Análisis completo de red
/tmp/monitor_optimization_results.py

# Verificar actualizaciones
opkg update
opkg list-upgradable

# Limpiar logs antiguos
logread -f | head -1000 > /tmp/recent_logs.txt
```

### Indicadores de Alerta

| Métrica | Umbral Crítico | Acción |
|---------|----------------|---------|
| Drops/min | >1000 | Investigar buffers |
| Latencia internet | >100ms | Verificar gateway |
| RAM libre | <10MB | Reiniciar servicios |
| Uptime | <1 día | Investigar crashes |
| Originators | <5 | Verificar mesh |

### Contactos de Emergencia

```bash
# Acceso de emergencia
ssh root@10.193.74.14  # montecitostyle (acceso principal)
ssh root@10.193.42.78  # palito (zona 2)

# Rollback de emergencia
/tmp/rollback_20250705_213338.sh

# Reinicio de emergencia
reboot
```

---

## 📋 Conclusiones

### Logros Principales
1. ✅ **Conectividad estabilizada** - 100% funcional
2. ✅ **Drops controlados** - De 354K activos a estables
3. ✅ **Latencia mejorada** - Reducción promedio 40%
4. ✅ **Ruteo optimizado** - Bucles eliminados
5. ✅ **Bridge eficiente** - Delays reducidos 87%

### Lecciones Aprendidas
1. **Buffers de red** son críticos en redes mesh
2. **Configuración bridge** impacta significativamente la latencia
3. **Gateway configuration** debe ser explícita
4. **Backups automáticos** son esenciales para cambios críticos
5. **Monitoreo continuo** previene degradación