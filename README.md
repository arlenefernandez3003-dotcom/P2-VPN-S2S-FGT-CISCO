# FortiGate 7.6.2 — VPN Site-to-Site con Router Cisco (Infraestructura 2)

### Arlene Fernández Herrera · Matrícula: 2025-0730

**Seguridad de Redes · ITLA**

---

## 🎥 Video Demostrativo

**[▶ Ver video de demostración](https://youtu.be/REEMPLAZAR-CON-TU-ID)**

---

---

## 📋 Tabla de Contenido

1. [Objetivo del Laboratorio](#1-objetivo-del-laboratorio)
2. [Topología y Direccionamiento](#2-topología-y-direccionamiento)
3. [Procedimiento paso a paso](#3-procedimiento-paso-a-paso)
   - [Paso 1. Nube PNET y PC local](#paso-1-nube-pnet-y-pc-local)
   - [Paso 2. Switch de Usuarios (VLAN 10)](#paso-2-switch-de-usuarios-vlan-10)
   - [Paso 3. Router Cisco: configuración base, VLAN 10 y DHCP](#paso-3-router-cisco-configuración-base-vlan-10-y-dhcp)
   - [Paso 4. Acceso inicial del FortiGate (CLI)](#paso-4-acceso-inicial-del-fortigate-cli)
   - [Paso 5. Interfaces del FortiGate (GUI)](#paso-5-interfaces-del-fortigate-gui)
   - [Paso 6. Verificar la conectividad de la nube](#paso-6-verificar-la-conectividad-de-la-nube)
   - [Paso 7. VPN IPsec en el FortiGate (GUI)](#paso-7-vpn-ipsec-en-el-fortigate-gui)
   - [Paso 8. Leer la propuesta de cifrado del túnel](#paso-8-leer-la-propuesta-de-cifrado-del-túnel)
   - [Paso 9. VPN IPsec en el Router Cisco (CLI)](#paso-9-vpn-ipsec-en-el-router-cisco-cli)
   - [Paso 10. Verificar rutas y políticas del FortiGate](#paso-10-verificar-rutas-y-políticas-del-fortigate)
   - [Paso 11. NAT hacia la WAN en ambos equipos](#paso-11-nat-hacia-la-wan-en-ambos-equipos)
   - [Paso 12. Web Server (HTTPS)](#paso-12-web-server-https)
   - [Paso 13. Pruebas de verificación](#paso-13-pruebas-de-verificación)
4. [Capturas de Pantalla](#4-capturas-de-pantalla)
5. [Estructura del Repositorio](#5-estructura-del-repositorio)

---

## 1. Objetivo del Laboratorio

Esta práctica implementa un **túnel VPN Site-to-Site (IPsec)** entre un **Router Cisco** (equipo de red, lado Usuarios) y un **FortiGate v7.6.2** (lado Servidor). Ambos se conectan entre sí y con la PC local a través de una **Nube de PNETLab** en `203.0.113.0/29`, que simula al ISP con IPs públicas.

* El **Usuario** (VLAN 10, `/25`, DHCP, detrás del Router Cisco y un switch) debe poder llegar al **Web Server** (`/28`, HTTPS, detrás del FortiGate) **únicamente a través del túnel VPN**. Se demuestra con `traceroute`, con el túnel activo y caído.
* Ambos equipos tienen **NAT** hacia la WAN. Se comprueba que el tráfico saliente hacia la nube lleva la IP pública de cada equipo y que ese NAT **no** interfiere con el tráfico cifrado de la VPN.

Toda la configuración y demostración del **FortiGate se hace por GUI**. El Router Cisco se configura por CLI. El laboratorio no requiere salida a Internet real.

---

## 2. Topología y Direccionamiento

> LAN internas derivadas de la matrícula **2025-0730** → base `20.25.30.0/24`. Red de la nube: `203.0.113.0/29` (rango reservado para documentación, RFC 5737).

### 2.1 Diagrama de Topología

```
                      ┌───────────────────────────┐
   PC local ──────────┤   Nube PNET (ISP / Cloud) │
   203.0.113.1        │       203.0.113.0/29      │
                      └─────┬───────────────┬─────┘
                            │               │
                     ┌──────┴───────┐ ┌─────┴────────┐
                     │ Router Cisco │ │   FortiGate  │
                     │ e0/0 (WAN)   │ │ port1 (WAN)  │
                     │ 203.0.113.2  │ │ 203.0.113.3  │
                     │ e0/1 (trunk) │ │ port2 (LAN)  │
                     │ └ Gi0/1.10   │ │ 20.25.30.130 │
                     │  20.25.30.2  │ │              │
                     └──────┬───────┘ └─────┬────────┘
                            │ trunk VLAN 10 │ 20.25.30.128/28
                     ┌──────┴───────┐ ┌─────┴────────┐
                     │ SW-USUARIOS  │ │  Web Server  │
                     │ e0/0 trunk   │ │  (Estática)  │
                     │ e0/1 acc.V10 │ │    HTTPS     │
                     └──────┬───────┘ └──────────────┘
                            │ VLAN 10 · 20.25.30.0/25
                     ┌──────┴───────┐
                     │   Usuario    │
                     │   (DHCP)     │
                     └──────────────┘

              ┄┄┄┄┄┄┄┄┄┄┄┄┄ VPN Site-to-Site (IPsec) ┄┄┄┄┄┄┄┄┄┄┄┄┄
                  túnel entre 203.0.113.2 ↔ 203.0.113.3
                  Router Cisco = crypto map (IKEv1)
                  FortiGate    = túnel creado con el VPN Wizard

  Política de comunicación:
  ┌───────────────────────────────────────────────────────────────────┐
  │ Usuario → Web Server : SOLO a través del túnel VPN (IPsec)        │
  │ El tráfico que coincide con la ACL de la VPN nunca sale sin cifrar│
  │ Si el túnel cae, el traceroute deja de completar hacia el servidor│
  │ NAT: el tráfico hacia la nube sale con la IP pública de cada lado,│
  │      excluyendo explícitamente el tráfico VPN                     │
  └───────────────────────────────────────────────────────────────────┘
```

### 2.2 Tabla de Interfaces

**Nube PNET:**

| Elemento | Rol | Dirección IP | Máscara |
|---|---|---|---|
| **Red de la nube** | Segmento compartido PC + Cisco + FortiGate (ISP) | 203.0.113.0 | /29 |
| **PC local** | Acceso a la GUI del FortiGate | 203.0.113.1 | /29 |

**SW-USUARIOS (switch L2):**

| Interfaz | Modo | VLAN | Conectado a |
|---|---|---|---|
| **e0/0** | Trunk (802.1Q) | 10 permitida | Router Cisco `Gi0/1` |
| **e0/1** | Access | 10 | Usuario |

**Router Cisco (lado Usuarios):**

| Interfaz | Rol | Dirección IP | Máscara |
|---|---|---|---|
| **Ethernet0/0** | WAN hacia la Nube | 203.0.113.2 | /29 |
| **Ethernet0/1** | Trunk hacia SW-USUARIOS (sin IP) | — | — |
| **Ethernet0/1.10** | Gateway VLAN 10 (encapsulación dot1Q 10) | 20.25.30.2 | /25 |

**FortiGate (lado Servidor):**

| Interfaz | Alias | Rol | Dirección IP | Máscara |
|---|---|---|---|---|
| **port1** | WAN-NUBE | WAN | 203.0.113.3 | /29 |
| **port2** | LAN-SERVIDOR | LAN | 20.25.30.130 | /28 |

### 2.3 Tabla de Dispositivos

| Dispositivo | Interfaz | Dirección IP | Máscara | Gateway | Método | Rol |
|---|---|---|---|---|---|---|
| **PC local** | Adaptador VMnet | 203.0.113.1 | /29 | — | Estática | Acceso a la GUI del FortiGate |
| **Router Cisco** | e0/0 | 203.0.113.2 | /29 | — | Estática | WAN, extremo local de la VPN |
| **Router Cisco** | e0/1.10 | 20.25.30.2 | /25 | — | Estática | Gateway VLAN 10 y servidor DHCP |
| **FortiGate** | port1 | 203.0.113.3 | /29 | — | Estática | WAN, extremo remoto de la VPN |
| **FortiGate** | port2 | 20.25.30.130 | /28 | — | Estática | Gateway LAN Servidor |
| **SW-USUARIOS** | e0/0 · e0/1 | — | — | — | — | Switch L2: trunk hacia el Cisco, access VLAN 10 al Usuario |
| **Usuario** | eth0 | 20.25.30.3 (rango) | /25 | 20.25.30.2 | **DHCP** | Cliente en VLAN 10 |
| **Web Server** | eth0 | 20.25.30.131 | /28 | 20.25.30.130 | **Estática** | Servidor HTTPS |

> El rango DHCP de VLAN 10 es `20.25.30.3 – 20.25.30.126`. En la red de la nube no hay gateway: los tres equipos están en el mismo segmento `/29` y no se necesita salida a Internet real.

---

## 3. Procedimiento paso a paso

Los pasos están en el orden en que se ejecutan. Cada uno depende de los anteriores.

> Los nombres de interfaz del Cisco (`Ethernet0/0`, `Ethernet0/1`) deben ajustarse a los que muestre `show ip interface brief` en la imagen usada en PNETLab.

---

### Paso 1. Nube PNET y PC local

Un nodo **Cloud** de PNETLab conecta `e0/0` del Router Cisco, `port1` del FortiGate y el adaptador virtual de la PC local, todos en `203.0.113.0/29`.

**Adaptador de la PC** (el que usa la VM de PNETLab, por ejemplo VMnet8 o Host-only):

| Campo | Valor |
|---|---|
| IP | `203.0.113.1` |
| Máscara | `255.255.255.248` |
| Gateway | *(vacío)* |

**En PNETLab:**

1. Clic derecho en el área de trabajo → `Add an object → Network`.
2. Type: `Management(Cloud0)`, nombre `Nube-PNET`.
3. Conectar `e0/0` del Router Cisco a `Nube-PNET`.
4. Conectar `port1` del FortiGate a `Nube-PNET`.
5. Conectar `e0/1` del Router Cisco a `e0/0` de `SW-USUARIOS` (Paso 2); `e0/1` del switch va al Usuario.
6. Conectar `port2` del FortiGate al Web Server.

---

### Paso 2. Switch de Usuarios (VLAN 10)

El puerto hacia el Router Cisco es un **trunk 802.1Q** y el puerto del Usuario es un **access en VLAN 10**. Consola del switch (script: [`scripts/sw-usuarios.txt`](scripts/sw-usuarios.txt)):

```bash
enable
configure terminal

hostname SW-USUARIOS

vlan 10
 name USUARIOS
exit

interface Ethernet0/0
 description Trunk hacia Router Cisco Gi0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10
 no shutdown
exit

interface Ethernet0/1
 description Usuario - VLAN 10
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
exit

end
write memory
```

**Verificación:**
```bash
show vlan brief
show interfaces trunk
```
Debe mostrar VLAN 10 `USUARIOS` con `Et0/1` y el trunk `Et0/0` activo con VLAN 10 permitida.

> Ver evidencia: [01_switch_vlan10.png](screenshots/01_switch_vlan10.png)

---

### Paso 3. Router Cisco: configuración base, VLAN 10 y DHCP

El Router Cisco es el gateway de la VLAN 10 (router-on-a-stick) y el servidor DHCP de los Usuarios. Consola del router (script: [`scripts/cisco-base.txt`](scripts/cisco-base.txt)):

```bash
enable
configure terminal

hostname R-CISCO
no ip domain-lookup

interface Ethernet0/0
 description WAN hacia la Nube PNET
 ip address 203.0.113.2 255.255.255.248
 no shutdown
exit

interface Ethernet0/1
 description Trunk hacia SW-USUARIOS
 no ip address
 no shutdown
exit

interface Ethernet0/1.10
 description VLAN 10 - Usuarios
 encapsulation dot1Q 10
 ip address 20.25.30.2 255.255.255.128
exit

ip dhcp excluded-address 20.25.30.1 20.25.30.2

ip dhcp pool USUARIOS-V10
 network 20.25.30.0 255.255.255.128
 default-router 20.25.30.2
 dns-server 8.8.8.8 8.8.4.4
 lease 1
exit

end
write memory
```

**Verificación:**
```bash
show ip interface brief
show ip dhcp pool
```
`e0/0` y `e0/1.10` deben estar `up/up`. El Usuario debe recibir una IP del rango `20.25.30.3 – 20.25.30.126` con gateway `20.25.30.2`.

> Ver evidencia: [02_cisco_interfaces.png](screenshots/02_cisco_interfaces.png), [03_cisco_dhcp.png](screenshots/03_cisco_dhcp.png)

---

### Paso 4. Acceso inicial del FortiGate (CLI)

Desde la consola del FortiGate (script: [`scripts/fortigate-cli.txt`](scripts/fortigate-cli.txt)):

```bash
config system interface
    edit "port1"
        set mode static
        set ip 203.0.113.3 255.255.255.248
        set allowaccess https ssh ping
        set role wan
    next
end
```

Acceder desde el navegador de la PC local a `https://203.0.113.3` con las credenciales por defecto (`admin` / contraseña vacía) y definir una contraseña segura.

> Ver evidencia: [04_cli_acceso_fortigate.png](screenshots/04_cli_acceso_fortigate.png)

---

### Paso 5. Interfaces del FortiGate (GUI)

**Ruta:** `Network → Interfaces`

**port1 — WAN-NUBE** (ya tiene IP desde el Paso 4; se completa el resto):

| Campo | Valor |
|---|---|
| Alias | `WAN-NUBE` |
| Role | `WAN` |
| Addressing mode | `Manual` |
| IP/Netmask | `203.0.113.3 / 255.255.255.248` |
| Administrative access | `HTTPS, SSH, Ping` |

**port2 — LAN-SERVIDOR:**

| Campo | Valor |
|---|---|
| Alias | `LAN-SERVIDOR` |
| Role | `LAN` |
| Addressing mode | `Manual` |
| IP/Netmask | `20.25.30.130 / 255.255.255.240` |
| Administrative access | `Ping` |

> Ver evidencia: [05_interfaces_fortigate.png](screenshots/05_interfaces_fortigate.png)

---

### Paso 6. Verificar la conectividad de la nube

Con las dos WAN configuradas, comprobar que la nube da conectividad real **antes** de crear la VPN.

Desde la PC local (CMD o PowerShell):
```
ping 203.0.113.2
ping 203.0.113.3
```

Desde el Router Cisco:
```bash
ping 203.0.113.3
```

Los tres deben responder. Si el ping de la PC falla, revisar el firewall de Windows (permitir ICMP) y la IP del adaptador de PNETLab (Paso 1).

> Ver evidencia: [06_ping_nube.png](screenshots/06_ping_nube.png)

---

### Paso 7. VPN IPsec en el FortiGate (GUI)
 
**Ruta:** `VPN → VPN Wizard`
 
En 7.6.2 el asistente es una sola pantalla con tres bloques (**VPN Tunnel**, **Remote Site**, **Local FortiGate**). Nombre del túnel: `VPN-to-Cisco`. Se llenan los tres bloques y se pulsa **Next** para ver el resumen antes de **Submit**.
 
**Bloque VPN Tunnel:**
 
| Campo | Valor |
|---|---|
| Authentication method | `Pre-shared key` |
| Pre-shared key | `Lab12345` *(la misma en el Cisco, Paso 9)* |
| IKE | `Version 1` |
| Transport | `Auto` |
| Use Fortinet encapsulation | Desactivado |
| NAT traversal | `Disable` |
 
> **IKE `Version 1`:** el Router Cisco usa el modelo clásico `crypto isakmp` (crypto map), que es IKEv1. Los dos extremos deben hablar la misma versión de IKE.
> **NAT traversal → `Disable`:** ningún equipo está detrás de un dispositivo que haga NAT; ambos están en el mismo segmento `203.0.113.0/29` y se ven con su IP real.
 
**Bloque Remote Site:**
 
| Campo | Valor |
|---|---|
| Remote site device type | `FortiGate` (ícono de FortiGate) |
| Remote site device | `Accessible and static` |
| IP/FQDN | `203.0.113.2` |
| Route this device's internet traffic through the remote site | Desactivado |
| Remote site subnets that can access VPN | `20.25.30.0/25` (red de Usuarios) |
 
> **Ícono de FortiGate:** el ícono solo elige la plantilla del asistente (sus valores por defecto); no cambia el protocolo. En la red el túnel es IPsec estándar y el peer sigue siendo el Router Cisco, que no ve qué ícono se eligió. Con la plantilla de FortiGate el asistente completa el túnel sin errores en 7.6.2.
 
**Bloque Local FortiGate:**
 
| Campo | Valor |
|---|---|
| Outgoing interface that binds to tunnel | `port1` |
| Create and add interface to zone | Activado (valor por defecto) |
| Local interface | `port2` (LAN-SERVIDOR) |
| Local subnets that can access VPN | `20.25.30.128/28` (red del Servidor) |
| Allow remote site's internet traffic through this device | Desactivado |
 
> Ver evidencia: [07_ipsec_tunel_fortigate.png](screenshots/07_ipsec_tunel_fortigate.png), [08_ipsec_remoto_local_fortigate.png](screenshots/08_ipsec_remoto_local_fortigate.png)
 
**Resumen y Submit:** en la pantalla **Review** el asistente lista los objetos que va a crear (grupos de direcciones `VPN-to-Cisco_local` y `VPN-to-Cisco_remote`, la interfaz de Fase 1 y Fase 2, la zona, las dos políticas y la ruta hacia la red remota). Pulsar **Submit** y esperar a que termine **sin mensajes de error**.
 
> El asistente crea todos los objetos en una sola operación. Si aparece un error (por ejemplo `phase1 interface object VPN-to-Cisco already exists`), el túnel quedó a medias: borrarlo desde `VPN → IPsec Tunnels` (primero las políticas `vpn_VPN-to-Cisco_*` en `Policy & Objects → Firewall Policy`) y volver a ejecutar el asistente completo. No completar la Fase 2 a mano por CLI.
 
> Ver evidencia: [09_ipsec_resumen_fortigate.png](screenshots/09_ipsec_resumen_fortigate.png)

---

### Paso 8. Leer la propuesta de cifrado del túnel

El cifrado, el hash y el grupo DH los define el asistente. Para que el Cisco coincida se leen (solo lectura, sin modificar nada) desde la CLI del FortiGate:

```bash
show vpn ipsec phase1-interface VPN-to-Cisco
show vpn ipsec phase2-interface VPN-to-Cisco
```

Anotar estos campos:

| Campo en el FortiGate | Valor esperado | Dónde se usa en el Cisco |
|---|---|---|
| `ike-version` (Fase 1) | `1` | Modelo `crypto isakmp` (IKEv1) |
| `proposal` (Fase 1) | `des-md5 des-sha1` | `crypto isakmp policy` 10 (`md5`) y 20 (`sha`), `encryption des` |
| `dhgrp` (Fase 1) | incluye `5` | `group 5` en las políticas ISAKMP |
| `proposal` (Fase 2) | `des-md5 des-sha1` | `transform-set` `TS-DES-MD5` y `TS-DES-SHA` |
| `pfs` y `dhgrp` (Fase 2) | `pfs enable`, `dhgrp` incluye `5` | `set pfs group5` en el crypto map |
| `src-name` / `dst-name` (Fase 2) | `VPN-to-Cisco_local` / `_remote` | ACL `VPN-TRAFFIC` en espejo (Paso 9) |
 
> **Cifrado bajo (low encryption):** el FortiGate solo admite **DES**, por eso ambas fases proponen `des-md5` y `des-sha1`. También se ve en `VPN → IPsec Tunnels → VPN-to-Cisco → Edit` (*Encryption - authentication* y *Diffie-Hellman groups*). El Cisco se configura al mismo nivel en el Paso 9: DES, MD5/SHA1 y grupo 5, sin AES.

> Ver evidencia: [10_propuesta_fortigate.png](screenshots/10_propuesta_fortigate.png)

---

### Paso 9. VPN IPsec en el Router Cisco (CLI)
 
Configuración clásica **IKEv1 + crypto map** al nivel de cifrado del FortiGate (DES), con la propuesta alineada a la del Paso 8. Se pega un bloque `configure terminal … end` a la vez (script completo: [`scripts/cisco-vpn.txt`](scripts/cisco-vpn.txt)).
 
**9.1 — Fase 1 (ISAKMP) y clave compartida**
 
```bash
enable
configure terminal
 
crypto isakmp policy 10
 encryption des
 hash md5
 authentication pre-share
 group 5
 lifetime 86400
exit
 
crypto isakmp policy 20
 encryption des
 hash sha
 authentication pre-share
 group 5
 lifetime 86400
exit
 
crypto isakmp key Lab12345 address 203.0.113.3
 
end
```
 
**9.2 — Fase 2 (transform-sets) y ACL del tráfico interesante**
 
```bash
configure terminal
 
crypto ipsec transform-set TS-DES-MD5 esp-des esp-md5-hmac
 mode tunnel
exit
 
crypto ipsec transform-set TS-DES-SHA esp-des esp-sha-hmac
 mode tunnel
exit
 
! Origen = red local del Cisco (Usuarios), destino = red del Servidor
ip access-list extended VPN-TRAFFIC
 permit ip 20.25.30.0 0.0.0.127 20.25.30.128 0.0.0.15
exit
 
end
```
 
**9.3 — Crypto map, interfaz WAN y ruta**
 
```bash
configure terminal
 
crypto map CM-VPN 10 ipsec-isakmp
 set peer 203.0.113.3
 set transform-set TS-DES-MD5 TS-DES-SHA
 set pfs group5
 match address VPN-TRAFFIC
exit
 
interface GigabitEthernet0/0
 crypto map CM-VPN
exit
 
! Ruta hacia la red del Servidor a través del FortiGate
ip route 20.25.30.128 255.255.255.240 203.0.113.3
 
end
write memory
```
 
> **Por qué dos políticas y dos transform-sets:** el FortiGate ofrece DES con MD5 y con SHA1; el Cisco acepta cualquiera de las dos combinaciones y el túnel usa la que coincida.
> **ACL en espejo:** el FortiGate declara Local `20.25.30.128/28` y Remote `20.25.30.0/25`; el `permit` del Cisco tiene origen y destino invertidos respecto a eso (`20.25.30.0/25` → `20.25.30.128/28`).
> **Ruta estática:** sin ella el router no tiene por dónde sacar el tráfico hacia `20.25.30.128/28` y el crypto map nunca se activa. Como el tráfico que coincide con `VPN-TRAFFIC` solo puede salir cifrado, si no hay túnel se descarta y nunca viaja en claro.
> **PFS:** `set pfs group5` usa un grupo incluido en el `dhgrp` de la Fase 2. Si el Paso 8 muestra `set pfs disable`, quitar esa línea; si el `dhgrp` no incluye el `5`, usar el grupo que muestre.
 
**Verificación:**
```bash
show crypto isakmp policy
show crypto map
```
Antes de generar tráfico, `show crypto isakmp sa` puede no mostrar nada: el túnel sube con el primer paquete interesante (Paso 13).
 
> Ver evidencia: [11_cisco_crypto_config.png](screenshots/11_cisco_crypto_config.png)

---

### Paso 10. Verificar rutas y políticas del FortiGate

El asistente crea rutas y políticas automáticamente; solo se verifican.

**Rutas — `Network → Static Routes`:** debe existir una ruta hacia `20.25.30.0/25` por la interfaz `VPN-to-Cisco`.

> Ver evidencia: [12_rutas_fortigate.png](screenshots/12_rutas_fortigate.png)

**Políticas — `Policy & Objects → Firewall Policy`:** deben existir las dos políticas del túnel, una por sentido, entre `port2 (LAN-SERVIDOR)` y `VPN-to-Cisco`, con `Action: ACCEPT` y **NAT deshabilitado**:

| Política | Sentido | NAT |
|---|---|---|
| `vpn_VPN-to-Cisco_local` | LAN-SERVIDOR → VPN-to-Cisco | ❌ Disabled |
| `vpn_VPN-to-Cisco_remote` | VPN-to-Cisco → LAN-SERVIDOR | ❌ Disabled |

> Ver evidencia: [13_politicas_vpn_fortigate.png](screenshots/13_politicas_vpn_fortigate.png)

---

### Paso 11. NAT hacia la WAN en ambos equipos

Se configura **después** de la VPN y se excluye explícitamente el tráfico cifrado, para que el NAT no interfiera con el túnel.

#### 11.1 FortiGate

**Ruta:** `Policy & Objects → Firewall Policy → Create New`

| Campo | Valor |
|---|---|
| Name | `Servidor-to-WAN` |
| Incoming Interface | `port2 (LAN-SERVIDOR)` |
| Outgoing Interface | `port1 (WAN-NUBE)` |
| Source | `all` |
| Destination | `all` |
| Schedule | `always` |
| Service | `ALL` |
| Action | `ACCEPT` |
| NAT | ✅ Enabled — `Use Outgoing Interface Address` |

> Esta política debe quedar **por debajo** de las políticas del túnel (Paso 10): así el tráfico hacia `20.25.30.0/25` sigue haciendo match primero con la política de la VPN. Si quedó arriba, arrastrarla debajo en la lista.

> Ver evidencia: [14_politica_nat_fortigate.png](screenshots/14_politica_nat_fortigate.png)

#### 11.2 Router Cisco (script: [`scripts/cisco-nat.txt`](scripts/cisco-nat.txt))

```bash
configure terminal

ip access-list extended NAT-NO-VPN
 deny ip 20.25.30.0 0.0.0.127 20.25.30.128 0.0.0.15
 permit ip 20.25.30.0 0.0.0.127 any
exit

interface GigabitEthernet0/0
 ip nat outside
exit

interface GigabitEthernet0/1.10
 ip nat inside
exit

ip nat inside source list NAT-NO-VPN interface GigabitEthernet0/0 overload

end
write memory
```

> El `deny` es la parte importante: le dice al router que **no** haga NAT al tráfico de la red de Usuarios hacia la red del Servidor (debe viajar intacto hacia el crypto map). El `permit` de abajo hace PAT (overload) al resto del tráfico saliente hacia la nube.

> Ver evidencia: [15_cisco_nat_config.png](screenshots/15_cisco_nat_config.png)

---

### Paso 12. Web Server (HTTPS)

Servidor Ubuntu con Apache + certificado autofirmado (script completo: [`scripts/webserver-https.sh`](scripts/webserver-https.sh)):

```bash
sudo apt update && sudo apt install -y apache2 openssl
sudo openssl req -x509 -nodes -days 825 -newkey rsa:2048 \
  -keyout /etc/ssl/private/webserver.key \
  -out /etc/ssl/certs/webserver.crt \
  -subj "/C=DO/ST=SantoDomingo/L=SantoDomingo/O=ITLA/CN=20.25.30.131" \
  -addext "basicConstraints=critical,CA:FALSE" \
  -addext "keyUsage=critical,digitalSignature,keyEncipherment" \
  -addext "subjectAltName=IP:20.25.30.131"
sudo a2enmod ssl
# apuntar SSLCertificateFile / SSLCertificateKeyFile a los archivos generados en default-ssl.conf
sudo a2ensite default-ssl
sudo systemctl restart apache2
```

Direccionamiento estático: `20.25.30.131/28`, gateway `20.25.30.130` (FortiGate, `port2`).

---

### Paso 13. Pruebas de verificación

**13.1 — Con el túnel activo**

Desde el Usuario (VLAN 10, con IP por DHCP):
```
traceroute 20.25.30.131
curl -k https://20.25.30.131/
```
El primer paquete interesante (el que coincide con la ACL `VPN-TRAFFIC`) dispara la negociación IKE; puede tardar 1–2 segundos la primera vez. Debe responder el Web Server.

Comprobar el túnel en ambos equipos:
```bash
show crypto isakmp sa
show crypto ipsec sa
```
`show crypto isakmp sa` debe mostrar `QM_IDLE` y `show crypto ipsec sa` contadores `#pkts encrypt` y `#pkts decrypt` mayores que cero. En el FortiGate: `Monitor → IPsec Monitor` con el túnel `Up`.

> Ver evidencia: [16_traceroute_tunel_activo.png](screenshots/16_traceroute_tunel_activo.png), [17_cisco_crypto_sa.png](screenshots/17_cisco_crypto_sa.png)

**13.2 — Con el túnel caído (no hay ruta alterna)**

En el FortiGate: `VPN → IPsec Tunnels → VPN-to-Cisco → Bring Down`. En el Cisco: `clear crypto isakmp` y `clear crypto sa`. Para que no se restablezca solo, dejar el túnel abajo mientras se prueba (por ejemplo `shutdown` en `Gi0/0` del Cisco).

Repetir `traceroute` y `curl` desde el Usuario: ambos deben **fallar o quedar colgados**. Luego restablecer (`no shutdown` en `Gi0/0` y `Bring Up`) y repetir la prueba para mostrar que se recupera.

> Ver evidencia: [18_traceroute_tunel_caido.png](screenshots/18_traceroute_tunel_caido.png), [19_ipsec_monitor_fortigate.png](screenshots/19_ipsec_monitor_fortigate.png)

**13.3 — Verificación de NAT**

Desde el Usuario, hacer ping a la PC local (permitir ICMP en el firewall de Windows si hace falta):
```
ping 203.0.113.1
```
En el Cisco, `show ip nat translations` debe mostrar la traducción a `203.0.113.2`. Desde el Web Server, `ping 203.0.113.1`: en `Log & Report → Forward Traffic` del FortiGate se ve la política `Servidor-to-WAN` con la IP de origen traducida a `203.0.113.3`.

> Ver evidencia: [20_prueba_nat.png](screenshots/20_prueba_nat.png)

**13.4 — Si el túnel no sube**

1. Confirmar que ambos lados usan **IKEv1** y la **misma propuesta** de Fase 1 y Fase 2 (Paso 8).
2. Confirmar que las subredes están **en espejo**: FortiGate Local `20.25.30.128/28` / Remote `20.25.30.0/25`; Cisco ACL `20.25.30.0/25` → `20.25.30.128/28`.
3. Confirmar la misma clave compartida en ambos equipos.
4. Ver la negociación en tiempo real:
   - FortiGate: `diagnose debug application ike -1` y `diagnose debug enable` (apagar con `diagnose debug disable`). `no proposal chosen` indica propuestas distintas; `no policy configured` indica que el túnel quedó a medias y hay que rehacerlo con el asistente.
   - Cisco: `debug crypto isakmp` (apagar con `undebug all`).
5. Limpiar el estado antes de reintentar: `diagnose vpn ike gateway flush name VPN-to-Cisco` en el FortiGate; `clear crypto isakmp` y `clear crypto sa` en el Cisco.

---

## 4. Capturas de Pantalla

Numeradas en el orden en que se toman durante el procedimiento.

| # | Archivo | Paso | Descripción |
|---|---|---|---|
| 01 | [`01_switch_vlan10.png`](screenshots/01_switch_vlan10.png) | 2 | SW-USUARIOS con `show vlan brief` y `show interfaces trunk`. |
| 02 | [`02_cisco_interfaces.png`](screenshots/02_cisco_interfaces.png) | 3 | `show ip interface brief` del Cisco: `Gi0/0` y `Gi0/1.10` en `up/up`. |
| 03 | [`03_cisco_dhcp.png`](screenshots/03_cisco_dhcp.png) | 3 | `show ip dhcp pool` del Cisco, rango `20.25.30.3–126`. |
| 04 | [`04_cli_acceso_fortigate.png`](screenshots/04_cli_acceso_fortigate.png) | 4 | CLI del FortiGate con la config inicial de `port1` (203.0.113.3/29). |
| 05 | [`05_interfaces_fortigate.png`](screenshots/05_interfaces_fortigate.png) | 5 | `Network → Interfaces` del FortiGate: port1 WAN y port2 LAN-SERVIDOR. |
| 06 | [`06_ping_nube.png`](screenshots/06_ping_nube.png) | 6 | Ping desde la PC local a `203.0.113.2` y `203.0.113.3`. |
| 07 | [`07_ipsec_tunel_fortigate.png`](screenshots/07_ipsec_tunel_fortigate.png) | 7 | Asistente 7.6.2, bloque VPN Tunnel (IKE Version 1). |
| 08 | [`08_ipsec_remoto_local_fortigate.png`](screenshots/08_ipsec_remoto_local_fortigate.png) | 7 | Asistente 7.6.2, bloques Remote Site y Local FortiGate. |
| 09 | [`09_ipsec_resumen_fortigate.png`](screenshots/09_ipsec_resumen_fortigate.png) | 7 | Pantalla Review del asistente con los objetos creados. |
| 10 | [`10_propuesta_fortigate.png`](screenshots/10_propuesta_fortigate.png) | 8 | `show vpn ipsec phase1-interface` y `phase2-interface` del FortiGate. |
| 11 | [`11_cisco_crypto_config.png`](screenshots/11_cisco_crypto_config.png) | 9 | Cisco con `crypto isakmp policy`, transform-sets y crypto map. |
| 12 | [`12_rutas_fortigate.png`](screenshots/12_rutas_fortigate.png) | 10 | `Network → Static Routes` con la ruta hacia el túnel. |
| 13 | [`13_politicas_vpn_fortigate.png`](screenshots/13_politicas_vpn_fortigate.png) | 10 | Políticas de firewall de la VPN en el FortiGate. |
| 14 | [`14_politica_nat_fortigate.png`](screenshots/14_politica_nat_fortigate.png) | 11.1 | Política `Servidor-to-WAN` con NAT habilitado, debajo de las de la VPN. |
| 15 | [`15_cisco_nat_config.png`](screenshots/15_cisco_nat_config.png) | 11.2 | Config de NAT del Cisco con la ACL `NAT-NO-VPN`. |
| 16 | [`16_traceroute_tunel_activo.png`](screenshots/16_traceroute_tunel_activo.png) | 13.1 | Traceroute exitoso del Usuario al Web Server con el túnel activo. |
| 17 | [`17_cisco_crypto_sa.png`](screenshots/17_cisco_crypto_sa.png) | 13.1 | `show crypto isakmp sa` y `show crypto ipsec sa` del Cisco. |
| 18 | [`18_traceroute_tunel_caido.png`](screenshots/18_traceroute_tunel_caido.png) | 13.2 | Traceroute fallido con el túnel caído. |
| 19 | [`19_ipsec_monitor_fortigate.png`](screenshots/19_ipsec_monitor_fortigate.png) | 13.2 | `Monitor → IPsec Monitor` con el túnel `Up` y luego `Down`. |
| 20 | [`20_prueba_nat.png`](screenshots/20_prueba_nat.png) | 13.3 | `show ip nat translations` del Cisco y Forward Traffic del FortiGate. |

---

## 5. Estructura del Repositorio

```
/
├── README.md                  ← este documento
├── screenshots/               ← capturas numeradas de cada configuración
├── scripts/
│   ├── sw-usuarios.txt        ← configuración del switch (VLAN 10, trunk/access)
│   ├── cisco-base.txt         ← interfaces, VLAN 10 y DHCP del router Cisco
│   ├── cisco-vpn.txt          ← ISAKMP + transform-sets + crypto map + ruta
│   ├── cisco-nat.txt          ← NAT del Cisco con exclusión del tráfico VPN
│   ├── fortigate-cli.txt      ← acceso inicial del FortiGate
│   └── webserver-https.sh     ← Apache + certificado autofirmado
├── running-configs/
│   ├── sw-usuarios-running-config.txt
│   ├── cisco-running-config.txt
│   └── fortigate-running-config.conf
```
