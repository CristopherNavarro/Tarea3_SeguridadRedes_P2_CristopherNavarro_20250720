# Práctica 2 - Infraestructura 2: VPN IPsec Site-to-Site Heterogénea (Cisco 7200 <---> FortiGate)

* **Estudiante:** Cristopher Navarro  
* **Matrícula:** 2025-0720  
* **Asignatura:** Seguridad de Redes  
* **Docente:** Jonathan Esteban Rondón Corniel  
* **Institución:** Instituto Tecnológico de Las Américas (ITLA)  
* **Repositorio GitHub:** https://github.com/CristopherNavarro/Tarea3_SeguridadRedes_P2_CristopherNavarro_20250720  
* **Fecha de Realización:** Octubre 2026  

---

## 1. Enlace al Video Demostrativo en YouTube
* **URL del Video:** https://youtu.be/4TQJ5wT-Vx4  
*(Video explicativo con cámara y micrófono activos, mostrando en pantalla fecha/hora, topología GNS3, CLI de Cisco, Web GUI de FortiOS y pruebas de consola en vivo).*

> **Nota aclaratoria del autor sobre la duración del video:**  
> El video tiene una duración de aproximadamente 10 minutos y 15 segundos, superando por escasos 15 segundos el tiempo límite previsto. Presento mis más sinceras disculpas al profesor por esta ligera variación; la misma obedeció a la rigurosidad técnica de desglosar detalladamente la interoperabilidad criptográfica heterogénea entre Cisco 7200 y FortiGate (asociaciones IKEv1 Fase 1 y Fase 2 en CLI y GUI), así como las pruebas completas de conectividad y enrutamiento sin omitir ningún criterio de la guía.

---

## 2. Propósito y Objetivos del Laboratorio

El propósito fundamental de esta práctica consistió en desplegar, configurar, auditar y certificar una arquitectura de comunicaciones seguras mediante un túnel **VPN IPsec Site-to-Site Heterogéneo**, interconectando dos plataformas perimetrales de fabricantes distintos: un router **Cisco serie 7200** (Cisco IOS Software 15.2) en el Sitio A y un firewall **FortiGate VM64-KVM** (FortiOS 7.0.9) en el Sitio B, atravesando un proveedor de servicios de Internet público simulado (ISP).

### Objetivos Específicos Alcanzados:
1. Diseñar e implementar el direccionamiento IP personalizado derivado estrictamente de mi número de matrícula institucional (`2025-0720`).
2. Configurar la segmentación mediante VLAN 10 (802.1Q) en el Sitio A utilizando subinterfaces en Cisco (`FastEthernet0/1.10`) y habilitar el servicio dinámico DHCP directamente en el router Cisco para clientes de usuario.
3. Superar y sincronizar los desafíos de compatibilidad criptográfica entre Cisco IOS y la licencia de evaluación de FortiOS KVM, logrando una convergencia perfecta en IKEv1 Fase 1 (`QM_IDLE`) y Fase 2 (`esp-des esp-sha256-hmac` con Diffie-Hellman Grupo 14 / PFS).
4. Implementar NAT Exemption mediante listas de acceso en Cisco para evitar que el tráfico corporativo destinado al túnel sea sobrecargado por la regla de NAT hacia Internet.
5. Aplicar políticas de firewall e inspección de estado en FortiGate para autorizar el tráfico del túnel hacia el segmento de servidores.
6. Desplegar un servicio de centro de datos seguro (Servidor Web HTTPS en el puerto 443 con certificado TLS) y validar su consumo exitoso desde la estación de trabajo remota.
7. Ejecutar pruebas de enforzamiento de seguridad y aislamiento mediante caída controlada de la asociación criptográfica en Cisco para verificar que el tráfico no viaje sin cifrar por la red pública.

---

## 3. Topología de Red y Diagrama de Conectividad

La topología fue construida y cableada en GNS3 respaldado por VMware Workstation:

```text
[PC-Usuario] (10.25.7.10/25 - VLAN 10)
      | (eth0)
      | (e1 - Access VLAN 10)
[SW-SiteA]
      | (e0 - 802.1Q Trunk)
      | (f0/1.10)
[R-Cisco] (Sitio A - Cisco 7200 IOS 15.2)
      | (f0/0: 203.25.7.2/29)
      |
      | (f0/0: 203.25.7.1/29)
    [ISP]
      | (f0/1: 203.25.7.9/29)
      |
      | (port1: 203.25.7.10/29)
[FortiGate] (Sitio B - FortiOS 7.0.9)
      | (port2: 10.25.7.129/28)
      | (eth0)
[Servidor-Web] (10.25.7.130/28 - HTTPS 443)
```

