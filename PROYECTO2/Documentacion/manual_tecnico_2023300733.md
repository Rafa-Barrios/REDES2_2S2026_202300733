# Manual técnico — Proyecto 2: Conectividad XXI

**Integración de Operadores Nacionales**

| | |
|---|---|
| Curso | Redes de Computadoras 2 — Segundo Semestre 2026 |
| Universidad | Universidad de San Carlos de Guatemala, Facultad de Ingeniería |
| Carné | 2023300733 |
| Estudiante | Angel Rafael Barrios González|
| Herramientas | Cisco Packet Tracer, Wireshark (en VMware con Ubuntu 24.04), GitHub |

---

## 1. Introducción y objetivos

Se diseña, configura y simula una infraestructura nacional de telecomunicaciones que interconecta **tres operadores (ISP)** mediante **BGP**. Cada operador usa una topología y un protocolo de enrutamiento interno distintos, y comparte servicios centralizados (DNS, DHCP y HTTP/HTTPS). Las reglas de comunicación entre departamentos se implementan con **ACLs extendidas** y se incluye redundancia de capa 3 (HSRP) y agregación de enlaces (LACP).

Objetivos cumplidos:

- Interconectar 3 ISP con BGP (sistemas autónomos 65001, 65002 y 65003).
- Topología en árbol + OSPF (FibraNet GT), jerárquica + OSPF + HSRP (AndinaCom) y hub-and-spoke + EIGRP + Wi-Fi (MoonLink).
- Mínimo 5 equipos de capa 3, 5 hosts y 2 enlaces LACP por ISP.
- Todo el direccionamiento de los equipos finales por DHCP.
- DNS y HTTP/HTTPS operativos desde cualquier punto de la red.
- Reglas de comunicación verificadas con ping y acceso a servicios.
- Captura y análisis de tráfico con Wireshark desde una máquina virtual.

### 1.1 Dónde está cada criterio de la rúbrica

| Criterio | Sección |
|---|---|
| ISP 1, 2 y 3 (topología, protocolo, LACP, subneteo, servicios) | 4, 5, 6 y 7 |
| Conexión inter-ISP (BGP) | 5.4 y 6.4 |
| Reglas de comunicación (ACLs) | 8 y 9 |
| Análisis de tráfico con Wireshark | 10 |
| Análisis en modo simulación | 11 |
| Seguimiento en GitHub | 3 |

---

## 2. Resumen de la solución

| | FibraNet GT | AndinaCom | MoonLink |
|---|---|---|---|
| Sistema autónomo | 65001 | 65002 | 65003 |
| Red asignada | 172.16.13.0/24 | 172.16.23.0/24 | 172.16.33.0/24 |
| Topología | Árbol | Jerárquica (3 capas) | Hub-and-spoke |
| Protocolo interno | OSPF (proceso 1, área 0) | OSPF (proceso 1, área 0) | EIGRP (AS 100) |
| Departamentos | Administración, Atención al Cliente | Ventas, Facturación | Soporte, Seguridad |
| Equipos de capa 3 | 5 | 5 | 5 |
| Enlaces LACP | 2 | 2 | 2 |
| Servicio especial | Servidor DNS y HTTP/HTTPS | Servidor DHCP, HSRP | Red inalámbrica (Wi-Fi) |
| Hosts finales | 5 PC | 5 PC | 4 PC + 1 laptop (Wi-Fi) |

Servidores (con IP estática, no son equipos finales): DNS/HTTP en FibraNet (`172.16.13.126`) y DHCP en AndinaCom (`172.16.23.130`).

---

## 3. Estructura del repositorio(Estrucutra o idea Inicial)

```
Proyecto_2/
├── README.md
├── Packet_Tracer/
│   └── Proyecto2_2023300733.pkt
├── Wireshark/
│   └── captura_wireshark_2023300733.pcapng
└── Documentacion/
    ├── manual_tecnico_2023300733.md
    ├── capturas/            (imágenes de pruebas)
    └── configs/             (running-config de cada)
```

El desarrollo se hizo por fases con commits progresivos (estructura, FibraNet, AndinaCom, MoonLink, BGP, ACLs, Wireshark y documentación).

---

## 4. Direccionamiento y subneteo

### 4.1 Interconexión entre ISP (BGP)

La red `192.168.33.0/24` (XX = 33, últimos dos dígitos del carné) se subnetea en bloques **/30**, la forma más eficiente para enlaces punto a punto (2 direcciones útiles por enlace).

| Enlace | Red | Lado A | Lado B |
|---|---|---|---|
| FibraNet ↔ AndinaCom | 192.168.33.0/30 | FN-CORE `.1` | AN-CORE1 `.2` |
| AndinaCom ↔ MoonLink | 192.168.33.4/30 | AN-CORE1 `.5` | MO-HUB `.6` |
| MoonLink ↔ FibraNet | 192.168.33.8/30 | MO-HUB `.9` | FN-CORE `.10` |

Quedan libres desde `192.168.33.12/30` para crecimiento.

### 4.2 FibraNet GT — `172.16.13.0/24`

Cada departamento debe soportar hasta 75 hosts: 2⁷ − 2 = 126 ≥ 75, por lo que se necesita una máscara **/25** (7 bits de host).

| Departamento | Red | Máscara | Rango útil | Broadcast | Gateway |
|---|---|---|---|---|---|
| Administración (VLAN 10) | 172.16.13.0/25 | 255.255.255.128 | .1 – .126 | .127 | .1 |
| Atención al Cliente (VLAN 20) | 172.16.13.128/25 | 255.255.255.128 | .129 – .254 | .255 | .129 |

El servidor DNS/HTTP usa `172.16.13.126` (estática) dentro de Administración.

Los dos /25 consumen todo el /24, por lo que **no queda espacio para los enlaces entre equipos**. Estos usan el rango privado `10.13.0.0/24` dividido en /30 (no se anuncia por BGP):

| Enlace | Red | Direcciones |
|---|---|---|
| FN-CORE ↔ FN-R1 | 10.13.0.0/30 | CORE `.1`, R1 `.2` |
| FN-CORE ↔ FN-R2 | 10.13.0.4/30 | CORE `.5`, R2 `.6` |
| FN-R1 ↔ FN-LEAF-ADM | 10.13.0.8/30 | R1 `.9`, LEAF-ADM `.10` |
| FN-R2 ↔ FN-LEAF-ATC | 10.13.0.12/30 | R2 `.13`, LEAF-ATC `.14` |

