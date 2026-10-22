# Unidad 1

## Remote Access Management

> **Cómo leer este manual:** los bloques `bash` se ejecutan en el **servidor** (Ubuntu). Los bloques `powershell` se ejecutan en el **cliente** (Windows). Cada paso: comandos para copiar y, debajo, qué comprobar.

---

## 1. Instalar SSH en el servidor

```bash
sudo apt update
sudo apt install -y openssh-server
```

El cliente (`openssh-client`) suele venir ya instalado.

## 2. Red de la máquina virtual

Utilizando el **adaptador puente** del host, configurar la IP manualmente en la MV. En modo gráfico se accede mediante el icono de red o la configuración, en la parte superior derecha de la pantalla.

### Desde terminal

Localizar el archivo de configuración en `/etc/netplan` (usualmente `00-installer-config` o `90-NM-*.yaml`):

```bash
ls /etc/netplan/
```

Ejemplo de contenido (sustituir interfaz, IP y puerta de enlace por los reales):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 172.16.6.20/24
      routes:
        - to: default
          via: 172.16.6.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

Aplicar y comprobar:

```bash
sudo netplan apply
ip -4 a
ip route | grep default
ping -c 2 8.8.8.8
```

✅ Debe salir la IP fija, una línea `default via <puerta de enlace>` y respuesta al ping.

> **La puerta de enlace tiene que existir en tu red.** Es la IP del router y debe estar dentro de la misma subred que la IP del servidor. Para saber cuál es: en Windows, `ipconfig` (campo *Puerta de enlace predeterminada* del adaptador); en otra máquina Linux de la misma red, `ip route | grep default`. Con una puerta de enlace equivocada, la máquina llega a la red local pero no a internet.

### Excepción en el firewall de Windows para poder hacer pings mutuos

Ejecutar en el cliente Windows, en PowerShell como administrador:

```powershell
Enable-NetFirewallRule -Name "FPS-ICMP4-ERQ-In"
```

## 3. Cambiar el puerto de SSH (lo primero tras instalarlo)

> **Por qué:** el puerto 22 es el primero que prueban los escáneres y bots. Cambiarlo reduce mucho el ruido en los logs. No es una medida de seguridad por sí sola, pero es el primer paso.

Ejemplo con el puerto `2222`. Si `ufw` está activo, abrir **antes** el puerto nuevo; si no, se pierde el acceso al reiniciar.

```bash
sudo ufw status
sudo ufw allow 2222/tcp
```

Fijar el puerto (fichero propio, sin tocar el `sshd_config` largo) y aplicar:

```bash
echo "Port 2222" | sudo tee /etc/ssh/sshd_config.d/10-puerto.conf
sudo sshd -t
sudo systemctl daemon-reload
sudo systemctl restart ssh
sudo ss -tlnp | grep sshd
```

✅ `sshd -t` no muestra nada si todo está bien, y `ss` debe mostrar `sshd` escuchando en `:2222`.

Si sigue escuchando en el 22 (en Ubuntu reciente SSH puede arrancar por *socket*), usar el servicio normal:

```bash
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh
sudo ss -tlnp | grep sshd
```

**Antes de cerrar la sesión actual**, abrir otra desde el cliente para comprobar que entra por el puerto nuevo:

```powershell
ssh -p 2222 jordi@IP_DE_LA_VM
```

A partir de ahora hay que indicar `-p 2222` en todos los comandos (`ssh`, y `-P 2222` en `scp`), o fijarlo en el alias de `~/.ssh/config` con `Port 2222`.

## 4. Probar la conexión

```powershell
ssh -p 2222 jordi@IP_DE_LA_VM
```

Si no se ha cambiado el puerto, sin `-p 2222`.

## 5. Claves (autenticación sin contraseña)

**Generar un par de claves.** En la máquina CLIENTE, en PowerShell:

```powershell
ssh-keygen -t ed25519 -C "jordi@cliente"
```