---

## 4. Esquema de Direccionamiento IP (Matrícula: 2025-0720)

Aplicando los parámetros exigidos en la guía académica con base en mi matrícula:

| Dispositivo | Interfaz Física / Lógica | Rol / Segmento | Dirección IP | Máscara de Red | Puerta de Enlace |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **PC-Usuario** | `eth0` | LAN Usuarios (VLAN 10) | `10.25.7.10` (DHCP) | `255.255.255.128` (/25) | `10.25.7.1` |
| **SW-SiteA** | `e0` (Trunk) / `e1` (Acceso) | Conmutación Local | N/A | N/A | N/A |
| **R-Cisco** | `FastEthernet0/1.10` | Gateway LAN Sitio A (VLAN 10) | `10.25.7.1` | `255.255.255.128` (/25) | N/A |
| **R-Cisco** | `FastEthernet0/0` | WAN Sitio A | `203.25.7.2` | `255.255.255.248` (/29) | `203.25.7.1` |
| **ISP** | `FastEthernet0/0` | Enlace WAN Sitio A | `203.25.7.1` | `255.255.255.248` (/29) | N/A |
| **ISP** | `FastEthernet0/1` | Enlace WAN Sitio B | `203.25.7.9` | `255.255.255.248` (/29) | N/A |
| **ISP** | `Loopback0` | Simulación DNS/Internet | `8.8.8.8` | `255.255.255.255` (/32) | N/A |
| **FortiGate** | `port1` | WAN Sitio B | `203.25.7.10` | `255.255.255.248` (/29) | `203.25.7.9` |
| **FortiGate** | `port2` | Gateway LAN Sitio B | `10.25.7.129` | `255.255.255.240` (/28) | N/A |
| **FortiGate** | `port3` | Gestión Web GUI | `192.168.6.202` | `255.255.255.0` (/24) | N/A |
| **Servidor-Web** | `eth0` | Servidor Web Seguro | `10.25.7.130` | `255.255.255.240` (/28) | `10.25.7.129` |

---

## 5. Procedimiento de Configuración Implementado

### 5.1. Configuración de R-Cisco (Sitio A - Cisco IOS 15.2)

1. **Subinterfaz LAN VLAN 10 y Servidor DHCP:**
```cisco
hostname R-Cisco
no ip domain-lookup

interface FastEthernet0/1
 no shutdown

interface FastEthernet0/1.10
 encapsulation dot1Q 10
 ip address 10.25.7.1 255.255.255.128
 ip nat inside
 no shutdown

ip dhcp excluded-address 10.25.7.1 10.25.7.9
ip dhcp excluded-address 10.25.7.51 10.25.7.126
ip dhcp pool POOL_VLAN10
 network 10.25.7.0 255.255.255.128
 default-router 10.25.7.1
 dns-server 8.8.8.8
```

2. **Interfaz WAN y Ruta por Defecto:**
```cisco
interface FastEthernet0/0
 description Enlace WAN hacia ISP
 ip address 203.25.7.2 255.255.255.248
 ip nat outside
 no shutdown

ip route 0.0.0.0 0.0.0.0 203.25.7.1
```

3. **Parámetros Criptográficos IPsec (Fase 1 y Fase 2):**
```cisco
crypto isakmp policy 10
 encr des
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key ClaveSegura2025 address 203.25.7.10

crypto ipsec transform-set TS-VPN esp-des esp-sha256-hmac
 mode tunnel

ip access-list extended ACL-VPN
 permit ip 10.25.7.0 0.0.0.127 10.25.7.128 0.0.0.15

ip access-list extended ACL-NAT
 deny ip 10.25.7.0 0.0.0.127 10.25.7.128 0.0.0.15
 permit ip 10.25.7.0 0.0.0.127 any

ip nat inside source list ACL-NAT interface FastEthernet0/0 overload

crypto map VPN-MAP 10 ipsec-isakmp
 set peer 203.25.7.10
 set transform-set TS-VPN
 set pfs group14
 match address ACL-VPN

interface FastEthernet0/0
 crypto map VPN-MAP
```

### 5.2. Configuración de FortiGate (Sitio B - FortiOS 7.0.9)

1. **Interfaces Físicas y Gestión:**
```fortios
config system interface
    edit "port1"
        set mode static
        set ip 203.25.7.10 255.255.255.248
        set allowaccess ping
    next
    edit "port2"
        set mode static
        set ip 10.25.7.129 255.255.255.240
        set allowaccess ping https ssh http
    next
    edit "port3"
        set mode static
        set ip 192.168.6.202 255.255.255.0
        set allowaccess ping https ssh http
    next
end

config router static
    edit 1
        set gateway 203.25.7.9
        set device "port1"
    next
end
```

