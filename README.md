# Laboratorio de Seguridad de Redes con FortiGate

**Estudiante:** Edwin De Paula
**Matrícula:** 2024-2415
**Carrera:** Seguridad Informática — ITLA
**Asignatura:** Seguridad de Redes

---

## Video

| Recurso | URL |
|---|---|
| Video YouTube | _pendiente_ |

---

## 1. Objetivo de la red

Diseñar y configurar, en su totalidad por interfaz gráfica (GUI), una topología de red segura utilizando FortiGate como firewall perimetral e interno. La red debe segmentar el tráfico entre una LAN de usuarios y una LAN de servidores, permitir salida controlada a Internet mediante NAT, y aplicar múltiples capas de seguridad: control de acceso por servicio, filtrado de aplicaciones, filtrado web, detección de escaneo de red (IPS/DoS) y protección de aplicaciones web (WAF).

El direccionamiento IP de toda la topología se derivó de la matrícula del estudiante (2024-2415), utilizando la red base `10.24.15.0/24`.

---

## 2. Topología de red

**[SCREENSHOT AQUÍ: diagrama de topología con nombre y matrícula visibles]**
`![Topología](screenshots/00-topologia.png)`

### 2.1 Descripción de la topología

| Nodo | Rol | Conexión |
|---|---|---|
| Kali Linux | Acceso a GUI del FortiGate / máquina de pruebas de escaneo | port1 (WAN) vía Cloud1 (VMware vmnet8) |
| FortiGate-VM64-KVM | Firewall / gateway | port1 (WAN), port2 (LAN Usuarios), port3 (LAN Servidores) |
| SW1 | Switch LAN Usuarios | port2 ↔ Linux-Desktop |
| Linux-Desktop | Cliente de pruebas (DHCP, navegación, escaneo nmap) | SW1 |
| SW2 | Switch LAN Servidores | port3 ↔ web Server |
| web Server (Metasploitable2 + DVWA) | Servidor web de destino | SW2 |

### 2.2 Direccionamiento IP

| Segmento / Interfaz | Red | Máscara | Gateway | Rango DHCP |
|---|---|---|---|---|
| WAN (port1) | Asignada por DHCP (vmnet8) | /24 | Aprendido automáticamente | — |
| LAN Usuarios (port2) | 10.24.15.0/25 | 255.255.255.128 | 10.24.15.1 | 10.24.15.10 – 10.24.15.100 |
| LAN Servidores (port3) | 10.24.15.128/28 | 255.255.255.240 | 10.24.15.129 | No aplica (estática) |
| web Server | 10.24.15.130 | 255.255.255.240 | 10.24.15.129 | Estática |

**Lógica de direccionamiento:** los dos últimos octetos (`24.15`) se derivan del año de ingreso (24) y los últimos dos dígitos de la matrícula (15) → red base `10.24.15.0/24`, subdividida en /25 para usuarios y /28 para servidores.

---

## 3. Configuración de interfaces

Todas las interfaces se configuraron desde **Network > Interfaces** en la GUI del FortiGate.

**[SCREENSHOT AQUÍ: tabla de interfaces con IP/Netmask, Administrative Access y DHCP Ranges]**
`![Interfaces](screenshots/01-interfaces-resumen.png)`

**[SCREENSHOT AQUÍ: detalle de configuración de port2]**
`![Port2](screenshots/02-port2-config.png)`

**[SCREENSHOT AQUÍ: detalle de configuración de port3]**
`![Port3](screenshots/03-port3-config.png)`

- `port1`: modo DHCP, conectado a la red externa (WAN).
- `port2`: IP estática `10.24.15.1/25`, rol LAN, DHCP Server habilitado.
- `port3`: IP estática `10.24.15.129/28`, rol LAN, sin DHCP (segmento de servidores con IP fija).

### 3.1 DHCP Server — LAN Usuarios

Configurado en **Network > DHCP Server**, asociado a `port2`.

**[SCREENSHOT AQUÍ: configuración del DHCP Server — rango, gateway, DNS]**
`![DHCP Server](screenshots/04-dhcp-server.png)`

- Rango: `10.24.15.10` – `10.24.15.100`
- Gateway entregado: `10.24.15.1`
- DNS: heredado del sistema

---

## 4. Ruta por defecto

El FortiGate aprendió automáticamente la ruta por defecto a través del DHCP de `port1`, visible en **Network > Static Routes**.

**[SCREENSHOT AQUÍ: tabla de Static Routes mostrando 0.0.0.0/0 vía port1]**
`![Ruta por defecto](screenshots/05-static-route.png)`

| Destino | Gateway | Interfaz | Estado |
|---|---|---|---|
| 0.0.0.0/0 | (asignado por DHCP en port1) | port1 | Enabled |

---

## 5. NAT y acceso a Internet