### 4.3 AndinaCom — `172.16.23.0/24`

El tamaño de las subredes es libre. Se usan /26 (62 hosts) para los departamentos, un /28 para servicios y /30 para los enlaces.

| Bloque | Red | Uso |
|---|---|---|
| Ventas (VLAN 10) | 172.16.23.0/26 | VIP HSRP `.1`, AN-DIST1 `.2`, AN-DIST2 `.3` |
| Facturación (VLAN 20) | 172.16.23.64/26 | VIP HSRP `.65`, AN-DIST1 `.66`, AN-DIST2 `.67` |
| Servicios | 172.16.23.128/28 | Gateway (AN-SRV) `.129`, servidor DHCP `.130` |
| Enlaces | 172.16.23.144/28 en adelante, /30 | Ver tabla |

| Enlace | Red | Direcciones |
|---|---|---|
| AN-CORE1 ↔ AN-CORE2 | 172.16.23.144/30 | `.145` / `.146` |
| AN-CORE1 ↔ AN-DIST1 | 172.16.23.148/30 | `.149` / `.150` |
| AN-CORE1 ↔ AN-DIST2 | 172.16.23.152/30 | `.153` / `.154` |
| AN-CORE2 ↔ AN-DIST1 | 172.16.23.156/30 | `.157` / `.158` |
| AN-CORE2 ↔ AN-DIST2 | 172.16.23.160/30 | `.161` / `.162` |
| AN-CORE1 ↔ AN-SRV | 172.16.23.164/30 | `.165` / `.166` |

### 4.4 MoonLink — `172.16.33.0/24`

Cada departamento debe soportar hasta 45 hosts: 2⁶ − 2 = 62 ≥ 45, por lo que la máscara es **/26**.

| Bloque | Red | Máscara | Rango útil | Gateway |
|---|---|---|---|---|
| Soporte (VLAN 10) | 172.16.33.0/26 | 255.255.255.192 | .1 – .62 | .1 (MO-S1) |
| Seguridad (VLAN 20) | 172.16.33.64/26 | 255.255.255.192 | .65 – .126 | .65 (MO-S2) |
| Enlaces | 172.16.33.128/25, en /30 | | | |

| Enlace | Red | Direcciones |
|---|---|---|
| MO-HUB ↔ MO-S1 | 172.16.33.128/30 | `.129` / `.130` |
| MO-HUB ↔ MO-S2 | 172.16.33.132/30 | `.133` / `.134` |
| MO-HUB ↔ MO-S3 | 172.16.33.136/30 | `.137` / `.138` |
| MO-HUB ↔ MO-S4 | 172.16.33.140/30 | `.141` / `.142` |

El router inalámbrico usa `172.16.33.5` (solo administración).

### 4.5 Identificadores de enrutamiento

| ISP | Equipos y router-id |
|---|---|
| FibraNet (OSPF) | FN-CORE `1.1.1.1`, FN-R1 `1.1.1.2`, FN-R2 `1.1.1.3`, FN-LEAF-ADM `1.1.1.4`, FN-LEAF-ATC `1.1.1.5` |
| AndinaCom (OSPF) | AN-CORE1 `2.2.2.1`, AN-CORE2 `2.2.2.2`, AN-DIST1 `2.2.2.3`, AN-DIST2 `2.2.2.4`, AN-SRV `2.2.2.5` |
| MoonLink (EIGRP) | MO-HUB `3.3.3.1`, MO-S1 `3.3.3.2`, MO-S2 `3.3.3.3`, MO-S3 `3.3.3.4`, MO-S4 `3.3.3.5` |

---

## 5. Topologías

### 5.1 FibraNet GT (árbol)

![Topología FibraNet](capturas/F1_01_topologia.png)

| Equipo | Modelo | Rol |
|---|---|---|
| FN-CORE | 3650-24PS | Raíz del árbol y borde BGP |
| FN-R1, FN-R2 | 2911 | Nivel intermedio |
| FN-LEAF-ADM, FN-LEAF-ATC | 3650-24PS | Hojas con la SVI de cada departamento |
| SW-ADM, SW-ATC | 2960-24TT | Acceso |

| Equipo | Puerto | Se conecta a |
|---|---|---|
| FN-CORE | Gi1/0/1 | FN-R1 Gi0/0 |
| FN-CORE | Gi1/0/2 | FN-R2 Gi0/0 |
| FN-R1 | Gi0/1 | FN-LEAF-ADM Gi1/0/1 |
| FN-R2 | Gi0/1 | FN-LEAF-ATC Gi1/0/1 |
| FN-LEAF-ADM | Gi1/0/2 y 1/0/3 | SW-ADM Gi0/1 y Gi0/2 (LACP Po1) |
| FN-LEAF-ATC | Gi1/0/2 y 1/0/3 | SW-ATC Gi0/1 y Gi0/2 (LACP Po1) |
| SW-ADM | Fa0/1, Fa0/2, Fa0/3 | Servidor, PC-ADM1, PC-ADM2 |
| SW-ATC | Fa0/1, Fa0/2, Fa0/3 | PC-ATC1, PC-ATC2, PC-ATC3 |

### 5.2 AndinaCom (jerárquica)

![Topología AndinaCom](capturas/F2_01_topologia.png)

| Capa | Equipos |
|---|---|
| Núcleo | AN-CORE1 (3650, borde BGP), AN-CORE2 (2911) |
| Servicios | AN-SRV (2911) con el servidor DHCP |
| Distribución | AN-DIST1, AN-DIST2 (3650, pareja HSRP) |
| Acceso | SW-ACC (2960) con 5 PC |

