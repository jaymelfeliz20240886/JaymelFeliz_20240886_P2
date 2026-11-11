**Demostración en Video:** https://youtu.be/40L4S8UZ8F0
# Laboratorio de Infraestructura de Red, Seguridad y Monitoreo
**Estudiante:** Matrícula 20240886  
**Entorno de Simulación:** GNS3  

---

## Índice
1. [Propósito del Laboratorio](#1-propósito-del-laboratorio)
2. [Topología de Red y Arquitectura](#2-topología-de-red-y-arquitectura)
3. [Tabla de Direccionamiento IP](#3-tabla-de-direccionamiento-ip)
4. [Diario de Configuración y Troubleshooting Operativo](#4-diario-de-configuración-y-troubleshooting-operativo)
5. [Configuraciones Implementadas Paso a Paso](#5-configuraciones-implementadas-paso-a-paso)
6. [Políticas de Seguridad (Firewall Policies & Hardening CIS Benchmark)](#6-políticas-de-seguridad-firewall-policies--hardening-cis-benchmark)
7. [Validación y Pruebas Analíticas](#7-validación-y-pruebas-analíticas)
8. [Diagramas y Evidencias Visuales](#8-diagramas-y-evidencias-visuales)
9. [Running Configurations Completas](#9-running-configurations-completas)
10. [Scripts y Archivos Utilizados](#10-scripts-y-archivos-utilizados)
11. [Conclusión](#11-conclusión)

---

## 1. Propósito del Laboratorio
Este proyecto tiene como objetivo diseñar, desplegar y auditar una infraestructura de red corporativa segura dentro de la plataforma GNS3, integrando servicios esenciales de capa 2, capa 3, autenticación centralizada (AAA/RADIUS), túneles criptográficos site-to-site y monitoreo proactivo mediante SNMP.

Toda la topología utiliza la estructura de direccionamiento IP basada en la matrícula **20240886**, implementando recomendaciones de endurecimiento (*Hardening*) alineadas al **CIS Cisco IOS Benchmark** y validando políticas de seguridad bidireccionales en FortiOS 7.0.x.

---

## 2. Topología de Red y Arquitectura
La red se divide en tres zonas principales:
* **Zona LAN Interna (`10.20.24.0/24`):** Alberga a los clientes finales (Webterm), el servidor de infraestructura Windows Server (Active Directory, DNS, NPS) y el servidor de monitoreo Zabbix.
* **Enlace WAN Inter-Sitio (`192.168.86.0/30`):** Conexión punto a punto dedicada entre el Router Cisco R1 y la interfaz WAN (`port1`) del FortiGate VM.
* **Zona Remota (`172.16.86.0/24`):** Red protegida detrás del FortiGate para simular la sede remota en el túnel VPN IPsec.

```text
               +-------------------------------------------------+
               |                ZONA LAN INTERNA                 |
               |                 10.20.24.0/24                   |
               |                                                 |
               |  [Webterm]   [WinServer AD/NPS]   [Zabbix]      |
               |      |               |               |          |
               +------|---------------|---------------|----------+
                      |               |               |
                      +---------------+---------------+
                                      | (Gi0/1, Gi0/2, Gi0/3)
                               +--------------+
                               | SW1-20240886 | (IOSvL2)
                               +--------------+
                                      | (Gi0/0)
                                      |
                                      | (Fa1/0 - 10.20.24.1)
                               +--------------+
                               | R1-20240886  | (Router Cisco)
                               +--------------+
                                      | (Fa0/0 - 192.168.86.1)
                                      |
                                      |  TÚNEL IPSEC (DES/SHA1/Group14)
                                      |  Red WAN: 192.168.86.0/30
                                      |
                                      | (port1 - 192.168.86.2)
                              +----------------+
                              | FortiGate-VM   | (FortiOS 7.0.x)
                              +----------------+
                                      | (port2 - 172.16.86.1)
                               +--------------+
                               | RED REMOTA   | (172.16.86.0/24)
                               +--------------+
```

---

## 3. Tabla de Direccionamiento IP

| Equipo | Interfaz | Dirección IP / Máscara | Gateway por Defecto | Rol / Propósito |
| :--- | :--- | :--- | :--- | :--- |
| **R1-20240886** | `FastEthernet1/0` | `10.20.24.1/24` | N/A | Gateway LAN, DHCP Server, Cliente RADIUS |
| **R1-20240886** | `FastEthernet0/0` | `192.168.86.1/30` | N/A | Extremo WAN / Crypto Map IPsec |
| **SW1-20240886** | `VLAN 1` | `10.20.24.2/24` | `10.20.24.1` | Switch L2, Agente SNMP (`RO-20240886`), SSH |
| **Windows Server** | `Ethernet0` | `10.20.24.10/24` | `10.20.24.1` | Active Directory (`lab.local`), DNS, NPS RADIUS |
| **Zabbix Docker** | `eth0` | `10.20.24.11/24` | `10.20.24.1` | Servidor de Monitoreo SNMPv2 |
| **Webterm (PC)** | `eth0` | DHCP (`10.20.24.50+`) | `10.20.24.1` | Cliente SSH de la LAN |
| **FortiGate VM** | `port1` | `192.168.86.2/30` | `192.168.86.1` | Extremo WAN VPN, GUI HTTPS |
| **FortiGate VM** | `port2` | `172.16.86.1/24` | N/A | Interfaz Gateway Red Remota |

---

## 4. Diario de Configuración y Troubleshooting Operativo

Durante el despliegue del laboratorio se resolvieron los siguientes desafíos técnicos:

1. **Negociación Cifrado SSH en Webterm (OpenSSH Moderno vs Cisco IOS):**
   * *Problema:* El cliente OpenSSH en Webterm abortaba la conexión arrojando `no matching key exchange method`, `no matching host key type` y `no matching cipher`.
   * *Solución:* Se creó el archivo `~/.ssh/config` en Webterm habilitando explícitamente los algoritmos legados compatibles con Cisco: `diffie-hellman-group14-sha1`, `ssh-rsa` y `aes256-cbc`.
2. **Rechazo de Sesión por Atributo RADIUS NPS (`Service-Type = Framed`):**
   * *Problema:* Al intentar conectarse por SSH utilizando credenciales del Active Directory, Cisco cerraba la sesión arrojando: `This line may not run PPP. Connection closed`.
   * *Solución:* En la política de red de Windows Server NPS, se cambió el atributo estándar `Service-Type` de `Framed` a `Login` y se agregó el atributo Vendor-Specific de Cisco `shell:priv-lvl=15`.
3. **Incompatibilidad de Cifrado en FortiGate VM (Licencia LENC):**
   * *Problema:* La imagen VM sin licencia de FortiGate restringía la propuesta de Fase 1 únicamente a `DES`. Intentar forzar `AES256` en CLI generaba error de sintaxis.
   * *Solución:* Se ajustó la propuesta ISAKMP y Transform-Set en el Router Cisco a `DES / SHA1 / Group 14`, logrando establecer exitosamente el túnel (`QM_IDLE`).
4. **Falta de Encapsulamiento IPsec (`pkts encaps: 0`):**
   * *Problema:* La Fase 1 y 2 estaban activas, pero el tráfico no se cifraba al hacer ping desde R1.
   * *Solución:* Se corrigió la máscara wildcard en la ACL 100 de Cisco (`0.0.0.255`), se agregó la ruta estática explícita hacia `172.16.86.0/24` vía `192.168.86.2`, y se forzó el parámetro de origen en el comando: `ping 172.16.86.1 source 10.20.24.1`.

---

## 5. Configuraciones Implementadas Paso a Paso

### 5.1 Router Cisco (R1-20240886)
* **Interfaces y DHCP:**
  ```text
  hostname R1-20240886
  ip domain-name lab.local
  
  interface FastEthernet1/0
   description CONEXION-LAN
   ip address 10.20.24.1 255.255.255.0
   no shutdown
  
  interface FastEthernet0/0
   description CONEXION-WAN-FORTIGATE
   ip address 192.168.86.1 255.255.255.252
   no shutdown

  ip dhcp excluded-address 10.20.24.1 10.20.24.20
  ip dhcp pool LAN_POOL
   network 10.20.24.0 255.255.255.0
   default-router 10.20.24.1
   dns-server 10.20.24.10
  ```
* **SSH v2 y AAA RADIUS:**
  ```text
  username admin secret admin123
  crypto key generate rsa modulus 2048
  ip ssh version 2
  ip ssh dh min size 2048

  aaa new-model
  radius server WIN-NPS
   address ipv4 10.20.24.10 auth-port 1812 acct-port 1813
   key cisco123

  aaa authentication login default group radius local
  aaa authorization exec default group radius local
  aaa accounting exec default start-stop group radius
  
  line vty 0 4
   transport input ssh
  ```
* **Túnel IPsec VPN (Fase 1, 2 y Crypto Map):**
  ```text
  access-list 100 permit ip 10.20.24.0 0.0.0.255 172.16.86.0 0.0.0.255

  crypto isakmp policy 10
   encr des
   hash sha
   authentication pre-share
   group 14
   lifetime 86400

  crypto isakmp key ClaveVPN20240886 address 192.168.86.2

  crypto ipsec transform-set TS-VPN esp-des esp-sha-hmac
   mode tunnel

  crypto map MAP-FORTIGATE 10 ipsec-isakmp
   set peer 192.168.86.2
   set transform-set TS-VPN
   match address 100

  interface FastEthernet0/0
   crypto map MAP-FORTIGATE

  ip route 172.16.86.0 255.255.255.0 192.168.86.2
  ```

### 5.2 Switch Cisco IOSvL2 (SW1-20240886)
* **Gestión, PortFast y SNMP:**
  ```text
  hostname SW1-20240886
  ip domain-name lab.local

  interface Vlan1
   ip address 10.20.24.2 255.255.255.0
   no shutdown
  ip default-gateway 10.20.24.1

  spanning-tree portfast default

  username admin secret admin123
  crypto key generate rsa modulus 2048
  ip ssh version 2

  line vty 0 4
   login local
   transport input ssh

  snmp-server community RO-20240886 RO
  ```

### 5.3 FortiGate VM (FortiOS 7.0.x CLI & GUI)
* **Interfaces y Ruta Estática de Retorno:**
  ```text
  config system interface
      edit "port1"
          set mode static
          set ip 192.168.86.2 255.255.255.252
          set allowaccess ping https http ssh
      next
      edit "port2"
          set mode static
          set ip 172.16.86.1 255.255.255.0
          set allowaccess ping https http ssh
      next
  end

  config router static
      edit 1
          set dst 10.20.24.0 255.255.255.0
          set gateway 192.168.86.1
          set device "port1"
      next
  end
  ```

---

## 6. Políticas de Seguridad (Firewall Policies & Hardening CIS Benchmark)

### 6.1 Políticas de Firewall en FortiGate (GUI)
1. **Política Entrante (`VPN-TO-LAN`):**
   * **Incoming Interface:** `VPN-CISCO` | **Outgoing Interface:** `port2`
   * **Source / Destination:** `all` | **Service:** `ALL`
   * **Action:** `ACCEPT` | **NAT:** `Disabled`
2. **Política Saliente (`LAN-TO-VPN`):**
   * **Incoming Interface:** `port2` | **Outgoing Interface:** `VPN-CISCO`
   * **Source / Destination:** `all` | **Service:** `ALL`
   * **Action:** `ACCEPT` | **NAT:** `Disabled`

### 6.2 Hardening Aplicado (CIS Cisco IOS Benchmark v4.0)

| Control CIS | Configuración Aplicada | Justificación de Seguridad | Evidencia de Cumplimiento |
| :--- | :--- | :--- | :--- |
| **1.1 Deshabilitar Telnet** | `transport input ssh` | Mitiga la captura de credenciales en texto claro mediante sniffing. | Intentos de conexión por puerto 23 son rechazados. |
| **1.2 Forzar SSHv2** | `ip ssh version 2` | Previene ataques de inyección y degradación de protocolo (SSHv1). | `show ip ssh` confirma versión 2.0 activa. |
| **1.3 Claves DH/RSA** | `crypto key generate rsa modulus 2048` | Evita ataques de de-criptación por fuerza bruta (Logjam). | Llaves criptográficas de 2048 bits configuradas. |
| **2.1 Contraseñas Cifradas**| `username admin secret <pass>` | Almacena credenciales con hash SHA-256 no reversible en la RAM. | La `running-config` muestra contraseñas tipo 5/6. |
| **3.1 AAA / RADIUS** | `aaa new-model`, `aaa accounting` | Control de acceso centralizado y trazabilidad completa de comandos. | Logs en NPS registran intentos `ACCESS-ACCEPT` y `REJECT`. |
| **4.2 STP PortFast** | `spanning-tree portfast default` | Evita ataques de manipulación de la raíz STP y optimiza la asignación DHCP. | Las PCs reciben concesión DHCP sin retardos de puerto. |

---

## 7. Validación y Pruebas Analíticas

### 7.1 Prueba de Túnel IPsec (Requisito 2)
* **Tráfico Activo:** Al ejecutar `ping 172.16.86.1 source 10.20.24.1 repeat 50` en R1, los contadores confirman el paso bidireccional cifrado:
  ```text
  #pkts encaps: 49, #pkts encrypt: 49, #pkts digest: 49
  #pkts decaps: 49, #pkts decrypt: 49, #pkts verify: 49
  ```
* **Demostración de Caída:** Al hacer clic en **Bring Down** en la GUI de FortiGate, el flujo de paquetes se interrumpe al instante (`Request timed out`), demostrando que la comunicación ocurre **únicamente** cuando la VPN está activa.

### 7.2 Prueba de Autenticación AAA RADIUS (Requisito 9)
* **Acceso Fallido:** Al intentar SSH con el usuario `admin20240886` e ingresar una clave errónea, la consola emite la traza:
  ```text
  *Nov 11 10:30:12.105: AAA/AUTHEN/LOGIN (0000000A): Status = FAIL
  *Nov 11 10:30:12.107: RADIUS/DEBU: Received Radius response - ACCESS-REJECT
  ```
* **Acceso Exitoso:** Al ingresar la contraseña correcta del Active Directory:
  ```text
  *Nov 11 10:31:04.220: RADIUS/DEBU: Received Radius response - ACCESS-ACCEPT
  *Nov 11 10:31:04.222: AAA/AUTHOR/EXEC (0000000B): Authorization Successful
  ```

---

## 8. Diagramas y Evidencias Visuales
*(Los archivos de imagen se encuentran alojados en la carpeta `/images` de este repositorio)*

* **Diagrama General de la Topología:**  
  `![Topologia](images/01_diagrama_topologia.png)`
* **Túnel VPN IPsec Activo en FortiGate GUI:**  
  `![VPN Up](images/02_vpn_ipsec_gui_up.png)`
* **Demostración de Caída de Tráfico (Bring Down):**  
  `![VPN Caida](images/03_vpn_ipsec_caida.png)`
* **Concesión de IP por DHCP en Webterm:**  
  `![DHCP Evidencia](images/04_dhcp_asignacion.png)`
* **Monitoreo SNMP del Switch en Zabbix (Estado Verde):**  
  `![Zabbix SNMP](images/06_zabbix_snmp_green.png)`
* **Logs de AAA / RADIUS (Acceso Exitoso y Fallido):**  
  `![Logs AAA RADIUS](images/08_logs_aaa_radius_exito_fallo.png)`

---

## 9. Running Configurations Completas

Los archivos de configuración completos e individuales de cada equipo se encuentran organizados dentro de la carpeta `/configs`:
* `configs/R1-20240886.txt`
* `configs/SW1-20240886.txt`
* `configs/FortiGate-20240886.conf`

---

## 10. Scripts y Archivos Utilizados

### 10.1 Script de Ajuste de SSH Client en Webterm (`~/.ssh/config`)
```bash
mkdir -p ~/.ssh
cat << 'EOF' > ~/.ssh/config
Host 10.20.24.*
    KexAlgorithms +diffie-hellman-group14-sha1,diffie-hellman-group-exchange-sha1
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedKeyTypes +ssh-rsa
    Ciphers +aes256-cbc,aes128-cbc
EOF
chmod 600 ~/.ssh/config
```

---

## 11. Conclusión
El laboratorio logró integrar exitosamente una arquitectura convergente que cumple con todos los estándares solicitados. Se validó la interoperabilidad entre equipamiento Cisco e imágenes de Fortinet, garantizando la seguridad perimetral mediante VPN IPsec, el control de acceso administrativo robusto soportado por Active Directory / NPS RADIUS, el cumplimiento de directivas de hardening del CIS Benchmark y la observabilidad continua de la infraestructura mediante el protocolo SNMP en Zabbix.
