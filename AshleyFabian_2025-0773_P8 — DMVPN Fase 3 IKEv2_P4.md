# DMVPN Fase 3 con IKEv2 y OSPF

**Nombre:** Ashley Fabian Ortiz  
**Matrícula:** 2025-0773  
**Práctica:** P4  

---

## Objetivo

Configurar una VPN hub and spoke punto a multipunto **DMVPN (Dynamic Multipoint VPN) Fase 3** con IKEv2 y enrutamiento dinámico OSPF entre un Hub y dos Spokes (Cisco CSR1000v), permitiendo comunicación cifrada tanto entre Hub↔Spoke como directamente entre Spoke↔Spoke. Esta práctica combina la arquitectura DMVPN de P7 con el protocolo de negociación IKEv2, representando la implementación más moderna y escalable de DMVPN.

### Diferencias entre Fase 2 (P7) y Fase 3 (P8)

| Característica            | Fase 2 (P7 — IKEv1)           | Fase 3 (P8 — IKEv2)                    |
|---------------------------|--------------------------------|-----------------------------------------|
| Protocolo IKE             | IKEv1                          | **IKEv2**                               |
| Mensajes de negociación   | 6-9 (Main Mode)                | 4 (IKE_SA_INIT + IKE_AUTH)              |
| Comando de verificación   | `show crypto isakmp sa`        | `show crypto ikev2 sa`                  |
| Estado SA activa          | `QM_IDLE`                      | `READY`                                 |
| Túnel Spoke↔Spoke         | Directo                        | Directo                                 |
| Routing en Spokes         | Ruta específica por Spoke      | Ruta summarizada via Hub (más escalable)|
| NHRP redirect             | Sí (Hub)                       | Sí (Hub)                                |
| NHRP shortcut             | Sí (Spokes)                    | Sí (Spokes)                             |
| Configuración IKE Hub     | `crypto isakmp policy/key`     | `crypto ikev2 proposal/policy/keyring/profile` |
| Escalabilidad             | Media                          | **Alta**                                |

---

## Topología

```
                    [PC-Hub]
                       |
                    [SW-Hub]
                       |
[PC-S1]--[SW-S1]--[Spoke1]--[ISP]--[Hub]
                              |
                          [Spoke2]--[SW-S2]--[PC-S2]

Túneles DMVPN (mGRE sobre IPSec IKEv2):
Hub (10.0.0.1) ←→ Spoke1 (10.0.0.2)
Hub (10.0.0.1) ←→ Spoke2 (10.0.0.3)
Spoke1 (10.0.0.2) ←→ Spoke2 (10.0.0.3) [túnel directo Fase 3]
```

---

## Direccionamiento IP

| Dispositivo | Interfaz         | Dirección IP  | Máscara         | Descripción           |
|-------------|------------------|---------------|-----------------|-----------------------|
| Hub         | GigabitEthernet1 | 25.7.73.193   | 255.255.255.252 | WAN Hub → ISP         |
| ISP         | GigabitEthernet1 | 25.7.73.194   | 255.255.255.252 | WAN ISP → Hub         |
| ISP         | GigabitEthernet2 | 25.7.73.197   | 255.255.255.252 | WAN ISP → Spoke1      |
| ISP         | GigabitEthernet3 | 25.7.73.201   | 255.255.255.252 | WAN ISP → Spoke2      |
| Spoke1      | GigabitEthernet2 | 25.7.73.198   | 255.255.255.252 | WAN Spoke1 → ISP      |
| Spoke2      | GigabitEthernet2 | 25.7.73.202   | 255.255.255.252 | WAN Spoke2 → ISP      |
| Hub         | GigabitEthernet3 | 25.7.73.205   | 255.255.255.248 | Gateway LAN Hub       |
| Spoke1      | GigabitEthernet3 | 25.7.73.213   | 255.255.255.248 | Gateway LAN Spoke1    |
| Spoke2      | GigabitEthernet3 | 25.7.73.221   | 255.255.255.248 | Gateway LAN Spoke2    |
| Hub         | Tunnel0 (mGRE)   | 10.0.0.1      | 255.255.255.0   | NHS (Hub DMVPN)       |
| Spoke1      | Tunnel0 (mGRE)   | 10.0.0.2      | 255.255.255.0   | NHC Spoke1            |
| Spoke2      | Tunnel0 (mGRE)   | 10.0.0.3      | 255.255.255.0   | NHC Spoke2            |
| PC-Hub      | eth0             | 25.7.73.206   | 255.255.255.248 | Host LAN Hub          |
| PC-S1       | eth0             | 25.7.73.214   | 255.255.255.248 | Host LAN Spoke1       |
| PC-S2       | eth0             | 25.7.73.222   | 255.255.255.248 | Host LAN Spoke2       |