| Equipo | Puerto | Se conecta a |
|---|---|---|
| AN-CORE1 | Gi1/0/1 | AN-CORE2 Gi0/0 |
| AN-CORE1 | Gi1/0/2 | AN-DIST1 Gi1/0/1 |
| AN-CORE1 | Gi1/0/3 | AN-DIST2 Gi1/0/1 |
| AN-CORE1 | Gi1/0/4 | AN-SRV Gi0/0 |
| AN-CORE2 | Gi0/1 | AN-DIST1 Gi1/0/2 |
| AN-CORE2 | Gi0/2 | AN-DIST2 Gi1/0/2 |
| AN-DIST1 | Gi1/0/3 y 1/0/4 | AN-DIST2 Gi1/0/3 y 1/0/4 (LACP Po1, troncal) |
| AN-DIST1 | Gi1/0/5 y 1/0/6 | SW-ACC Gi0/1 y Gi0/2 (LACP Po2, troncal) |
| AN-DIST2 | Gi1/0/5 | SW-ACC Fa0/24 (troncal de respaldo) |
| AN-SRV | Gi0/1 | SRV-DHCP |
| SW-ACC | Fa0/1 a Fa0/3 | PC-VEN1, PC-VEN2, PC-VEN3 (VLAN 10) |
| SW-ACC | Fa0/4 y Fa0/5 | PC-FAC1, PC-FAC2 (VLAN 20) |

El enlace de respaldo entre AN-DIST2 y SW-ACC aparece bloqueado por Spanning Tree (punto naranja en Packet Tracer). Es el comportamiento esperado: se activa solo si Po2 o AN-DIST1 fallan.

### 5.3 MoonLink (hub-and-spoke)

![Topología MoonLink](capturas/F3_01_topologia.png)

| Equipo | Modelo | Rol |
|---|---|---|
| MO-HUB | 3650-24PS | Centro y borde BGP |
| MO-S1 | 3650-24PS | Spoke de Soporte (SVI VLAN 10) |
| MO-S2 | 3650-24PS | Spoke de Seguridad (SVI VLAN 20) |
| MO-S3, MO-S4 | 2911 | Spokes adicionales |
| SW-SOP, SW-SEG | 2960-24TT | Acceso |
| Router inalámbrico | WRT300N | Punto de acceso Wi-Fi en la red de Soporte |

| Equipo | Puerto | Se conecta a |
|---|---|---|
| MO-HUB | Gi1/0/1 a Gi1/0/4 | MO-S1, MO-S2, MO-S3 y MO-S4 (Gi1/0/1 en S1 y S2, Gi0/0 en S3 y S4) |
| MO-S1 | Gi1/0/2 y 1/0/3 | SW-SOP Gi0/1 y Gi0/2 (LACP Po1) |
| MO-S2 | Gi1/0/2 y 1/0/3 | SW-SEG Gi0/1 y Gi0/2 (LACP Po1) |
| SW-SOP | Fa0/1, Fa0/2, Fa0/3 | PC-SOP1, PC-SOP2, router inalámbrico (Ethernet 1) |
| SW-SEG | Fa0/1, Fa0/2 | PC-SEG1, PC-SEG2 |

La laptop (LAPTOP-WIFI) se conecta por Wi-Fi al router inalámbrico.

### 5.4 Interconexión entre ISP

![Topología completa](capturas/F4_01_topologia_completa.png)

| Equipo | Puerto | Se conecta a |
|---|---|---|
| FN-CORE | Gi1/0/23 | AN-CORE1 Gi1/0/23 |
| AN-CORE1 | Gi1/0/24 | MO-HUB Gi1/0/23 |
| MO-HUB | Gi1/0/24 | FN-CORE Gi1/0/24 |

---

## 6. Configuraciones

Los comandos aplicados en cada equipo (20 dispositivos de red) están en el archivo configuraciones_aplicadas.md, organizados por ISP y por equipo. A continuación se documentan los comandos clave por rol; los demás equipos del mismo rol siguen el mismo patrón con sus propias direcciones (secciones 4 y 5).

### 6.1 Resumen por dispositivo

| Dispositivo | Rol |
|---|---|
| FN-CORE | Raíz del árbol, borde BGP, origina ruta por defecto en OSPF |
| FN-R1, FN-R2 | Routers OSPF |
| FN-LEAF-ADM, FN-LEAF-ATC | Gateway del departamento, LACP, DHCP relay, ACL |
| SW-ADM, SW-ATC | Acceso, LACP |
| AN-CORE1 | Núcleo, borde BGP, origina ruta por defecto en OSPF |
| AN-CORE2, AN-SRV | Routers OSPF |
| AN-DIST1, AN-DIST2 | Distribución: HSRP, LACP, STP, DHCP relay, ACL |
| SW-ACC | Acceso, LACP, troncales |
| MO-HUB | Centro, borde BGP, ruta por defecto en EIGRP |
| MO-S1, MO-S2 | Gateway del departamento, LACP, DHCP relay, ACL |
| MO-S3, MO-S4 | Routers EIGRP |
| SW-SOP, SW-SEG | Acceso, LACP |

### 6.2 Hoja con departamento: gateway, LACP, DHCP relay y OSPF (FN-LEAF-ADM)

```
hostname FN-LEAF-ADM
ip routing
vlan 10
 name ADMINISTRACION
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.13.0.10 255.255.255.252
 no shutdown
interface range GigabitEthernet1/0/2 - 3
 switchport mode access
 switchport access vlan 10
 channel-group 1 mode active
interface Port-channel1
 switchport mode access
 switchport access vlan 10
interface Vlan10
 ip address 172.16.13.1 255.255.255.128
 ip helper-address 172.16.23.130
 no shutdown
router ospf 1
 router-id 1.1.1.4
 passive-interface Vlan10
 network 10.13.0.8 0.0.0.3 area 0
 network 172.16.13.0 0.0.0.127 area 0
```

FN-LEAF-ATC es igual con VLAN 20, `172.16.13.129/25`, enlace `10.13.0.14/30` y router-id `1.1.1.5`. La ACL se aplica en el puerto `Gi1/0/1` (sección 8).

### 6.3 Switch de acceso (SW-ADM)

```
hostname SW-ADM
vlan 10
 name ADMINISTRACION
interface range GigabitEthernet0/1 - 2
 switchport mode access
 switchport access vlan 10
 channel-group 1 mode passive
interface Port-channel1
 switchport mode access
 switchport access vlan 10
interface range FastEthernet0/1 - 3
 switchport mode access
 switchport access vlan 10
```

