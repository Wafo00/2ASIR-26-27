# Instalación y configuración de DHCP en Windows Server 2022

> Antes de instalar: comprobar que no hay otro servicio DHCP activo en la misma red (evita conflictos de IP).

## 1. Instalar el rol DHCP

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
```

Instala el servicio y la consola de gestión (`DHCP Manager`). También puede configurarse desde esa consola gráfica (`dhcpmgmt.msc`); aquí se documenta en PowerShell por ser más rápido de repetir.

## 2. Autorizar el servidor (solo si hay Active Directory)

```powershell
Add-DhcpServerInDC -DnsName "servidor.dominio.local" -IPAddress <IP_DEL_SERVIDOR>
```

En un dominio AD, un DHCP sin autorizar es ignorado por los clientes como medida de seguridad. En un servidor independiente (sin AD) este paso no existe y debe omitirse; el comando fallaría al no encontrar un controlador de dominio.

## 3. Crear el ámbito (scope) y el rango

```powershell
Add-DhcpServerv4Scope -Name "Nombre_del_ambito" -StartRange <IP_INICIO> -EndRange <IP_FIN> -SubnetMask 255.255.255.0
```

Un *scope* define la red y el rango de IPs que el servidor puede repartir.

## 4. Configurar puerta de enlace

```powershell
Set-DhcpServerv4OptionValue -ScopeId <RED> -Router <IP_GATEWAY>
```

## 5. Configurar DNS

```powershell
Set-DhcpServerv4OptionValue -ScopeId <RED> -DnsServer <IP_DNS_1>,<IP_DNS_2>
```

Este cmdlet valida que la IP indicada responda realmente como servidor DNS (puerto 53), no solo que sea alcanzable por red. Puede fallar con el error `"<IP> no es un servidor DNS válido"` en dos casos:

- **La IP no tiene salida de red** (en una VM con la interfaz en modo "red interna" de VirtualBox, por ejemplo, ninguna IP externa será alcanzable nunca por diseño).
- **La IP es alcanzable pero no tiene el rol DNS instalado** (no hay nada escuchando en el puerto 53).

Para saber si el rol DNS está instalado:

```powershell
Get-WindowsFeature -Name DNS
```

Si no lo está, dos opciones:

**A. Instalar el rol DNS** (recomendado; permite resolución de nombres real):

```powershell
Install-WindowsFeature -Name DNS -IncludeManagementTools
Set-DhcpServerv4OptionValue -ScopeId <RED> -DnsServer <IP_DEL_PROPIO_SERVIDOR>
```

**B. Forzar el valor sin validar** (válido solo si el objetivo es probar el reparto de DHCP, sin resolución de nombres real):

```powershell
Set-DhcpServerv4OptionValue -ScopeId <RED> -DnsServer <IP_DNS> -Force
```

Si se instala el rol DNS, crear además una zona para poder registrar nombres de los equipos de la red:

```powershell
Add-DnsServerPrimaryZone -Name "dominio.local" -ZoneFile "dominio.local.dns"
```

## 6. Activar el ámbito

```powershell
Set-DhcpServerv4Scope -ScopeId <RED> -State Active
```

Un scope creado pero no activado existe en la configuración, pero no reparte IPs.

## 7. Reservar IPs fijas (equipos, impresoras, servidores...)

```powershell
Add-DhcpServerv4Reservation -ScopeId <RED> -IPAddress <IP_FIJA> -ClientId "<MAC_DEL_EQUIPO>" -Description "Descripción del equipo"
```

Una reserva asigna siempre la misma IP a una MAC concreta: combina la gestión centralizada del DHCP con la previsibilidad de una IP fija. Es el método habitual para impresoras, servidores internos o cualquier equipo que deba localizarse siempre en la misma dirección. La MAC se obtiene con `ipconfig /all` en el equipo cliente, o con `arp -a` desde el servidor si el equipo ya ha generado tráfico.

## 8. Validar y comprobar

```powershell
Get-DhcpServerv4Scope
Get-DhcpServerv4Statistics
```

Muestra el estado del ámbito y cuántas IPs están libres o concedidas.

## 9. Verificar desde un cliente

```cmd
ipconfig /release
ipconfig /renew
```

El cliente debe recibir una IP dentro del rango configurado, salvo que tenga reserva, en cuyo caso recibirá siempre la misma.

## Consultas habituales de administración

```powershell
Get-DhcpServerv4Lease -ScopeId <RED>
```
Listado de concesiones activas.

```powershell
Get-DhcpServerv4Reservation -ScopeId <RED>
```
Listado de reservas configuradas.

```powershell
Remove-DhcpServerv4Lease -IPAddress <IP>
```
Libera manualmente una concesión activa, sin esperar a que caduque.

```powershell
Get-DhcpServerv4Statistics
```
Porcentaje de uso del ámbito; útil para detectar si el rango se está quedando corto.

---

| Síntoma | Causa probable |
|---|---|
| El cliente no recibe IP | Ámbito no activado, o servidor no autorizado en AD |
| Siempre recibe la misma IP | Comportamiento normal si tiene reserva, o la concesión aún no ha caducado |
| El servicio no reparte en la red esperada | Interfaz de red del servidor en la VLAN/red equivocada |
| Error "`<IP>` no es un servidor DNS válido" al configurar DNS | La IP no es alcanzable (red interna de una VM) o no tiene el rol DNS instalado — ver sección "Configurar DNS" |
