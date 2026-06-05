# Ataque STP Root Claim Attack

**Autor:** Roger Rodriguez  
**Matrícula:** 20250757  
**Fecha:** Junio 2026  

---

## Objetivo del laboratorio

Demostrar cómo un atacante puede manipular el protocolo STP (Spanning Tree Protocol)
enviando BPDUs (Bridge Protocol Data Units) falsos con prioridad más baja que el
Root Bridge legítimo, intentando que los switches lo elijan como nuevo Root Bridge
y así alterar la topología de la red para interceptar o interrumpir el tráfico.

---

## Objetivo del script

El script `stp_root.py` realiza las siguientes acciones:

1. Construye BPDUs falsos con prioridad extremadamente baja (0x0001)
2. Los envía al multicast STP `01:80:c2:00:00:00`
3. Intenta convencer a los switches de que el atacante es el nuevo Root Bridge
4. Si tiene éxito altera el árbol de spanning tree y redirige el tráfico

---

## Parámetros usados

| Parámetro | Valor | Descripción |
|---|---|---|
| `INTERFAZ` | `eth0` | Interfaz de red del atacante |
| `PAQUETES` | `100` | Cantidad de BPDUs falsos enviados |
| `DELAY` | `0.1` | Tiempo entre BPDUs (100ms) |
| MAC destino | `01:80:c2:00:00:00` | Multicast STP IEEE |
| MAC origen | `00:00:00:00:00:01` | MAC falsa con prioridad baja |
| Prioridad | `0x0001` | Prioridad más baja posible |

---

## Requisitos para utilizar la herramienta

### Software
- Kali Linux
- Python 3.x
- Librería Scapy

### Instalación de dependencias
```bash
sudo apt update && sudo apt install python3-scapy -y
```

### Permisos
```bash
sudo python3 stp_root.py
```

---

## Documentación del funcionamiento del script

### ¿Cómo funciona STP Root Claim?

STP elige un Root Bridge basándose en la prioridad del switch (menor = mejor).
Por defecto la prioridad es 32768. Si un atacante envía BPDUs con prioridad
menor (como 1), los switches pueden recalcular el árbol y elegir al atacante
como Root Bridge, alterando todos los caminos de la red.

### Flujo del ataque

```
NORMAL:
SW-CORE (prioridad 32769) es el Root Bridge
Todo el tráfico fluye a través de SW-CORE

CON ATAQUE:
Kali envía BPDUs con prioridad 1
Los switches evalúan: 1 < 32769
Si aceptan los BPDUs → Kali se convierte en Root Bridge
Todo el tráfico se redirige por Kali
```

### Estructura del BPDU falso

```
Ethernet Frame:
  dst: 01:80:c2:00:00:00 (STP multicast)
  src: 00:00:00:00:00:01 (MAC falsa)

LLC Header:
  dsap: 0x42
  ssap: 0x42
  ctrl: 0x03

BPDU Payload:
  Protocol ID:    0x0000
  Version:        0x00
  Type:           0x00 (Configuration BPDU)
  Flags:          0x00
  Root ID:        0x0001 + MAC atacante (prioridad 1)
  Root Path Cost: 0x00000000
  Bridge ID:      0x0001 + MAC atacante
  Port ID:        0x0001
  Message Age:    0x000a
  Max Age:        0x000f
  Hello Time:     0x0014
  Forward Delay:  0x0001
```

### Diagrama del ataque

```
ATTACKER (Kali)         SW-ACCESS-1           SW-CORE
20.25.7.100             (Root actual)
      |                      |                    |
      |--BPDU prio=1-------->|                    |
      |                      | evalua: 1 < 32769  |
      |                      |--BPDU prio=1------->|
      |                      |                    | evalua: 1 < 32769
      |                      |                    |
      |         Switches recalculan topologia STP  |
      |         Kali podria convertirse en Root    |
```

### Pasos del script

1. **Construcción BPDU:** Arma el payload con prioridad 1
2. **Frame Ethernet:** Destino multicast STP con MAC falsa
3. **LLC Header:** Headers requeridos por el protocolo STP
4. **Envío:** Manda 100 BPDUs con 100ms de intervalo
5. **Reporte:** Muestra progreso cada 10 BPDUs

---

## Documentación de la red

### Topología

```
                    Router-GW
                   20.25.7.1/24
                        |
                    SW-CORE (Root Bridge)
                   Priority: 32769
                   /              \
            SW-ACCESS-1        SW-ACCESS-2
            /       \               \
        ATTACKER    PC1             PC2
      20.25.7.100  20.25.7.10     20.25.7.20
```

