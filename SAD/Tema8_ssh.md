# SAD · Tema 8 — SSH: claves, certificados y administración

Manual de consulta. Los comandos van **en el orden en que se usan**. Lo que va entre `< >` hay que sustituirlo por el valor propio.
Cliente: Windows (PowerShell), salvo que se indique. Servidor: Ubuntu.

---

## 1. La idea en dos listas

SSH guarda claves en dos sitios, con sentidos contrarios:

| Archivo | Dónde está | Qué guarda | Para qué sirve |
|---|---|---|---|
| `known_hosts` | Cliente (`~/.ssh/`) | Claves públicas de los **servidores** que conozco | Saber que me conecto al servidor correcto |
| `authorized_keys` | Servidor, una por cuenta (`~/.ssh/`) | Claves públicas de los **usuarios** a los que dejo entrar | Saber que quien llama es quien dice ser |

Y dos tipos de claves:

- **Claves de host** (`/etc/ssh/ssh_host_*`): identifican al servidor. Se crean al instalar OpenSSH.
- **Claves de usuario** (`~/.ssh/id_*`): identifican a la persona. Las crea cada usuario con `ssh-keygen`.

**La analogía del portero.** Mi clave privada es como un DNI. Cada cuenta del servidor es una puerta con un portero que tiene su propia lista de invitados (su `authorized_keys`). El DNI no dice a qué puertas puedo entrar: lo decide cada portero con su lista. Si apunto mi clave pública en las listas de dos cuentas, el mismo DNI abre las dos. Si la quito de una, esa puerta deja de abrirse y la otra sigue igual.

La **huella** (*fingerprint*) es el resumen corto de una clave pública. Sirve para comprobar que una clave es la correcta sin leerla entera.

---

## 2. Instalar y arrancar el servidor

```bash
sudo apt update && sudo apt install -y openssh-server
sudo systemctl enable --now ssh
systemctl status ssh --no-pager
sudo ss -tlnp | grep sshd
```

- `enable --now`: arranca el servicio y hace que inicie con el sistema.
- `ss -tlnp`: comprueba que escucha en el puerto 22.

En RHEL / Rocky: `sudo dnf install -y openssh-server` y el servicio se llama `sshd`.

**Cortafuegos**

```bash
sudo ufw allow OpenSSH          # o: sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status verbose
```

En RHEL / Rocky (firewalld):

