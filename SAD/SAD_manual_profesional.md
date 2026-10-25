# SAD · Manual profesional

Consulta rápida por tarea. Detalle en `Temas1-6_fundamentos.md`, `Tema7_criptografia_gpg.md` y `Tema8_ssh.md`. Los `< >` se sustituyen.

---

## 1. Servidor nuevo: checklist

- [ ] Actualizar: `sudo apt update && sudo apt upgrade` (probar antes en pruebas si es producción)
- [ ] Usuario propio con `sudo`; no trabajar como root
- [ ] SSH endurecido (sección 2)
- [ ] Cortafuegos: solo los puertos necesarios
- [ ] Quitar servicios y puertos que sobren: `ss -tulpn`
- [ ] Copias de seguridad probadas (**RAID ≠ copia**; una copia fuera del sitio)
- [ ] Hora sincronizada y *logs* revisables
- [ ] Contraseñas por defecto cambiadas; cuentas sin uso fuera

```bash
sudo ufw allow <puerto>/tcp
sudo ufw enable
sudo ufw status verbose
ss -tulpn                        # qué escucha y quién
```

---

## 2. SSH: instalar y endurecer

```bash
sudo apt install openssh-server
sudo systemctl enable --now ssh
sudo nano /etc/ssh/sshd_config
sudo sshd -t                     # validar ANTES de reiniciar
sudo systemctl restart ssh
```

| Opción | Valor | Para qué |
|---|---|---|
| `Port` | `<puerto>` | Cambiar el 22 (si usa `ssh.socket`, el puerto se cambia también allí) |
| `PermitRootLogin` | `no` | Sin acceso directo de root |
| `PasswordAuthentication` | `no` | Solo clave (**después** de probar la clave) |
| `PermitEmptyPasswords` | `no` | Nada de contraseñas vacías |
| `AllowUsers` | `<usuario>` | Lista blanca |
| `MaxAuthTries` | `3` | Menos intentos |

> Regla de oro: abre **otra sesión** antes de reiniciar `sshd`. Si te equivocas, no te quedas fuera.

---

## 3. Claves SSH

```bash
ssh-keygen -t ed25519 -C "<comentario>" -f ~/.ssh/<nombre>    # crear (Ed25519 por defecto; RSA: -t rsa -b 4096)
ssh-keygen -p -f ~/.ssh/<nombre>                               # cambiar passphrase
ssh-keygen -y -f ~/.ssh/<nombre>                               # sacar la pública de la privada
ssh-keygen -lf ~/.ssh/<nombre>.pub                             # huella
ssh-copy-id -i ~/.ssh/<nombre>.pub <usuario>@<ip>              # Linux
```

Desde Windows (PowerShell), sin `ssh-copy-id`:

