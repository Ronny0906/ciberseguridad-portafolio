[writeup-003-ssh-hardening.md](https://github.com/user-attachments/files/33747739/writeup-003-ssh-hardening.md)
# Write-up 003 — Auditoría y hardening de SSH

**Autor:** Ronny Fajardo — KCRON TECH
**Fecha:** 20 oct 2026 · **Fase:** 1 (Fundamentos de seguridad) · **Duración:** ~1.5 h
**Entorno:** Laboratorio propio en VirtualBox, red aislada Host-Only (192.168.56.0/24). Pruebas con autorización sobre equipos propios.

## Objetivo

Auditar la configuración criptográfica del servicio SSH del servidor objetivo, identificar algoritmos débiles, corregirlos mediante hardening y verificar de forma independiente que la corrección fue efectiva.

## Topología

| Rol | Equipo | IP (Host-Only) |
| --- | --- | --- |
| Analista | Kali Linux 2026.2 | 192.168.56.102 |
| Objetivo | Ubuntu Server 26.04 (OpenSSH 10.2p1) | 192.168.56.101 |

---

## 1. Detección — auditoría de algoritmos

Desde Kali, enumeración de algoritmos SSH con el script NSE correspondiente:

```bash
sudo nmap --script ssh2-enum-algos 192.168.56.101
```

> Nota metodológica: los scripts NSE y la detección fina de Nmap requieren privilegios de root (`sudo`). Sin ellos, el escaneo entrega resultados recortados.

**Resultado (MACs antes de la corrección) — 10 algoritmos, incluidos débiles:**
```
mac_algorithms: (10)
   umac-64-etm@openssh.com
   umac-128-etm@openssh.com
   hmac-sha2-256-etm@openssh.com
   hmac-sha2-512-etm@openssh.com
   hmac-sha1-etm@openssh.com        <-- débil
   umac-64@openssh.com
   umac-128@openssh.com
   hmac-sha2-256
   hmac-sha2-512
   hmac-sha1                        <-- débil
```

Los algoritmos de intercambio de claves y de cifrado resultaron modernos y sin observaciones.

## 2. Evaluación — hallazgo

| # | Hallazgo | Riesgo | Detalle |
| --- | --- | --- | --- |
| 1 | SSH acepta MAC obsoletos (`hmac-sha1`, `hmac-sha1-etm`) | Bajo | SHA-1 tiene colisiones demostradas (2017); debe deshabilitarse en un servidor endurecido |

**Interpretación:** aunque los algoritmos modernos se negocian primero, aceptar SHA-1 amplía innecesariamente la superficie criptográfica. En una auditoría formal se reporta como observación de hardening.

## 3. Corrección — hardening

Todo el procedimiento se hizo sobre el servidor Ubuntu, con tres controles de seguridad:

**3.1 Respaldo previo (nunca editar sin copia):**
```bash
cd /etc/ssh
sudo cp sshd_config sshd_config.backup
```

**3.2 Edición de la configuración** (`sudo nano sshd_config`), añadiendo al final:
```
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com
```
Esto restringe los MAC a tres algoritmos fuertes de tipo encrypt-then-MAC (`-etm`).

**3.3 Validación de sintaxis antes de aplicar:**
```bash
sudo sshd -t      # sin salida = configuración válida
```

**3.4 Aplicación del cambio:**
```bash
sudo systemctl restart ssh
```

> Buena práctica aplicada: no cerrar la sesión SSH activa hasta confirmar que el cambio funciona, para no perder el acceso.

## 4. Verificación — re-escaneo independiente

Desde Kali, se repitió la enumeración:

```bash
sudo nmap --script ssh2-enum-algos 192.168.56.101
```

**Resultado (MACs después de la corrección) — 3 algoritmos, todos fuertes:**
```
mac_algorithms: (3)
   hmac-sha2-512-etm@openssh.com
   hmac-sha2-256-etm@openssh.com
   umac-128-etm@openssh.com
```

**El algoritmo `hmac-sha1` ya no aparece.** Corrección confirmada.

## Conclusión

El servidor pasó de ofrecer 10 algoritmos MAC (dos de ellos obsoletos) a solo 3, todos modernos. El hallazgo quedó remediado y verificado de forma independiente.

## Ciclo aplicado

| Etapa | Acción |
| --- | --- |
| Detectar | Enumeración de algoritmos con NSE |
| Evaluar | Identificación de `hmac-sha1` como débil |
| Corregir | Hardening de `sshd_config` con respaldo y validación |
| Verificar | Re-escaneo confirmando la eliminación |

## Conceptos aprendidos

- Los scripts NSE de Nmap requieren `sudo` para funcionar completos.
- `ssh2-enum-algos` enumera los algoritmos criptográficos del servicio SSH.
- SHA-1 es un algoritmo obsoleto; un servidor endurecido no debe aceptarlo.
- Hardening seguro: respaldar → editar → validar (`sshd -t`) → aplicar → verificar.
- Encrypt-then-MAC (`-etm`) es el método de integridad más seguro en SSH.

## Comandos de referencia

| Comando | Para qué |
| --- | --- |
| `sudo nmap --script ssh2-enum-algos <ip>` | Enumerar algoritmos SSH |
| `sudo cp sshd_config sshd_config.backup` | Respaldar configuración |
| `sudo nano sshd_config` | Editar configuración |
| `sudo sshd -t` | Validar sintaxis sin aplicar |
| `sudo systemctl restart ssh` | Aplicar cambios |