**Importante (LACP):** con ambos extremos en `active` el canal no se formó en Packet Tracer; el 3650 queda en `active` y el 2960 en `passive` en todos los canales del proyecto.

### 6.4 Routers de borde: BGP (FN-CORE)

```
hostname FN-CORE
ip routing
interface GigabitEthernet1/0/23
 no switchport
 ip address 192.168.33.1 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/24
 no switchport
 ip address 192.168.33.10 255.255.255.252
 no shutdown
ip route 172.16.13.0 255.255.255.0 Null0
ip route 0.0.0.0 0.0.0.0 Null0
router ospf 1
 default-information originate
router bgp 65001
 bgp router-id 1.1.1.1
 neighbor 192.168.33.2 remote-as 65002
 neighbor 192.168.33.9 remote-as 65003
 network 172.16.13.0 mask 255.255.255.0
```

- La ruta `Null0` del /24 permite que BGP anuncie la red completa aunque internamente esté dividida en /25 (BGP solo anuncia un `network` si esa ruta exacta está en la tabla).
- La ruta por defecto hacia `Null0` más `default-information originate` entrega a los equipos internos una ruta por defecto que apunta al borde, que sí conoce los demás ISP por BGP.

Los otros dos bordes:

| | AN-CORE1 | MO-HUB |
|---|---|---|
| Puertos y direcciones | Gi1/0/23 `192.168.33.2/30`, Gi1/0/24 `192.168.33.5/30` | Gi1/0/23 `192.168.33.6/30`, Gi1/0/24 `192.168.33.9/30` |
| BGP | `router bgp 65002`, router-id `2.2.2.1`, vecinos `.1` (65001) y `.6` (65003), red `172.16.23.0/24` | `router bgp 65003`, router-id `3.3.3.1`, vecinos `.5` (65002) y `.10` (65001), red `172.16.33.0/24` |
| Ruta por defecto interna | `default-information originate` en OSPF | `redistribute static metric 1000000 10 255 1 1500` en EIGRP |

### 6.5 Distribución con HSRP (AN-DIST1)

```
hostname AN-DIST1
ip routing
spanning-tree mode rapid-pvst
vlan 10
 name VENTAS
vlan 20
 name FACTURACION
spanning-tree vlan 10,20 priority 4096
interface range GigabitEthernet1/0/3 - 4
 switchport mode trunk
 channel-group 1 mode active
interface range GigabitEthernet1/0/5 - 6
 switchport mode trunk
 channel-group 2 mode active
interface Vlan10
 ip address 172.16.23.2 255.255.255.192
 ip helper-address 172.16.23.130
 standby 10 ip 172.16.23.1
 standby 10 priority 110
 standby 10 preempt
interface Vlan20
 ip address 172.16.23.66 255.255.255.192
 ip helper-address 172.16.23.130
 standby 20 ip 172.16.23.65
 standby 20 priority 110
 standby 20 preempt
router ospf 1
 router-id 2.2.2.3
 passive-interface Vlan10
 passive-interface Vlan20
```

AN-DIST2 es igual con IP `.3` y `.67`, prioridad HSRP por defecto (100) con `preempt`, prioridad STP 8192 y los puertos de LACP en `passive`. Con esto AN-DIST1 es el gateway activo y raíz de Spanning Tree, y AN-DIST2 queda en espera.

### 6.6 Centro hub-and-spoke (MO-HUB) y spoke (MO-S1)

```
hostname MO-HUB
ip routing
interface GigabitEthernet1/0/1
 no switchport
 ip address 172.16.33.129 255.255.255.252
router eigrp 100
 eigrp router-id 3.3.3.1
 no auto-summary
 network 172.16.33.128 0.0.0.3
 network 172.16.33.132 0.0.0.3
 network 172.16.33.136 0.0.0.3
 network 172.16.33.140 0.0.0.3
 redistribute static metric 1000000 10 255 1 1500
```

```
hostname MO-S1
interface Vlan10
 ip address 172.16.33.1 255.255.255.192
 ip helper-address 172.16.23.130
router eigrp 100
 eigrp router-id 3.3.3.2
 no auto-summary
 passive-interface Vlan10
 network 172.16.33.128 0.0.0.3
 network 172.16.33.0 0.0.0.63
```

### 6.7 Router inalámbrico (punto de acceso)

Se usa como punto de acceso dentro de la red de Soporte, por lo que su DHCP interno está desactivado y los clientes reciben dirección del servidor DHCP central.

| Parámetro | Valor |
|---|---|
| IP de administración | 172.16.33.5 / 255.255.255.192 |
| DHCP interno | Deshabilitado |
| SSID | MoonLink-WiFi |
| Seguridad | WPA2 Personal, cifrado AES |
| Conexión a la red | Puerto LAN (Ethernet 1) hacia SW-SOP Fa0/3 |

---

## 7. Servicios centralizados

### 7.1 DNS y HTTP/HTTPS (FibraNet, `172.16.13.126`)

| Servicio | Configuración |
|---|---|
| DNS | Registro A: `www.proyecto2_2023300733.com` → `172.16.13.126` |
| HTTP y HTTPS | Habilitados; `index.html` con los datos del estudiante y del curso |

![DNS](capturas/F1_12_servidor_dns.png)
![HTTP](capturas/F1_10_web_http.png)

### 7.2 DHCP (AndinaCom, `172.16.23.130`)

Los equipos de los tres ISP reciben dirección mediante **relay** (`ip helper-address 172.16.23.130` en la interfaz de cada gateway). El servidor usa un pool por departamento y entrega siempre el DNS `172.16.13.126`:

| Pool | Gateway | Inicio | Máscara | Máx. |
|---|---|---|---|---|
| FN-ADM | 172.16.13.1 | 172.16.13.10 | 255.255.255.128 | 70 |
| FN-ATC | 172.16.13.129 | 172.16.13.130 | 255.255.255.128 | 70 |
| AN-VEN | 172.16.23.1 (VIP HSRP) | 172.16.23.10 | 255.255.255.192 | 40 |
| AN-FAC | 172.16.23.65 (VIP HSRP) | 172.16.23.70 | 255.255.255.192 | 40 |
| MO-SOP | 172.16.33.1 | 172.16.33.10 | 255.255.255.192 | 40 |
| MO-SEG | 172.16.33.65 | 172.16.33.70 | 255.255.255.192 | 40 |