Política de firewall que permite la salida de la LAN de Usuarios hacia Internet, con NAT de origen habilitado.

**[SCREENSHOT AQUÍ: política Usuarios_a_Internet con NAT activado]**
`![Política NAT](screenshots/06-policy-internet-nat.png)`

**Configuración:**
- **Nombre:** `Usuarios_a_Internet`
- **Incoming Interface:** port2
- **Outgoing Interface:** port1
- **Source / Destination:** all
- **Service:** ALL
- **Action:** ACCEPT
- **NAT:** Enable — Use Outgoing Interface Address
- **Log Allowed Traffic:** habilitado

**[SCREENSHOT AQUÍ: prueba de ping desde Linux-Desktop hacia 8.8.8.8 exitosa]**
`![Prueba Internet](screenshots/07-ping-internet-ok.png)`

---

## 6. Control de tráfico LAN Usuarios → LAN Servidores (solo HTTP)

Se implementaron dos políticas explícitas, en orden estricto, para permitir únicamente HTTP y bloquear el resto de forma explícita.

**[SCREENSHOT AQUÍ: lista de políticas mostrando el orden HTTP_Allow arriba de DENY_ALL]**
`![Orden de políticas](screenshots/08-policy-order.png)`

### 6.1 Política de permiso (HTTP)

- **Nombre:** `Usuarios_a_Servidores_HTTP_Allow`
- **Incoming Interface:** port2
- **Outgoing Interface:** port3
- **Service:** HTTP
- **Action:** ACCEPT
- **Inspection Mode:** Proxy-based (requerido para habilitar WAF, ver sección 10)

### 6.2 Política de bloqueo explícito

- **Nombre:** `Usuarios_a_Servidores_DENY_ALL`
- **Incoming Interface:** port2
- **Outgoing Interface:** port3
- **Service:** ALL
- **Action:** DENY
- **Log Violation Traffic:** habilitado

**[SCREENSHOT AQUÍ: HTTP cargando correctamente desde el navegador hacia 10.24.15.130]**
`![HTTP permitido](screenshots/09-http-allowed.png)`

**[SCREENSHOT AQUÍ: ping/ssh fallando hacia 10.24.15.130]**
`![Resto bloqueado](screenshots/10-resto-bloqueado.png)`

---

## 7. Application Control — Redes sociales y llamadas de WhatsApp

**Perfil:** `Bloqueo_RedesSociales_WhatsApp` (Security Profiles > Application Control)

**[SCREENSHOT AQUÍ: categoría Social.Media en modo Block]**
`![Bloqueo redes sociales](screenshots/11-appctrl-socialmedia.png)`

**[SCREENSHOT AQUÍ: Application Override con firma WhatsApp_VoIP.Call en Block]**
`![Bloqueo WhatsApp](screenshots/12-appctrl-whatsapp.png)`

- **Categoría bloqueada:** `Social.Media` (cubre Facebook, Instagram, X, entre otras).
- **Firma específica bloqueada:** `WhatsApp_VoIP.Call` — se optó por bloquear la firma granular de llamadas en lugar de la aplicación completa, para cumplir exactamente el requisito ("bloquear llamadas") sin afectar la mensajería de texto.
- Perfil aplicado a la política `Usuarios_a_Internet`, con SSL Inspection (`certificate-inspection`) habilitada, necesaria porque estas aplicaciones operan sobre HTTPS.

**[SCREENSHOT AQUÍ: intento de acceso a Facebook/Instagram bloqueado por FortiGuard]**
`![Prueba de bloqueo redes sociales](screenshots/13-test-socialmedia.png)`

---

## 8. Web Filter — Bloqueo de itla.edu.do y subdominios

**Perfil:** `Bloqueo_ITLA` (Security Profiles > Web Filter > Static URL Filter)

**[SCREENSHOT AQUÍ: entradas del Static URL Filter con itla.edu.do y *.itla.edu.do en Block]**
`![Static URL Filter](screenshots/14-webfilter-itla.png)`

| URL | Tipo | Acción |
|---|---|---|
| `itla.edu.do` | Wildcard | Block |
| `*.itla.edu.do` | Wildcard | Block |

**[SCREENSHOT AQUÍ: intento de acceso a itla.edu.do bloqueado]**
`![Prueba de bloqueo ITLA](screenshots/15-test-itla.png)`

---

## 9. Detección y bloqueo de escáneres de red

Implementado mediante **DoS Policy** (Policy & Objects > DoS Policy), que detecta patrones de escaneo por anomalía de tráfico en lugar de depender de firmas IPS descargadas de FortiGuard.

**Perfil:** `AntiScan_Servidores`

**[SCREENSHOT AQUÍ: configuración de la DoS Policy con tcp_port_scan, icmp_sweep en Block]**
`![DoS Policy](screenshots/16-dos-policy-config.png)`