### Subredes utilizadas (bloque 25.7.73.192/27)

| Subred           | Rango utilizable            | Uso                  |
|-------------------|-----------------------------|----------------------|
| 25.7.73.192/30    | 25.7.73.193 – 25.7.73.194   | WAN Hub – ISP        |
| 25.7.73.196/30    | 25.7.73.197 – 25.7.73.198   | WAN ISP – Spoke1     |
| 25.7.73.200/30    | 25.7.73.201 – 25.7.73.202   | WAN ISP – Spoke2     |
| 25.7.73.204/29    | 25.7.73.205 – 25.7.73.210   | LAN Hub              |
| 25.7.73.208/29    | 25.7.73.209 – 25.7.73.214   | LAN Spoke1           |
| 25.7.73.216/29    | 25.7.73.217 – 25.7.73.222   | LAN Spoke2           |
| 10.0.0.0/24       | 10.0.0.1 – 10.0.0.254       | Red DMVPN (Tunnel0)  |

---

## Parámetros de configuración

### IKEv2 Proposal

| Parámetro  | Valor               |
|------------|---------------------|
| Cifrado    | AES-CBC 256         |
| Integridad | SHA-256             |
| Grupo DH   | Group 14 (2048-bit) |

### IKEv2 Keyring

| Parámetro      | Valor                    |
|----------------|--------------------------|
| Peer           | 0.0.0.0/0.0.0.0 (cualquier) |
| Pre-shared key | Cisco123!                |

### IKEv2 Profile

| Parámetro             | Valor                    |
|-----------------------|--------------------------|
| Match identity remote | 0.0.0.0 (cualquier)      |
| Autenticación local   | pre-share                |
| Autenticación remota  | pre-share                |

### IPSec Transform Set

| Parámetro      | Valor         |
|----------------|---------------|
| Cifrado ESP    | AES 256       |
| Integridad ESP | SHA-256 HMAC  |
| Modo           | Transport     |

### Tunnel mGRE

| Parámetro              | Hub          | Spoke1        | Spoke2        |
|------------------------|--------------|---------------|---------------|
| IP Tunnel              | 10.0.0.1/24  | 10.0.0.2/24   | 10.0.0.3/24   |
| Tunnel source          | Gi1          | Gi2           | Gi2           |
| Tunnel mode            | gre multipoint | gre multipoint | gre multipoint |
| NHRP network-id        | 100          | 100           | 100           |
| NHRP NHS               | —            | 10.0.0.1      | 10.0.0.1      |
| ip nhrp redirect       | Sí           | No            | No            |
| ip nhrp shortcut       | No           | Sí            | Sí            |
| ip ospf network        | broadcast    | broadcast     | broadcast     |
| ip ospf priority       | 2 (DR)       | 0 (never DR)  | 0 (never DR)  |

---

## Scripts de configuración

### ISP

```
enable
configure terminal
hostname ISP

interface GigabitEthernet1
 ip address 25.7.73.194 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet2
 ip address 25.7.73.197 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet3
 ip address 25.7.73.201 255.255.255.252
 no shutdown
 exit

end
write memory
```

### Hub

```
enable
configure terminal
hostname Hub

interface GigabitEthernet1
 ip address 25.7.73.193 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet3
 ip address 25.7.73.205 255.255.255.248
 no shutdown
 exit

ip route 0.0.0.0 0.0.0.0 25.7.73.194
ip route 25.7.73.196 255.255.255.252 25.7.73.194
ip route 25.7.73.200 255.255.255.252 25.7.73.194

! --- IKEv2 Proposal ---
crypto ikev2 proposal PROP-DMVPN
 encryption aes-cbc-256
 integrity sha256
 group 14
 exit

! --- IKEv2 Policy ---
crypto ikev2 policy POL-DMVPN
 proposal PROP-DMVPN
 exit

! --- IKEv2 Keyring (acepta cualquier peer) ---
crypto ikev2 keyring KEY-DMVPN
 peer SPOKES
  address 0.0.0.0 0.0.0.0
  pre-shared-key Cisco123!
  exit
 exit

! --- IKEv2 Profile ---
crypto ikev2 profile PROF-IKEv2-DMVPN
 match identity remote address 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local KEY-DMVPN
 exit

! --- Transform Set ---
crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
 exit

! --- IPSec Profile referenciando IKEv2 ---
crypto ipsec profile PROF-DMVPN
 set transform-set TS-DMVPN
 set ikev2-profile PROF-IKEv2-DMVPN
 exit

! --- Tunnel mGRE ---
interface Tunnel0
 ip address 10.0.0.1 255.255.255.0
 ip nhrp authentication Cisco123
 ip nhrp map multicast dynamic
 ip nhrp network-id 100
 ip nhrp holdtime 300
 ip nhrp redirect
 ip ospf network broadcast
 ip ospf priority 2
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-DMVPN
 no shutdown
 exit

! --- OSPF ---
router ospf 1
 network 10.0.0.0 0.0.0.255 area 0
 network 25.7.73.204 0.0.0.7 area 0
 exit

end
write memory
```