Los rangos de inicio dejan libres las direcciones de gateway y la `.126` del servidor DNS. Los 15 hosts finales obtienen su dirección por DHCP; solo los servidores y la administración del router inalámbrico son estáticos.

![DHCP en AndinaCom](capturas/F2_10_dhcp_pc_ven1.png)
![DHCP en FibraNet](capturas/F4_10_dhcp_fn_adm.png)

---

## 8. Reglas de comunicación (ACLs extendidas)

### 8.1 Interpretación de la tabla del enunciado

Origen en la fila, destino en la columna (✅ permitido, ❌ bloqueado):

| Origen ↓ / Destino → | ADM | ATC | VEN | FAC | SOP | SEG |
|---|---|---|---|---|---|---|
| **ADM** | – | ✅ | ✅ | ✅ | ✅ | ❌ |
| **ATC** | ✅ | – | ✅ | ❌ | ✅ | ❌ |
| **VEN** | ✅ | ✅ | – | ✅ | ✅ | ❌ |
| **FAC** | ✅ | ❌ | ✅ | – | ✅ | ❌ |
| **SOP** | ✅ | ✅ | ✅ | ✅ | – | ❌ |
| **SEG** | ✅ | ✅ | ✅ | ✅ | ✅ | – |

Protocolos permitidos: ICMP, TCP/80 y TCP/443 entre los pares que lo indica la tabla; entre Atención ↔ Ventas, Facturación ↔ Ventas y Ventas hacia Atención/Facturación solo ICMP y TCP/80.

**Conflicto resuelto.** Las filas de Soporte y Administración dicen "cualquier departamento", pero la fila de Seguridad indica que ningún departamento puede iniciar hacia Seguridad. Se aplicó la regla **más restrictiva**: Soporte → Seguridad y Administración → Seguridad quedan bloqueados.

**"Seguridad inicia".** Seguridad puede iniciar contacto con todos y las respuestas regresan, porque cada ACL permite el tráfico de retorno (`echo-reply` para ICMP y `established` para TCP) antes de las denegaciones.

### 8.2 Dónde y en qué sentido se aplicó cada ACL

Cada ACL filtra el tráfico **originado** en su departamento (por dirección de origen) y se aplica en **sentido saliente (`out`) del puerto de subida** del equipo que sirve de gateway.

| ACL | Equipo | Interfaz | Sentido | Departamento filtrado |
|---|---|---|---|---|
| ACL_ADM_IN | FN-LEAF-ADM | Gi1/0/1 | out | Administración |
| ACL_ATC_IN | FN-LEAF-ATC | Gi1/0/1 | out | Atención al Cliente |
| ACL_AND_OUT | AN-DIST1 y AN-DIST2 | Gi1/0/1 y Gi1/0/2 | out | Ventas y Facturación |
| ACL_SOP_IN | MO-S1 | Gi1/0/1 | out | Soporte |
| ACL_SEG_IN | MO-S2 | Gi1/0/1 | out | Seguridad |

**Justificación:**

- **Filtrar en el origen** evita que tráfico prohibido recorra la red, y hace natural la regla "quién inicia".
- **Puerto de subida, sentido `out`.** Todo lo que el departamento envía hacia otras redes sale por ese puerto. Originalmente se aplicaron `in` en la SVI de cada departamento, pero en este Packet Tracer el `ip access-group` de una interfaz VLAN **no se conserva al guardar y reabrir** el archivo; en un puerto físico routed sí. El sufijo `_IN` de algunos nombres quedó por ese historial.
- **AndinaCom lleva una sola ACL combinada en dos puertos.** Ventas y Facturación comparten los mismos puertos de subida y una interfaz solo admite una ACL por sentido. Se aplica en las dos DIST (si cae la activa, la otra mantiene el filtrado) y en sus dos puertos hacia el núcleo.
- **El tráfico entre Ventas y Facturación** se enruta dentro del mismo equipo de distribución y no pasa por esos puertos; como ese par está permitido, el resultado es el mismo.
- **Infraestructura permitida:** DHCP (UDP 67/68), consultas DNS (UDP 53 hacia `172.16.13.126`) y los protocolos de enrutamiento (`ospf` o `eigrp`).
- **Efecto buscado:** el servidor DHCP no pertenece a ningún departamento, así que no responde a ping desde los hosts; el servicio DHCP sí funciona (por relay).

### 8.3 Listas de acceso aplicadas

**ACL_ADM_IN** (FN-LEAF-ADM, `interface GigabitEthernet1/0/1` → `ip access-group ACL_ADM_IN out`)

```
ip access-list extended ACL_ADM_IN
 permit ospf any any
 permit udp host 172.16.13.1 host 172.16.23.130 eq 67
 permit udp any eq 68 any eq 67
 permit udp host 172.16.13.126 eq 53 any
 permit icmp 172.16.13.0 0.0.0.127 host 172.16.13.1
 permit tcp 172.16.13.0 0.0.0.127 any established
 permit icmp 172.16.13.0 0.0.0.127 any echo-reply
 deny ip 172.16.13.0 0.0.0.127 172.16.33.64 0.0.0.63
 permit icmp 172.16.13.0 0.0.0.127 any
 permit tcp 172.16.13.0 0.0.0.127 any eq 80
 permit tcp 172.16.13.0 0.0.0.127 any eq 443
 deny ip any any
```

**ACL_ATC_IN** (FN-LEAF-ATC, `Gi1/0/1` out)