```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

**Archivos importantes**

| Ruta | Contenido |
|---|---|
| `/etc/ssh/sshd_config` | Configuración del servidor |
| `/etc/ssh/sshd_config.d/*.conf` | Fragmentos de configuración (recomendado en sistemas modernos) |
| `/etc/ssh/ssh_host_*` | Claves de host |
| `/var/log/auth.log` (Debian/Ubuntu) · `/var/log/secure` (RHEL) | Registro de accesos y errores |

---

## 3. Llegar a la máquina virtual (VirtualBox)

- **Adaptador puente:** la MV tiene IP propia en la red (fija o por DHCP) y se conecta con `ssh usuario@<IP>`. Es lo que ven los demás equipos.
- **NAT con reenvío de puertos:** funciona en cualquier red, sin depender de la IP.
  Configuración → Red → Adaptador → NAT → Avanzadas → Reenvío de puertos → TCP, IP anfitrión `127.0.0.1`, puerto anfitrión `2222`, puerto invitado `22`.

```powershell
ssh -p 2222 usuario@127.0.0.1
```

La IP interna de la NAT (por ejemplo `10.0.3.15`) **no** es accesible desde Windows: solo se llega por el reenvío.

Combinarlos: un adaptador en puente (IP fija, ruta por defecto con métrica alta, `optional: true`) y otro en NAT (`dhcp4: true`, `route-metric` baja, `optional: true`) en netplan permite mover la MV entre redes sin tocar nada.

---

## 4. Claves de host y huella

```bash
sudo ls -l /etc/ssh/ssh_host_*
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Las privadas (`ssh_host_*_key`) son `-rw-------`; las `.pub` son legibles por todos.

**Primera conexión: comparar la huella antes de aceptar**

```powershell
ssh usuario@<IP>
```

SSH muestra la huella del servidor y pregunta `yes/no`. Se compara **entera** con la del comando anterior. Si coincide, `yes`. Si no, `no` y se investiga.

Si el servidor ya está en `known_hosts` y coincide, conecta en silencio: solo muestra la huella si el servidor es desconocido o su clave ha cambiado.

**Consultar y limpiar `known_hosts` (en el cliente)**

```powershell
ssh-keygen -F <host>        # ver lo guardado de ese servidor
ssh-keygen -lF <host>       # lo mismo, mostrando huellas
ssh-keygen -R <host>        # borrar sus entradas
```

- Con un puerto distinto del 22, el host se escribe entre corchetes y comillas: `"[127.0.0.1]:2222"`.
- Salen **tres entradas** (Ed25519, RSA y ECDSA): tras autenticarse, el cliente aprende todas las claves de host del servidor.
- `-R` solo borra en el cliente; no cambia nada en el servidor.
- `ssh -o VisualHostKey=yes ...` o `ssh-keygen -lv` muestran la huella también como dibujo (*randomart*).

---

## 5. Crear claves de usuario

```powershell
ssh-keygen -t ed25519 -C "usuario@equipo" -f $env:USERPROFILE\.ssh\id_ed25519_nombre
ssh-keygen -t rsa -b 3072 -C "usuario@equipo-rsa" -f $env:USERPROFILE\.ssh\id_rsa_nombre
dir $env:USERPROFILE\.ssh\id_*nombre*
ssh-keygen -lf $env:USERPROFILE\.ssh\id_ed25519_nombre.pub
```

Se crean dos archivos: la **privada** (sin extensión, nunca se comparte) y la **pública** (`.pub`). La passphrase protege la privada en el disco.

| Opción | Para qué |
|---|---|
| `-t` | Tipo de algoritmo: `ed25519` (recomendado), `rsa`, `ecdsa` |
| `-b` | Tamaño en bits (RSA: 3072 o 4096). Con Ed25519 no se usa, es fijo |
| `-C` | Comentario que identifica al dueño de la clave |
| `-f` | Nombre y ruta del archivo |
| `-N` | Passphrase desde la línea de comandos (`-N ""` = sin passphrase) |
| `-a` | Rondas de derivación de la passphrase: más rondas, más difícil de romper por fuerza bruta |
| `-p` | Cambiar la passphrase de una clave existente |
| `-y` | Regenerar la `.pub` a partir de la privada |
| `-l` / `-E` | Mostrar la huella / elegir el hash (`sha256`, `md5`) |

**Ed25519 frente a RSA:** Ed25519 usa curvas elípticas, así que sus claves son mucho más cortas (la pública ocupa unos 100 bytes frente a unos 600 de una RSA de 3072) y rápidas con una seguridad equivalente o mayor. RSA solo se usa por compatibilidad con sistemas antiguos.

---

## 6. Autenticación por clave pública

**Copiar la clave pública al servidor**

Windows (no existe `ssh-copy-id`):

```powershell
type $env:USERPROFILE\.ssh\id_ed25519_nombre.pub | ssh usuario@<IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && tr -d '\r' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

`tr -d '\r'` quita los retornos de carro de Windows. `>>` añade al final sin borrar lo que había.

Linux:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_nombre.pub usuario@<IP>
```

**Conectar con una clave concreta**

```powershell
ssh -i $env:USERPROFILE\.ssh\id_ed25519_nombre -p <puerto> usuario@<IP>
```

**Forzar solo clave pública** (para comprobar que funciona de verdad, sin caer a la contraseña):

```powershell
ssh -o PreferredAuthentications=publickey -o PasswordAuthentication=no -o IdentitiesOnly=yes -i <clave> usuario@<IP>
```

**Permisos**: si `~/.ssh` (700) o `authorized_keys` (600) están más abiertos, `sshd` rechaza la clave. Y si la privada del cliente es demasiado abierta, también.

**Dos contraseñas distintas:** la del usuario vive en el servidor y abre la cuenta; la passphrase vive en el cliente y solo desbloquea el archivo de la clave. Nunca viaja por la red.

---

## 7. Varias claves y revocación

Un mismo usuario puede tener **varias claves autorizadas**: una por equipo, para rotarlas sin quedarse fuera, o para retirar solo una si se pierde un portátil.

**Revocar una clave** = quitar su línea de `authorized_keys` en el servidor (se reconoce por el comentario del final):

```bash
nano ~/.ssh/authorized_keys             # Ctrl+K corta la línea, Ctrl+O guarda, Ctrl+X sale
sudo sed -i '/<comentario>/d' <ruta>/authorized_keys     # alternativa en una línea
```

- No hace falta borrar la privada del cliente: el acceso lo controla el servidor.
- Al intentarlo con la clave revocada: `Permission denied (publickey).` Es el resultado correcto.

---

## 8. Una misma clave en varias cuentas

```bash
sudo adduser alumno2
id alumno2
sudo mkdir -p /home/alumno2/.ssh
grep "<comentario>" ~/.ssh/authorized_keys | sudo tee /home/alumno2/.ssh/authorized_keys
sudo chown -R alumno2:alumno2 /home/alumno2/.ssh
sudo chmod 700 /home/alumno2/.ssh
sudo chmod 600 /home/alumno2/.ssh/authorized_keys
```

La misma clave privada entra en las dos cuentas (ver la analogía del portero). Compartir clave **no** comparte permisos: cada cuenta tiene su UID, su carpeta personal y sus grupos.

```bash
getent passwd usuario1 alumno2                      # UID (campo 3) y carpeta personal (campo 6)
id usuario1; id alumno2                             # UID y grupos
sudo ls -l /home/usuario1/.ssh/authorized_keys /home/alumno2/.ssh/authorized_keys
```

Ventaja: una sola clave que gestionar. Inconveniente: si la roban, se accede a todas las cuentas a la vez, y hay que revocarla una a una.

---

## 9. Endurecer el servidor (`sshd_config`)

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo nano /etc/ssh/sshd_config
```

| Parámetro | Efecto |
|---|---|
| `PermitRootLogin no` | Prohíbe entrar directamente como root |
| `PasswordAuthentication no` | Obliga a usar clave |
| `PubkeyAuthentication yes` | Activa la autenticación por clave pública |
| `PermitEmptyPasswords no` | Prohíbe contraseñas vacías |
| `MaxAuthTries 3` | Intentos fallidos antes de cortar |
| `LoginGraceTime 30s` | Tiempo máximo para autenticarse |
| `X11Forwarding no` | Desactiva la interfaz gráfica remota si no hace falta |
| `AllowUsers u1 u2` / `AllowGroups g` | Solo esos usuarios o grupos pueden entrar |
| `Port <n>` | Cambia el puerto (opcional; reduce el ruido de escaneos) |
| `Banner /etc/issue.net` | Muestra un aviso legal antes del login |

**Aplicar**

```bash
sudo sshd -t                    # comprueba la sintaxis; sin salida = correcto
sudo systemctl restart ssh
```

**Antes de apretar, ojo con esto**

- Instalar y **probar la clave primero**; solo entonces desactivar `PasswordAuthentication`. Si no, uno se queda fuera.
- `AllowUsers` con un nombre de ejemplo copiado tal cual deja fuera al usuario real.
- Probar siempre en una **segunda terminal** antes de cerrar la sesión abierta.
- Si se cambia el `Port`: abrirlo en el cortafuegos y actualizar el reenvío de VirtualBox (puerto invitado). En Ubuntu recientes el servicio puede arrancar por socket (`ssh.socket`) y entonces el `Port` de `sshd_config` se ignora: comprobar con `systemctl status ssh.socket` y cambiarlo en el socket o desactivarlo.
- `Protocol 2` ya no hace falta: las versiones actuales solo hablan v2.

**Extra:** `fail2ban` bloquea IPs con muchos intentos fallidos (`sudo apt install -y fail2ban && sudo systemctl enable --now fail2ban`).

---

## 10. Certificados SSH firmados por una CA

En vez de copiar la clave de cada usuario a cada servidor, una **autoridad certificadora (CA)** firma las claves públicas (con caducidad y usuarios permitidos) y los servidores solo confían en la CA.

Tres papeles: CA, cliente y servidor.

```bash
# 1. En la CA: crear su par de claves (la privada, muy protegida)
ssh-keygen -t ed25519 -f ~/.ssh/ssh_ca -C "CA"

# 2. En el cliente: su par normal, y envía la .pub a la CA
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "usuario@equipo"

# 3. En la CA: firmar la clave pública del usuario
ssh-keygen -s ~/.ssh/ssh_ca -I "id_usuario" -n "usuario_servidor" -V +5d ~/.ssh/id_ed25519.pub
#   crea id_ed25519-cert.pub; se deja junto a la privada en ~/.ssh/ del cliente

# 4. En el servidor: confiar en la CA
sudo cp ssh_ca.pub /etc/ssh/ssh_ca.pub && sudo chmod 644 /etc/ssh/ssh_ca.pub
#   en sshd_config:  TrustedUserCAKeys /etc/ssh/ssh_ca.pub
sudo systemctl restart ssh

# 5. En el cliente: entrar sin haber copiado la clave a authorized_keys
ssh usuario_servidor@<IP>
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub        # ver los datos del certificado
```

| Opción | Para qué |
|---|---|
| `-s` | Clave privada de la CA con la que se firma |
| `-I` | Identificador del certificado (aparece en los logs del servidor) |
| `-n` | Usuarios (*principals*) con los que se puede entrar |
| `-V +5d` | Validez (caduca solo) |
| `-z` | Número de serie, para poder revocar certificados concretos |
| `-O` | Restricciones (`-O clear` quita todo y se permite lo necesario) |
| `-L` | Inspeccionar un certificado |

**Certificados de servidor:** `-h` firma la clave de host (`ssh-keygen -s host_ca -h -I web01 -n web01.local ssh_host_ed25519_key.pub`). El servidor lo carga con `HostCertificate`, y el cliente confía en la CA con una línea `@cert-authority <dominio> <clave pública de la CA>` en `known_hosts`, sin pregunta de huella.

**Revocar antes de que caduquen** (lista KRL):

```bash
ssh-keygen -k -f /etc/ssh/revoked_keys clave_comprometida.pub
ssh-keygen -Q -f /etc/ssh/revoked_keys clave.pub            # consultar
#   en sshd_config:  RevokedKeys /etc/ssh/revoked_keys
```

Diferencia con un par de claves: un par es solo la pareja pública/privada y cada servidor debe tener la pública en `authorized_keys`. Un certificado es una clave pública **firmada por una autoridad**, con caducidad y usuarios permitidos; el servidor solo necesita confiar en la CA.

---

## 11. OpenSSH en Windows Server (2019 o superior)

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'
Get-Service sshd
```

Regla del cortafuegos (si no existe):

```powershell
New-NetFirewallRule -Name 'OpenSSH-Server-In-TCP' -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

Shell por defecto PowerShell (en vez de `cmd`):

```powershell
New-ItemProperty -Path 'HKLM:\SOFTWARE\OpenSSH' -Name DefaultShell -Value 'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe' -PropertyType String -Force
```

- Configuración: `C:\ProgramData\ssh\sshd_config`. Tras cambiarla: `Restart-Service sshd`.
- Usuarios normales: `%USERPROFILE%\.ssh\authorized_keys`.
- **Administradores:** se usa un archivo central, `C:\ProgramData\ssh\administrators_authorized_keys`, con permisos solo para SYSTEM y Administrators:

```powershell
icacls.exe "C:\ProgramData\ssh\administrators_authorized_keys" /inheritance:r /grant "Administrators:F" /grant "SYSTEM:F"
```

- Certificados de CA: `TrustedUserCAKeys C:\ProgramData\ssh\ssh_ca.pub`.
- Conexión: `ssh Administrador@<IP>` (cuenta local) o `ssh "DOMINIO\usuario@<IP>"` (cuenta de dominio).

| | Linux | Windows Server |
|---|---|---|
| Permisos | POSIX (`chmod`, `chown`) | ACL de NTFS (`icacls`) |
| Claves de administradores | `~/.ssh/authorized_keys` | `administrators_authorized_keys` |
| Autenticación | PAM | LSA, NTLM, Kerberos, Active Directory |
| Jaula `ChrootDirectory` | Sí | No de forma nativa |
| Registros | `auth.log` / `journalctl -u ssh` | Visor de eventos → OpenSSH → Operational |
| Rutas | `/`, distingue mayúsculas | `\` (admite `/`), no distingue |

---

## 12. Diagnóstico y errores típicos

```bash
sudo ss -tlnp | grep sshd                 # ¿escucha en el puerto esperado?
sudo tail -f /var/log/auth.log | grep sshd
sudo journalctl -u ssh -f
```

```powershell
ssh -vvv -p <puerto> usuario@<IP>         # modo depuración en el cliente
Test-NetConnection 127.0.0.1 -Port 2222   # ¿responde el puerto reenviado?
```

| Síntoma | Causa probable y qué hacer |
|---|---|
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | El servidor presenta otra clave. **No ignorar:** verificar la huella nueva en el servidor y, solo si es correcta, `ssh-keygen -R <host>` y aceptar. Causas legítimas: reinstalación, VM clonada, otra máquina con esa IP |
| `Permission denied (publickey)` | La clave no está en `authorized_keys`, usuario equivocado, o permisos demasiado abiertos (`.ssh` 700, `authorized_keys` 600) |
| Pide contraseña en vez de passphrase | El servidor no ha aceptado la clave: revisar lo anterior |
| `Connection refused` | `sshd` parado, otro puerto, cortafuegos o reenvío de puertos mal configurado |
| `Connection timed out` | IP o red incorrectas, o cortafuegos descartando el tráfico |
| Dos máquinas con la misma IP | Aviso de huella cambiada que no tiene lógica: comprobar con `arp -a <IP>` que responde la MAC esperada |
| `scp` no conecta al puerto | En `scp` el puerto es `-P` mayúscula (en `ssh` es `-p`) |
| Una VM clonada tiene las mismas claves de host que la original | `sudo rm /etc/ssh/ssh_host_*` y `sudo dpkg-reconfigure openssh-server` |

---

## 13. Chuleta de `known_hosts` y `authorized_keys`

| Quiero... | Comando |
|---|---|
| Ver el servidor en `known_hosts` | `ssh-keygen -F <host>` |
| Olvidar un servidor | `ssh-keygen -R <host>` |
| Ver la huella de una clave | `ssh-keygen -lf <archivo.pub>` |
| Ver las claves autorizadas de una cuenta | `cat ~/.ssh/authorized_keys` |
| Autorizar una clave | `type clave.pub \| ssh ... "cat >> ~/.ssh/authorized_keys"` |
| Revocar una clave | Quitar su línea de `authorized_keys` |
| Ocultar los nombres de `known_hosts` | `ssh-keygen -H -f ~/.ssh/known_hosts` |