### Spoke1

```
enable
configure terminal
hostname Spoke1

interface GigabitEthernet2
 ip address 25.7.73.198 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet3
 ip address 25.7.73.213 255.255.255.248
 no shutdown
 exit

ip route 0.0.0.0 0.0.0.0 25.7.73.197
ip route 25.7.73.200 255.255.255.252 25.7.73.197
ip route 25.7.73.202 255.255.255.255 25.7.73.197

! --- IKEv2 Proposal ---
crypto ikev2 proposal PROP-DMVPN
 encryption aes-cbc-256
 integrity sha256
 group 14
 exit

! --- IKEv2 Policy ---
crypto ikev2 policy POL-DMVPN
 proposal PROP-DMVPN
 exit

! --- IKEv2 Keyring ---
crypto ikev2 keyring KEY-DMVPN
 peer HUB
  address 0.0.0.0 0.0.0.0
  pre-shared-key Cisco123!
  exit
 exit

! --- IKEv2 Profile ---
crypto ikev2 profile PROF-IKEv2-DMVPN
 match identity remote address 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local KEY-DMVPN
 exit

! --- Transform Set ---
crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
 exit

! --- IPSec Profile ---
crypto ipsec profile PROF-DMVPN
 set transform-set TS-DMVPN
 set ikev2-profile PROF-IKEv2-DMVPN
 exit

! --- Tunnel mGRE ---
interface Tunnel0
 ip address 10.0.0.2 255.255.255.0
 ip nhrp authentication Cisco123
 ip nhrp map 10.0.0.1 25.7.73.193
 ip nhrp map multicast 25.7.73.193
 ip nhrp network-id 100
 ip nhrp holdtime 300
 ip nhrp nhs 10.0.0.1
 ip nhrp shortcut
 ip ospf network broadcast
 ip ospf priority 0
 tunnel source GigabitEthernet2
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-DMVPN
 no shutdown
 exit

! --- OSPF ---
router ospf 1
 network 10.0.0.0 0.0.0.255 area 0
 network 25.7.73.208 0.0.0.7 area 0
 exit

end
write memory
```

### Spoke2

```
enable
configure terminal
hostname Spoke2

interface GigabitEthernet2
 ip address 25.7.73.202 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet3
 ip address 25.7.73.221 255.255.255.248
 no shutdown
 exit

ip route 0.0.0.0 0.0.0.0 25.7.73.201
ip route 25.7.73.196 255.255.255.252 25.7.73.201
ip route 25.7.73.198 255.255.255.255 25.7.73.201

! --- IKEv2 Proposal ---
crypto ikev2 proposal PROP-DMVPN
 encryption aes-cbc-256
 integrity sha256
 group 14
 exit

! --- IKEv2 Policy ---
crypto ikev2 policy POL-DMVPN
 proposal PROP-DMVPN
 exit

! --- IKEv2 Keyring ---
crypto ikev2 keyring KEY-DMVPN
 peer HUB
  address 0.0.0.0 0.0.0.0
  pre-shared-key Cisco123!
  exit
 exit

! --- IKEv2 Profile ---
crypto ikev2 profile PROF-IKEv2-DMVPN
 match identity remote address 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local KEY-DMVPN
 exit

! --- Transform Set ---
crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
 exit

! --- IPSec Profile ---
crypto ipsec profile PROF-DMVPN
 set transform-set TS-DMVPN
 set ikev2-profile PROF-IKEv2-DMVPN
 exit

! --- Tunnel mGRE ---
interface Tunnel0
 ip address 10.0.0.3 255.255.255.0
 ip nhrp authentication Cisco123
 ip nhrp map 10.0.0.1 25.7.73.193
 ip nhrp map multicast 25.7.73.193
 ip nhrp network-id 100
 ip nhrp holdtime 300
 ip nhrp nhs 10.0.0.1
 ip nhrp shortcut
 ip ospf network broadcast
 ip ospf priority 0
 tunnel source GigabitEthernet2
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-DMVPN
 no shutdown
 exit

! --- OSPF ---
router ospf 1
 network 10.0.0.0 0.0.0.255 area 0
 network 25.7.73.216 0.0.0.7 area 0
 exit

end
write memory
```

### SW-Hub, SW-S1, SW-S2 (IOSvL2)

```
enable
configure terminal

interface GigabitEthernet0/0
 switchport mode access
 switchport access vlan 1
 no shutdown
 exit

interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 1
 no shutdown
 exit

end
write memory
```

