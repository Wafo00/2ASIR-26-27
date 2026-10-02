# Instalación y configuración de DHCP con Kea en Linux (Ubuntu/Debian)

#instalación-y-configuración-de-dhcp-con-kea
> Antes de instalar: comprobar que no hay otro servicio DHCP activo en la misma red (evita conflictos de IP).

## 1. Instalar

#1-instalar

```
sudo apt update
sudo apt install kea
```

Durante la instalación aparece un prompt para configurar la contraseña de `kea-ctrl-agent` (su API REST). Elegir **"configure with a given password"** e introducir una propia.

Una contraseña generada al azar queda guardada en un fichero y hay que consultarla cada vez que se necesita (`sudo cat /etc/kea/kea-api-password`). Poniéndola, se conoce desde el principio y se puede usar directamente si en el futuro se accede a la API (`kea-shell`, recarga en caliente, herramientas de gestión), sin depender de tener acceso root para consultarla.

## 2. Configurar interfaz y rango

#2-configurar-interfaz-y-rango

Hacer copia de seguridad del fichero a editar:

```
sudo cp /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.bak
```

> **Recomendado:** en vez de editar con `nano` (el copiar y pegar desde otro sitio puede introducir comillas tipográficas o cortar líneas, provocando errores de sintaxis difíciles de ver), sobrescribir el fichero de una vez con:
> ```
> sudo tee /etc/kea/kea-dhcp4.conf > /dev/null << 'EOF'
> { ... contenido del JSON de abajo ... }
> EOF
> ```
> Las comillas simples en `'EOF'` evitan que el terminal interprete el contenido, lo escribe tal cual (End Of File).

Alternativamente, puede editarse a mano:

```
sudo nano /etc/kea/kea-dhcp4.conf
```

Pueden rellenarse los espacios en blanco del propio archivo, o borrarlo todo e insertar el código propuesto:

```
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": ["enp0s3"]
    },
    "control-socket": {
      "socket-type": "unix",
      "socket-name": "/run/kea/kea4-ctrl-socket"
    },
    "lease-database": {
      "type": "memfile",
      "lfc-interval": 3600
    },
    "valid-lifetime": 600,
    "max-valid-lifetime": 7200,
    "subnet4": [
      {
        "id": 1,
        "subnet": "192.168.1.0/24",
        "pools": [
          { "pool": "192.168.1.100 - 192.168.1.200" }
        ],
        "option-data": [
          { "name": "routers", "data": "192.168.1.1" },
          { "name": "domain-name-servers", "data": "8.8.8.8, 8.8.4.4" }
        ]
      }
    ]
  }
}
```

Sustituir interfaz (`ip a` para verla), red y rango por los propios.

### Cambiar tiempos de concesión (lease time)

```json
"valid-lifetime": 600,
"max-valid-lifetime": 7200,
```

`valid-lifetime`: duración por defecto de una concesión, en segundos. `max-valid-lifetime`: el máximo permitido, aunque el cliente pida más.

Editar en `/etc/kea/kea-dhcp4.conf`, validar y aplicar sin cortar el servicio:

```
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
sudo systemctl reload kea-dhcp4-server
```

## 3. Validar sintaxis

#3-validar-sintaxis

```
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

Detecta errores de JSON (comas de más/menos) sin arrancar el servicio.

### Incidencia conocida: "Unable to open file"

Si el comando anterior devuelve `Syntax check failed with: Unable to open file /etc/kea/kea-dhcp4.conf` aunque el fichero exista con permisos correctos (`root:root`, `644`), el bloqueo no es de sintaxis ni de permisos: es el perfil de **AppArmor** del paquete, que impide al proceso `kea-dhcp4` leer el fichero.

Confirmarlo revisando el log del kernel:

```
sudo dmesg | grep -i apparmor | grep -i kea
```

Si aparecen líneas `DENIED` con `capname="dac_override"` o `dac_read_search`, se confirma la causa.

Solución:

```
sudo apt install apparmor-utils
sudo aa-complain /usr/sbin/kea-dhcp4
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

`aa-complain` pone el perfil en modo "registra pero no bloquea", en vez de desactivarlo del todo. Para producción sería preferible ajustar el perfil (`/etc/apparmor.d/usr.sbin.kea-dhcp4`) y volver a modo estricto con `sudo aa-enforce /usr/sbin/kea-dhcp4`; en un entorno de prácticas, dejarlo en `complain` es suficiente.

## 4. Arrancar y comprobar

#4-arrancar-y-comprobar

```
sudo systemctl restart kea-dhcp4-server
sudo systemctl status kea-dhcp4-server
```

Debe salir `active (running)`. Si falla:

```
journalctl -u kea-dhcp4-server -e
```

## 5. Verificar desde un cliente

#5-verificar-desde-un-cliente

