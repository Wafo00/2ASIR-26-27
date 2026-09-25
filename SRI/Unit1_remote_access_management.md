# Unidad 1 

## Remote Access Management  

Configurar en el servidor (en este caso Ubuntu en máquina vritual)  
* sudo apt update  
* sudo apt install openssh-client openssh-server  
(el cliente no suele ser necesario instalarlo)  

## Configurar conexión máquina virtual  
Utilizando el adaptador puente del Host, configurar la ip manualmente en la MV. En modo gráfico se accede mediante el icono de red o la configuración, en la parte superior derecha de la pantalla.  

## Desde terminal
Localizar el archivo de configuración, en /etc/netplan  
usualmente 00-installer-config  

network:
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
## Excepción en el firewall de windows para poder hacer pings mutuos  
Enable-NetFirewallRule -Name "FPS-ICMP4-ERQ-In"  

### Generar un par de claves
En la máquina CLIENTE, se escribe en la terminal:
```yaml
ssh-keygen -t ed25519 -C "jordi@172.16.5.20"

```
Se crea unarchivo enla carpeta de usuario si no se cambia

```yaml
type id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGHyZg35Ipsc6kF6MFRdfYAAvnsFmstZlRpH1ALVN7wE jordi@172.16.5.20
```

Copiar la clave pública al servidor
```yaml
ssh-copy-id -i ~/.ssh/id_ed25519.pub jordi@IP_DE_LA_VM
```

Probar 
```yaml
ssh jordi@IP_DE_LA_VM
```

Endurecer
```yaml
(Opcional) Endurecer: en /etc/ssh/sshd_config → PasswordAuthentication no, PermitRootLogin no → sudo systemctl restart ssh
```