Se crea en la carpeta de usuario (`~/.ssh`) si no se cambia la ruta. La clave pública se puede ver con:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub
```

**Copiar la clave pública al servidor** (un solo comando; en Windows no existe `ssh-copy-id`):

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 jordi@IP_DE_LA_VM "mkdir -p ~/.ssh && chmod 700 ~/.ssh && tr -d '\r' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

✅ Desde ahora, `ssh -p 2222 jordi@IP_DE_LA_VM` entra sin pedir contraseña.

## 6. Endurecer (opcional)

> **Probar antes** que la entrada por clave funciona. Si se desactiva la contraseña sin tener clave, se pierde el acceso.

Mismo método que el puerto: un fichero propio (no hace falta buscar nada dentro de `sshd_config`):

```bash
printf "PasswordAuthentication no\nPermitRootLogin no\n" | sudo tee /etc/ssh/sshd_config.d/20-endurecer.conf
sudo sshd -t
sudo systemctl restart ssh
```

Comprobar:

```bash
sudo sshd -T | grep -i -E "passwordauthentication|permitrootlogin|^port"
```

✅ Debe salir `passwordauthentication no`, `permitrootlogin no` y `port 2222`.

## 7. Configurar el servicio para autoarranque

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
sudo systemctl status ssh
```

## Argumentos habituales de `ssh`

### Conexión e identidad

| Argumento | Comando de ejemplo | Explicación |
|---|---|---|
| `-i` | `ssh -i ~/.ssh/id_ed25519 usuario@IP` | Indica qué clave privada usar |
| `-p` | `ssh -p 2222 usuario@IP` | Puerto distinto al 22 (típico con NAT y reenvío de puertos) |
| `-l` | `ssh -l usuario IP` | Especifica el usuario; equivale a `usuario@IP` |
| `-F` | `ssh -F otro_config usuario@IP` | Usa un fichero de configuración distinto a `~/.ssh/config` |
| `-J` | `ssh -J usuario@bastion usuario@IP_INTERNA` | Salto a través de una máquina intermedia |
| `-4` / `-6` | `ssh -4 usuario@IP` | Fuerza IPv4 o IPv6 |

### Terminal y ejecución

| Argumento | Comando de ejemplo | Explicación |
|---|---|---|
| `-t` | `ssh -t usuario@IP "sudo -i"` | Fuerza una terminal interactiva (necesaria para `sudo`, `nano`, `top`...) |
| `-tt` | `ssh -tt usuario@IP "sudo -i"` | Igual que `-t`, pero obligatorio aunque el cliente no detecte terminal local |
| `-T` | `ssh -T usuario@IP` | Desactiva la terminal; útil en scripts y automatizaciones |
| `-N` | `ssh -N -L 8080:localhost:80 usuario@IP` | No ejecuta comando remoto; solo mantiene la conexión (túneles) |
| `-f` | `ssh -f -N -L 8080:localhost:80 usuario@IP` | Pasa a segundo plano tras autenticarse |
| `-C` | `ssh -C usuario@IP` | Comprime el tráfico; útil en conexiones lentas |

### Túneles y reenvío

| Argumento | Comando de ejemplo | Explicación |
|---|---|---|
| `-L` | `ssh -L 8080:localhost:80 usuario@IP` | Túnel local: un puerto tuyo apunta a un servicio remoto |
| `-R` | `ssh -R 9000:localhost:3000 usuario@IP` | Túnel remoto: un puerto del servidor apunta a un servicio tuyo |
| `-D` | `ssh -D 1080 usuario@IP` | Proxy SOCKS dinámico a través del servidor |
| `-A` | `ssh -A usuario@IP` | Reenvía el agente SSH (usar solo con máquinas de confianza) |
| `-X` / `-Y` | `ssh -X usuario@IP` | Reenvía aplicaciones gráficas X11 (`-Y` es la variante "de confianza") |