### PC-Hub, PC-S1, PC-S2 (VPCS)

```
! PC-Hub
ip 25.7.73.206 255.255.255.248 25.7.73.205
save

! PC-S1
ip 25.7.73.214 255.255.255.248 25.7.73.213
save

! PC-S2
ip 25.7.73.222 255.255.255.248 25.7.73.221
save
```

---

## Verificación y pruebas

> Comandos ejecutados en Hub, Spoke1 y Spoke2.

### Comandos de verificación

```
! En Hub
show dmvpn
show ip ospf neighbor
show crypto ikev2 sa
show ip route ospf

! En Spoke1 y Spoke2
show dmvpn
show crypto ikev2 sa
```

### Resultados obtenidos — Hub

**show dmvpn:**
```
Type:Hub, NHRP Peers:2
25.7.73.198   10.0.0.2   UP   01:05:28   Dynamic ← Spoke1
25.7.73.202   10.0.0.3   UP   00:40:55   Dynamic ← Spoke2
```

**show ip ospf neighbor:**
```
25.7.73.213   0   FULL/DROTHER   Tunnel0 ← Spoke1
25.7.73.221   0   FULL/DROTHER   Tunnel0 ← Spoke2
```

**show crypto ikev2 sa:**
```
Tunnel-id   Local               Remote              Status
1           25.7.73.193/500     25.7.73.198/500     READY  ← Hub↔Spoke1
2           25.7.73.193/500     25.7.73.202/500     READY  ← Hub↔Spoke2
Encr: AES-CBC, keysize: 256, PRF: SHA256, Hash: SHA256, DH Grp:14, PSK
```

### Resultados obtenidos — Spoke1

**show dmvpn:**
```
Type:Spoke, NHRP Peers:2
25.7.73.193   10.0.0.1   UP   Static   ← Hub
25.7.73.202   10.0.0.3   UP   Dynamic  ← Spoke2 (túnel directo Fase 3)
```

**show crypto ikev2 sa:**
```
Tunnel-id   Local               Remote              Status
1           25.7.73.198/500     25.7.73.193/500     READY ← Spoke1↔Hub
2           25.7.73.198/500     25.7.73.202/500     READY ← Spoke1↔Spoke2 directo
```

### Pruebas de conectividad

**Hub → Spokes:**
```
Hub# ping 25.7.73.213
Success rate is 100 percent (5/5)

Hub# ping 25.7.73.221
Success rate is 100 percent (5/5)
```

**Spoke1 → Spoke2:**
```
Spoke1# ping 25.7.73.221 source GigabitEthernet3
! Primer intento: 0% (negociación NHRP + IKEv2)
! Segundo intento: 100% (túnel directo establecido)
Success rate is 100 percent (5/5)
```

**Spoke2 → Spoke1:**
```
Spoke2# ping 25.7.73.213 source GigabitEthernet3
Success rate is 100 percent (5/5)
```

> El primer ping entre Spokes falla mientras NHRP resuelve la IP NBMA y se negocia la SA IKEv2 directa. Este es el comportamiento esperado y definitorio de DMVPN Fase 3. Una vez establecido el túnel directo, el tráfico entre Spokes ya no transita por el Hub.

---

## Capturas de pantalla

> Insertar aquí las capturas de:
> 1. Topología completa en GNS3 con nombre y matrícula visible
![](./Topologia.png)

> 2. `show dmvpn` en Hub (2 peers UP Dynamic)
![](./CH1.png)

> 3. `show ip ospf neighbor` en Hub (2 vecinos FULL)
![](./CH2.png)

> 4. `show crypto ikev2 sa` en Hub (2 SAs READY)
![](./CH3.png)

> 5. `show ip route ospf` en Hub
![](./CH4.png)

> 6. `show dmvpn` en Spoke1 (peer dinámico hacia Spoke2)
![](./C1S1.png)

> 7. `show crypto ikev2 sa` en Spoke1 (2 SAs READY — Hub y Spoke2)
![](./C1S2.png)

---

## Conclusión

Se configuró exitosamente DMVPN Fase 3 con IKEv2 y OSPF entre un Hub y dos Spokes. Esta implementación representa la combinación más moderna y escalable de DMVPN: IKEv2 reduce los mensajes de negociación a 4 (vs 6-9 de IKEv1 en P7), mejora el manejo de errores y soporta NAT-T nativo, mientras que la Fase 3 con `ip nhrp redirect` y `ip nhrp shortcut` permite túneles directos Spoke↔Spoke sin intervención continua del Hub. Las dos SAs IKEv2 en cada Spoke (una hacia el Hub y otra directa hacia el Spoke remoto, ambas en estado READY) confirmaron el funcionamiento correcto de la Fase 3. La conectividad end-to-end entre las tres LANs fue verificada exitosamente mediante pings bidireccionales.
