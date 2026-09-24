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
&nbsp;&nbsp;&nbsp; ethernets:
&nbsp;&nbsp;&nbsp;&nbsp; enps3: (o la tarjeta de red correspondiente)

## Excepción en el firewall de windows para poder hacer pings mutuos  
Enable-NetFirewallRule -Name "FPS-ICMP4-ERQ-In"  

### Generar un par de claves
