[writeup-004-fuerza-bruta-ssh.md](https://github.com/user-attachments/files/34108106/writeup-004-fuerza-bruta-ssh.md)
# Write-up 004 — Ataque de fuerza bruta SSH: ataque, detección y remediación

**Autor:** Ronny Fajardo — KCRON TECH
**Fecha:** 25 oct 2026 · **Fase:** 1 (Fundamentos de seguridad) · **Duración:** ~1.5 h
**Entorno:** Laboratorio propio en VirtualBox, red aislada Host-Only (192.168.56.0/24). Ejercicio ofensivo autorizado sobre equipos propios.

> **Nota ética:** este ejercicio se realizó exclusivamente sobre sistemas propios, en una red aislada y con autorización. Realizar estas técnicas contra sistemas de terceros sin permiso escrito es un delito.

## Objetivo

Ejecutar un ataque de fuerza bruta contra el servicio SSH de un servidor propio, detectar la huella del ataque en los registros del sistema (blue team), identificar la causa raíz del compromiso y remediarla, verificando después que la corrección es efectiva.

## Topología

| Rol | Equipo | IP (Host-Only) |
| --- | --- | --- |
| Atacante | Kali Linux 2026.2 | 192.168.56.102 |
| Objetivo | Ubuntu Server 26.04 (OpenSSH 10.2p1) | 192.168.56.101 |

---

## 1. Ataque (Red Team)

Se preparó un diccionario de contraseñas comunes (`wordlist.txt`) y se lanzó Hydra contra el SSH del objetivo:

```bash
hydra -l ronny -P wordlist.txt ssh://192.168.56.101
```

- `-l ronny`: usuario objetivo.
- `-P wordlist.txt`: lista de contraseñas a probar.
- `ssh://192.168.56.101`: servicio y objetivo.

**Resultado:**
```
[22][ssh] host: 192.168.56.101   login: ronny   password: ********
1 of 1 target successfully completed, 1 valid password found
```

La cuenta fue comprometida en segundos. La contraseña era débil: **basada en un nombre propio más un número**, y además **reutilizada** en otra máquina del laboratorio.

## 2. Detección (Blue Team)

Observando `/var/log/auth.log` en el servidor durante el ataque:

```bash
sudo grep "Failed password" /var/log/auth.log
sudo grep "Accepted password" /var/log/auth.log
```

**Intentos fallidos — todos en el mismo segundo:**
```
23:25:55.096  Failed password for ronny from 192.168.56.102
23:25:55.112  Failed password for ronny from 192.168.56.102
23:25:55.246  Failed password for ronny from 192.168.56.102
...
```

**Acceso exitoso:**
```
23:25:53  Accepted password for ronny from 192.168.56.102 port 51084 ssh2
```

### Firma del ataque

| Señal | Interpretación |
| --- | --- |
| Múltiples `Failed` en milisegundos | Herramienta automatizada, no un humano |
| Una sola IP de origen (192.168.56.102) | Un único atacante |
| Misma cuenta objetivo (`ronny`) | Ataque dirigido |
| `Accepted` tras los `Failed` | **Ataque exitoso — incidente crítico** |

## 3. Remediación

Causa raíz: contraseña débil y reutilizada. Acciones:

1. Cambio de la contraseña de la cuenta `ronny` por una frase larga (16+ caracteres, mayúsculas, números y símbolos), con `passwd`.
2. Cambio de la contraseña reutilizada en la otra máquina, por una **distinta**.

## 4. Verificación

Se relanzó el mismo ataque con el diccionario original:

```bash
hydra -l ronny -P wordlist.txt ssh://192.168.56.101
```

**Resultado:**
```
1 of 1 target completed, 0 valid password found
```

El ataque fracasó. Además, OpenSSH activó una defensa automática observada en el log:
```
sshd: srclimit_penalise: 192.168.56.102/32: activating ipv4 penalty of 20 seconds for failed authentication
```
El servidor penalizó y cerró las conexiones del atacante tras detectar los fallos repetidos (rate limiting / mitigación de fuerza bruta nativa).

## Hallazgos

| # | Hallazgo | Riesgo | Estado |
| --- | --- | --- | --- |
| 1 | Cuenta SSH con contraseña débil (nombre + número) | Crítico | Remediado |
| 2 | Reutilización de contraseña entre máquinas | Alto | Remediado |

## Conclusión

Una contraseña débil basada en información personal fue crackeada por fuerza bruta en segundos. Tras aplicar una contraseña fuerte y única, el mismo ataque resultó inviable, reforzado por la mitigación nativa de OpenSSH. El ejercicio demuestra el ciclo completo: ataque → detección → diagnóstico → remediación → verificación.

## Conceptos aprendidos

- **Fuerza bruta:** prueba automatizada de contraseñas; vence a las claves débiles en segundos.
- **Firma en logs:** ráfaga de `Failed password` en milisegundos desde una IP = ataque automatizado.
- **Reutilización de contraseñas:** un solo compromiso expone todos los sistemas que comparten la clave.
- **Rate limiting:** OpenSSH moderno penaliza automáticamente a los atacantes de fuerza bruta.
- **Defensa adicional recomendada:** `fail2ban` (baneo de IP tras X fallos), claves SSH en vez de contraseña, deshabilitar login por contraseña.

## Recomendaciones de hardening para SSH

1. Usar contraseñas largas y únicas, o mejor, **autenticación por clave pública** (deshabilitar contraseñas).
2. Instalar **fail2ban** para banear IPs con intentos repetidos.
3. Limitar los usuarios que pueden entrar por SSH (`AllowUsers`).
4. Considerar cambiar el puerto por defecto y usar MFA.

## Comandos de referencia

| Comando | Para qué |
| --- | --- |
| `hydra -l <user> -P <lista> ssh://<ip>` | Ataque de fuerza bruta SSH |
| `sudo grep "Failed password" /var/log/auth.log` | Ver intentos fallidos |
| `sudo grep "Accepted password" /var/log/auth.log` | Ver accesos exitosos |
| `tail -f /var/log/auth.log` | Monitorear logins en vivo |
| `passwd` | Cambiar contraseña |
