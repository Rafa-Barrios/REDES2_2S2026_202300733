# Configuraciones aplicadas por dispositivo — Proyecto 2 (carné 2023300733)

Comandos de configuración aplicados por CLI en cada equipo de Cisco Packet Tracer, en su estado final. Las direcciones y puertos corresponden a las secciones 4 y 5 del manual técnico. Las listas de acceso completas están en la sección 8.3 del manual.

Convenciones del proyecto:

- LACP: el switch multicapa 3650 en `active` y el otro extremo en `passive`.
- Los gateways de cada departamento llevan `ip helper-address 172.16.23.130` (servidor DHCP de AndinaCom).
- Todos los equipos se guardaron con `write memory`.

Índice: [FibraNet](#1-fibranet-gt) · [AndinaCom](#2-andinacom) · [MoonLink](#3-moonlink) · [Servidores y hosts](#4-servidores-y-hosts)

---

## 1. FibraNet GT

### FN-CORE (3650-24PS, raíz del árbol y borde BGP)

```
hostname FN-CORE
no ip domain-lookup
ip routing
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.13.0.1 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/2
 no switchport
 ip address 10.13.0.5 255.255.255.252
 no shutdown
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
 router-id 1.1.1.1
 default-information originate
 network 10.13.0.0 0.0.0.3 area 0
 network 10.13.0.4 0.0.0.3 area 0
router bgp 65001
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 neighbor 192.168.33.2 remote-as 65002
 neighbor 192.168.33.9 remote-as 65003
 network 172.16.13.0 mask 255.255.255.0
```

### FN-R1 (2911)

```
hostname FN-R1
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 10.13.0.2 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 ip address 10.13.0.9 255.255.255.252
 no shutdown
router ospf 1
 router-id 1.1.1.2
 network 10.13.0.0 0.0.0.3 area 0
 network 10.13.0.8 0.0.0.3 area 0
```

### FN-R2 (2911)

```
hostname FN-R2
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 10.13.0.6 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 ip address 10.13.0.13 255.255.255.252
 no shutdown
router ospf 1
 router-id 1.1.1.3
 network 10.13.0.4 0.0.0.3 area 0
 network 10.13.0.12 0.0.0.3 area 0
```

### FN-LEAF-ADM (3650-24PS, gateway de Administración)

```
hostname FN-LEAF-ADM
no ip domain-lookup
ip routing
vlan 10
 name ADMINISTRACION
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.13.0.10 255.255.255.252
 ip access-group ACL_ADM_IN out
 no shutdown
interface range GigabitEthernet1/0/2 - 3
 switchport mode access
 switchport access vlan 10
 channel-group 1 mode active
 no shutdown
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

Lista de acceso `ACL_ADM_IN`: sección 8.3 del manual.

### FN-LEAF-ATC (3650-24PS, gateway de Atención al Cliente)

```
hostname FN-LEAF-ATC
no ip domain-lookup
ip routing
vlan 20
 name ATENCION_CLIENTE
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.13.0.14 255.255.255.252
 ip access-group ACL_ATC_IN out
 no shutdown
interface range GigabitEthernet1/0/2 - 3
 switchport mode access
 switchport access vlan 20
 channel-group 1 mode active
 no shutdown
interface Port-channel1
 switchport mode access
 switchport access vlan 20
interface Vlan20
 ip address 172.16.13.129 255.255.255.128
 ip helper-address 172.16.23.130
 no shutdown
router ospf 1
 router-id 1.1.1.5
 passive-interface Vlan20
 network 10.13.0.12 0.0.0.3 area 0
 network 172.16.13.128 0.0.0.127 area 0
```

Lista de acceso `ACL_ATC_IN`: sección 8.3 del manual.

### SW-ADM (2960-24TT)

```
hostname SW-ADM
no ip domain-lookup
vlan 10
 name ADMINISTRACION
interface range GigabitEthernet0/1 - 2
 switchport mode access
 switchport access vlan 10
 channel-group 1 mode passive
 no shutdown
interface Port-channel1
 switchport mode access
 switchport access vlan 10
interface range FastEthernet0/1 - 3
 switchport mode access
 switchport access vlan 10
```

### SW-ATC (2960-24TT)

```
hostname SW-ATC
no ip domain-lookup
vlan 20
 name ATENCION_CLIENTE
interface range GigabitEthernet0/1 - 2
 switchport mode access
 switchport access vlan 20
 channel-group 1 mode passive
 no shutdown
interface Port-channel1
 switchport mode access
 switchport access vlan 20
interface range FastEthernet0/1 - 3
 switchport mode access
 switchport access vlan 20
```

---

## 2. AndinaCom

### AN-CORE1 (3650-24PS, núcleo y borde BGP)

```
hostname AN-CORE1
no ip domain-lookup
ip routing
interface GigabitEthernet1/0/1
 no switchport
 ip address 172.16.23.145 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/2
 no switchport
 ip address 172.16.23.149 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/3
 no switchport
 ip address 172.16.23.153 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/4
 no switchport
 ip address 172.16.23.165 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/23
 no switchport
 ip address 192.168.33.2 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/24
 no switchport
 ip address 192.168.33.5 255.255.255.252
 no shutdown
ip route 172.16.23.0 255.255.255.0 Null0
ip route 0.0.0.0 0.0.0.0 Null0
router ospf 1
 router-id 2.2.2.1
 default-information originate
 network 172.16.23.144 0.0.0.3 area 0
 network 172.16.23.148 0.0.0.3 area 0
 network 172.16.23.152 0.0.0.3 area 0
 network 172.16.23.164 0.0.0.3 area 0
router bgp 65002
 bgp router-id 2.2.2.1
 bgp log-neighbor-changes
 neighbor 192.168.33.1 remote-as 65001
 neighbor 192.168.33.6 remote-as 65003
 network 172.16.23.0 mask 255.255.255.0
```

### AN-CORE2 (2911)

```
hostname AN-CORE2
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 172.16.23.146 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 ip address 172.16.23.157 255.255.255.252
 no shutdown
interface GigabitEthernet0/2
 ip address 172.16.23.161 255.255.255.252
 no shutdown
router ospf 1
 router-id 2.2.2.2
 network 172.16.23.144 0.0.0.3 area 0
 network 172.16.23.156 0.0.0.3 area 0
 network 172.16.23.160 0.0.0.3 area 0
```

### AN-SRV (2911, router de servicios)

```
hostname AN-SRV
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 172.16.23.166 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 ip address 172.16.23.129 255.255.255.240
 no shutdown
router ospf 1
 router-id 2.2.2.5
 passive-interface GigabitEthernet0/1
 network 172.16.23.164 0.0.0.3 area 0
 network 172.16.23.128 0.0.0.15 area 0
```

### AN-DIST1 (3650-24PS, distribución: HSRP activo y raíz de STP)

```
hostname AN-DIST1
no ip domain-lookup
ip routing
spanning-tree mode rapid-pvst
vlan 10
 name VENTAS
vlan 20
 name FACTURACION
spanning-tree vlan 10,20 priority 4096
interface GigabitEthernet1/0/1
 no switchport
 ip address 172.16.23.150 255.255.255.252
 ip access-group ACL_AND_OUT out
 no shutdown
interface GigabitEthernet1/0/2
 no switchport
 ip address 172.16.23.158 255.255.255.252
 ip access-group ACL_AND_OUT out
 no shutdown
interface range GigabitEthernet1/0/3 - 4
 switchport mode trunk
 channel-group 1 mode active
 no shutdown
interface Port-channel1
 switchport mode trunk
interface range GigabitEthernet1/0/5 - 6
 switchport mode trunk
 channel-group 2 mode active
 no shutdown
interface Port-channel2
 switchport mode trunk
interface Vlan10
 ip address 172.16.23.2 255.255.255.192
 ip helper-address 172.16.23.130
 standby 10 ip 172.16.23.1
 standby 10 priority 110
 standby 10 preempt
 no shutdown
interface Vlan20
 ip address 172.16.23.66 255.255.255.192
 ip helper-address 172.16.23.130
 standby 20 ip 172.16.23.65
 standby 20 priority 110
 standby 20 preempt
 no shutdown
router ospf 1
 router-id 2.2.2.3
 passive-interface Vlan10
 passive-interface Vlan20
 network 172.16.23.148 0.0.0.3 area 0
 network 172.16.23.156 0.0.0.3 area 0
 network 172.16.23.0 0.0.0.63 area 0
 network 172.16.23.64 0.0.0.63 area 0
```

Lista de acceso `ACL_AND_OUT`: sección 8.3 del manual.

### AN-DIST2 (3650-24PS, distribución: HSRP en espera)

```
hostname AN-DIST2
no ip domain-lookup
ip routing
spanning-tree mode rapid-pvst
vlan 10
 name VENTAS
vlan 20
 name FACTURACION
spanning-tree vlan 10,20 priority 8192
interface GigabitEthernet1/0/1
 no switchport
 ip address 172.16.23.154 255.255.255.252
 ip access-group ACL_AND_OUT out
 no shutdown
interface GigabitEthernet1/0/2
 no switchport
 ip address 172.16.23.162 255.255.255.252
 ip access-group ACL_AND_OUT out
 no shutdown
interface range GigabitEthernet1/0/3 - 4
 switchport mode trunk
 channel-group 1 mode passive
 no shutdown
interface Port-channel1
 switchport mode trunk
interface GigabitEthernet1/0/5
 switchport mode trunk
 no shutdown
interface Vlan10
 ip address 172.16.23.3 255.255.255.192
 ip helper-address 172.16.23.130
 standby 10 ip 172.16.23.1
 standby 10 preempt
 no shutdown
interface Vlan20
 ip address 172.16.23.67 255.255.255.192
 ip helper-address 172.16.23.130
 standby 20 ip 172.16.23.65
 standby 20 preempt
 no shutdown
router ospf 1
 router-id 2.2.2.4
 passive-interface Vlan10
 passive-interface Vlan20
 network 172.16.23.152 0.0.0.3 area 0
 network 172.16.23.160 0.0.0.3 area 0
 network 172.16.23.0 0.0.0.63 area 0
 network 172.16.23.64 0.0.0.63 area 0
```

### SW-ACC (2960-24TT, acceso)

```
hostname SW-ACC
no ip domain-lookup
spanning-tree mode rapid-pvst
vlan 10
 name VENTAS
vlan 20
 name FACTURACION
interface range GigabitEthernet0/1 - 2
 switchport mode trunk
 channel-group 2 mode passive
 no shutdown
interface Port-channel2
 switchport mode trunk
interface FastEthernet0/24
 switchport mode trunk
 no shutdown
interface range FastEthernet0/1 - 3
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
interface range FastEthernet0/4 - 5
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
```

---

## 3. MoonLink

### MO-HUB (3650-24PS, centro y borde BGP)

```
hostname MO-HUB
no ip domain-lookup
ip routing
interface GigabitEthernet1/0/1
 no switchport
 ip address 172.16.33.129 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/2
 no switchport
 ip address 172.16.33.133 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/3
 no switchport
 ip address 172.16.33.137 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/4
 no switchport
 ip address 172.16.33.141 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/23
 no switchport
 ip address 192.168.33.6 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/24
 no switchport
 ip address 192.168.33.9 255.255.255.252
 no shutdown
ip route 172.16.33.0 255.255.255.0 Null0
ip route 0.0.0.0 0.0.0.0 Null0
router eigrp 100
 eigrp router-id 3.3.3.1
 no auto-summary
 network 172.16.33.128 0.0.0.3
 network 172.16.33.132 0.0.0.3
 network 172.16.33.136 0.0.0.3
 network 172.16.33.140 0.0.0.3
 redistribute static metric 1000000 10 255 1 1500
router bgp 65003
 bgp router-id 3.3.3.1
 bgp log-neighbor-changes
 neighbor 192.168.33.5 remote-as 65002
 neighbor 192.168.33.10 remote-as 65001
 network 172.16.33.0 mask 255.255.255.0
```

### MO-S1 (3650-24PS, gateway de Soporte)

```
hostname MO-S1
no ip domain-lookup
ip routing
vlan 10
 name SOPORTE
interface GigabitEthernet1/0/1
 no switchport
 ip address 172.16.33.130 255.255.255.252
 ip access-group ACL_SOP_IN out
 no shutdown
interface range GigabitEthernet1/0/2 - 3
 switchport mode access
 switchport access vlan 10
 channel-group 1 mode active
 no shutdown
interface Port-channel1
 switchport mode access
 switchport access vlan 10
interface Vlan10
 ip address 172.16.33.1 255.255.255.192
 ip helper-address 172.16.23.130
 no shutdown
router eigrp 100
 eigrp router-id 3.3.3.2
 no auto-summary
 passive-interface Vlan10
 network 172.16.33.128 0.0.0.3
 network 172.16.33.0 0.0.0.63
```

Lista de acceso `ACL_SOP_IN`: sección 8.3 del manual.

### MO-S2 (3650-24PS, gateway de Seguridad)

```
hostname MO-S2
no ip domain-lookup
ip routing
vlan 20
 name SEGURIDAD
interface GigabitEthernet1/0/1
 no switchport
 ip address 172.16.33.134 255.255.255.252
 ip access-group ACL_SEG_IN out
 no shutdown
interface range GigabitEthernet1/0/2 - 3
 switchport mode access
 switchport access vlan 20
 channel-group 1 mode active
 no shutdown
interface Port-channel1
 switchport mode access
 switchport access vlan 20
interface Vlan20
 ip address 172.16.33.65 255.255.255.192
 ip helper-address 172.16.23.130
 no shutdown
router eigrp 100
 eigrp router-id 3.3.3.3
 no auto-summary
 passive-interface Vlan20
 network 172.16.33.132 0.0.0.3
 network 172.16.33.64 0.0.0.63
```

Lista de acceso `ACL_SEG_IN`: sección 8.3 del manual.

### MO-S3 (2911)

```
hostname MO-S3
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 172.16.33.138 255.255.255.252
 no shutdown
router eigrp 100
 eigrp router-id 3.3.3.4
 no auto-summary
 network 172.16.33.136 0.0.0.3
```

### MO-S4 (2911)

```
hostname MO-S4
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 172.16.33.142 255.255.255.252
 no shutdown
router eigrp 100
 eigrp router-id 3.3.3.5
 no auto-summary
 network 172.16.33.140 0.0.0.3
```

### SW-SOP (2960-24TT)

```
hostname SW-SOP
no ip domain-lookup
vlan 10
 name SOPORTE
interface range GigabitEthernet0/1 - 2
 switchport mode access
 switchport access vlan 10
 channel-group 1 mode passive
 no shutdown
interface Port-channel1
 switchport mode access
 switchport access vlan 10
interface range FastEthernet0/1 - 3
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
```

### SW-SEG (2960-24TT)

```
hostname SW-SEG
no ip domain-lookup
vlan 20
 name SEGURIDAD
interface range GigabitEthernet0/1 - 2
 switchport mode access
 switchport access vlan 20
 channel-group 1 mode passive
 no shutdown
interface Port-channel1
 switchport mode access
 switchport access vlan 20
interface range FastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
```

### Router inalámbrico (WRT300N, punto de acceso)

| Parámetro | Valor |
|---|---|
| IP de administración | 172.16.33.5 / 255.255.255.192 |
| DHCP interno | Deshabilitado |
| SSID | MoonLink-WiFi |
| Seguridad | WPA2 Personal, cifrado AES |
| Conexión | Puerto LAN (Ethernet 1) a SW-SOP Fa0/3 |

---

## 4. Servidores y hosts

| Dispositivo | Configuración |
|---|---|
| Servidor DNS/HTTP (SW-ADM Fa0/1) | IP estática 172.16.13.126/25, gateway 172.16.13.1, DNS 172.16.13.126. DNS: registro A `www.proyecto2_2023300733.com` → 172.16.13.126. HTTP y HTTPS habilitados con `index.html` del estudiante |
| Servidor DHCP (AN-SRV Gi0/1) | IP estática 172.16.23.130/28, gateway 172.16.23.129, DNS 172.16.13.126. Seis pools: FN-ADM, FN-ATC, AN-VEN, AN-FAC, MO-SOP y MO-SEG (sección 7.2 del manual) |
| Hosts finales (15) | DHCP: PC-ADM1, PC-ADM2, PC-ATC1 a 3, PC-VEN1 a 3, PC-FAC1 y 2, PC-SOP1 y 2, LAPTOP-WIFI, PC-SEG1 y 2 |