```
ip access-list extended ACL_ATC_IN
 permit ospf any any
 permit udp host 172.16.13.129 host 172.16.23.130 eq 67
 permit udp any eq 68 any eq 67
 permit udp 172.16.13.128 0.0.0.127 host 172.16.13.126 eq 53
 permit icmp 172.16.13.128 0.0.0.127 host 172.16.13.129
 permit tcp 172.16.13.128 0.0.0.127 any established
 permit icmp 172.16.13.128 0.0.0.127 any echo-reply
 permit icmp 172.16.13.128 0.0.0.127 172.16.13.0 0.0.0.127
 permit tcp 172.16.13.128 0.0.0.127 172.16.13.0 0.0.0.127 eq 80
 permit tcp 172.16.13.128 0.0.0.127 172.16.13.0 0.0.0.127 eq 443
 permit icmp 172.16.13.128 0.0.0.127 172.16.23.0 0.0.0.63
 permit tcp 172.16.13.128 0.0.0.127 172.16.23.0 0.0.0.63 eq 80
 permit icmp 172.16.13.128 0.0.0.127 172.16.33.0 0.0.0.63
 permit tcp 172.16.13.128 0.0.0.127 172.16.33.0 0.0.0.63 eq 80
 permit tcp 172.16.13.128 0.0.0.127 172.16.33.0 0.0.0.63 eq 443
 deny ip any any
```

**ACL_AND_OUT** (AN-DIST1 y AN-DIST2, `Gi1/0/1` y `Gi1/0/2` out)

```
ip access-list extended ACL_AND_OUT
 permit ospf any any
 permit udp 172.16.23.0 0.0.0.127 host 172.16.23.130 eq 67
 permit udp any eq 68 any eq 67
 permit udp 172.16.23.0 0.0.0.127 host 172.16.13.126 eq 53
 permit tcp 172.16.23.0 0.0.0.127 any established
 permit icmp 172.16.23.0 0.0.0.127 any echo-reply
 permit icmp 172.16.23.0 0.0.0.63 172.16.23.64 0.0.0.63
 permit tcp 172.16.23.0 0.0.0.63 172.16.23.64 0.0.0.63 eq 80
 permit icmp 172.16.23.64 0.0.0.63 172.16.23.0 0.0.0.63
 permit tcp 172.16.23.64 0.0.0.63 172.16.23.0 0.0.0.63 eq 80
 permit icmp 172.16.23.0 0.0.0.63 172.16.13.128 0.0.0.127
 permit tcp 172.16.23.0 0.0.0.63 172.16.13.128 0.0.0.127 eq 80
 permit icmp 172.16.23.0 0.0.0.127 172.16.13.0 0.0.0.127
 permit tcp 172.16.23.0 0.0.0.127 172.16.13.0 0.0.0.127 eq 80
 permit tcp 172.16.23.0 0.0.0.127 172.16.13.0 0.0.0.127 eq 443
 permit icmp 172.16.23.0 0.0.0.127 172.16.33.0 0.0.0.63
 permit tcp 172.16.23.0 0.0.0.127 172.16.33.0 0.0.0.63 eq 80
 permit tcp 172.16.23.0 0.0.0.127 172.16.33.0 0.0.0.63 eq 443
 deny ip any any
```

Facturación (`172.16.23.64/26`) no tiene ningún permiso hacia Atención (`172.16.13.128/25`), mientras que Ventas sí.

**ACL_SOP_IN** (MO-S1, `Gi1/0/1` out)

```
ip access-list extended ACL_SOP_IN
 permit eigrp any any
 permit udp host 172.16.33.1 host 172.16.23.130 eq 67
 permit udp any eq 68 any eq 67
 permit udp 172.16.33.0 0.0.0.63 host 172.16.13.126 eq 53
 permit icmp 172.16.33.0 0.0.0.63 host 172.16.33.1
 permit tcp 172.16.33.0 0.0.0.63 any established
 permit icmp 172.16.33.0 0.0.0.63 any echo-reply
 deny ip 172.16.33.0 0.0.0.63 172.16.33.64 0.0.0.63
 permit icmp 172.16.33.0 0.0.0.63 any
 permit tcp 172.16.33.0 0.0.0.63 any eq 80
 permit tcp 172.16.33.0 0.0.0.63 any eq 443
 deny ip any any
```

**ACL_SEG_IN** (MO-S2, `Gi1/0/1` out)

```
ip access-list extended ACL_SEG_IN
 permit eigrp any any
 permit udp host 172.16.33.65 host 172.16.23.130 eq 67
 permit udp any eq 68 any eq 67
 permit udp 172.16.33.64 0.0.0.63 host 172.16.13.126 eq 53
 permit icmp 172.16.33.64 0.0.0.63 host 172.16.33.65
 permit tcp 172.16.33.64 0.0.0.63 any established
 permit icmp 172.16.33.64 0.0.0.63 any
 permit tcp 172.16.33.64 0.0.0.63 any eq 80
 permit tcp 172.16.33.64 0.0.0.63 any eq 443
 deny ip any any
```

![Contadores de ACL](capturas/F5_09_show_access_lists.png)

---

## 9. Pruebas de comunicación

Se usó un host representante por departamento y se comparó con la matriz de la sección 8.1.

| Departamento | Host | IP aproximada (DHCP) |
|---|---|---|
| Administración | PC-ADM1 | 172.16.13.10 |
| Atención al Cliente | PC-ATC1 | 172.16.13.130 |
| Ventas | PC-VEN1 | 172.16.23.10 |
| Facturación | PC-FAC1 | 172.16.23.70 |
| Soporte | PC-SOP1 | 172.16.33.10 |
| Seguridad | PC-SEG1 | 172.16.33.70 |

![Pings desde Administración](capturas/F5_01_pings_adm1.png)
![Pings desde Atención](capturas/F5_02_pings_atc1.png)
![Pings desde Ventas](capturas/F5_03_pings_ven1.png)
![Pings desde Facturación](capturas/F5_04_pings_fac1.png)
![Pings desde Soporte](capturas/F5_05_pings_sop1.png)
![Pings desde Seguridad](capturas/F5_06_pings_seg1.png)

### 9.1 Otras verificaciones

