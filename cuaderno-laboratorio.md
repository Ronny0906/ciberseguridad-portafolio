# Cuaderno de Laboratorio

**Ronny Fajardo — KCRON TECH · Formación Analista SOC**

Actualizado: 2026-10-18

---

## Datos del laboratorio

Laboratorio de ciberseguridad en VirtualBox sobre un host Windows con 12 GB de RAM. Regla de memoria: encender como máximo dos VMs a la vez para no pasar de ~7–8 GB.

### Máquinas virtuales

| VM | Sistema | RAM | Disco | IP laboratorio (Host-Only) | IP NAT | Rol en el lab |
| --- | --- | --- | --- | --- | --- | --- |
| ssrv-lab | Ubuntu Server 26.04 | 2 GB | 25 GB | 192.168.56.101 | 10.0.2.15 | Servidor a monitorear / objetivo |
| kali | Kali Linux 2026.2 | 2–3 GB | 25 GB | 192.168.56.102 | NAT | Estación de análisis y escaneo |
| win10 | Windows 10 | 3–4 GB | 50 GB | 192.168.56.103 | NAT | Endpoint corporativo |

> Nota: las IPs 192.168.56.x las asigna el DHCP y podrían cambiar al reiniciar. Si un ping no responde, revisa con `ip a` (Linux) o `ipconfig` (Windows).

### Credenciales

> Anota tus usuarios y contraseñas en tu copia privada. **No subas contraseñas reales a GitHub.**

| VM | Usuario | Contraseña | Nota |
| --- | --- | --- | --- |
| ssrv-lab | ronny | *(privada)* | Cambiable con `passwd` |
| kali | kali | *(cambiada)* | Default pública era kali/kali |
| win10 | *(privada)* | *(privada)* | Sin clave de producto (lab) |

### Host

- Equipo real: Windows, 12 GB de RAM
- VirtualBox como hipervisor
- Red del laboratorio: Host-Only (192.168.56.0/24) + NAT para internet

---

## Bitácora 001 — Montaje del laboratorio

**Fecha:** 18 oct 2026 · **Fase:** 0 (Montar el laboratorio) · **Duración:** ~3 h

**Objetivo:** Dejar el laboratorio base armado en VirtualBox con las VMs de Kali, Ubuntu Server y Windows, conectadas en red, más las cuentas y el repositorio de portafolio.

### Qué se hizo

1. Instalación de VirtualBox en el host Windows.
2. Creación de las VMs: `kali`, `win10` y `server ubuntu`.
3. Instalación completa de **Ubuntu Server 26.04** en `ssrv-lab`:
    - Disco de 25 GB con LVM, sin cifrado LUKS.
    - Usuario `ronny`, red NAT (IP 10.0.2.15).
    - **OpenSSH server** instalado.
    - Sistema actualizado con `sudo apt update && sudo apt upgrade -y`.
4. Importación de la imagen pre-armada de **Kali Linux 2026.2** y cambio de la contraseña por defecto.
5. Instalación de **Windows 10**.
6. Configuración de red: a cada VM se le agregó un **Adaptador 2 Host-Only** (además del NAT), creando la red aislada 192.168.56.0/24.
7. Regla de firewall en Windows para permitir ping ICMPv4 entrante (solo para el laboratorio).
8. Cuentas creadas en **TryHackMe**, **LetsDefend** y **GitHub**; repositorio **ciberseguridad-portafolio** creado.

### Resultado

Las tres VMs operativas y comunicándose entre sí por la red Host-Only (ping en ambos sentidos, 0% de pérdida). Cada VM mantiene su salida a internet por NAT.

### Problemas y solución

| Problema | Causa | Solución |
| --- | --- | --- |
| No aparecía la ISO de Ubuntu | Se descomprimió con 7-Zip en una carpeta | Descargar la ISO y montarla sin extraer |
| SSH figuraba `inactive (dead)` | Activación por socket (ssh.socket) | Normal: el servicio se enciende al conectarse |
| Login de Kali "password incorrect" | No se usó el usuario por defecto | Entrar con `kali` / `kali` |
| Ubuntu no hacía ping a Windows | Firewall de Windows bloquea ICMP | Regla `netsh` para permitir ping (solo lab) |
| Error en el comando `netsh` | Espacio después de la coma en `icmpv4:8,any` | Escribirlo sin espacio |

### Concepto aprendido

Un host Windows puede parecer "apagado" en un escaneo porque **no responde a ICMP (ping) por defecto**. No significa que no exista; el firewall lo oculta.

---

## Pendientes y próximos pasos

- [x] Instalar Windows 10
- [x] Cambiar la contraseña por defecto de Kali
- [x] Conectar las tres VMs en red interna
- [x] Probar SSH / ping entre VMs
- [ ] Instalar las Guest Additions de VirtualBox donde haga falta
- [ ] Fijar IPs estáticas en la red del laboratorio
- [ ] Dibujar el diagrama del laboratorio y subirlo a GitHub

Luego empieza la **Fase 1**: comandos de Linux/Windows para seguridad, Wireshark y Nmap de VM a VM, y los primeros write-ups de TryHackMe.

---

## Plantilla para próximas entradas

```markdown
## Bitácora 00X — [Título de la práctica]

**Fecha:** [día] · **Fase:** [número] · **Duración:** [horas]

**Objetivo:** [qué querías lograr]

### Qué se hizo
1. [paso]

### Comandos usados
- `comando` — [para qué sirvió]

### Resultado / hallazgo
[Qué observaste. Pega una captura si aplica.]

### Problemas y solución
| Problema | Causa | Solución |
| --- | --- | --- |
| | | |

### Pendientes
- [ ] [lo que queda para la próxima]
```

**Regla de oro:** cada entrada debería terminar subida a GitHub. Si no quedó registro, la práctica no cuenta como experiencia.
