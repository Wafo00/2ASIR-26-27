# Instalación y configuración de DHCP con Kea

> Antes de instalar: comprobar que no hay otro servicio DHCP activo en la misma red.

## 1. Instalar

```
sudo apt update
```
```
sudo apt install kea
```

Durante la instalación, al configurar la contraseña de `kea-ctrl-agent`, elegir **"configure with a given password"** e introducir una propia (evita depender de `sudo cat /etc/kea/kea-api-password` más adelante).

## 2. Configurar interfaz y rango

```
sudo cp /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.bak
```

Editar con `nano`, o sobrescribir de una vez (más seguro frente a errores de copiar/pegar):

```
sudo tee /etc/kea/kea-dhcp4.conf > /dev/null << 'EOF'
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
          { "name": "domain-name-servers", "data": "192.168.1.1" }
        ]
      }
    ]
  }
}
EOF
```

Sustituir interfaz (`ip a`), red y rango por los propios. La IP del servidor debe quedar fuera del pool.

## 3. Validar sintaxis

```
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

### Incidencia: "Unable to open file"

Si falla con `Syntax check failed with: Unable to open file` pese a tener el fichero con permisos correctos (`root:root`, `644`), es un bloqueo de **AppArmor**, no de sintaxis ni permisos. Confirmar:

```
sudo dmesg | grep -i apparmor | grep -i kea
```

Líneas `DENIED` con `dac_override`/`dac_read_search` confirman la causa. Solución:

```
sudo apt install apparmor-utils
sudo aa-complain /usr/sbin/kea-dhcp4
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

## 4. Arrancar y comprobar

```
sudo systemctl restart kea-dhcp4-server
```
```
sudo systemctl status kea-dhcp4-server
```

Si falla: `journalctl -u kea-dhcp4-server -e`

## 5. Configurar para autoarranque

```
sudo systemctl enable kea-dhcp4-server
```
```
systemctl is-enabled kea-dhcp4-server
```

Debe responder `enabled`. Suele venir así por defecto tras la instalación.

## 6. Verificar desde un cliente

En el cliente, modo DHCP en `/etc/netplan/*.yaml`:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
```

```
sudo netplan apply
```
```
ip a
```

Debe recibir IP del pool, marcada `dynamic`. Si no, o quedan IPs mezcladas:

```
sudo ip addr flush dev enp0s3
```
```
sudo netplan apply
```

Para renovar en cliente con NetworkManager:

```
sudo nmcli device disconnect enp0s3
```
```
sudo nmcli device connect enp0s3
```

Si el cliente es Windows:
```powershell
ipconfig /release
```
```powershell
ipconfig /renew
```

Si repite la misma IP, es comportamiento normal (*sticky lease*: mientras no caduque y la MAC coincida, el servidor intenta devolver la misma).

## Operaciones habituales
```
nmcli -f ALL device show | grep -i dhcp_server_identifier
```
Servicor DHCP del que recibe IP el cliente

```
sudo cat /var/lib/kea/kea-leases4.csv
```
Concesiones activas.

```
sudo journalctl -u kea-dhcp4-server -f
```
Log en tiempo real.

```
sudo systemctl reload kea-dhcp4-server
```
Aplica cambios de configuración sin cortar concesiones activas (preferible a `restart` con clientes ya conectados).

### Cambiar tiempos de concesión

```json
"valid-lifetime": 600,
"max-valid-lifetime": 7200
```
En segundos. Editar, validar y aplicar:

```
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```
```
sudo systemctl reload kea-dhcp4-server
```

### Cambiar el pool

Editar `"pools"` dentro de `subnet4` en `kea-dhcp4.conf`, validar y `reload` igual que arriba.

---

| Síntoma | Causa probable |
|---|---|
| El servicio no arranca | Error de sintaxis JSON, o AppArmor bloqueando el fichero |
| Arranca pero no reparte IPs | Interfaz equivocada, u otro DHCP compitiendo en la red |
| El cliente no recibe IP | VMs en redes distintas, o interfaz con IPs residuales |  

# Instalación y configuración de DHCP en Windows Server 2022

> Antes de instalar: comprobar que no hay otro servicio DHCP activo en la misma red. El adaptador de red de la VM debe estar en modo **Red interna** o **Bridge**, nunca NAT.

## 1. Configurar IP estática

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress <IP_SERVIDOR> -PrefixLength 24 -DefaultGateway <IP_GATEWAY>
```
```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses <IP_SERVIDOR>
```

