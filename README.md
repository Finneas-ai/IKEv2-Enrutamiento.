# VPN Site-to-Site IKEv2 basada en Enrutamiento (VTI)

Video de referencia: https://youtu.be/l3-B7s3R65U

## Descripción

Versión route-based (VTI) del escenario anterior, migrada a **IKEv2**. Se combina `crypto ikev2 profile` con un `crypto ipsec profile` (VTI-PROF2) aplicado directamente a la interfaz `Tunnel0`, sin necesidad de ACL ni crypto map.

## Topología

```
LAN A (10.6.82.0/25) -- R1 --Tunnel0 (30.6.82.0/30)-- ISP -- R2 --Tunnel0-- LAN B (10.6.82.128/25)
```

## Direccionamiento IP

| Dispositivo | Interfaz | IP / Máscara            |
|-------------|----------|--------------------------|
| ISP         | e0/0     | 20.6.82.1 /30            |
| ISP         | e0/1     | 20.6.82.5 /30            |
| R1 (Peer A) | e0/0     | 20.6.82.2 /30            |
| R1 (Peer A) | e0/1     | 10.6.82.1 /25 (LAN A)    |
| R1 (Peer A) | Tunnel0  | 30.6.82.1 /30            |
| R2 (Peer B) | e0/0     | 20.6.82.6 /30            |
| R2 (Peer B) | e0/1     | 10.6.82.129 /25 (LAN B)  |
| R2 (Peer B) | Tunnel0  | 30.6.82.2 /30            |
| PC1 (VPCS)  | -        | 10.6.82.10/25, gw 10.6.82.1   |
| PC2 (VPCS)  | -        | 10.6.82.138/25, gw 10.6.82.129 |

## Parámetros IKEv2 / IPsec

| Parámetro   | Valor                              |
|-------------|-------------------------------------|
| encryption  | aes-cbc-256                        |
| integrity   | sha256                             |
| group (DH)  | 14                                  |
| keyring PSK | CLAVE0682                          |
| ipsec profile | VTI-PROF2 (transform-set + ikev2-profile) |

## Configuración

### ISP

```
interface e0/0
 ip address 20.6.82.1 255.255.255.252
 no shut
interface e0/1
 ip address 20.6.82.5 255.255.255.252
 no shut
```

### R1 (Peer A)

```
hostname R1

interface e0/0
 ip address 20.6.82.2 255.255.255.252
 no shutdown
interface e0/1
 ip address 10.6.82.1 255.255.255.128
 no shutdown
ip route 0.0.0.0 0.0.0.0 20.6.82.1

crypto ikev2 proposal PROP0682
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL0682
 proposal PROP0682

crypto ikev2 keyring KEY0682
 peer R2
  address 20.6.82.6
  pre-shared-key CLAVE0682

crypto ikev2 profile PROF0682
 match identity remote address 20.6.82.6 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KEY0682

crypto ipsec transform-set TS0682 esp-aes 256 esp-sha256-hmac
 mode tunnel

crypto ipsec profile VTI-PROF2
 set transform-set TS0682
 set ikev2-profile PROF0682

interface Tunnel0
 ip address 30.6.82.1 255.255.255.252
 tunnel source e0/0
 tunnel destination 20.6.82.6
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile VTI-PROF2

ip route 10.6.82.128 255.255.255.128 Tunnel0
```

### R2 (Peer B)

```
hostname R2

interface e0/0
 ip address 20.6.82.6 255.255.255.252
 no shutdown
interface e0/1
 ip address 10.6.82.129 255.255.255.128
 no shutdown
ip route 0.0.0.0 0.0.0.0 20.6.82.5

crypto ikev2 proposal PROP0682
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL0682
 proposal PROP0682

crypto ikev2 keyring KEY0682
 peer R1
  address 20.6.82.2
  pre-shared-key CLAVE0682

crypto ikev2 profile PROF0682
 match identity remote address 20.6.82.2 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KEY0682

crypto ipsec transform-set TS0682 esp-aes 256 esp-sha256-hmac
 mode tunnel

crypto ipsec profile VTI-PROF2
 set transform-set TS0682
 set ikev2-profile PROF0682

interface Tunnel0
 ip address 30.6.82.2 255.255.255.252
 tunnel source e0/0
 tunnel destination 20.6.82.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile VTI-PROF2

ip route 10.6.82.0 255.255.255.128 Tunnel0
```

### Hosts finales (VPCS)

```
PC1: ip 10.6.82.10/25 10.6.82.1
PC2: ip 10.6.82.138/25 10.6.82.129
```

## Verificación

```
show interface tunnel 0
show crypto ikev2 sa
show crypto ipsec sa
show ip route
ping 10.6.82.138 source 10.6.82.10
```