```
ip a
```

Para renovar la IP en un cliente Ubuntu de escritorio (con NetworkManager):

```
sudo nmcli device disconnect enp0s3
sudo nmcli device connect enp0s3
```

(Sustituir `enp0s3` por la interfaz real del cliente.)

> **Nota:** si un cliente vuelve a solicitar IP y recibe la misma que antes, es el comportamiento normal de DHCP ("sticky lease"): mientras la concesión no haya caducado (`valid-lifetime`) y la MAC coincida, el servidor intenta devolver la misma IP. No es un fallo.

Para ver las IPs concedidas por Kea:

```
sudo cat /var/lib/kea/kea-leases4.csv
```

## Configurar para autoarranque

#configurar-para-autoarranque

```
sudo systemctl enable kea-dhcp4-server
sudo systemctl start kea-dhcp4-server
```

Comprobar que quedó activado:

```
systemctl is-enabled kea-dhcp4-server
```

Debe responder `enabled`. Es probable que ya lo esté por defecto tras la instalación; en ese caso no hace falta tocar nada.

---

| Síntoma                     | Causa probable                                                                                     |
| ---------------------------- | --------------------------------------------------------------------------------------------------- |
| El servicio no arranca      | Error de sintaxis JSON (`kea-dhcp4 -t`), o AppArmor bloqueando el fichero (ver incidencia arriba) |
| Arranca pero no reparte IPs | Interfaz equivocada, u otro DHCP compitiendo en la red                                             |
| El cliente no recibe IP     | VMs en redes distintas (modo de red de VirtualBox)                                                 |


## Consultas habituales de administración

```
sudo systemctl status kea-dhcp4-server
```
Estado del servicio: activo, parado, o con errores recientes.

```
sudo cat /var/lib/kea/kea-leases4.csv
```
Listado de todas las concesiones de IP activas, con su MAC y fecha de caducidad.

```
sudo journalctl -u kea-dhcp4-server -f
```
Sigue el log del servicio en tiempo real; útil para ver en directo cómo llegan las peticiones DHCP de los clientes.

```
sudo journalctl -u kea-dhcp4-server --since "10 min ago"
```
Log de los últimos minutos, sin tener que revisar todo el histórico.

```
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```
Valida la configuración actual sin reiniciar el servicio (repetir tras cualquier cambio manual del fichero).

```
sudo systemctl reload kea-dhcp4-server
```
Aplica cambios de configuración sin cortar las concesiones ya activas (a diferencia de `restart`, que sí interrumpe el servicio un instante).

```
ip -s -4 addr show enp0s3
```
Estadísticas de la interfaz por la que escucha el servidor; útil para confirmar que recibe tráfico.

```
sudo grep <MAC_del_cliente> /var/lib/kea/kea-leases4.csv
```
Busca la concesión de un cliente concreto por su dirección MAC (visible con `ip a` en el propio cliente).  
  

# Instalación y configuración de DHCP en Windows Server 2022

> Antes de instalar: comprobar que no hay otro servicio DHCP activo en la misma red (evita conflictos de IP).

## 1. Configurar IP estática en el servidor

Comprobar primero que esa IP no está en uso por otro equipo de la red (evita conflictos como el descrito en la incidencia del paso 6):

```powershell
Test-NetConnection -ComputerName <IP_SERVIDOR> -Port 53
Resolve-DnsName -Name localhost -Server <IP_SERVIDOR>
ping <IP_SERVIDOR>
```

Si alguno responde, esa IP ya está en uso por otro equipo; elegir otra antes de continuar.

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress <IP_SERVIDOR> -PrefixLength 24 -DefaultGateway <IP_GATEWAY>
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses <IP_SERVIDOR>
```

El servidor DHCP nunca debe obtener su propia IP por DHCP, por lo que se configura de forma manual y estática. `-InterfaceAlias` es el nombre del adaptador de red (comprobar con `Get-NetAdapter` si hay dudas).

> **Importante:** en una VM VirtualBox, el adaptador de red debe estar en modo **Red interna** (o Bridge, según el diseño de la red), nunca en NAT — en NAT la IP la asigna VirtualBox automáticamente (típicamente `10.0.2.x`) y no es posible fijar una IP propia de la red del aula. Revisar esto en VirtualBox → Configuración → Red antes del paso 1.

## 2. Instalar el rol DHCP

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
```

Instala el servicio y la consola de gestión (`DHCP Manager`). También puede configurarse desde esa consola gráfica (`dhcpmgmt.msc`); aquí se documenta en PowerShell por ser más rápido de repetir.