```powershell
type $env:USERPROFILE\.ssh\<nombre>.pub | ssh <usuario>@<ip> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && tr -d '\r' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Conectar: `ssh -i ~/.ssh/<nombre> -p <puerto> <usuario>@<ip>`

### Dos listas, no las mezcles

| Archivo | Dónde | Guarda |
|---|---|---|
| `known_hosts` | **Cliente** | Servidores en los que confío |
| `authorized_keys` | **Servidor** | Usuarios que pueden entrar |

Analogía del portero: la clave privada es tu DNI; cada cuenta es una puerta con un portero y su lista de invitados (`authorized_keys`). La misma clave pública apuntada en dos cuentas abre las dos; quitarla de una no afecta a la otra.

```bash
ssh-keygen -F "[<ip>]:<puerto>"      # buscar un host en known_hosts
ssh-keygen -R "[<ip>]:<puerto>"      # borrarlo (tras reinstalar un servidor)
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub   # huella del servidor (para compararla al conectar)
```

Revocar un acceso: borrar su línea de `authorized_keys`.

### Certificados SSH (CA propia)

```bash
ssh-keygen -t ed25519 -f ca                                    # crear la CA
ssh-keygen -s ca -I <id> -n <usuario> -V +52w <usuario>.pub     # firmar clave de usuario
# Servidor: TrustedUserCAKeys /etc/ssh/ca.pub  (en sshd_config)
ssh-keygen -s ca -h -I <host> -n <host> /etc/ssh/ssh_host_ed25519_key.pub   # certificado de host
# Cliente: @cert-authority * <contenido de ca.pub>   (en known_hosts)
ssh-keygen -L -f <certificado>                                  # inspeccionar
```

Revocación: `RevokedKeys` (lista KRL, `ssh-keygen -k`).

---

## 4. GPG

```bash
gpg --full-generate-key                          # crear (Ed25519 o RSA 3072+, con caducidad)
gpg -k ; gpg -K                                  # listar públicas / privadas
gpg --fingerprint <correo>                       # huella (compararla por otro canal)
gpg --export --armor <correo> > <nombre>_publica.asc
gpg --import <fichero>.asc
gpg --gen-revoke <correo> > revocacion.asc       # guardarla fuera del equipo
gpg --edit-key <correo>  →  trust                # nivel de confianza (5 = mis claves)
```

| Quiero... | Comando |
|---|---|
| Cifrar para alguien | `gpg --encrypt --armor -r <correo> <f>` |
| Descifrar | `gpg --decrypt <f>.asc > <f>` |
| Cifrar con contraseña | `gpg -c <f>` |
| Firmar aparte | `gpg --detach-sign --armor <f>` |
| Verificar | `gpg --verify <f>.asc <f>` |
| Firmar + cifrar | `gpg --sign --encrypt -r <correo> <f>` |

- Cifrar usa la **pública del destinatario**; firmar usa **mi privada**; verificar usa la **pública del firmante**.
- `Good signature` = íntegro y de quien dice. `BAD signature` = modificado.
- Hash: `sha256sum <f>` (SHA-256 sí; MD5 y SHA-1 no).

---

## 5. Diagnóstico rápido

| Síntoma | Qué mirar |
|---|---|
| `Connection refused` | ¿Corre `sshd`? `systemctl status ssh` · ¿puerto? `ss -tlnp` · ¿cortafuegos? |
| `Connection timed out` | Red/IP/NAT; desde Windows: `Test-NetConnection <ip> -Port <puerto>` |
| `Permission denied (publickey)` | Permisos: `~/.ssh` 700 y `authorized_keys` 600; propietario correcto; ¿clave en el usuario correcto? |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | Servidor reinstalado → `ssh-keygen -R`; si no lo has tocado, **sospecha** |
| Detalle de una conexión | `ssh -vvv ...` · en el servidor: `journalctl -u ssh` o `/var/log/auth.log` |
| VirtualBox NAT | Conectar a `127.0.0.1` con `-p <puerto redirigido>`; la IP NAT (10.0.3.15) no se alcanza desde el host |
| `scp` falla por puerto | `scp -P` (mayúscula; `ssh` usa `-p`) |
| GPG `unsafe permissions` | `chmod 700 ~/.gnupg` |
| GPG `No public key` | Importar la pública del firmante/destinatario |
| GPG en SSH pide contraseña y falla | `export GPG_TTY=$(tty)` |

---

## 6. Tablas de bolsillo

**Propiedades:** Confidencialidad (nadie no autorizado lo ve) · Integridad (no se altera) · Disponibilidad (está cuando se necesita).

| Disponibilidad | Parada máx./año |
|---|---|
| 99 % | ~3,65 días |
| 99,9 % | ~8,76 h |
| 99,99 % | ~52 min |
| 99,999 % | ~5 min |

| TIER | Idea |
|---|---|
| I | Todo básico, una vía; mantener = parar |
| II | Componentes redundantes, una sola vía de distribución |
| III | Varias vías; mantenimiento sin parar |
| IV | Tolerante a fallos |

**Cadena de riesgo:** Activo → Vulnerabilidad → Amenaza → Ataque → Incidente → Impacto.

**Estándares:** CVE identifica · CVSS puntúa gravedad · CWE tipo de debilidad · CPE producto afectado · NVD amplía los CVE.

**Medidas por fase:** Prevención → Detección → Respuesta → Recuperación.

**SPOF:** componente único cuyo fallo para todo el servicio → se elimina con redundancia. **RAID ≠ copia de seguridad.**

### SAI

| Tipo | Uso |
|---|---|
| Offline | Equipos personales |
| Line-interactive | Oficina y servidores pequeños |
| Online (doble conversión) | CPD y sistemas críticos |

```
P(W) = S(VA) × FP        S(VA) = P(W) / FP
Dimensionar: sumar W → pasar a VA con el FP → +20 % de margen → modelo comercial igual o superior
E(Wh) = V × Ah           Autonomía(h) ≈ E / P   (real: menos)
```

Ejemplo: 1250 W, FP 0,8 → 1562 VA → ×1,2 ≈ 1875 VA → elegir un SAI de capacidad superior.

El SAI da tiempo para un **apagado ordenado**; no es alta disponibilidad por sí solo.