## 2. Instalar el rol DHCP

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
```

Si tarda mucho sin terminar (intenta descargar de Windows Update sin salida a internet), cancelar con `Ctrl+C` y repetir — es idempotente y seguro.

## 3. Autorizar en AD (solo si hay dominio)

```powershell
Add-DhcpServerInDC -DnsName "servidor.dominio.local" -IPAddress <IP_SERVIDOR>
```

Omitir si no hay Active Directory.

## 4. Crear el ámbito

```powershell
Add-DhcpServerv4Scope -Name "Nombre" -StartRange <IP_INICIO> -EndRange <IP_FIN> -SubnetMask 255.255.255.0
```

La IP del servidor debe quedar fuera del rango.

## 5. Configurar gateway

```powershell
Set-DhcpServerv4OptionValue -ScopeId <RED> -Router <IP_GATEWAY>
```

## 6. Configurar DNS

```powershell
Get-WindowsFeature -Name DNS
```

Si no está instalado:

```powershell
Install-WindowsFeature -Name DNS -IncludeManagementTools
```
```powershell
Set-DhcpServerv4OptionValue -ScopeId <RED> -DnsServer <IP_SERVIDOR>
```

Si da error `"<IP> no es un servidor DNS válido"` con el rol ya instalado, confirmar que el DNS funciona de verdad:

```powershell
Resolve-DnsName -Name localhost -Server <IP_DNS>
```

Si responde, es un falso positivo del cmdlet — forzar:

```powershell
Set-DhcpServerv4OptionValue -ScopeId <RED> -DnsServer <IP_DNS> -Force
```

## 7. Activar el ámbito

```powershell
Set-DhcpServerv4Scope -ScopeId <RED> -State Active
```

## 8. Reservar IPs fijas (equipos, impresoras...)

```powershell
Add-DhcpServerv4Reservation -ScopeId <RED> -IPAddress <IP_FIJA> -ClientId "<MAC>" -Description "Descripción"
```

MAC del cliente: `ipconfig /all`, o `arp -a` en el servidor si ya hubo tráfico.

## 9. Validar y comprobar

```powershell
Get-DhcpServerv4Scope
```
```powershell
Get-Service -Name DHCPServer
```

> La consola gráfica (`dhcpmgmt.msc`) puede mostrar *"Requiere configuración para Servidor DHCP en \<equipo\>"*. En un servidor sin AD es un falso positivo (`Get-DhcpServerInDC` lo confirma con error si no hay dominio) y no afecta al servicio.

## 10. Verificar desde un cliente

En el cliente (Ubuntu), modo DHCP y comprobar con `ip a`. La concesión se confirma en el servidor:

```powershell
Get-DhcpServerv4Lease -ScopeId <RED>
```

En un cliente Windows:
```powershell
ipconfig /release
```
```powershell
ipconfig /renew
```
(Lista todos los ámbitos del servidor, y las concesiones del rango indicado)  

En un cliente Ubuntu (NetworkManager):
```bash
sudo nmcli device disconnect enp0s3
```
```
sudo nmcli device connect enp0s3
```

## Operaciones habituales

```powershell
Set-DhcpServerv4Scope -ScopeId <RED> -LeaseDuration 8.00:00:00
```
Tiempo de concesión (formato `días.horas:minutos:segundos`).

```powershell
Set-DhcpServerv4Scope -ScopeId <RED> -StartRange <IP> -EndRange <IP>
```
Cambiar el rango del pool.

```powershell
Get-DhcpServerv4Reservation -ScopeId <RED>
```
```powershell
Remove-DhcpServerv4Reservation -IPAddress <IP>
```
Listar / eliminar reservas.

```powershell
Get-DhcpServerv4Lease -ScopeId <IPRed>
```
(Comprueba las ips concedidas)

```powershell
Remove-DhcpServerv4Lease -IPAddress <IP>
```
Liberar una concesión sin esperar su caducidad.

```powershell
Get-DhcpServerv4Statistics
```
Porcentaje de uso del ámbito.

```powershell
Set-DhcpServerv4Scope -ScopeId <RED> -State InActive
```
Desactivar temporalmente el reparto sin borrar la configuración.

---

| Síntoma | Causa probable |
|---|---|
| El cliente no recibe IP | Ámbito inactivo, servidor no autorizado en AD, o servicio parado |
| Siempre recibe la misma IP | Tiene reserva, o la concesión aún no ha caducado |
| No reparte en la red esperada | Adaptador en modo NAT en vez de Red interna/Bridge |
| Error DNS "no es un servidor válido" | IP no alcanzable, rol DNS no instalado, o falso positivo del cmdlet |
| IPs inconsistentes en el cliente | Hay más de un servidor DHCP activo en la red |

# Instalación y configuración de DHCP en Windows Server 2022 (interfaz gráfica)

> Antes de instalar: comprobar que no hay otro servicio DHCP activo en la misma red. El adaptador de red de la VM debe estar en modo **Red interna** o **Bridge**, nunca NAT.

## 1. Configurar IP estática

Panel de control → Centro de redes y recursos compartidos → Cambiar configuración del adaptador → clic derecho sobre el adaptador → Propiedades → Protocolo de Internet versión 4 (TCP/IPv4) → Propiedades.

Marcar **"Usar la siguiente dirección IP"** y rellenar IP, máscara, puerta de enlace. En **"Usar las siguientes direcciones de servidor DNS"**, poner la propia IP del servidor.

## 2. Instalar el rol DHCP

Administrador del servidor → **Agregar roles y características** → Siguiente hasta "Roles de servidor" → marcar **Servidor DHCP** → Agregar características (si lo pide) → Siguiente → Instalar.

Al terminar, clic en el aviso amarillo de la esquina superior → **Completar configuración de DHCP** → Confirmar (sin añadir credenciales de dominio si no hay AD) → Cerrar.

## 3. Autorizar en AD (solo si hay dominio)

Herramientas → DHCP → clic derecho sobre el nombre del servidor → **Autorizar**.

Omitir si no hay Active Directory.

## 4. Crear el ámbito

Herramientas → DHCP → expandir el servidor → clic derecho en **IPv4** → **Ámbito nuevo** → Siguiente.

- Nombre del ámbito → el que se quiera.
- Rango de direcciones IP → IP inicial, IP final, máscara de subred.
- Exclusiones → (opcional) dejar fuera IPs reservadas para equipos fijos.
- Duración de la concesión → por defecto 8 días.

## 5. Configurar gateway

Dentro del mismo asistente (paso "Puerta de enlace predeterminada"): añadir la IP del router/gateway → Agregar → Siguiente.

## 6. Configurar DNS

Dentro del mismo asistente (paso "Dominio y servidor DNS"): el nombre de dominio puede dejarse en blanco si no hay AD. En "Servidor IP", escribir la IP del propio servidor → **Agregar** (si no se resuelve el nombre automáticamente, escribir la IP directamente y pulsar Agregar sin esperar la resolución).

Si el rol DNS no está instalado: Administrador del servidor → Agregar roles y características → marcar **Servidor DNS** → Instalar (mismo proceso que el paso 2, pero con DNS).

## 7. Activar el ámbito

Dentro del asistente, paso "Activar el ámbito ahora" → marcar **Sí** → Finalizar.

Si ya se creó sin activar: clic derecho sobre el ámbito → **Activar**.

## 8. Reservar IPs fijas (equipos, impresoras...)

Expandir el ámbito → clic derecho en **Reservas** → **Reserva nueva**.

Rellenar: nombre de la reserva, dirección IP deseada, dirección MAC del equipo (sin guiones ni dos puntos, formato `0800279ABC8E`), descripción → Agregar.

MAC del cliente: `ipconfig /all` en el propio equipo.

## 9. Validar y comprobar

Clic en el nombre del ámbito → panel derecho debe mostrar el rango y el estado **Activo**.

Clic derecho sobre el servidor → **Todas las tareas** → **Reiniciar** (opcional, para confirmar que arranca bien tras cualquier cambio).

> Si aparece el aviso *"Requiere configuración para Servidor DHCP en \<equipo\>"* al abrir la consola, y no hay AD en el entorno, es un falso positivo — no afecta al servicio si el ámbito está activo y reparte IPs.

## 10. Verificar desde un cliente

## 10. Verificar desde un cliente

En un cliente Windows, renovar la IP:

```powershell
ipconfig /release
```
```powershell
ipconfig /renew
```

En un cliente Ubuntu (NetworkManager):

```bash
sudo nmcli device disconnect enp0s3
```
```bash
sudo nmcli device connect enp0s3
```

Comprobar la concesión en el servidor: expandir el ámbito → **Concesiones de direcciones** → debe aparecer la IP y el nombre/MAC del cliente.

## Operaciones habituales

**Cambiar tiempo de concesión:** clic derecho en el ámbito → Propiedades → pestaña General → ajustar duración.

**Cambiar el rango del pool:** clic derecho en el ámbito → Propiedades → pestaña General → editar rango de direcciones (puede requerir eliminar y recrear el ámbito si se reduce por debajo de concesiones activas).

**Añadir/eliminar reserva:** Reservas → clic derecho → Nueva reserva / Eliminar.

**Ver concesiones activas:** Concesiones de direcciones (dentro del ámbito).

**Liberar una concesión:** Concesiones de direcciones → clic derecho sobre la IP → **Eliminar**.

**Ver estadísticas de uso:** clic derecho sobre el servidor → **Mostrar estadísticas**.

**Desactivar el ámbito temporalmente:** clic derecho sobre el ámbito → **Desactivar**.

---

| Síntoma | Causa probable |
|---|---|
| El cliente no recibe IP | Ámbito inactivo, servidor no autorizado en AD, o servicio parado |
| Siempre recibe la misma IP | Tiene reserva, o la concesión aún no ha caducado |
| No reparte en la red esperada | Adaptador en modo NAT en vez de Red interna/Bridge |
| Falta la opción "Agregar roles y características" | No se está ejecutando como Administrador |

## Preguntas y respuestas
### ¿Puede haber dos servidores DHCP en una misma red?
Sí, siempre que estén coordinados en modo failover (reparto o respaldo del mismo ámbito entre ambos). Sin esa coordinación, dos servidores DHCP independientes en la misma red compiten y dan asignaciones inconsistentes.