Si la instalación tarda más de unos minutos sin terminar, puede deberse a que el sistema está intentando descargar los ficheros de origen desde Windows Update en vez de usarlos localmente — algo que falla o se alarga mucho sin salida a internet (red interna). Cancelar con `Ctrl+C` y volver a lanzar el mismo comando suele resolverlo, ya que `Install-WindowsFeature` es idempotente y seguro de repetir. Si persiste, indicar una fuente local explícita (el ISO de instalación montado):

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools -Source "D:\sources\sxs"
```

## 3. Autorizar el servidor (solo si hay Active Directory)

```powershell
Add-DhcpServerInDC -DnsName "servidor.dominio.local" -IPAddress <IP_SERVIDOR>
```

En un dominio AD, un DHCP sin autorizar es ignorado por los clientes como medida de seguridad. En un servidor independiente (sin AD) este paso no existe y debe omitirse; el comando fallaría al no encontrar un controlador de dominio.

## 4. Crear el ámbito (scope) y el rango

```powershell
Add-DhcpServerv4Scope -Name "Nombre_del_ambito" -StartRange <IP_INICIO> -EndRange <IP_FIN> -SubnetMask 255.255.255.0
```

Un *scope* define la red y el rango de IPs que el servidor puede repartir, ya sea de forma dinámica o mediante reservas (ver paso 7). **La IP del propio servidor debe quedar fuera de este rango.**

## 5. Configurar puerta de enlace

```powershell
Set-DhcpServerv4OptionValue -ScopeId <RED> -Router <IP_GATEWAY>
```

## 6. Configurar DNS

```powershell
Set-DhcpServerv4OptionValue -ScopeId <RED> -DnsServer <IP_DNS_1>,<IP_DNS_2>
```

Este cmdlet valida que la IP indicada responda realmente como servidor DNS, y en algunos entornos (especialmente redes internas sin salida y sin forwarders configurados) puede rechazar una IP que en realidad funciona correctamente. Puede dar el error `"<IP> no es un servidor DNS válido"` en varios casos:

- **La IP no tiene salida de red** (VM en modo "red interna" de VirtualBox; ninguna IP externa, como `8.8.8.8`, será alcanzable nunca por diseño).
- **La IP es alcanzable pero no tiene el rol DNS instalado** (no hay nada escuchando en el puerto 53).
- **El rol DNS está instalado y funciona, pero el cmdlet igualmente lo rechaza** (validación interna adicional, p. ej. intenta resolución recursiva hacia internet). Confirmar que el DNS funciona de verdad antes de forzar:
```powershell
  Resolve-DnsName -Name localhost -Server <IP_DNS>
```
  Si responde correctamente, el servidor DNS es válido y el rechazo del cmdlet es un falso positivo.

Para saber si el rol DNS está instalado:

```powershell
Get-WindowsFeature -Name DNS
```

Si no lo está, instalarlo (recomendado; permite resolución de nombres real):

```powershell
Install-WindowsFeature -Name DNS -IncludeManagementTools
Set-DhcpServerv4OptionValue -ScopeId <RED> -DnsServer <IP_SERVIDOR>
```

Si el DNS ya está confirmado como funcional (con `Resolve-DnsName`) y el cmdlet lo sigue rechazando, saltar la validación:

```powershell
Set-DhcpServerv4OptionValue -ScopeId <RED> -DnsServer <IP_DNS> -Force
```

Si se instala el rol DNS, crear además una zona para poder registrar nombres de los equipos de la red:

```powershell
Add-DnsServerPrimaryZone -Name "dominio.local" -ZoneFile "dominio.local.dns"
```

## 7. Activar el ámbito

```powershell
Set-DhcpServerv4Scope -ScopeId <RED> -State Active
```

Un scope creado pero no activado existe en la configuración, pero no reparte IPs.

## 8. Reservar IPs fijas (equipos, impresoras, servidores...)

Una **reserva** no es lo mismo que el rango del scope: el scope es el conjunto de IPs que el servidor puede repartir dinámicamente; una reserva fija una IP concreta de ese rango a una MAC concreta, para que ese equipo reciba siempre la misma dirección. No existe un comando que reserve un rango completo de golpe sin especificar MACs — se repite una vez por equipo:

```powershell
Add-DhcpServerv4Reservation -ScopeId <RED> -IPAddress <IP_FIJA> -ClientId "<MAC_DEL_EQUIPO>" -Description "Descripción del equipo"
```

Es el método habitual para impresoras, servidores internos o cualquier equipo que deba localizarse siempre en la misma dirección. La MAC se obtiene con `ipconfig /all` en el equipo cliente, o con `arp -a` desde el servidor si el equipo ya ha generado tráfico.

Si en cambio lo que se busca es simplemente limitar el rango dinámico (sin atar IPs a equipos concretos), basta con ajustar `-EndRange` al crear el scope (paso 4), sin usar reservas.

## 9. Validar y comprobar

```powershell
Get-DhcpServerv4Scope
Get-DhcpServerv4Statistics
Get
