# SRI: Servicios en Red e Internet

Manuales de las prácticas del módulo. Cada uno lleva un índice al principio para ir directo al apartado que se necesite.

## Máquinas virtuales e IPs

Todas en la red `172.16.6.0/24` (puerta de enlace `172.16.6.1`). Rango asignado: `172.16.6.20` a `172.16.6.29`.

| Máquina | IP | Para qué sirve | Nombres de servicio |
|---|---|---|---|
| `server` | `172.16.6.20` | SRI: DNS primario, DHCP primario y web | `dns.`, `dhcp.`, `www.` |
| `server2` | `172.16.6.21` | SRI: DNS secundario, DHCP secundario y web | `dns2.`, `dhcp2.`, `www2.` |
| SAD | `172.16.6.22` | Seguridad y Alta Disponibilidad | |
| ASO | `172.16.6.24` | Administración de Sistemas Operativos | |
| IAW | `172.16.6.25` | Implantación de aplicaciones web | |
| `client` | DHCP (pool `.200`-`.250`) o fija `172.16.6.26` | SRI: cliente para probar DHCP y DNS | |

Los nombres de servicio van seguidos de la zona `jordi.asir2.iespoveda.local` (por ejemplo, `www.jordi.asir2.iespoveda.local`). La `.23` y de la `.27` a la `.29` están libres.

## Manuales

| Manual | Contenido |
|---|---|
| [Unit0_introduccion.md](Unit0_introduccion.md) | Introducción: distribuciones Linux oficiales de las comunidades autónomas, en qué se basan las distribuciones y modos de red de VirtualBox |
| [Unit1_remote_access_management.md](Unit1_remote_access_management.md) | Acceso remoto con SSH: instalación, red de la máquina virtual, cambio de puerto, claves, endurecimiento y formas de conectarse |
| [Unit2_DHCP_server.md](Unit2_DHCP_server.md) | DHCP: Kea en Ubuntu (con servidor secundario en alta disponibilidad) y rol DHCP en Windows Server 2022 (línea de comandos y gráfico). Anexo con el fichero de configuración de Kea explicado |
| [Unit3_dns.md](Unit3_dns.md) | DNS con BIND9: instalación, zonas, reenvío, servidor secundario y transferencia de zonas. Incluye cómo se forman los nombres de dominio |
| [Unit3_escenario_simulado_dns.md](Unit3_escenario_simulado_dns.md) | DNS con BIND9 en un escenario simulado, con variables, y gestión diaria de dominios (añadir, consultar, cambiar, eliminar) |
| [Unit4_servidor_web.md](Unit4_servidor_web.md) | Servidor web con Nginx: virtual hosts por puerto y por nombre, firewall, HTTPS con certificado autofirmado y problemas típicos |
| [Comprobaciones_servicios.md](Comprobaciones_servicios.md) | Comprobación rápida de qué servicios hay, dónde están y si funcionan: tabla de servicios con su dominio (`servicios.sh`), script de demostración y comandos sueltos |

## Entorno de las prácticas

| Dato | Valor |
|---|---|
| Red | `172.16.6.0/24` |
| Puerta de enlace | `172.16.6.1` |
| Zona DNS de los servicios | `jordi.asir2.iespoveda.local` |
| Sistema | Ubuntu Server, en máquinas virtuales de VirtualBox |

## Orden recomendado

1. **Unit 0**: introducción y modos de red de las máquinas virtuales.
2. **Unit 1**: acceso remoto, para poder administrar los servidores por SSH.
3. **Unit 2**: DHCP, para que los clientes reciban su configuración de red.
4. **Unit 3**: DNS, para que los clientes resuelvan nombres.
5. **Unit 4**: servidor web, el servicio que se publica con esos nombres.
6. **Comprobaciones**: para verificar o demostrar que todo está funcionando.