| Prueba | Evidencia |
|---|---|
| Vecinos OSPF en FibraNet | ![](capturas/F1_02_ospf_vecinos_core.png) |
| Rutas OSPF en el core de FibraNet | ![](capturas/F1_03_rutas_ospf_core.png) |
| LACP en FibraNet | ![](capturas/F1_04_lacp_leaf_adm.png) |
| Vecinos OSPF y HSRP en AndinaCom | ![](capturas/F2_02_ospf_vecinos_core1.png) ![](capturas/F2_07_hsrp_dist1.png) |
| Conmutación HSRP | ![](capturas/F2_13_hsrp_ping_falla.png) ![](capturas/F2_14_hsrp_dist2_activo.png) |
| Vecinos EIGRP en MoonLink | ![](capturas/F3_02_eigrp_vecinos_hub.png) |
| Red inalámbrica | ![](capturas/F3_09_wifi_router_ssid.png) |
| Sesiones BGP | ![](capturas/F4_02_bgp_summary_fn.png) |
| Tabla BGP | ![](capturas/F4_05_bgp_tabla_fn.png) |
| Web por HTTPS desde otro ISP | ![](capturas/F5_07_web_desde_seguridad.png) |
| DHCP tras aplicar las ACL | ![](capturas/F5_08_dhcp_renew.png) |

**Prueba de HSRP:** con un `ping -t` continuo desde PC-VEN1, se apagaron las SVI de AN-DIST1; el ping se interrumpió brevemente y se restableció cuando AN-DIST2 pasó a `Active`. Al restaurar AN-DIST1, recuperó el control por `preempt`.

**Prueba de redundancia BGP:** con el enlace directo FibraNet ↔ AndinaCom apagado, la ruta a `172.16.23.0/24` pasó a recorrerse por MoonLink (camino de AS `65003 65002`).

---

## 10. Análisis de tráfico con Wireshark

### 10.1 Entorno de captura

| Dato | Valor |
|---|---|
| Máquina virtual | Ubuntu 24.04 LTS en VMware |
| Red de la VM | **NAT** (VMnet8), la VM usa `192.168.188.129` y DNS/gateway `192.168.188.2` |
| Interfaz capturada | La interfaz de red de la VM (ens33/ens160) |
| Archivo | `Wireshark/captura_wireshark_2023300733.pcapng` |


El tráfico se generó dentro de la VM con estos comandos, en este orden:

```
curl -v http://example.com
nslookup example.com 8.8.8.8
dig @8.8.8.8 example.com A
curl -v https://example.com
ping -c 4 8.8.8.8
```

La captura se filtró (Export Specified Packets) para conservar solo los paquetes de estos cuatro escenarios; se descartó el tráfico ajeno a la prueba (por ejemplo, multidifusión del anfitrión y navegación). Los números de paquete que siguen son los del archivo entregado.

### 10.2 Escenario 1 — Handshake TCP (paquetes 5, 6 y 7)