### Depuración y opciones

| Argumento | Comando de ejemplo | Explicación |
|---|---|---|
| `-v` / `-vv` / `-vvv` | `ssh -v usuario@IP` | Modo detallado; cada `v` añade nivel de información para depurar |
| `-q` | `ssh -q usuario@IP` | Modo silencioso; suprime avisos y mensajes |
| `-G` | `ssh -G vm` | Muestra la configuración efectiva sin conectar |
| `-o` | `ssh -o StrictHostKeyChecking=no usuario@IP` | Pasa una opción de `ssh_config` por línea de comandos |

## Formas de conectarse por SSH

### Según el usuario

| Método | Comando de ejemplo | Explicación |
|---|---|---|
| Usuario normal + sudo | `ssh usuario@IP` | Forma recomendada; se usa `sudo` solo para tareas puntuales |
| Subir a root tras entrar | `ssh usuario@IP sudo -i` | Cambia a shell de root; queda registrado en los logs quién lo hizo |
| Directo a shell de root | `ssh -t usuario@IP "sudo -i"` | Conecta y aterriza ya como root, sin abrir el login de root |
| Login directo como root | `ssh root@IP` | Requiere `PermitRootLogin prohibit-password` o `yes`; no recomendado |

### Según la autenticación

| Método | Comando de ejemplo | Explicación |
|---|---|---|
| Contraseña | `ssh usuario@IP` | Requiere `PasswordAuthentication yes`; vulnerable a fuerza bruta |
| Clave pública | `ssh -i ~/.ssh/id_ed25519 usuario@IP` | La clave privada nunca sale del cliente; es el método estándar |
| Clave + ssh-agent | `ssh-add` | Se introduce la passphrase una sola vez por sesión |
| FIDO2 | `ssh-keygen -t ed25519-sk` | Clave respaldada por una llave física (tipo YubiKey) |
| 2FA (PAM) | `sudo apt install libpam-google-authenticator` | Añade un segundo factor con códigos temporales |

### Según cómo se lanza

| Método | Comando de ejemplo | Explicación |
|---|---|---|
| Comando suelto | `ssh usuario@IP "df -h"` | Ejecuta un comando en remoto y cierra la conexión |
| Alias en `config` | `ssh vm` | Define `Host`, `HostName`, `User` e `IdentityFile` en `~/.ssh/config` |
| Alias directo a root | `ssh vm` | Añadiendo `RequestTTY yes` y `RemoteCommand sudo -i` al alias |
| Puerto distinto (NAT) | `ssh -p 2222 usuario@127.0.0.1` | Necesario si la VM usa reenvío de puertos |
| Salto por otra máquina | `ssh -J usuario@bastion usuario@IP_INTERNA` | Llega a una máquina interna pasando por un intermediario |
| Túnel local | `ssh -L 8080:localhost:80 usuario@IP` | Accede a un servicio de la VM sin exponerlo a la red |
| Túnel remoto | `ssh -R 9000:localhost:3000 usuario@IP` | Expone un servicio local en el servidor remoto |
| Proxy SOCKS | `ssh -D 1080 usuario@IP` | Usa la VM como proxy para el tráfico del cliente |

### Otros clientes

| Cliente | Comando de ejemplo | Explicación |
|---|---|---|
| PuTTY / KiTTY | `putty -ssh usuario@IP -i clave.ppk` | Cliente gráfico para Windows; usa claves en formato `.ppk` |
| VS Code Remote-SSH | `code --remote ssh-remote+vm` | Edita ficheros de la VM como si fueran locales |
| scp / sftp | `scp fichero usuario@IP:/ruta/` | Transferencia de ficheros sobre SSH |
| WinSCP | (interfaz gráfica) | Cliente gráfico de transferencia de ficheros para Windows |
| Mosh | `mosh usuario@IP` | Tolera cortes de red y cambios de IP; requiere instalarlo en ambos lados |
