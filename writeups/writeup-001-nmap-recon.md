[writeup-001-nmap-recon.md](https://github.com/user-attachments/files/33671187/writeup-001-nmap-recon.md)
# Write-up 001 — Reconocimiento con Nmap

**Autor:** Ronny Fajardo — KCRON TECH
**Fecha:** 19 oct 2026 · **Fase:** 1 (Fundamentos de seguridad) · **Duración:** ~1 h
**Entorno:** Laboratorio propio en VirtualBox, red aislada Host-Only (192.168.56.0/24). Todas las pruebas con autorización sobre equipos propios.

## Objetivo

Hacer el reconocimiento de un host objetivo usando Nmap, de lo general a lo específico: confirmar que está vivo, descubrir sus puertos abiertos e identificar el servicio y versión que expone.

## Objetivo del escaneo

| Dato | Valor |
| --- | --- |
| Atacante (desde) | Kali Linux 2026.2 — 192.168.56.102 |
| Objetivo | Ubuntu Server 26.04 — 192.168.56.101 |

## Procedimiento y resultados

### Paso 1 — Descubrimiento de host (host discovery)

```bash
nmap -sn 192.168.56.101
```

- `-sn` = ping scan: solo averigua si el equipo está vivo, sin escanear puertos.

**Resultado:**
```
Host is up (0.0012s latency).
MAC Address: 08:00:27:82:66:05 (Oracle VirtualBox virtual NIC)
```

**Interpretación:** el host responde. La MAC con prefijo `08:00:27` (OUI de Oracle) revela que es una máquina virtual de VirtualBox.

### Paso 2 — Escaneo de puertos

```bash
nmap 192.168.56.101
```

**Resultado:**
```
Not shown: 999 closed tcp ports (reset)
PORT     STATE   SERVICE
22/tcp   open    ssh
```

**Interpretación:** de los 1000 puertos más comunes, 999 están cerrados y solo el **22/tcp (SSH)** está abierto. Superficie de ataque mínima: el servidor solo expone lo necesario para su administración.

### Paso 3 — Detección de servicio y versión (fingerprinting)

```bash
nmap -sV 192.168.56.101
```

- `-sV` = service version: interroga el puerto para identificar el programa y su versión.

**Resultado:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.2p1 Ubuntu 2ubuntu3.8 (Ubuntu Linux; protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Interpretación:** el servicio es **OpenSSH 10.2p1** sobre **Ubuntu Linux**, usando SSH protocolo 2.0 (el seguro). El CPE permite automatizar la búsqueda de vulnerabilidades conocidas.

## Hallazgos

| # | Hallazgo | Riesgo | Observación |
| --- | --- | --- | --- |
| 1 | Puerto 22/TCP (SSH) abierto | Informativo | Único servicio expuesto; esperado para administración |
| 2 | OpenSSH 10.2p1, protocolo 2.0 | Bajo | Versión reciente y parcheada; sin CVE críticos conocidos |

## Conclusión

El host objetivo expone una superficie de ataque mínima (un solo puerto) con un servicio actualizado. El ejercicio demuestra la metodología de reconocimiento escalonado: **vivo → puertos → versión**. Un administrador obtiene de este escaneo exactamente lo que un atacante vería, por eso escanear la propia red es una práctica defensiva.

## Conceptos aprendidos

- **Host discovery:** confirmar que el objetivo existe antes de escanear puertos.
- **Superficie de ataque:** cada puerto abierto es una puerta potencial; menos puertos = más seguro.
- **OUI / MAC:** el prefijo de la MAC identifica al fabricante.
- **Fingerprinting:** identificar programa y versión permite buscar vulnerabilidades (CVE).
- Las opciones de Nmap distinguen mayúsculas (`-sV`) de minúsculas (`-v`).

## Comandos de referencia

| Comando | Para qué |
| --- | --- |
| `nmap -sn <ip>` | Saber si el host está vivo |
| `nmap <ip>` | Escanear los 1000 puertos más comunes |
| `nmap -sV <ip>` | Detectar servicio y versión |
| `nmap -p- <ip>` | Escanear los 65535 puertos (próxima práctica) |
