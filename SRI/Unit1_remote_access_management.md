# Unidad 1 

## Remote Access Management  

Configurar en el servidor (en este caso Ubuntu en máquina vritual)  
```yaml
sudo apt update  
sudo apt install openssh-client openssh-server  
```
No suele ser necesario instalar el cliente  

### Configurar conexión máquina virtual  
Utilizando el **adaptador puente** del Host, configurar la IP manualmente en la MV. En modo gráfico se accede mediante el icono de red o la configuración, en la parte superior derecha de la pantalla.  

### Desde terminal
Localizar el archivo de configuración en /etc/netplan  
usualmente llamado "00-installer-config"  

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 172.16.5.20/24
      routes:
        - to: default
          via: 172.16.0.1
      nameservers:
        address: [8.8.8.8, 8.8.4.4]
```
### Excepción en el firewall de windows para poder hacer pings mutuos  
```yaml
Enable-NetFirewallRule -Name "FPS-ICMP4-ERQ-In"  
```

## Configurar SSH

Probar la instalación de SSH
```yaml
ssh jordi@IP_DE_LA_VM
```
Generar un par de claves
En la máquina CLIENTE, se escribe en la terminal:
```yaml
ssh-keygen -t ed25519 -C "jordi@172.16.5.20"

```
Se crea unarchivo en la carpeta de usuario si no se cambia
Una vez creada, puede verse la clave:
```yaml
type id_ed25519.pub
```

Copiar la clave pública al servidor
```yaml
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh jordi@IP_DE_LA_VM "mkdir -p ~/.ssh && chmod 700 ~/.ssh && tr -d '\r' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Endurecer (Opcional)  
/etc/ssh/sshd_config → PasswordAuthentication no, PermitRootLogin no
```yaml
sudo systemctl restart ssh
```  

Comprobar
```yaml
sudo sshd -T | grep -i passwordauthentication
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
