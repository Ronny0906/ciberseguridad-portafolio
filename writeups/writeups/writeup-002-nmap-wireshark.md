[writeup-002-nmap-wireshark.md](https://github.com/user-attachments/files/33747541/writeup-002-nmap-wireshark.md)
# Write-up 002 — Reconocimiento con Nmap y análisis con Wireshark

**Autor:** Ronny Fajardo — KCRON TECH
**Fecha:** 19 oct 2026 · **Fase:** 1 (Fundamentos de seguridad) · **Duración:** ~2 h
**Entorno:** Laboratorio propio en VirtualBox, red aislada Host-Only (192.168.56.0/24). Pruebas con autorización sobre equipos propios.

## Objetivo

Realizar un reconocimiento completo de un host objetivo con Nmap (de lo general a lo específico) y luego observar el tráfico real de ese escaneo con Wireshark, conectando la herramienta con lo que ocurre en la red.

## Topología

| Rol | Equipo | IP (Host-Only) |
| --- | --- | --- |
| Atacante / analista | Kali Linux 2026.2 | 192.168.56.102 |
| Objetivo | Ubuntu Server 26.04 | 192.168.56.101 |
| Servidor DHCP | VirtualBox | 192.168.56.100 |

---

## Parte 1 — Reconocimiento con Nmap

### 1.1 Descubrimiento de host

```bash
nmap -sn 192.168.56.101
```
- `-sn`: ping scan, solo comprueba si el host está vivo (sin escanear puertos).

**Resultado:** `Host is up`. MAC `08:00:27:...` → OUI de Oracle VirtualBox (es una VM).

### 1.2 Escaneo de puertos comunes

```bash
nmap 192.168.56.101
```

**Resultado:** 999 puertos cerrados, **22/tcp open ssh**.

### 1.3 Escaneo de todos los puertos

```bash
nmap -p- 192.168.56.101
```
- `-p-`: escanea los 65535 puertos.

**Resultado:** 65534 cerrados, solo **22/tcp** abierto. Certeza total: no hay servicios ocultos en puertos altos. Tardó ~30 s frente a 1 s del escaneo común.

### 1.4 Detección de versión

```bash
nmap -sV 192.168.56.101
```
- `-sV`: identifica programa y versión del servicio.

**Resultado:** `OpenSSH 10.2p1 Ubuntu 2ubuntu3.8 (Ubuntu Linux; protocol 2.0)`.

### 1.5 Escaneo agresivo (todo en uno)

```bash
nmap -A 192.168.56.101
```
- `-A`: versión + detección de SO (`-O`) + scripts (`-sC`) + traceroute.

**Resultado:** mejor estimación de SO **Linux 4.15–5.19 (97%)**, traceroute de **1 salto** (conexión directa). La detección de SO es probabilística, no exacta.

---

## Parte 2 — Análisis de tráfico con Wireshark

Captura en la interfaz `eth1` (red del laboratorio), con filtro de visualización:

```
ip.addr == 192.168.56.101
```

### 2.1 Tráfico de base observado

| Protocolo | Qué hace |
| --- | --- |
| DHCP | Asignación automática de IP (Request → ACK) |
| ARP | Traducción de IP a dirección MAC en la red local |

### 2.2 Tráfico durante un escaneo Nmap

Dejando Wireshark capturando, se lanzó `nmap 192.168.56.101` desde una segunda terminal. Patrón observado:

| Dirección | Bandera TCP | Significado |
| --- | --- | --- |
| Kali → Ubuntu | `[SYN]` | "¿puerto abierto?" (sondeo) |
| Ubuntu → Kali | `[RST, ACK]` | "puerto cerrado" (portazo) |
| Ubuntu → Kali (puerto 22) | `[SYN, ACK]` | "puerto abierto" |

**Hallazgo clave:** el mensaje de Nmap `Not shown: 999 closed tcp ports (reset)` corresponde exactamente a los cientos de paquetes `[RST, ACK]` capturados. La herramienta resume en una línea lo que en la red son cientos de respuestas.

## Conceptos aprendidos

- **Reconocimiento escalonado:** vivo → puertos comunes → todos los puertos → versión → agresivo.
- **Superficie de ataque:** el objetivo solo expone el puerto 22; configuración limpia.
- **ARP y DHCP:** cómo una red local asigna IPs y traduce IP a MAC.
- **TCP handshake (SYN / SYN-ACK / RST-ACK):** base de toda conexión; Nmap lo usa para sondear el estado de los puertos.
- **Firma de un escaneo:** un host enviando SYN a muchos puertos en milisegundos es un patrón que un IDS (Suricata) detecta como escaneo de puertos.

## Valor defensivo

Escanear la propia red muestra exactamente lo que vería un atacante. Leer una captura de red (pcap) es una habilidad central de un analista SOC: durante un incidente, la captura es la evidencia.

## Comandos de referencia

| Comando | Para qué |
| --- | --- |
| `nmap -sn <ip>` | ¿Host vivo? |
| `nmap <ip>` | Puertos comunes |
| `nmap -p- <ip>` | Todos los puertos (65535) |
| `nmap -sV <ip>` | Versión del servicio |
| `nmap -A <ip>` | Reconocimiento completo |
| Filtro Wireshark `ip.addr == <ip>` | Ver solo el tráfico de un host |
| Filtro Wireshark `tcp.flags.syn == 1` | Ver solo paquetes SYN (sondeos) |