- **Incoming Interface:** port2
- **Anomalías activadas:** `tcp_port_scan`, `icmp_sweep` (acción: Block)
- **Threshold ajustado:** reducido respecto al valor por defecto para detectar escaneos de bajo volumen típicos de un entorno de laboratorio.

### Prueba de escaneo (nmap desde Linux-Desktop)

```bash
nmap -sS -Pn 10.24.15.130
```

**[SCREENSHOT AQUÍ: log de Anomaly mostrando tcp_port_scan desde 10.24.15.10, acción clear_session]**
`![Detección de escaneo](screenshots/17-dos-log.png)`

| Fecha/Hora | Severidad | Origen | Protocolo | Acción | Ataque |
|---|---|---|---|---|---|
| — | Alta | 10.24.15.10 | TCP (6) | clear_session | tcp_port_scan |

---

## 10. WAF (Web Application Firewall) al servidor Web

Requiere que la política `Usuarios_a_Servidores_HTTP_Allow` opere en **modo de inspección Proxy-based** (en FortiOS 6.4.x el modo de inspección se configura por política individual, no de forma global).

**Perfil:** `WAF_WebServer` (Security Profiles > Web Application Firewall)

**[SCREENSHOT AQUÍ: perfil WAF con categorías SQL Injection y XSS en modo Block, nivel High]**
`![Perfil WAF](screenshots/18-waf-profile.png)`

- **Categorías bloqueadas:** SQL Injection, Cross Site Scripting (XSS) — nivel de sensibilidad **High**.
- **Aplicado a:** política `Usuarios_a_Servidores_HTTP_Allow`.

### Pruebas realizadas contra DVWA (`http://10.24.15.130/dvwa/`)

**SQL Injection:**
```
1' OR '1'='1
```

**[SCREENSHOT AQUÍ: bloqueo de SQL Injection en DVWA]**
`![Bloqueo SQLi](screenshots/19-waf-sqli-block.png)`

**Cross Site Scripting (XSS):**
```
<script>alert('test')</script>
```

**[SCREENSHOT AQUÍ: bloqueo de XSS en DVWA]**
`![Bloqueo XSS](screenshots/20-waf-xss-block.png)`

**Nota técnica:** con el nivel de seguridad de DVWA en "Medium", la sanitización parcial de la propia aplicación camuflaba el patrón del ataque lo suficiente como para que la firma WAF no lo reconociera. Al subir DVWA a nivel "High", el payload llega sin modificar al servidor y la firma WAF lo detecta correctamente. Esto evidencia la diferencia entre protección a nivel de aplicación y protección perimetral.

**[SCREENSHOT AQUÍ: log de Web Application Firewall mostrando ambos eventos bloqueados]**
`![Log WAF](screenshots/21-waf-log.png)`

---

## 11. Limitaciones conocidas del entorno

- **Acceso administrativo HTTPS a la GUI:** se identificó un bug conocido de corrupción de paquetes TLS en el vNIC `e1000` emulado por VMware/PNetLab, que producía el error `SSL_ERROR_RX_RECORD_TOO_LONG` al acceder por HTTPS. Se utilizó HTTP (`allowaccess http`) para la administración durante el laboratorio, documentando esta limitación del entorno virtual.
- **Prueba en vivo de llamada de WhatsApp:** se intentó desplegar un nodo Windows 10 en PNetLab para instalar WhatsApp Desktop y demostrar la llamada bloqueada en tiempo real. La imagen disponible presentó un error de arranque (`0xc000000f`), probablemente por incompatibilidad de modo de arranque (UEFI/GPT) con la emulación BIOS Legacy del template QEMU de PNetLab. Dado el alcance de tiempo del laboratorio, se optó por no continuar troubleshooting de este componente aislado. El bloqueo de llamadas de WhatsApp queda demostrado mediante la firma `WhatsApp_VoIP.Call` configurada en modo Block dentro del perfil de Application Control (ver sección 7), verificable en la configuración aunque no se ejecutó una llamada real.

---

## 12. Credenciales de laboratorio

> Únicamente se documentan credenciales de servicios con valores por defecto públicamente conocidos (Metasploitable, DVWA). La contraseña de administrador del FortiGate **no se incluye** en este documento por buenas prácticas de seguridad.

| Servicio | Usuario | Password |
|---|---|---|
| Metasploitable2 (consola) | `msfadmin` | `msfadmin` |
| DVWA (web) | `admin` | `password` |
| FortiGate GUI | `admin` | *(no documentada, ver nota anterior)* |

---

## 13. Estructura del repositorio

```
FortiGate-Network-Security-Lab-2024-2415/
├── README.md
├── documentacion-tecnica.pdf
├── screenshots/
│   ├── 00-topologia.png
│   ├── 01-interfaces-resumen.png
│   └── ...
└── video/
    └── enlace-video.md   (o el video embebido/enlazado si el tamaño lo permite)
```

---