### Interfaces y direccionamiento

| Dispositivo | Interfaz | IP | VLAN |
|---|---|---|---|
| Router-GW | e0/0 | 20.25.7.1/24 | 1 |
| ATTACKER | eth0 | 20.25.7.100/24 | 1 |
| PC1 | eth0 | 20.25.7.10/24 | 1 |
| PC2 | eth0 | 20.25.7.20/24 | 1 |

### Estado STP antes del ataque

```
SW-CORE# show spanning-tree
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     aabb.cc00.2000
             This bridge is the root
```

---

## Ejecución del ataque

```bash
# Clonar el repositorio
git clone https://github.com/tu-usuario/stp-root-attack

# Entrar al directorio
cd stp-root-attack

# Ejecutar el script
sudo python3 stp_root.py
```

### Resultado esperado
```
==================================================
 STP Root Claim Attack
 Autor    : Roger Rodriguez
 Matricula: 20250757
 Interfaz : eth0
 Paquetes : 100
==================================================
[*] Iniciando ataque...
[+] BPDUs enviados: 10/100
[+] BPDUs enviados: 20/100
[+] BPDUs enviados: 30/100
...
[+] BPDUs enviados: 100/100
==================================================
[*] Ataque finalizado
[+] Enviados : 100
[-] Errores  : 0
==================================================
```

### Verificar en SW-CORE durante el ataque
```
SW-CORE# show spanning-tree
SW-CORE# show spanning-tree detail
```

---

## Contra-medida

### Descripción
Configurar **BPDU Guard** en los puertos de acceso para que cualquier
puerto que reciba un BPDU sea deshabilitado automáticamente. Los puertos
de acceso (clientes) nunca deben recibir BPDUs legítimos.

### Implementación en SW-CORE
```
enable
configure terminal
spanning-tree portfast bpduguard default
end
write memory
```

### Implementación en SW-ACCESS-1 y SW-ACCESS-2
```
enable
configure terminal
interface e0/1
 spanning-tree bpduguard enable
interface e0/2
 spanning-tree bpduguard enable
end
write memory
```

### Verificación
```
SW-CORE# show spanning-tree summary
PortFast BPDU Guard Default    is enabled
```

### ¿Por qué funciona?
BPDU Guard detecta cuando un puerto de acceso recibe un BPDU (lo cual
nunca debería ocurrir en un puerto de usuario final) y lo deshabilita
inmediatamente en estado `err-disabled`, bloqueando cualquier intento
de manipulación del árbol STP.

### Recuperar puerto después de BPDU Guard
```
enable
configure terminal
interface e0/1
 shutdown
 no shutdown
end
```

---

## Resumen de todos los ataques realizados

| # | Ataque | Script | Contra-medida |
|---|---|---|---|
| 1 | DoS via CDP | `cdp_dos.py` | `no cdp run` |
| 2 | MitM via ARP | `arp_mitm.py` | `ip arp inspection` |
| 3 | DHCP Spoofing | `dhcp_spoof.py` | `ip dhcp snooping` |
| 4 | DHCP Starvation | `dhcp_starvation.py` | `ip dhcp snooping limit rate` |
| 5 | MAC Flooding | `mac_flooding.py` | `switchport port-security` |
| 6 | STP Root Claim | `stp_root.py` | `spanning-tree bpduguard` |

---

## Capturas de pantalla

| Archivo | Descripción |
|---|---|
| <img width="907" height="497" alt="image" src="https://github.com/user-attachments/assets/c28bf57c-7880-410d-939c-6a4cfd27582d" /> | Topología en EVE-NG |
|<img width="764" height="786" alt="image" src="https://github.com/user-attachments/assets/4b7f2b7d-0480-478d-bed0-34c0ca9312dd" />  | Script corriendo con 100 BPDUs enviados |
| <img width="951" height="814" alt="image" src="https://github.com/user-attachments/assets/e0b23c23-4fa1-40d2-aa01-03955fbd900e" /> | Estado STP en SW-CORE |
|<img width="912" height="498" alt="image" src="https://github.com/user-attachments/assets/9edda326-8ffc-4696-851d-8f08333c3697" /> | BPDU Guard habilitado en SW-CORE |

---

## Referencias

- [STP Root Bridge Attack](https://www.geeksforgeeks.org/stp-root-bridge-election/)
- [Cisco BPDU Guard](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst6500/ios/12-2SX/configuration/guide/book/spantree.html)
- [Scapy Documentation](https://scapy.readthedocs.io/)
