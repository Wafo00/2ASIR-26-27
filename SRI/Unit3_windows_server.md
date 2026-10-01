# Instalación y configuración de DHCP en Windows Server 2022

> Antes de instalar: comprobar que no hay otro servicio DHCP activo en la misma red (evita conflictos de IP).

## 1. Instalar el rol DHCP

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
```

Instala el servicio y la consola de gestión (`DHCP Manager`). Para configurarlo también puede usarse esa consola gráfica (`dhcpmgmt.msc`); aquí se documenta en PowerShell por ser más rápido de repetir y de pegar en pruebas.

## 2. Autorizar el servidor (si hay Active Directory)

```powershell
Add-DhcpServerInDC -DnsName "servidor.dominio.local" -IPAddress 172.16.5.20
```

En un dominio AD, un DHCP sin autorizar es ignorado por los clientes como medida de seguridad (evita servidores DHCP "pirata" en la red). En un servidor independiente (sin AD) este paso no existe y se puede omitir.

## 3. Crear el ámbito (scope) y el rango

```powershell
Add-DhcpServerv4Scope -Name "Aula" -StartRange 172.16.5.21 -EndRange 172.16.5.30 -SubnetMask 255.255.255.0
```

Un *scope* equivale al `subnet4` de Kea: define la red y el rango de IPs que el servidor puede repartir.

## 4. Configurar opciones (puerta de enlace y DNS)

```powershell
Set-DhcpServerv4OptionValue -ScopeId 172.16.5.0 -Router 172.16.5.1 -DnsServer 8.8.8.8,8.8.4.4
```

Equivalente al `option-data` de Kea: indica a los clientes su puerta de enlace y servidores DNS.

## 5. Activar el ámbito

```powershell
Set-DhcpServerv4Scope -ScopeId 172.16.5.0 -State Active
```

Un scope creado pero no activado existe en la configuración, pero no reparte IPs.

## 6. Reservar IPs fijas (equipos, impresoras...)

```powershell
Add-DhcpServerv4Reservation -ScopeId 172.16.5.0 -IPAddress 172.16.5.5 -ClientId "08-00-27-AA-BB-CC" -Description "Impresora aula"
```

Una reserva asigna siempre la misma IP a una MAC concreta, combinando lo cómodo del DHCP (gestión centralizada) con lo predecible de una IP fija. Es el método habitual en entornos reales para impresoras, servidores internos o equipos que necesitan localizarse siempre en la misma dirección. El `ClientId` es la MAC del equipo (se obtiene con `ipconfig /all` en el cliente, o `arp -a` desde el servidor si ya ha tenido tráfico).

## 7. Validar y comprobar

```powershell
Get-DhcpServerv4Scope
Get-DhcpServerv4Statistics
```

Muestra el estado del scope y cuántas IPs están libres/concedidas, sin necesidad de ir cliente por cliente.

## 8. Verificar desde un cliente

```cmd
ipconfig /release
ipconfig /renew
```

El cliente debe recibir una IP dentro del rango `172.16.5.21-30`, salvo que tenga reserva, en cuyo caso recibirá siempre la misma.

## Consultas habituales de administración

```powershell
Get-DhcpServerv4Lease -ScopeId 172.16.5.0
```
Listado de concesiones activas, equivalente al `kea-leases4.csv` de Kea.

```powershell
Get-DhcpServerv4Reservation -ScopeId 172.16.5.0
```
Listado de reservas configuradas.

```powershell
Remove-DhcpServerv4Lease -IPAddress 172.16.5.24
```
Libera manualmente una concesión activa, sin esperar a que caduque.

```powershell
Get-DhcpServerv4Statistics
```
Porcentaje de uso del ámbito; útil para detectar si el rango se está quedando corto.

---

| Síntoma | Causa probable |
|---|---|
| El cliente no recibe IP | Scope no activado, o servidor no autorizado en AD |
| Siempre recibe la misma IP | Comportamiento normal si tiene reserva, o lease aún no caducado |
| El servicio no reparte en la red esperada | Interfaz de red del servidor en la VLAN/red equivocada |