2. **Túnel VPN IPsec Site-to-Site Simétrico:**
```fortios
config vpn ipsec phase1-interface
    edit "VPN-S2S-FGT"
        set interface "port1"
        set peertype any
        set net-device disable
        set dpd on-idle
        set dhgrp 14
        set proposal des-sha256
        set remote-gw 203.25.7.2
        set psksecret ClaveSegura2025
    next
end

config vpn ipsec phase2-interface
    edit "VPN-S2S-FGT"
        set phase1name "VPN-S2S-FGT"
        set proposal des-sha256
        set dhgrp 14
        set src-subnet 10.25.7.128 255.255.255.240
        set dst-subnet 10.25.7.0 255.255.255.128
        set auto-negotiate enable
    next
end

config router static
    edit 2
        set dst 10.25.7.0 255.255.255.128
        set device "VPN-S2S-FGT"
    next
end
```

3. **Políticas de Firewall:**
```fortios
config firewall address
    edit "LAN_SITE_B"
        set subnet 10.25.7.128 255.255.255.240
    next
    edit "LAN_SITE_A"
        set subnet 10.25.7.0 255.255.255.128
    next
end

config firewall policy
    edit 1
        set name "LAN_to_VPN"
        set srcintf "port2"
        set dstintf "VPN-S2S-FGT"
        set action accept
        set srcaddr "LAN_SITE_B"
        set dstaddr "LAN_SITE_A"
        set schedule "always"
        set service "ALL"
    next
    edit 2
        set name "VPN_to_LAN"
        set srcintf "VPN-S2S-FGT"
        set dstintf "port2"
        set action accept
        set srcaddr "LAN_SITE_A"
        set dstaddr "LAN_SITE_B"
        set schedule "always"
        set service "ALL"
    next
end
```

---

## 6. Evidencias de Verificación y Auditoría en Vivo

### 6.1. Inspección de Asociaciones de Seguridad en Cisco (`R-Cisco`)
```text
R-Cisco#show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
203.25.7.2      203.25.7.10     QM_IDLE           1001 ACTIVE

R-Cisco#show crypto ipsec sa
interface: FastEthernet0/0
    Crypto map tag: VPN-MAP, local addr 203.25.7.2
   protected vrf: (none)
   local  ident (addr/mask/prot/port): (10.25.7.0/255.255.255.128/0/0)
   remote ident (addr/mask/prot/port): (10.25.7.128/255.255.255.240/0/0)
   current_peer 203.25.7.10 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 22, #pkts encrypt: 22, #pkts digest: 22
    #pkts decaps: 17, #pkts decrypt: 17, #pkts verify: 17
    #send errors 0, #recv errors 0
     local crypto endpt.: 203.25.7.2, remote crypto endpt.: 203.25.7.10
     path mtu 1500, ip mtu 1500, ip mtu idb FastEthernet0/0
     PFS (Y/N): Y, DH group: group14
     Status: ACTIVE(ACTIVE)
```

### 6.2. Inspección del Arrendamiento DHCP en Cisco
```text
R-Cisco#show ip dhcp binding
Bindings from all pools not associated with VRF:
IP address      Client-ID/ 		Lease expiration 	Type       State      Interface
		Hardware address/
		User name
10.25.7.10      0102.42e2.1425.00       Oct 03 2026 12:14 AM    Automatic  Active     FastEthernet0/1.10
```

### 6.3. Verificación de IP en PC-Usuario
```text
/ # ip addr show dev eth0
6: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN qlen 1000
    link/ether 02:42:e2:14:25:00 brd ff:ff:ff:ff:ff:ff
    inet 10.25.7.10/25 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:e2ff:fe14:2500/64 scope link 
       valid_lft forever preferred_lft forever
```

### 6.4. Conectividad Extremo a Extremo a través del Túnel Heterogéneo (Ping)
```text
/ # ping -c 5 10.25.7.130
PING 10.25.7.130 (10.25.7.130): 56 data bytes
64 bytes from 10.25.7.130: seq=0 ttl=62 time=70.338 ms
64 bytes from 10.25.7.130: seq=1 ttl=62 time=46.458 ms
64 bytes from 10.25.7.130: seq=2 ttl=62 time=43.170 ms
64 bytes from 10.25.7.130: seq=3 ttl=62 time=35.140 ms
64 bytes from 10.25.7.130: seq=4 ttl=62 time=29.022 ms

--- 10.25.7.130 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 29.022/44.825/70.338 ms
```