| Capa | SYN (#5) | SYN-ACK (#6) | ACK (#7) |
|---|---|---|---|
| 2 — Enlace (Ethernet II) | Trama de la VM hacia el gateway virtual de VMware | Trama de regreso | Trama de la VM |
| 3 — Red (IPv4) | 192.168.188.129 → 104.20.23.154, protocolo 6 (TCP), TTL 64 | 104.20.23.154 → 192.168.188.129, TTL 128 | Igual que el SYN |
| 4 — Transporte (TCP) | Puerto 34778 → 80; flags **SYN**; seq 0 (real 2327958096); ventana 64240 | Puerto 80 → 34778; flags **SYN, ACK**; seq 0 (real 1318349535); ack 1 (real 2327958097) | Flags **ACK**; seq 1, ack 1 (reales 2327958097 / 1318349536) |
| 7 — Aplicación | Sin datos | Sin datos | Sin datos |

Los números de secuencia que muestra Wireshark son **relativos** (0 y 1); los valores reales son los indicados entre paréntesis. El ACK final es siempre el número de secuencia del otro extremo más uno, lo que confirma que ambos pueden enviar y recibir. La conexión se cierra después con segmentos FIN (paquetes 12 a 15).

### 10.3 Escenario 2 — Resolución DNS contra 8.8.8.8 (paquetes 16 a 21)

| Capa | Consulta | Respuesta |
|---|---|---|
| 2 — Enlace | Ethernet II | Ethernet II |
| 3 — Red | 192.168.188.129 → 8.8.8.8, protocolo 17 (UDP) | 8.8.8.8 → 192.168.188.129 |
| 4 — Transporte (UDP) | Puerto origen 60825 (nslookup) o 51062 (dig) → destino **53** | Puerto 53 → el puerto de origen de la consulta |
| 7 — Aplicación (DNS) | Pregunta `example.com`, tipo **A** (también AAAA), clase IN, recursión deseada | Respuesta con registros **A** `172.66.147.243` y `104.20.23.154`, **TTL 300 s** (AAAA: `2606:4700:10::6814:179a` y `2606:4700:10::ac42:93f3`) |

El TTL de 300 segundos indica durante cuánto tiempo puede guardarse la respuesta en caché. DNS usa UDP porque la consulta y la respuesta caben en un solo datagrama. Aparte, `curl` resolvió el nombre a través del DNS de VMware (`192.168.188.2`, paquetes 1 a 4, TTL 5 s).

### 10.4 Escenario 3 — HTTP frente a HTTPS

**HTTP (paquetes 8 y 10)**


| Capa | Contenido |
|---|---|
| 2 y 3 | Igual que el escenario 1 (VM ↔ 104.20.23.154) |
| 4 — TCP | Puerto 34778 → **80**, flags PSH, ACK |
| 7 — HTTP | Petición `GET / HTTP/1.1`, `Host: example.com`, `User-Agent: curl/8.5.0`. Respuesta `HTTP/1.1 200 OK`, `Content-Type: text/html; charset=utf-8`, `Server: cloudflare`, `Transfer-Encoding: chunked` |

Todo el contenido es **legible en texto plano**, incluida la página HTML ("Example Domain").

**HTTPS (paquetes 26 a 31)**


| Paso | Detalle |
|---|---|
| Handshake TCP (#26–#28) | 192.168.188.129:54604 → 172.66.147.243:**443** |
| Client Hello (#29) | Indica el nombre del servidor (SNI) `example.com`, ofrece TLS 1.3 y 1.2, 31 suites de cifrado y ALPN (`h2`, `http/1.1`) |
| Server Hello (#31) | Elige **TLS 1.3** y la suite **TLS_AES_256_GCM_SHA384** (0x1302) |
| Después | `Change Cipher Spec` (compatibilidad) y registros que Wireshark etiqueta como **Application Data** |

**Por qué el contenido de HTTPS aparece cifrado:** durante el handshake, cliente y servidor acuerdan claves de sesión mediante un intercambio de claves; desde el Server Hello, todo lo que sigue se cifra con la suite elegida (AES-256 en modo GCM). En TLS 1.3 incluso el certificado y el resto del handshake viajan cifrados, de modo que un observador solo ve el Client Hello, el Server Hello y datos ilegibles. Por eso el filtro `tls.handshake` muestra únicamente esos dos mensajes.

### 10.5 Escenario 4 — ICMP (paquetes 55 a 62)

![ICMP](capturas/F6_06_icmp.png)

| Capa | Echo Request | Echo Reply |
|---|---|---|
| 2 — Enlace | Ethernet II | Ethernet II |
| 3 — Red | 192.168.188.129 → 8.8.8.8, protocolo 1 (ICMP), TTL 64 | 8.8.8.8 → 192.168.188.129, TTL 128 |
| 4 — ICMP | Tipo **8**, código 0, identificador 4669 (0x123d), secuencias 1 a 4 | Tipo **0**, código 0, mismo identificador y secuencia |

ICMP no usa puertos ni TCP/UDP: viaja directamente sobre IP. El tiempo de ida y vuelta fue de unos 30 a 40 ms. Todas las respuestas llegan con TTL 128 aunque provienen de servidores distintos (Google y Cloudflare); lo más probable es que se deba al NAT de VMware, que reenvía los paquetes, aunque no se comprobó.

---

## 11. Análisis en modo simulación de Packet Tracer

En modo simulación se inspeccionaron paquetes de DHCP (DORA), DNS, HTTP/HTTPS, OSPF/EIGRP y BGP usando la ventana *PDU Information* (pestaña *OSI Model*).

| Paquete | Capa 2 | Capa 3 | Capa 4 | Capa 7 |
|---|---|---|---|---|
| DHCP Discover | MAC origen del PC, destino broadcast | 0.0.0.0 → 255.255.255.255 | UDP 68 → 67 | DHCP Discover |
| DHCP Offer / Ack | Unicast (o broadcast) al cliente | Servidor o relay → IP ofrecida | UDP 67 → 68 | IP, máscara, gateway y DNS |
| Relay | MAC del router | Gateway → `172.16.23.130` | UDP 67 → 67 | Reenvío del Discover con la subred de origen |
| DNS consulta / respuesta | MAC del PC y del gateway | PC ↔ `172.16.13.126` | UDP puerto aleatorio ↔ **53** | Registro A de `www.proyecto2_2023300733.com` |
| HTTP / HTTPS | MAC del PC y del gateway | PC ↔ servidor | TCP → **80** / **443** | GET de la página / tráfico TLS |
| OSPF Hello | Multicast `01:00:5E:00:00:05` | Protocolo **89**, destino `224.0.0.5` | No usa TCP ni UDP | Hello: área 0, router-id, intervalo y vecinos |
| EIGRP Hello | Multicast `01:00:5E:00:00:0A` | Protocolo **88**, destino `224.0.0.10` | No usa TCP ni UDP | Hello del AS 100 |
| BGP | MAC de los routers vecinos | Unicast entre IP de `192.168.33.0/24` | TCP **179** | OPEN, KEEPALIVE y UPDATE (redes /24 y camino de AS) |

Los valores exactos pueden variar ligeramente según la versión del simulador.

![DHCP en simulación](capturas/F6_07_sim_dhcp.png)
![DNS o HTTP en simulación](capturas/F6_08_sim_dns_http.png)
![OSPF o EIGRP en simulación](capturas/F6_09_sim_ospf_eigrp.png)
![BGP en simulación](capturas/F6_10_sim_bgp.png)

---

## 12. Problemas encontrados y decisiones de diseño

| Tema | Qué pasó | Decisión |
|---|---|---|
| LACP | Con ambos extremos en `active` el canal quedaba en `SD`/`(s)` | 3650 en `active` y 2960 en `passive` en todos los canales |
| Estado de Packet Tracer | En una ocasión una ruta OSPF no se anunciaba | Guardar, cerrar y reabrir el archivo lo resolvió |
| Direccionamiento de FibraNet | Dos /25 consumen todo el /24 | Enlaces internos en `10.13.0.0/24` (no anunciado por BGP) |
| Fibra óptica | El 3650 de este simulador no permitió conectar con fibra | Enlaces de interconexión con cobre cruzado (Gi1/0/23 y Gi1/0/24) |
| Interconexión `192.168.XX.0/16` | Un /16 no admite "XX" en el tercer octeto | Se usa `192.168.33.0/24` dividida en /30 |
| ACL en interfaces VLAN | El `ip access-group` de las SVI no se conservaba al reabrir | ACL en sentido `out` del puerto de subida (sección 8) |
| Conflicto en la tabla de reglas | Soporte y Administración "con cualquier depto" frente a "nadie inicia hacia Seguridad" | Prevalece la regla más restrictiva (Seguridad) |
| Ping desde routers de FibraNet | Un ping originado en un router usa una IP `10.13.x.x` no anunciada por BGP y no regresa | Los hosts (que usan `172.16.13.x`) no se ven afectados; las pruebas se hacen desde PCs |
| 5 routers por ISP | Los 3650 cuentan como equipos de capa 3 (la rúbrica habla de "routers/switches multicapa") | Cada ISP tiene 5 equipos de capa 3 |

---

## 13. Conclusiones

- Se integraron tres operadores con topologías y protocolos internos distintos bajo un único dominio de enrutamiento BGP, con redundancia en el triángulo de interconexión.
- Los servicios compartidos (DNS, HTTP/HTTPS y DHCP) funcionan desde cualquier ISP gracias al relay DHCP y a las rutas por defecto que reparten los bordes.
- HSRP y LACP aportan continuidad de servicio y mayor capacidad en los puntos críticos.
- Las ACL extendidas permiten implementar la política de comunicación por origen, con retorno de tráfico controlado y sin romper DHCP, DNS ni el enrutamiento.
- El análisis con Wireshark permitió observar capa por capa el handshake TCP, DNS, HTTP frente a HTTPS (TLS 1.3) e ICMP, y entender por qué el contenido de HTTPS no es legible.