### 6.5. Traza de Saltos de Red (Traceroute)
```text
/ # traceroute -n 10.25.7.130
traceroute to 10.25.7.130 (10.25.7.130), 30 hops max, 46 byte packets
 1  10.25.7.1  13.635 ms  10.063 ms  11.343 ms
 2  *  *  *
 3  10.25.7.130  44.555 ms  39.353 ms  41.495 ms
```

### 6.6. Consumo del Servicio Web Seguro (HTTPS 443)
```text
/ # curl -k -i https://10.25.7.130/
HTTP/1.1 200 OK
Server: SimpleHTTP/0.6 Python/3.14.8
Date: Fri, 02 Oct 2026 00:15:35 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1006
Connection: close

<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Servidor Web Seguro - Practica 2</title>
</head>
<body style="font-family: Arial, sans-serif; background-color: #f4f6f9; color: #333; padding: 30px;">
    <div style="background: white; border: 1px solid #ddd; padding: 25px; border-radius: 8px; max-width: 700px; margin: 0 auto; box-shadow: 0 4px 6px rgba(0,0,0,0.1);">
        <h1 style="color: #0b5394;">Servidor Web Seguro (HTTPS)</h1>
        <hr style="border: 0; border-top: 1px solid #ccc;">
        <p><strong>Estudiante:</strong> Cristopher Navarro</p>
        <p><strong>Matricula:</strong> 2025-0720</p>
        <p><strong>Asignatura:</strong> Seguridad de Redes</p>
        <p><strong>Docente:</strong> Jonathan Esteban Rondon Corniel</p>
        <p><strong>Segmento Servidor:</strong> 10.25.7.128/28 (IP: 10.25.7.130)</p>
        <p style="color: #274e13; font-weight: bold;">Acceso exitoso a traves del tunel VPN IPsec Site-to-Site!</p>
    </div>
</body>
</html>
```

### 6.7. Prueba de Enforzamiento de Seguridad y Aislamiento (Drop Test en Cisco)
Para certificar que el tráfico corporativo depende exclusivamente del mapa criptográfico y no puede filtrarse por la WAN sin cifrado:
1. **Desaplicación del crypto map en `FastEthernet0/0` de R-Cisco:**
   ```text
   R-Cisco(config)# interface FastEthernet0/0
   R-Cisco(config-if)# no crypto map VPN-MAP
   ```
   En la consola de `PC-Usuario`:
   ```text
   / # ping -c 3 -W 1 10.25.7.130
   PING 10.25.7.130 (10.25.7.130): 56 data bytes

   --- 10.25.7.130 ping statistics ---
   3 packets transmitted, 0 packets received, 100% packet loss
   ```
   *Resultado:* El tráfico cae al **100 % de pérdida**, comprobando el descarte absoluto.

2. **Reaplicación del crypto map en `FastEthernet0/0` de R-Cisco:**
   ```text
   R-Cisco(config)# interface FastEthernet0/0
   R-Cisco(config-if)# crypto map VPN-MAP
   ```
   En la consola de `PC-Usuario`:
   ```text
   / # ping -c 3 10.25.7.130
   PING 10.25.7.130 (10.25.7.130): 56 data bytes
   64 bytes from 10.25.7.130: seq=0 ttl=62 time=50.445 ms
   64 bytes from 10.25.7.130: seq=1 ttl=62 time=27.389 ms
   64 bytes from 10.25.7.130: seq=2 ttl=62 time=41.462 ms

   --- 10.25.7.130 ping statistics ---
   3 packets transmitted, 3 packets received, 0% packet loss
   round-trip min/avg/max = 27.389/39.765/50.445 ms
   ```
   *Resultado:* El túnel renegocia y restablece el flujo de inmediato al **0 % de pérdida**.

---

## 7. Conclusiones Técnicas

1. **Interoperabilidad Multi-Vendor Certificada:** Se demostró la viabilidad técnica y operativa de interconectar soluciones Cisco y Fortinet mediante estándares abiertos (RFC 2409 / RFC 4301), logrando un canal seguro de alto rendimiento.
2. **Importancia del NAT Exemption:** En arquitecturas de borde donde coexisten acceso a Internet mediante NAT Overload y túneles IPsec, la denegación explícita del tráfico corporativo en la ACL de NAT es mandatoria para preservar las direcciones IP privadas que disparan las políticas criptográficas.
3. **Resiliencia y Seguridad Determinista:** La combinación de Diffie-Hellman Grupo 14 con PFS garantiza que el compromiso de claves pasadas no afecte sesiones futuras, proporcionando el más alto nivel de confidencialidad para la organización.
